# Integration (OAuth & Bağlantılar)

**Rota:** `/oauth-integrations`

Git sağlayıcıları (GitHub, GitLab, Bitbucket, Azure) ve container registry bağlantılarının yönetildiği alan. Proje oluşturma ve servis keşfi akışlarında kullanılan bağlantılar buradan tanımlanır.

---

## İçerik

- **Git bağlantıları** — GitHub, Bitbucket, GitLab, Azure DevOps OAuth bağlantıları; bağlantı durumu, repo listesi.
- **Provider detayı** — Seçilen sağlayıcıya ait repo'lar, repository içerik dialog'u (branch, Dockerfile keşfi).
- **Registry bağlantıları** — Docker Hub, GitHub/GitLab Container Registry, ACR, ECR, GCR, custom registry; proje/servis deploy için image push/pull.

---

## Kavramlar

- **Git Connection** — OAuth ile kurulan Git sağlayıcı bağlantısı; proje oluşturma sihirbazında "Servis bağlantısı" adımında kullanılır.
- **Registry Connection** — Container registry bağlantısı; build edilen image'ların push edildiği ve deploy sırasında pull edildiği kayıt.
- **Service Connection** — Backend tarafında proje/repo ile eşleşen bağlantı kaydı; Angular tarafında git/registry listesi ve repo seçimi bu verilerle beslenir.

---

## İlgili Akışlar

- **Project Creation** — `/project-creation` sihirbazında "Servis bağlantısı" adımında buradaki Git bağlantılarından biri seçilir; repo listesi ve servis keşfi bu bağlantı üzerinden yapılır.
- **Repo listesi** — OAuth Integrations içinde provider seçilince ilgili kullanıcının repo'ları listelenir; repository contents (branch, dosya) dialog ile gösterilir.

---

*[overview.md](./overview.md) — Genel bakış.*
