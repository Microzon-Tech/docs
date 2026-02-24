# PostgreSQL Timeout Hatası - Analiz ve Çözüm Önerileri

## Hata Özeti

```
System.InvalidOperationException: An exception has been raised that is likely due to a transient failure.
 ---> Npgsql.NpgsqlException: Exception while reading from stream
 ---> System.TimeoutException: Timeout during reading attempt
```

**Etkilenen işlem:** `UPDATE "AbpSessions" SET "IpAddresses" = @p0, "LastAccessed" = @p1 WHERE "Id" = @p2`  
**Süre:** 120,019ms (CommandTimeout limitine ulaşıldı)

---

## Kök Neden Analizi

### 1. **Geçici (Transient) Bağlantı Sorunları**
- PostgreSQL ile uygulama arasındaki TCP bağlantısı okuma sırasında zaman aşımına uğruyor
- Npgsql'in varsayılan `NpgsqlRetryingExecutionStrategy` devreye giriyor ancak yeniden denemeler de başarısız oluyor

### 2. **Olası Nedenler**

| Neden | Açıklama |
|------|----------|
| **Connection pool tükenmesi** | Tüm bağlantılar kullanımda, yeni istekler 120 sn bekleyip timeout alıyor |
| **Ağ gecikmesi** | `postgres-cluster-rw.database.svc` uzak/gecikmeli ağda olabilir |
| **PostgreSQL yükü** | Ağır sorgular, lock contention, vacuum eksikliği |
| **Kopuk bağlantılar** | Pool'daki bağlantılar PostgreSQL tarafından kapatılmış ama Npgsql fark etmemiş olabilir |
| **Session güncelleme sıklığı** | Her HTTP isteğinde AbpSessions UPDATE tetikleniyor (IdentitySession) |

### 3. **Mevcut Konfigürasyon**

**Connection String (appsettings.json):**
```
Timeout=90; Keepalive=30; MaxPoolSize=200; MinPoolSize=20; 
ConnectionIdleLifetime=300; ConnectionPruningInterval=10
```

**EF Core:**
- CommandTimeout: 120 saniye
- EnableRetryOnFailure: **Devre dışı** (ABP UoW uyumluluğu nedeniyle)

**IdentitySession:**
- `MinUpdateIntervalSeconds: 600` (10 dk)
- `DisableTouchUpdates: true` ✅ (Zaten optimize)

---

## Önerilen Çözümler

### Öncelik 1: Connection String İyileştirmeleri

```json
// Önerilen parametreler
"ConnectionStrings": {
  "Default": "Host=...;Timeout=120;Command Timeout=120;Keepalive=30;MaxPoolSize=200;MinPoolSize=20;ConnectionIdleLifetime=300;ConnectionPruningInterval=10;Connection Idle Lifetime=300;Pooling=true;Enlist=true;"
}
```

- `Timeout` ve `Command Timeout` değerlerini tutarlı yapın (120)
- `Connect Timeout` ekleyin (bağlantı kurulumu için): `Connect Timeout=15`

### Öncelik 2: Npgsql Enabling Retry (Dikkatli Değerlendirme)

EntityFrameworkCoreModule'da yorum satırı:
```csharp
// EnableRetryOnFailure kullanılmıyor: ABP Unit of Work ve manuel transaction'lar
// NpgsqlRetryingExecutionStrategy ile uyumlu değil (InvalidOperationException).
```

**Not:** Npgsql zaten varsayılan olarak `NpgsqlRetryingExecutionStrategy` kullanıyor. Sorun retry eksikliği değil, tüm denemelerin timeout'a düşmesi.

### Öncelik 3: Session Update Yükünü Azaltma

`IdentitySessionUpdateThrottleInterceptor` şu an **yorum satırında**. Bu interceptor session güncellemelerini throttle ederek DB yükünü azaltabilir. ABP Identity Pro modülü ile uyumluluğu kontrol edilerek tekrar etkinleştirilebilir.

### Öncelik 4: PostgreSQL Tarafı Kontrolleri

1. **Vacuum:** Replikasyon dead tuple'ları vacuum'u yavaşlatıyor olabilir
2. **Bağlantı sayısı:** `SELECT count(*) FROM pg_stat_activity;`
3. **Uzun süren sorgular:** `pg_stat_activity` ile tespit
4. **Lock'lar:** `pg_locks` tablosu

### Öncelik 5: Connection Pool İzleme

- Uygulama loglarına connection pool metrikleri eklenebilir
- Hangfire (5 worker) + API + diğer servisler aynı connection string'i kullanıyorsa toplam bağlantı ihtiyacı artar

---

## Uygulanan Değişiklikler (✅ Tamamlandı)

1. **appsettings.json** - Connection string güncellendi:
   - `Timeout=90` → `Timeout=30` (bağlantı kurulumu için daha hızlı fail)
   - `Command Timeout=120` eklendi (EF Core ile uyumlu)
   - `ConnectionIdleLifetime=300` → `ConnectionIdleLifetime=180` (stale bağlantıları 3 dk'da temizle)
   - `Pooling=true` açıkça tanımlandı

2. **DevOpsZonEntityFrameworkCoreModule** - `ConfigureConnectionStringResilience`:
   - Env var ile override edilse bile Command Timeout, ConnectionIdleLifetime, Keepalive eklenir

3. **DevOpsZonHttpApiHostModule** - `EnsurePostgresResilienceParams`:
   - Hangfire connection string'ine aynı resilience parametreleri uygulanır

## Ek Uygulanan Değişiklik (Session)

- **MinUpdateIntervalSeconds: 86400** (24 saat) - Session güncellemesi en fazla günde 1 kez yapılır
- Bu, AbpSessions UPDATE sıklığını ~24 kat azaltır (600 sn → 86400 sn)

## Altyapı Kontrolleri (Manuel - Önemli)

Hata devam ediyorsa aşağıdakileri kontrol edin:

1. **PostgreSQL konumu** - API ve PostgreSQL aynı K8s cluster'da mı? Farklı cluster/network = yüksek gecikme
2. **max_connections** - `SHOW max_connections;` - 200+ pool için yeterli olmalı
3. **Aktif bağlantılar** - `SELECT count(*) FROM pg_stat_activity;`
4. **PgBouncer** - Connection pooler kullanılıyorsa transaction mode'da timeout ayarları
5. **Vacuum** - Replikasyon dead tuple'ları: `SELECT * FROM pg_stat_user_tables WHERE relname = 'AbpSessions';`
