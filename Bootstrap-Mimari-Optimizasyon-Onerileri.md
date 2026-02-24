# Bootstrap ve Dashboard – Mimari Optimizasyon Önerileri

Bu doküman, API yükünü azaltmak ve çok kullanıcılı senaryoda timeout sorunlarını çözmek için mimari seçenekleri değerlendirir.

---

## 1. Mevcut Sorunlar

| Sorun | Açıklama |
|-------|----------|
| **Tek kullanıcıda bile timeout** | Bootstrap sıralı çalışıyor; N servis × 5 sn → uzun süre |
| **API thread blokajı** | Her bootstrap isteği API thread'ini N×5 sn bloke ediyor |
| **Çok kullanıcıda yığılma** | 10 kullanıcı aynı anda proje seçerse → 10× bootstrap paralel çalışır |
| **K8s API baskısı** | Her servis için kubectl çağrısı → K8s API overload |
| **PostgreSQL lock** | Her istek sonunda AbpSessions UPDATE → aynı satırda lock contention |

---

## 2. Seçenekler ve Değerlendirme

### 2.1 Seçenek A: Minimalist Yaklaşım (Hızlı Uygulama)

**Fikir:** Bootstrap’ı hafiflet, sadece gerekli veriyi ver.

| Değişiklik | Etki |
|------------|------|
| Servis listesi DB’den | Pod/health olmadan sadece servis metadata (id, name, namespace, clusterId) – tek DB sorgusu |
| Pod/health lazy | Sadece seçilen servis için istek; veya SignalR ile canlı güncelleme |
| Timeout 5 sn → 3 sn | Yanıt süresi kısalır; yavaş cluster’larda timeout artabilir |

**Artıları:** Az kod değişikliği, hızlı uygulanır.  
**Eksileri:** UX değişir (dropdown hemen dolar ama health geç gelir). Çok kullanıcıda hâlâ yük oluşur.

---

### 2.2 Seçenek B: Redis Cache (Orta Zorluk)

**Fikir:** Aynı proje için tekrarlayan bootstrap isteklerini cache’le.

```
Cache Key: bootstrap:{projectId}
TTL: 30–60 saniye
```

| Senaryo | Davranış |
|---------|----------|
| Cache HIT | Anında yanıt, K8s çağrısı yok |
| Cache MISS | Mevcut akış çalışır, sonuç cache’e yazılır |

**Artıları:** Aynı projeye hızlı geçişlerde yük azalır.  
**Eksileri:** İlk istek ve cache süresi dolunca yine yük var. Çok farklı projelerde etkisi sınırlı.

---

### 2.3 Seçenek C: RabbitMQ Queue (Async Bootstrap)

**Fikir:** Bootstrap’ı API thread’inden çıkar, arka planda işle; sonucu SignalR ile ilet.

```
[Frontend] → POST /bootstrap/request { projectId } → 202 Accepted { jobId }
[API]      → RabbitMQ'ya mesaj publish → Hemen 202 dön
[Consumer] → Bootstrap işle → Redis'e yaz + SignalR ile project-{projectId} grubuna gönder
[Frontend] → SignalR "BootstrapCompleted" dinle → NGRX güncelle
```

| Bileşen | Rol |
|---------|-----|
| **API** | Sadece mesaj publish, anında 202 döner |
| **BootstrapConsumer** | Queue’dan alır, GetBootstrapDataAsync çağırır, sonucu Redis + SignalR ile iletir |
| **SignalR** | `Clients.Group("project-{projectId}").SendAsync("BootstrapCompleted", data)` |
| **Frontend** | `bootstrapRequested` → API çağrısı yerine "request + SignalR dinle" |

**Artıları:**
- API thread bloklanmaz
- İstekler sıraya girer, ani yığılma azalır
- Çok kullanıcıda ölçeklenebilir

**Eksileri:**
- Frontend değişikliği (async pattern)
- SignalR bağlantısı yoksa fallback gerekir (polling veya tekrar deneme)

---

