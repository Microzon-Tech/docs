# Project Management

**Rota:** `/project-management`

Projeler, repo'lar ve servis listesinin yönetildiği ana giriş noktası.

---

## İçerik

- Proje listesi ve dropdown ile proje seçimi
- Cluster ve registry seçimi
- Seçilen projeye ait servis listesi
- Hızlı deploy tetikleme

---

## İlgili Akışlar

| Akış | Rota | Açıklama |
|------|------|----------|
| Yeni proje oluşturma | `/project-creation` | Sihirbaz: Temel bilgi → Servis bağlantısı → Repo → Servis keşfi → Kaynak/Cluster → Env → Özet |
| Repo ekleme | `/repo-add` | Mevcut projeye yeni repo ekleme |
| Servis keşfi | `/service-discovery` | Repo'lardaki Dockerfile'lara göre servis keşfi ve onay |

---

## Kavramlar

- **Project** — Üst seviye organizasyon birimi; altında repo'lar ve servisler
- **Repo** — Git repository bağlantısı
- **Service (ProjectRepoService)** — Repo'dan keşfedilen, deploy edilebilir uygulama (Dockerfile ile tanımlı)

---

*Detaylı kavramlar için [ANGULAR_APP_CONCEPTS.md](../DevopsZon.API/angular/docs/ANGULAR_APP_CONCEPTS.md) ve [overview.md](./overview.md) bakınız.*
