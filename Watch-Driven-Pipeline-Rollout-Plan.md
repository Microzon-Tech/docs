# Watch-Driven Pipeline Rollout Mimarisi — Dönüşüm Planı

## Özet

Mevcut sistemde `rollout-status` pipeline task dot'u **polling tabanlı** iki ayrı kaynak tarafından güncelleniyor.
Bu plan, bu iki kaynağı kaldırıp **halihazırda var olan** `ServiceRolloutWatcher` (Kubernetes Watch, Redis leader election) 
altyapısını pipeline task güncellemesine bağlar. Sonuç: polling yok, race condition yok, milisaniye gecikme.

---

## Mevcut Durum (AS-IS)

```
Pipeline tamamlanır
    │
    ├─► PipelineRolloutTracker.PollRolloutUntilCompletionAsync()
    │     → Her 5s Kubernetes API sorgu
    │     → TaskStatusChanged MassTransit mesajı
    │     → PipelineHistoryConsumer → DB yazar
    │
    └─► ServicesController.PollAndPersistRolloutStatusAfterPromoteAsync()
          → Her 3s Kubernetes API sorgu (promote sonrası)
          → PersistAndBroadcastRolloutStatusSucceededAfterPromoteAsync()
          → DB yazar + SignalR yayınlar

ServiceRolloutWatcher (şu an yalnızca "Canlı Trafik Analizi" için)
    → Kubernetes Watch (event-driven, sıfır polling)
    → RolloutStatusHub → Angular "Canlı Trafik Analizi"
    ✗ Pipeline rollout-status task DOT'unu GÜNCELLEMİYOR
```

**Sorunlar:**
- 2 polling kaynağı × N servis × M backend pod = çok Kubernetes API isteği
- Race condition (PrefetchCount=10 + eş zamanlı yazma)
- En az 3-5 saniye gecikme
- `Succeeded → Running` geri dönüşü (polling sıralamasından)

---

## Hedef Durum (TO-BE)

```
Pipeline tamamlanır
    │
    └─► RolloutMonitoringBackgroundService.EnsureServiceWatchedAsync(serviceId, pipelineRunId)
          → Mevcut ServiceRolloutWatcher başlatılır (eğer çalışmıyorsa)
          → ServiceRolloutWatcher zaten çalışıyorsa: sadece pipelineRunId kaydedilir

ServiceRolloutWatcher (tek kaynak, her zaman aktif)
    → Kubernetes Watch (event-driven, sıfır polling)
    │
    ├─► RolloutStatusHub → Angular "Canlı Trafik Analizi" (mevcut, değişmez)
    │
    └─► Phase değişikliği → PipelineTaskStatusGateway.UpdateRolloutTaskAsync()
          → RolloutTaskStateMapper.Map(phase) → RolloutTaskState
          → PipelineHistoryMessage (TaskStatusChanged) → MassTransit
          → PipelineHistoryConsumer → DB (atomic UPDATE)
          → TektonWatchHub → Angular Pipeline DOT

ServicesController.PromoteRolloutAsync()
    → Promote komutu gönderilir
    → PollAndPersistRolloutStatusAfterPromoteAsync() KALDIRILIR
    → ServiceRolloutWatcher zaten phase değişikliğini yakalar

PipelineRolloutTracker
    → PollRolloutUntilCompletionAsync() KALDIRILIR
    → ApplyRolloutStatusToArgoCdSyncAsync() → Initial status için tek sefer çalışır
    → Poll loop tamamen kaldırılır
```

---

## Detaylı Uygulama Adımları

### Adım 1 — `PipelineTaskStatusGateway` Servisi (YENİ)

**Dosya:** `src/DevOpsZon.Application/Deployments/Monitoring/PipelineTaskStatusGateway.cs`

