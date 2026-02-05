# Erişim Kontrolü (Access Control)

**Rota:** `/access-control`

Kullanıcı bazlı satır (entity) seviyesinde yetkilendirme. Hangi kullanıcının hangi proje, servis, cluster veya entegrasyona Read/Edit yetkisi olduğu bu sayfadan yönetilir.

---

## İçerik

- **Sekmeler:** Project, Service, Cluster, Integration — Her sekmede ilgili entity listesi ve yetkiler gösterilir.
- **Sol panel — Kullanıcılar:** Sistemdeki kullanıcı listesi; arama; kullanıcı seçildiğinde sağ panelde o kullanıcının entity bazlı yetkileri listelenir.
- **Sağ panel — Yetkilendirmeler:** Seçilen entity tipine göre (proje, servis, cluster, entegrasyon) kayıt listesi; her kayıt için **Read** ve **Edit** checkbox'ları; "Tümüne Read/Edit ver" ve kaydet.

---

## Yetki Tipleri

| Yetki | Açıklama |
|-------|----------|
| **Read** | İlgili proje/servis/cluster/entegrasyonu görüntüleme |
| **Edit** | İlgili kaydı değiştirme (güncelleme, silme, deploy tetikleme vb.) |

Yetkiler entity bazında (satır bazında) atanır; kullanıcı seçilir, ilgili sekmedeki listeden hangi kayıtlara Read/Edit verileceği işaretlenir ve kaydedilir.

---

## Kavramlar

- **Entity** — Project, Service, Cluster veya Integration (OAuth/Service Connection) kaydı.
- **ACL (Access Control List)** — Backend'de kullanıcı–entity–Read/Edit eşlemesi; bu sayfa ACL kayıtlarını listeler ve günceller.

---

*[overview.md](./overview.md) — Genel bakış.*