### 2.4 Seçenek D: Hibrit (Cache + Queue + Minimalist)

**Fikir:** Tüm teknikleri birleştir.

1. **Cache-first:** Redis’te varsa anında dön.
2. **Queue:** Cache MISS ise queue’ya at, API 202 dönsün.
3. **Minimalist fallback:** Servis listesi her zaman DB’den; pod/health opsiyonel veya lazy.

```
┌─────────────────────────────────────────────────────────────────────────┐
│  GET /bootstrap?projectId=X                                              │
├─────────────────────────────────────────────────────────────────────────┤
│  1. Redis cache HIT?  → 200 OK (anında)                                  │
│  2. Redis cache MISS? → Publish to RabbitMQ → 202 Accepted { jobId }    │
│     Consumer işler → Redis'e yazar → SignalR "BootstrapCompleted"       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Önerilen Yol: Aşamalı Uygulama

### Faz 1: Hızlı Kazanımlar (1–2 gün)

1. **Redis cache** – Bootstrap sonucu 30–60 sn cache’le.
2. **Servis listesi ayrı endpoint** – `GET /projects/{id}/services` sadece metadata (DB), dropdown için.
3. **Bootstrap sadece pod/health** – Servis listesi zaten gelmişse bootstrap sadece pod + health dönsün.

**Sonuç:** Aynı projeye tekrar geçişlerde yük azalır. Dropdown hızlı dolar.

---

### Faz 2: Queue ile Async (3–5 gün)

1. **BootstrapRequest mesajı** – `projectId`, `userId`/`connectionId` (SignalR için).
2. **BootstrapConsumer** – MassTransit consumer, `GetBootstrapDataAsync` çağırır.
3. **API değişikliği** – Cache MISS ise publish + 202; Consumer bitince SignalR ile gönder.
4. **Frontend** – `bootstrapRequested` → API 202 alınca SignalR’dan `BootstrapCompleted` bekle.

**Sonuç:** API thread bloklanmaz, yük kuyruğa dağılır.

---

### Faz 3: Minimalist Bootstrap (Opsiyonel)

1. **Lazy pod/health** – İlk yüklemede sadece servis listesi; pod/health seçilen servis için veya SignalR ile.
2. **Paralel servis sorguları** – Scope izolasyonu ile `Task.WhenAll` (DbContext paylaşımı yok).

---

## 4. Teknik Detaylar

### 4.1 Redis Cache (Faz 1)

```csharp
// Cache key
var cacheKey = $"bootstrap:project:{projectId}";

// TTL: 45 saniye (çok taze değil, çok eski de değil)
var cacheOptions = new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromSeconds(45) };

// Okuma
var cached = await _cache.GetStringAsync(cacheKey);
if (!string.IsNullOrEmpty(cached))
    return JsonSerializer.Deserialize<BootstrapResponseDto>(cached);

// Yazma (bootstrap tamamlandığında)
await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(response), cacheOptions);
```

### 4.2 RabbitMQ Mesaj (Faz 2)

```csharp
// DevOpsZon.Application.Contracts/Queue/IBootstrapRequestMessage.cs
public interface IBootstrapRequestMessage
{
    Guid ProjectId { get; }
    string? SignalRConnectionId { get; }  // Opsiyonel: sadece bu connection'a gönder
}
```

### 4.3 Consumer → SignalR Akışı

```csharp
// BootstrapConsumer
public async Task Consume(ConsumeContext<IBootstrapRequestMessage> context)
{
    var data = await _monitoringAppService.GetBootstrapDataAsync(context.Message.ProjectId);
    
    // Redis cache
    await _cache.SetStringAsync($"bootstrap:project:{data.ProjectId}", 
        JsonSerializer.Serialize(data), 
        new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromSeconds(45) });
    
    // SignalR: project grubuna gönder (o projeyi seçen tüm client'lar alır)
    await _hubContext.Clients.Group($"project-{data.ProjectId}")
        .SendAsync("BootstrapCompleted", data);
}
```

### 4.4 Frontend Değişikliği (Faz 2)

```typescript
// monitoring.effects.ts - bootstrapRequested$
// Eski: HTTP GET, await response
// Yeni: HTTP POST (veya GET) → 202 + jobId → SignalR "BootstrapCompleted" dinle