```csharp
/// <summary>
/// ServiceRolloutWatcher'dan gelen phase event'lerini pipeline task güncellemesine dönüştürür.
/// Tek sorumluluk: RolloutStatusDto → PipelineHistoryMessage (TaskStatusChanged)
/// </summary>
public class PipelineTaskStatusGateway
{
    // Servis başına aktif pipelineRunId kaydı (Watch başladıktan sonra set edilir)
    private readonly ConcurrentDictionary<Guid, string> _activePipelineRuns = new();

    public void RegisterPipeline(Guid serviceId, string pipelineRunId)
        => _activePipelineRuns[serviceId] = pipelineRunId;

    public void UnregisterPipeline(Guid serviceId)
        => _activePipelineRuns.TryRemove(serviceId, out _);

    public async Task HandleRolloutPhaseChangedAsync(
        Guid serviceId, string serviceIdStr, RolloutStatusDto status,
        IPublishEndpoint publishEndpoint, ILogger logger)
    {
        if (!_activePipelineRuns.TryGetValue(serviceId, out var pipelineRunId))
            return; // Bu servis için aktif pipeline yok

        var decision = RolloutTaskStateMapper.Map(status);
        var taskStatus = decision.State.ToTaskStatus(); // "Running" | "Paused" | "Succeeded" | "Failed"

        // AutoPromote için Succeeded yalnızca AutoPromoteQueueConsumer'dan gelir
        if (status.RolloutStrategy == 3 && decision.State == RolloutTaskState.Succeeded)
        {
            // Tracker gibi: WaitingForAutoPromotePromotedEvent yayınla, Succeeded'a geçme
            taskStatus = "Running";
        }

        // Succeeded olduğunda pipeline kaydını temizle (artık izlemeye gerek yok)
        if (decision.State == RolloutTaskState.Succeeded || decision.State == RolloutTaskState.Failed)
            UnregisterPipeline(serviceId);

        var msg = new PipelineHistoryMessage
        {
            EventType     = PipelineHistoryEventType.TaskStatusChanged,
            PipelineRunName = pipelineRunId,
            Namespace     = "devopszon-pipelines",
            ServiceId     = serviceIdStr,
            Task = new PipelineHistoryTaskDto
            {
                TaskRunId            = $"{pipelineRunId}-argocd-sync",
                TaskName             = "rollout-status",
                Status               = taskStatus,
                Reason               = decision.Reason,
                IsDeploymentReadyTask = true,
                StartTime            = DateTime.UtcNow,
            }
        };

        await publishEndpoint.Publish(msg);
        logger.LogInformation(
            "[PIPELINE-GATEWAY] rollout-status → {Status}/{Reason} | Pipeline={PipelineRunId} Service={ServiceId}",
            taskStatus, decision.Reason, pipelineRunId, serviceId);
    }
}
```

---

### Adım 2 — `ServiceRolloutWatcher`'a Gateway Entegrasyonu

**Dosya:** `src/DevOpsZon.Application/Deployments/Monitoring/ServiceRolloutWatcher.cs`

`NotifyStatusUpdate` metoduna **pipeline gateway çağrısı eklenir**:

```csharp
private async Task NotifyStatusUpdate(RolloutStatusDto status, List<StatusChange> changes)
{
    // Mevcut: RolloutStatusHub → Angular "Canlı Trafik Analizi"
    using var scope = _scopeFactory.CreateScope();
    var hubContext = scope.ServiceProvider.GetRequiredService<IHubContext<RolloutStatusHub>>();
    // ... mevcut SignalR kodu değişmez ...

    // YENİ: Pipeline task DOT güncellemesi
    var gateway = scope.ServiceProvider.GetRequiredService<PipelineTaskStatusGateway>();
    var publishEndpoint = scope.ServiceProvider.GetRequiredService<IPublishEndpoint>();
    await gateway.HandleRolloutPhaseChangedAsync(
        _serviceId, _serviceId.ToString(), status, publishEndpoint, _logger);
}
```

---

### Adım 3 — `RolloutMonitoringBackgroundService`'e Pipeline Kayıt Metodu

**Dosya:** `src/DevOpsZon.Application/Deployments/Monitoring/RolloutMonitoringBackgroundService.cs`

```csharp
/// <summary>
/// Pipeline tamamlandığında çağrılır. Watch zaten aktifse yalnızca pipelineRunId kaydedilir.
/// Watch aktif değilse başlatılır (Canlı Trafik Analizi'nden bağımsız olarak).
/// </summary>
public async Task EnsureServiceWatchedForPipelineAsync(
    Guid tenantId, Guid serviceId, string pipelineRunId,
    string rolloutName, string @namespace, string kubeConfig)
{
    // PipelineTaskStatusGateway'e pipelineRunId kaydet
    _gateway.RegisterPipeline(serviceId, pipelineRunId);

    // Watcher'ı başlat (zaten çalışıyorsa idempotent, hiçbir şey yapmaz)
    await AddSubscriptionAsync(tenantId, serviceId, rolloutName, @namespace, kubeConfig, 
        connectionId: $"pipeline-{pipelineRunId}");
}
```

---

### Adım 4 — `PipelineRolloutTracker` Sadeleştirmesi

**Dosya:** `src/DevOpsZon.Application/DevopsZonAppServices/TektonWatch/PipelineRolloutTracker.cs`

Kaldırılacaklar:
- `PollRolloutUntilCompletionAsync` → tamamen silinir
- `ApplyRolloutStatusToArgoCdSyncAsync` → poll başlatmak yerine `RolloutMonitoringBackgroundService.EnsureServiceWatchedForPipelineAsync` çağrılır

```csharp
// ÖNCE (polling):
if (decision.State != RolloutTaskState.Succeeded && decision.State != RolloutTaskState.Failed)
{
    _ = PollRolloutUntilCompletionAsync(dto, serviceGuid, rolloutStrategy, effectiveNamespace, rolloutName);
}

// SONRA (watch):
await _rolloutMonitoringService.EnsureServiceWatchedForPipelineAsync(
    tenantId, serviceGuid, dto.Id, rolloutName, effectiveNamespace, kubeConfig);
```

