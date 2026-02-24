# GetBootstrapDataAsync – Görev ve Gereklilik Analizi

Bu doküman, `GetBootstrapDataAsync` metodunun görevini, frontend kullanımını ve gerekliliğini açıklar.

---

## 1. Özet

| Özellik | Değer |
|---------|-------|
| **Endpoint** | `GET /api/devopszon/services/monitoring/bootstrap?projectId={guid}` |
| **Backend Metod** | `MonitoringAppService.GetBootstrapDataAsync(Guid projectId)` |
| **Amaç** | Projedeki tüm servislerin pod durumlarını ve health bilgilerini Kubernetes’ten alıp ilk monitoring state’ini oluşturmak |
| **Tetiklenme** | Proje seçildiğinde, Home sayfası yüklendiğinde, Service-dashboard core bootstrap sonrası |

---

## 2. Backend – Ne Yapıyor?

### 2.1 Akış

1. **DB sorgusu:** `_projectRepository.GetProjectRepoAsync(projectId)` ile projedeki tüm repo’lar ve `ProjectRepoServices` alınır.
2. **Servis listesi:** Repo’lardan düzleştirilmiş servis listesi oluşturulur.
3. **Her servis için (sıralı):**
   - `CreateScope()` ile izole scope
   - `GetServiceBootstrapDataByServiceIdAsync(serviceId)` çağrılır
   - Servis başına **5 saniye** timeout
   - Timeout veya hata durumunda `DeploymentStatus: "Timeout"` veya `"Unknown"` ile boş pod listesi döner
4. **Sonuç:** `BootstrapResponseDto` – `Services`, `TotalPods`, `RunningPods`, `PendingPods`, `FailedPods`

### 2.2 GetServiceBootstrapDataByServiceIdAsync (Her Servis İçin)

- `GetServiceBootstrapDataAsync(ProjectRepoService)` private metodunu çağırır
- Bu metod **kubectl** ile:
  - **Rollout** (önce): `kubectl get rollout` → spec.selector.matchLabels → Pod listesi (Argo Rollouts kullanılıyor, Deployment denenmez)
  - Fallback: service label, app label, tüm pod'lardan filtreleme
  - Her strateji için 10 sn timeout
- Dönen veri: `ServiceBootstrapDto` (ServiceId, ServiceName, Namespace, ClusterId, Pods, Health, RolloutStrategy, ClusterType)

### 2.3 Performans Özellikleri

- **Sıralı işlem:** Servisler paralel değil, sırayla işlenir (DbContext izolasyonu için)
- **Servis başına timeout:** 5 sn
- **Strateji başına timeout (kubectl):** 10 sn
- **Toplam süre:** N servis × (en fazla 5 sn) → Örn. 10 servis ≈ 50 sn üst sınır

---

## 3. Frontend – Nerede ve Nasıl Kullanılıyor?

### 3.1 Tetikleme Zinciri

| Tetikleyici | Dosya | Açıklama |
|-------------|-------|----------|
| **projectSelected** | `monitoring.effects.ts` → `projectChanged$` | Proje seçildiğinde `bootstrapRequested` dispatch edilir |
| **loadProjectsSuccess** | `monitoring.effects.ts` → `autoSelectFirstProject$` | İlk proje otomatik seçilir → `projectSelected` → bootstrap |
| **requestMonitoringBootstrapIfPossible** | `service-dashboard.component.ts` | Core dashboard bootstrap yüklendikten sonra `bootstrapRequested` dispatch edilir |

### 3.2 API Çağrısı

```typescript
// monitoring.effects.ts (satır 55-74)
bootstrapRequested$ = createEffect(() =>
  this.actions$.pipe(
    ofType(MonitoringActions.bootstrapRequested),
    switchMap(({ projectId }) => {
      const url = `${environment.apis.default.url}/api/devopszon/services/monitoring/bootstrap?projectId=${projectId}`;
      return this.http.get<BootstrapResponse>(url).pipe(
        map((data) => MonitoringActions.bootstrapCompleted({ data })),
        catchError(...)
      );
    })
  )
);
```

### 3.3 bootstrapCompleted Sonrası Ne Oluyor?

| Effect | Açıklama |
|--------|----------|
| **monitoring.reducer** | `podsByService`, `serviceHealthMap`, `isBootstrapComplete: true` güncellenir |
| **extractServicesFromBootstrap$** | `loadServicesSuccess` dispatch → NGRX store’daki `availableServices` (dropdown için) güncellenir |
| **autoSelectFirstService$** | İlk servis otomatik seçilir |
| **startPipelineWatching$** | Her servis için `TektonWatchService.watchApplication()` çağrılır (SignalR pipeline watch) |

### 3.4 Veri Kullanım Yerleri

