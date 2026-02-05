# Alert Management

**Rota:** `/alert-management` (global)  
**Servis içi:** `/service-management` → **Alert Management** menü öğesi

Uygulama genelinde ve isteğe bağlı olarak servis bazında alert kural yönetimi.

---

## İçerik

- Alert kural listesi (cluster / servis filtreleme)
- Kural oluşturma ve düzenleme:
  - Metrik tabanlı (Prometheus, PromQL)
  - Log tabanlı (Loki, LogQL)
- Severity, süre, özet, bildirim kanalları
- Şablonlar (global / cluster / service scope)
- Kural testi ve Kubernetes'e deploy (PrometheusRule vb.)
- Servis dashboard'da alert özeti ve geçmişi

---

## Kavramlar

- **Alert Rule** — Koşul sağlandığında tetiklenen kural; metrik veya log sorgusu ile tanımlanır
- **Severity** — Info, Warning, Critical, Emergency
- **Notification Channel** — Webhook, e-posta vb. bildirim kanalları

---

*Detaylı kavramlar için [ANGULAR_APP_CONCEPTS.md](../DevopsZon.API/angular/docs/ANGULAR_APP_CONCEPTS.md) ve [overview.md](./overview.md) bakınız.*
