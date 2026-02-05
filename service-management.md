# Service Management

**Rota:** `/service-management`

Servis seçildikten sonra servis detayı, pipeline, alert, history & rollback ve diğer operasyonel sayfaların yönetildiği alan. Sidebar ile alt sayfalara geçiş yapılır.

---

## Sidebar Menü (Standard Preset)

| Menü Öğesi | Açıklama |
|------------|----------|
| **Dashboard** | Pod durumu, alert sayısı, son deployment revizyonu, HPA, kaynaklar, env, portlar, health probe özeti |
| **Pipeline Summary** | Tekton pipeline çalıştırmaları (canlı + geçmiş), tetikleme, durum takibi |
| **Environment Variable** | Ortam değişkenleri yönetimi |
| **Custom Patches** | Kustomize vb. özel patch'ler (varsa) |
| **Deployment History & Rollback** | Argo Rollouts revizyon geçmişi ve rollback |
| **Ingress Management** | Ingress kuralları ve hostname'ler |
| **Health Probes** | Liveness / Readiness probe yapılandırması |
| **Alert Management** | Bu servise özel alert kuralları |
| **Live Monitoring** | Canlı metrikler |
| **Logs** | Log görüntüleme (Loki vb.) |

SaaS preset'te menü farklılık gösterir (ör. Resource Plan, Invoice; History & Rollback yok).

---

## Kavramlar

- **Service (ProjectRepoService)** — Bir projeye bağlı repodan keşfedilen, Tekton ile build/deploy, Argo Rollouts ile Kubernetes'te çalışan uygulama
- **Pipeline** — Tekton pipeline run'ları
- **Rollout / History & Rollback** — Argo Rollouts revizyonları ve geri alma

---

*Detaylı kavramlar için [ANGULAR_APP_CONCEPTS.md](../DevopsZon.API/angular/docs/ANGULAR_APP_CONCEPTS.md) ve [overview.md](./overview.md) bakınız.*