// Seçenek 1: API 202 döner, SignalR zaten project grubunda
// Client sadece SignalR'dan BootstrapCompleted bekler (ek endpoint gerekmez)

// Seçenek 2: API hâlâ sync dönebilir (cache HIT), ama cache MISS'te 202 + SignalR
// Frontend: Önce GET dene. 202 ise SignalR dinle. 200 ise direkt bootstrapCompleted dispatch.
```

---

## 5. Karar Matrisi

| Kriter | A: Minimalist | B: Cache | C: Queue | D: Hibrit |
|--------|---------------|----------|----------|-----------|
| Uygulama süresi | 1 gün | 1 gün | 3–5 gün | 5–7 gün |
| API yükü azalması | Orta | Yüksek (cache HIT) | Yüksek | Çok yüksek |
| Çok kullanıcı dayanımı | Düşük | Orta | Yüksek | Çok yüksek |
| Frontend değişikliği | Orta | Yok | Orta | Orta |
| Mevcut altyapı kullanımı | - | Redis ✓ | RabbitMQ ✓ | İkisi ✓ |

---

## 6. Öneri

**Öncelik sırası:**

1. **Faz 1 (hemen):** Redis cache + servis listesi ayrı endpoint. Düşük risk, hızlı kazanım.
2. **Faz 2 (kısa vadede):** Queue + SignalR async bootstrap. Çok kullanıcı için gerekli.
3. **Faz 3 (opsiyonel):** Minimalist/lazy pod yükleme, paralel servis sorguları.

**Neden bu sıra?**
- Redis ve RabbitMQ zaten kullanılıyor.
- Faz 1 ile tek kullanıcıda bile cache HIT ile iyileşme sağlanır.
- Faz 2 ile çok kullanıcı senaryosu güvenli hale gelir.
- Faz 3, ek performans için ihtiyaç olursa eklenebilir.

---

## 7. Dashboard Bootstrap (GetServiceDashboardBootstrapAsync)

Service-dashboard için ayrı endpoint (`/services/{id}/dashboard/bootstrap`) da benzer şekilde:

- **Cache:** `dashboard-bootstrap:service:{serviceId}` – 30 sn TTL.
- **Queue:** Gerekirse aynı pattern; ancak tek servis olduğu için öncelik düşük.
- **Paralel:** Ingress + Rollout + Pipeline history ayrı scope’larda paralel çekilebilir.

---

## 8. Sonuç

| Hedef | Çözüm |
|-------|-------|
| API’yi yormamak | Cache + Queue |
| İsteklerin yığılmaması | RabbitMQ ile sıralama |
| Çok kullanıcı dayanımı | Async pattern + Cache |
| Hızlı başlangıç | Faz 1 (Cache) ile hemen iyileşme |

Önerilen ilk adım: **Faz 1 (Redis cache)**. Sonrasında ihtiyaca göre **Faz 2 (Queue + SignalR)** eklenebilir.

---

## 9. Uygulama Durumu (Güncel)

Tüm fazlar uygulandı:

| Faz | Durum | Dosyalar |
|-----|-------|----------|
| **Faz 1** | ✅ | BootstrapCacheService, IBootstrapCacheService, Dashboard bootstrap cache |
| **Faz 2** | ✅ | BootstrapConsumer, BootstrapRequestMessage, devopszon-bootstrap-queue |
| **Faz 3** | ✅ | MonitoringAppService: paralel servis sorguları (Task.WhenAll) |
| **Frontend** | ✅ | 202 + SignalR BootstrapCompleted + polling fallback |

**API parametreleri:**
- `?sync=true`: Cache MISS'te senkron bekle (geriye dönük uyumluluk)
- `?sync=false` (varsayılan): Async - 202 döner, SignalR veya polling ile sonuç