| Bileşen | Kullanım |
|---------|----------|
| **Home** | `selectIsBootstrapComplete` → bootstrap tamamlandığında health status güncellenir; servis dropdown’ı bootstrap’tan beslenir |
| **Service-dashboard** | `requestMonitoringBootstrapIfPossible` → core bootstrap sonrası proje bazlı bootstrap tetiklenir; NGRX’teki `selectedService` (rolloutStrategy, healthStatus) bootstrap’tan gelir |
| **Argo-rollouts-management** | `rolloutStrategy` NGRX `selectedService`’ten (bootstrap’tan) alınır |

---

## 4. İki Farklı Bootstrap Endpoint

| Endpoint | Metod | Kapsam | Kullanım |
|----------|-------|--------|----------|
| `GET .../monitoring/bootstrap?projectId=` | `GetBootstrapDataAsync` | Proje – tüm servisler | Home, proje seçimi, NGRX initial state |
| `GET .../services/{serviceId}/dashboard/bootstrap` | `GetServiceDashboardBootstrapAsync` | Tek servis | Service-dashboard (Ingress, Rollout, Pipeline history) |

**GetBootstrapDataAsync** sadece pod + health bilgisi verir.  
**GetServiceDashboardBootstrapAsync** ek olarak Ingress, Rollout ve son pipeline run bilgisini içerir.

---

## 5. Gereklilik Değerlendirmesi

### 5.1 Neden Gerekli?

1. **İlk state:** SignalR bağlantısı kurulmadan önce mevcut pod durumlarını göstermek için gerekli.
2. **Servis listesi:** Proje seçildiğinde dropdown’daki servisler ve health durumları bu endpoint’ten geliyor.
3. **Pipeline watch:** `startPipelineWatching$` bootstrap tamamlandığında her servis için Tekton watch başlatıyor.
4. **Health baseline:** Home sayfasındaki servis health göstergeleri bootstrap verisiyle besleniyor.

### 5.2 Alternatifler / Optimizasyon Fırsatları

| Seçenek | Açıklama |
|---------|----------|
| **Lazy loading** | Servis listesi DB’den alınabilir; pod durumları sadece seçilen servis için istenebilir. Bu durumda bootstrap sadece “seçilen servis” için çalışır. |
| **Paralel servis sorguları** | Şu an sıralı; scope izolasyonu korunarak paralel hale getirilebilir (örn. `Task.WhenAll` + her biri kendi scope’unda). |
| **Timeout azaltma** | Servis başına 5 sn → 3 sn gibi düşürülebilir; yanıt süresi kısalır, bazı yavaş servisler timeout alabilir. |
| **Cache** | Kısa TTL (örn. 30 sn) ile cache eklenebilir; aynı proje için tekrarlayan istekler azalır. |
| **Sadece metadata** | İlk yüklemede sadece servis listesi + health özeti; pod detayları lazy veya SignalR ile güncellenebilir. |

### 5.3 Kaldırılabilir mi?

**Hayır.** En azından şu anki mimaride:

- Home sayfası servis listesi ve health durumları için buna bağımlı.
- `loadServicesSuccess` ve `startPipelineWatching` bootstrap’tan tetikleniyor.
- Bootstrap olmadan NGRX state eksik kalır, dropdown ve health göstergeleri çalışmaz.

Kaldırılması için:

- Servis listesinin başka bir API’den (örn. `GET /api/devopszon/projects/{id}/services`) alınması,
- Pod/health bilgisinin lazy veya sadece SignalR ile yönetilmesi,
- Pipeline watch’ın proje seçildiğinde farklı bir mekanizmayla başlatılması gerekir.

---

## 6. Timeout ve Dashboard İlişkisi

Dashboard açılırken yaşanan timeout’ların bir kısmı:

1. **GetBootstrapDataAsync:** Proje seçildiğinde tetiklenir; N servis × 5 sn → uzun sürebilir.
2. **GetServiceDashboardBootstrapAsync:** Her servis dashboard’u açıldığında tetiklenir (farklı endpoint).
3. **AbpSessions lock:** Her istek sonunda `UnitOfWork.SaveChanges` → `UPDATE AbpSessions` → aynı satırda lock contention.

Bootstrap’ın sıralı çalışması ve servis sayısının fazlalığı toplam süreyi artırıyor. Paralel çalışma (scope izolasyonu korunarak) veya lazy loading ile iyileştirme yapılabilir.

---

## 7. Özet Tablo

| Soru | Cevap |
|------|-------|
| **Ne yapıyor?** | Projedeki tüm servislerin pod + health bilgisini K8s’ten alıp NGRX store’a yazıyor |
| **Kim tetikliyor?** | Proje seçimi, ilk proje auto-select, service-dashboard (core bootstrap sonrası) |
| **Gerekli mi?** | Evet – servis listesi, health baseline, pipeline watch başlatma için kullanılıyor |
| **Optimize edilebilir mi?** | Evet – paralel servis sorguları, timeout azaltma, cache, lazy loading |
| **Kaldırılabilir mi?** | Mimari değişiklik gerekir; alternatif API’ler ve akışlar tanımlanmalı |