Kalacaklar:
- `EnsureArgoCdSyncTrackingAsync` → initial Pending/Running tespiti için
- `UpsertArgoCdSyncTask` + `PublishArgoCdSyncTaskStatusIfChangedAsync` → initial yayın için

---

### Adım 5 — `ServicesController` Sadeleştirmesi

**Dosya:** `src/DevOpsZon.HttpApi/Controllers/ServicesController.cs`

Kaldırılacaklar:
- `PollAndPersistRolloutStatusAfterPromoteAsync` → tamamen silinir (~80 satır)
- `PersistAndBroadcastRolloutStatusSucceededAfterPromoteAsync` → kaldırılır

`PromoteRolloutAsync` içinde:
```csharp
// ÖNCE:
_ = PollAndPersistRolloutStatusAfterPromoteAsync(...);

// SONRA:
// Promote komutu gönderilir; ServiceRolloutWatcher phase değişikliğini otomatik yakalar.
// Ek polling gerekmez.
_logger.LogInformation(
    "[PROMOTE] Promote komutu gönderildi. ServiceRolloutWatcher phase değişikliğini izleyecek. Service={ServiceId}",
    serviceId);
```

---

## Mimari Avantajlar

| Özellik | Polling (Mevcut) | Watch (Hedef) |
|---|---|---|
| Gecikme | 3-5 sn | <500ms |
| Kubernetes API yükü | N×M×interval | 1 connection / servis |
| Race condition | Yüksek | Neredeyse imkansız |
| Kaynak sayısı (rollout-status) | 4 (2 poller + 2 consumer) | 1 (Watch + Gateway) |
| Multi-cluster | Her pod tüm cluster'ları sorgular | Redis leader: 1 pod per servis |
| Multi-instance backend | Race condition kaçınılmaz | Atomic SQL + leader election |
| Kod karmaşıklığı | Çok yüksek | Düşük |

---

## Sıralama ve Bağımlılıklar

```
Sprint 1 (bu branch):
  1. PipelineTaskStatusGateway sınıfı yaz (bağımsız, test edilebilir)
  2. ServiceRolloutWatcher.NotifyStatusUpdate içine Gateway çağrısı ekle
  3. RolloutMonitoringBackgroundService.EnsureServiceWatchedForPipelineAsync ekle
  4. PipelineRolloutTracker'dan poll loop kaldır → Watch'a geç
  
Sprint 2:
  5. ServicesController'dan PollAndPersistRolloutStatusAfterPromoteAsync kaldır
  6. Regresyon testleri: AutoPromote, BlueGreen, Canary stratejileri
  7. Load test: 50 servis × 3 cluster → Watch connection sayısı ve Redis leader TTL doğrula

Opsiyonel Sprint 3:
  8. RolloutMonitoringBackgroundService'i "daima aktif" moduna al
     (şu an yalnızca Canlı Trafik Analizi izlendiğinde açılıyor)
     Pipeline tamamlandığında otomatik başlasın, kullanıcı ekrana bakmasa bile
```

---

## Kabul Kriterleri

- [ ] BlueGreen manual promote sonrası rollout-status DOT sarıdan yeşile geçer, geri düşmez
- [ ] Canary promote sonrası rollout-status DOT sarıdan yeşile geçer, geri düşmez
- [ ] AutoPromote tamamlandığında rollout-status DOT yeşile geçer
- [ ] DB'de tek `rollout-status` task kaydı bulunur (upsert, duplicate yok)
- [ ] Dashboard ve Pipeline Summary aynı pipelineRunId için birebir aynı DOT dizisi gösterir
- [ ] 3 backend pod çalışırken event yalnızca 1 kez işlenir (Redis leader election)
- [ ] Kubernetes API'ye açık watch connection sayısı: (toplam aktif servis) adet, N×M değil

---

## Risk Azaltma

**Risk:** `ServiceRolloutWatcher` pipeline tamamlanmadan başlamazsa initial state eksik olur.
**Çözüm:** `PipelineRolloutTracker.EnsureArgoCdSyncTrackingAsync` ilk "Running" state'i MassTransit ile yayınlar.
Watch başlayana kadar (~100ms) pipeline DOT "Running" (sarı) gösterir. Bu kabul edilebilir.

**Risk:** Watch bağlantısı kesilirse phase değişikliği kaçırılır.
**Çözüm:** `ServiceRolloutWatcher` zaten otomatik reconnect yapıyor (5s sonra). Reconnect'te initial state tekrar yayınlanır.

**Risk:** `PipelineTaskStatusGateway._activePipelineRuns` in-memory; backend restart'ta sıfırlanır.
**Çözüm:** Backend restart'ta `RolloutMonitoringBackgroundService` başlar ve tüm aktif pipeline'lar için
`PipelineRunHistory` DB'sinden "Running" durumundaki pipeline'ları okuyup yeniden kaydeder.
(Sprint 2'de eklenecek: `RestoreActivePipelineWatchersAsync`)
