# Service Management Dashboard - Timeout ve Lock Analizi

## Özet

Dashboard yüklenirken **çok sayıda paralel API isteği** atılıyor. Her istek sonunda **aynı kullanıcı oturumu** için `AbpSessions` satırı güncelleniyor. Bu da **row-level lock contention** ve **connection beklemeleri** ile timeout’lara yol açıyor.

---

## Dashboard Yüklenirken Yapılan API Çağrıları

| Endpoint | Açıklama | DB | K8s |
|----------|----------|-----|-----|
| `GET /bootstrap?projectId=` | Proje bootstrap – tüm servisler için pod durumu | ✓ | ✓ (her servis için) |
| `GET /{serviceId}/dashboard/bootstrap` | Servis dashboard bootstrap | ✓ | ✓ (Ingress, Rollout, kubectl) |
| `GET /` | Servis listesi | ✓ | - |
| `GET /{id}` | Servis detayı | ✓ | - |
| `GET /{serviceId}/ingress` | Ingress konfigürasyonu | ✓ | ✓ |
| `GET /pipeline-run-history/get-list-by-service-id` | Pipeline geçmişi | ✓ | - |
| `GET /argocd/history/{app}` | ArgoCD geçmişi | ✓ | ✓ |
| Alerts, Health probes, vb. | Diğer detaylar | ✓ | ✓ |

**Sonuç:** Tek bir dashboard açılışında **10–20+ eşzamanlı istek** tetikleniyor.

---

## Kök Neden: AbpSessions Row Lock Contention

1. Her istek **aynı kullanıcı oturumu** ile geliyor.
2. Her isteğin sonunda `UnitOfWork.SaveChanges` → `UPDATE AbpSessions SET IpAddresses=..., LastAccessed=... WHERE Id=@sessionId` çalışıyor.
3. Tüm bu UPDATE’ler **aynı satırı** güncelliyor.
4. PostgreSQL bu satır için **row-level lock** alıyor; aynı anda sadece bir UPDATE çalışabiliyor.
5. Diğer istekler bu lock’u bekliyor.
6. Bekleme süresi 120 saniyeyi aşınca **Timeout during reading attempt** hatası oluşuyor.

```
Request 1: ... → SaveChanges → UPDATE AbpSessions (row lock) → 120s timeout
Request 2: ... → SaveChanges → UPDATE AbpSessions (bekliyor) → 120s timeout  
Request 3: ... → SaveChanges → UPDATE AbpSessions (bekliyor) → 120s timeout
...
```

---

## Ek Faktörler

### 1. Bootstrap’ın Sıralı Çalışması

`GetBootstrapDataAsync` servisleri **sırayla** işliyor:

```csharp
foreach (var service in services)
{
    var serviceTask = monitoringApp.GetServiceBootstrapDataByServiceIdAsync(...);
    var completedTask = await Task.WhenAny(serviceTask, timeoutTask); // 5 sn timeout
}
```

10 servis × 5 sn = en fazla ~50 sn. Bu süre boyunca bu istek DB bağlantısını tutuyor.

### 2. GetServiceDashboardBootstrap – Çoklu K8s Çağrıları

- `GetServiceBootstrapDataAsync` → kubectl
- `GetIngressConfiguration` → Cluster bilgisi + K8s
- `GetRolloutStatus` → kubectl
- `GetPipelineHistory` → DB

K8s çağrıları yavaş veya timeout olursa, istek uzun süre açık kalıyor.

### 3. GetIngressConfiguration

Log’larda sık görülen:

```
[GET-CLUSTER-INFO] ServiceId=..., TotalKubeConfigs=0, ActiveKubeConfig=False
[INGRESS-CONFIG] KubeConfig not found for ServiceId: ...
```

KubeConfig bulunamadığında bile işlem devam ediyor; bu sırada DB bağlantısı tutuluyor.

---

## Önerilen Çözümler

### Öncelik 1: Session Güncelleme Sıklığını Azaltma (Uygulandı)

- `MinUpdateIntervalSeconds: 86400` (24 saat)
- `DisableTouchUpdates: true`

### Öncelik 2: Frontend – Paralel İstekleri Azaltma

- Bootstrap tamamlanmadan diğer istekleri başlatmamak
- Servis bazlı verileri **sıralı** veya **lazy** yüklemek
- Gerekirse tek bir **dashboard/bootstrap** endpoint’i ile tüm veriyi toplu almak

### Öncelik 3: Backend – Bootstrap Optimizasyonu

- `GetBootstrapDataAsync` içinde servis başına 5 sn timeout yerine daha kısa süre (örn. 2–3 sn)
- Ingress / Rollout gibi K8s çağrılarını **fire-and-forget** veya **background** yapmak (session update’i bloke etmemek için)

### Öncelik 4: PostgreSQL Lock Kontrolü

```sql
-- AbpSessions üzerinde bekleyen lock'lar
SELECT blocked_locks.pid AS blocked_pid,
       blocking_locks.pid AS blocking_pid,
       blocked_activity.query AS blocked_query,
       blocking_activity.query AS blocking_query,
       now() - blocked_activity.query_start AS blocked_duration
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks 
  ON blocking_locks.locktype = blocked_locks.locktype
  AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
  AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
  AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted
  AND blocked_activity.query LIKE '%AbpSessions%';
```

---

## Hızlı Test

Dashboard açılırken tarayıcı Network sekmesinde:

- Kaç istek eşzamanlı gidiyor?
- Hangi istekler 120 sn civarında timeout alıyor?

Bu bilgi, lock contention hipotezini doğrulamaya yardımcı olur.
