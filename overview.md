# DevOpsZon Angular Uygulaması — Genel Bakış

Bu doküman, DevOpsZon Angular uygulamasının amacını, kapsamını ve ana giriş noktalarını özetler. Uygulama, projeleri, servisleri, cluster'ları ve ilgili operasyonları (pipeline, alert, rollback, ingress vb.) tek bir panel üzerinden yönetmek için kullanılır.

---

## Uygulamanın Amacı

DevOpsZon Angular uygulaması:

- **Projeleri** (Git repo bağlantıları, servis keşfi, kaynak ve cluster ataması) yönetir.
- **Servisleri** (deploy, pipeline, ortam değişkenleri, ingress, health probe, rollback) yönetir.
- **Kubernetes cluster'larını** (bağlantı, kubeconfig, addon'lar) yönetir.
- **Alert kurallarını** (metrik ve log tabanlı, bildirim kanalları, şablonlar) uygulama genelinde ve servis bazında yönetir.
- **Entegrasyonları** (Git OAuth, container registry bağlantıları) yönetir.
- **Erişim kontrolünü** (kullanıcı bazında proje/servis/cluster/entegrasyon Read/Edit yetkileri) yönetir.

Kullanıcılar bu panel üzerinden SaaS veya Hibrit modelde proje oluşturabilir, servisleri keşfedip deploy edebilir, pipeline ve rollout geçmişini takip edebilir, alert kuralları tanımlayıp bildirim alabilir.

---

## Ana Giriş Noktaları

| Alan | Rota | Kısa Açıklama |
|------|------|----------------|
| **Project Management** | `/project-management` | Projeler, repo'lar ve servis listesi; proje/cluster seçimi; deploy tetikleme |
| **Service Management** | `/service-management` | Servis detayı, pipeline, alert, history & rollback, ingress, health probe, log, monitoring |
| **Cluster Management** | `/cluster-management` | Kubernetes cluster'ları; kubeconfig, namespace, addon ve kaynak yönetimi |
| **Alert Management** | `/alert-management` | Uygulama genelinde alert kural yönetimi; şablonlar, test, deploy |
| **Integration** | `/oauth-integrations` | Git (GitHub/GitLab/Bitbucket/Azure) ve container registry bağlantıları |
| **Erişim Kontrolü** | `/access-control` | Kullanıcı bazında Project / Service / Cluster / Integration yetkilendirme (Read/Edit) |

Aşağıda her bir giriş noktası kısaca açıklanmaktadır.

---

### Project Management

- **Konum:** Ana sayfa (`/project-management`).
- **İçerik:** Proje listesi ve dropdown ile proje seçimi; cluster ve registry seçimi; seçilen projeye ait servis listesi; hızlı deploy.
- **İlgili akışlar:**
  - **Yeni proje:** `/project-creation` — Temel bilgi → Servis bağlantısı → Repo seçimi → Servis keşfi → Kaynak/Cluster → Ortam değişkenleri → Özet.
  - **Repo ekleme:** `/repo-add` — Mevcut projeye yeni repo ekleme.
  - **Servis keşfi:** `/service-discovery` — Repo'lardaki Dockerfile'lara göre servis keşfi ve onay.

Proje, üst seviye organizasyon birimidir; altında repo'lar ve bu repo'lardan keşfedilen servisler yer alır.

---

### Service Management

- **Konum:** Servis seçildikten sonra (`/service-management`); sidebar ile alt sayfalara geçiş.
- **İçerik:** Seçilen servise özel:
  - **Dashboard** — Pod durumu, alert sayısı, son deployment revizyonu, HPA, kaynaklar, env, portlar, health probe özeti.
  - **Pipeline Summary** — Tekton pipeline çalıştırmaları (canlı + geçmiş), tetikleme, durum takibi.
  - **Environment Variable** — Ortam değişkenleri yönetimi.
  - **Custom Patches** — Kustomize vb. özel patch'ler (varsa).
  - **Deployment History & Rollback** — Argo Rollouts revizyon geçmişi ve rollback.
  - **Ingress Management** — Ingress kuralları ve hostname'ler.
  - **Health Probes** — Liveness / Readiness probe yapılandırması.
  - **Alert Management** — Bu servise özel alert kuralları.
  - **Live Monitoring / Logs** — Canlı metrikler ve log görüntüleme.

Menü yapısı **preset**'e göre değişir (Standard vs SaaS); örneğin SaaS'ta Resource Plan, Invoice vardır; History & Rollback ve bazı operasyonel menüler farklılık gösterebilir.

---

### Cluster Management

- **Konum:** `/cluster-management`.
- **İçerik:** Kubernetes cluster'larının listesi ve yönetimi; kubeconfig veya bağlantı bilgileri; namespace'ler; cluster bazlı addon'lar ve kaynak yapılandırması. Hibrit modelde kullanıcı kendi cluster'ını buradan ekleyip proje/servis atamasında kullanır.

---

### Alert Management

- **Konum:** `/alert-management` (global); servis içinde **Service Management** → **Alert Management** menü öğesi (servis bazlı).
- **İçerik:** Alert kural listesi (cluster/servis filtreleme); kural oluşturma/düzenleme (metrik PromQL / log LogQL); severity, süre, bildirim kanalları; şablonlar; kural testi ve Kubernetes'e deploy (PrometheusRule vb.). Servis dashboard'da da alert özeti ve geçmişi gösterilir.

---

### Integration (OAuth & Bağlantılar)

- **Konum:** `/oauth-integrations`.
- **İçerik:** Git sağlayıcıları (GitHub, GitLab, Bitbucket, Azure) OAuth bağlantıları; repo listesi ve repository içerik dialog'u; container registry bağlantıları (Docker Hub, GHCR, ACR, ECR, GCR, custom). Proje oluşturma sihirbazında "Servis bağlantısı" adımında bu bağlantılar kullanılır.

---

### Erişim Kontrolü (Access Control)

- **Konum:** `/access-control`.
- **İçerik:** Kullanıcı listesi (sol panel) ve entity bazlı yetkilendirme (sağ panel). Sekmeler: Project, Service, Cluster, Integration. Her entity için Read / Edit yetkisi atanır; değişiklikler kaydedilir. Hangi kullanıcının hangi proje, servis, cluster veya entegrasyona erişebileceği buradan yönetilir.

---

## Dokümantasyon Yapısı

Bu `overview.md` dosyası, Angular uygulaması dokümantasyonunun giriş noktasıdır.

- **Ana giriş noktaları (detay sayfaları):**
  - [Project Management](./project-management.md)
  - [Service Management](./service-management.md)
  - [Cluster Management](./cluster-management.md)
  - [Alert Management](./alert-management.md)
  - [Integration](./integration.md)
  - [Erişim Kontrolü](./access-control.md)
- **Kavramlar ve sayfa eşlemesi:** Angular projesi içindeki `angular/docs/ANGULAR_APP_CONCEPTS.md` (Project, Service, Alert, History & Rollback ve rota özeti).

---

## Özet

| Ne yönetilir? | Nerede? |
|---------------|--------|
| Projeler, repo'lar, servis listesi | Project Management |
| Servis detayı, pipeline, alert, history, ingress, probe, log | Service Management |
| Kubernetes cluster'ları | Cluster Management |
| Alert kuralları (global + servis bazlı) | Alert Management |
| Git ve registry bağlantıları | Integration |
| Kullanıcı yetkilendirme (proje/servis/cluster/entegrasyon) | Erişim Kontrolü |

DevOpsZon Angular uygulaması, bu ana giriş noktaları üzerinden proje → servis → cluster, alert, entegrasyon ve erişim kontrolü yaşam döngüsünü tek panelde toplar.
