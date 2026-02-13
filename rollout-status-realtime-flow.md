# 📡 Rollout Status Real-Time Flow Dokümantasyonu

Bu doküman, DevOpsZon sisteminde rollout durumunun nasıl tespit edildiği, takip edildiği ve Angular tarafına SignalR ile nasıl iletildiğini detaylı açıklar.

---

## 🎯 1. Backend Tarafı: Servisin Rollout Stratejisini Nasıl Anlıyor?

### 1.1. Veritabanı Kaynağı
Rollout stratejisi **veritabanında** (`ProjectRepoService` tablosunda) `RolloutStrategy` enum field'ı olarak saklanır:

```csharp
public enum RolloutStrategy
{
    Canary = 1,          // Canary deployment (aşamalı yayın)
    BlueGreen = 2,       // Blue-Green deployment (anlık geçiş)
    AutoPromote = 3      // BlueGreen + Auto-promotion (zero-downtime)
}
```

### 1.2. Strateji Tespiti (`GetRolloutStrategyAsync`)

```csharp
// Lokasyon: RolloutAppService.cs, satır 101-144
public async Task<(string Strategy, bool IsAutoPromote)> GetRolloutStrategyAsync(
    Guid? serviceId,
    CancellationToken cancellationToken = default)
{
    // 1️⃣ ServiceId ile veritabanından servis bilgisini çek
    var service = await _projectRepoServiceRepository.GetAsync(serviceId.Value);
    
    // 2️⃣ Enum'u Kubernetes manifest formatına dönüştür
    string strategy;
    var isAutoPromote = service.RolloutStrategy == RolloutStrategy.AutoPromote;
    
    if (service.RolloutStrategy == RolloutStrategy.Canary)
    {
        strategy = "canary";
    }
    else if (service.RolloutStrategy == RolloutStrategy.BlueGreen || 
             service.RolloutStrategy == RolloutStrategy.AutoPromote)
    {
        strategy = "blueGreen";
    }
    
    return (strategy, isAutoPromote);
}
```

**🔑 Anahtar Noktalar:**
- ✅ Strateji **veritabanından** gelir (user seçimi)
- ✅ Enum değeri **Kubernetes manifest formatına** çevrilir (`canary` veya `blueGreen`)
- ✅ AutoPromote bir flag olarak döndürülür (BlueGreen + AutoPromotion)

---

## 🔍 2. Rollout Durumunun Tespiti ve Angular Tarafına İletimi

### 2.1. Status Tespiti Akışı

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ROLLOUT STATUS TESPİTİ AKIŞI                     │
└─────────────────────────────────────────────────────────────────────┘

  ┌─────────────────┐
  │  Angular Client │
  │  (UI)           │
  └────────┬────────┘
           │
           │ 1️⃣ HTTP Request: GET /api/services/{id}/rollouts/status
           │
  ┌────────▼─────────┐
  │ ServicesController│
  │                   │
  └────────┬─────────┘
           │
           │ 2️⃣ Call: GetRolloutStatusAsync(serviceId, rolloutName, namespace)
           │
  ┌────────▼──────────────┐
  │ RolloutAppService     │
  │ (Business Logic)      │
  │                       │
  │ Adım 1: Strateji Çek  │───► GetRolloutStrategyAsync(serviceId)
  │                       │      ▼
  │                       │    DB: service.RolloutStrategy
  │                       │
  │ Adım 2: Kubectl       │───► kubectl get rollout {name} -n {namespace} -o json
  │                       │      ▼
  │                       │    Kubernetes API Server
  │                       │      ▼
  │                       │    Argo Rollout Object (JSON)
  │                       │
  │ Adım 3: Parse JSON    │───► RolloutStatusParser.ParseRolloutStatus(jsonData)
  │                       │      ▼
  │                       │    RolloutStatusDto (structured)
  │                       │
  │ Adım 4: Enrich Data   │───► status.RolloutStrategy = (int)service.RolloutStrategy
  │                       │      status.Phase = CalculatePhase(...)
  │                       │      status.StableImage = GetImageFromReplicaSet(...)
  │                       │      status.CanaryImage = GetImageFromReplicaSet(...)
  │                       │
  │ Adım 5: SignalR Push  │───► NotifyRolloutStatusUpdateAsync(serviceId, status)
  │                       │      ▼
  └───────────────────────┘    SignalR Hub
                                ▼
                          ┌─────────────────┐
                          │ RolloutStatusHub│
                          │ (Real-time)     │
                          └────────┬────────┘
                                   │
                                   │ 3️⃣ Broadcast: "RolloutStatusUpdated"
                                   │    to Group: "rollout-service-{serviceId}"
                                   │
  ┌────────────────────────────────▼────────────────────────────────┐
  │  All Connected Angular Clients                                  │
  │  (listening to serviceId group)                                 │
  │                                                                  │
  │  ┌─────────────────────────────────────────────────────────┐   │
  │  │  RolloutStatusSignalRService                            │   │
  │  │  - rolloutStatusUpdate$.next(update)                    │   │
  │  │  - Updates service-dashboard component                  │   │
  │  │  - Re-renders "Canlı Trafik Analizi"                    │   │
  │  └─────────────────────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────────────────────┘
```

---

### 2.2. Rollout Status Tespiti Detayları (`GetRolloutStatusAsync`)

**Lokasyon:** `RolloutAppService.cs`, satır 341-791

```csharp
public async Task<RolloutStatusDto> GetRolloutStatusAsync(
    Guid serviceId,
    string rolloutName,
    string @namespace,
    CancellationToken cancellationToken)
{
    // ═══════════════════════════════════════════════════════════
    // ADIM 1: Kubernetes'ten Rollout Object'ini Çek
    // ═══════════════════════════════════════════════════════════
    var rolloutJson = await ExecuteKubectlCommandAsync(
        $"get rollout {rolloutName} -n {@namespace} -o json",
        kubeConfig
    );
    
    // ═══════════════════════════════════════════════════════════
    // ADIM 2: JSON Parse (RolloutStatusParser kullanarak)
    // ═══════════════════════════════════════════════════════════
    var status = RolloutStatusParser.ParseRolloutStatus(rolloutJson);
    
    // ═══════════════════════════════════════════════════════════
    // ADIM 3: Stratejiyi Veritabanından Ekle
    // ═══════════════════════════════════════════════════════════
    using (_dataFilter.Disable<IMultiTenant>())
    {
        var service = await _projectRepoServiceRepository
            .GetProjectRepoServiceWithById(serviceId, cancellationToken);
        
        if (service != null)
        {
            status.RolloutStrategy = (int)service.RolloutStrategy;
        }
        else
        {
            status.RolloutStrategy = 3; // ✅ DEFAULT: AutoPromote
        }
    }
    
    // ═══════════════════════════════════════════════════════════
    // ADIM 4: Phase/Status Hesaplamaları (Strateji-Specific)
    // ═══════════════════════════════════════════════════════════
    // Canary için: currentWeight, desiredWeight, canaryStep
    // BlueGreen için: activeRevision, previewRevision
    // AutoPromote için: autoPromotionEnabled = true
    
    // ═══════════════════════════════════════════════════════════
    // ADIM 5: ReplicaSet'lerden Gerçek Image'ları Çek
    // ═══════════════════════════════════════════════════════════
    status.StableImage = GetImageFromReplicaSet("stable", namespace);
    status.CanaryImage = GetImageFromReplicaSet("canary", namespace);
    
    // ═══════════════════════════════════════════════════════════
    // ADIM 6: RolloutStrategyStatus Hesapla (UI için)
    // ═══════════════════════════════════════════════════════════
    // Pending → Rollout yok/preparing
    // Progressing → Dağıtım devam ediyor
    // Paused → Manuel müdahale bekliyor
    // Completed → Dağıtım tamamlandı
    // Degraded → Hata durumu
    if (status.Status == "Healthy" && status.Phase == "Completed")
    {
        status.RolloutStrategyStatus = "Completed";
    }
    else if (status.Phase == "Paused")
    {
        status.RolloutStrategyStatus = "Paused";
    }
    else if (status.Phase == "Progressing")
    {
        status.RolloutStrategyStatus = "Progressing";
    }
    
    // ═══════════════════════════════════════════════════════════
    // ADIM 7: SignalR ile Angular'a Bildir (Real-time)
    // ═══════════════════════════════════════════════════════════
    await NotifyRolloutStatusUpdateAsync(serviceId, status, cancellationToken);
    
    return status;
}
```

---

### 2.3. SignalR Bildirimi (`NotifyRolloutStatusUpdateAsync`)

**Lokasyon:** `RolloutAppService.cs`, satır 2181-2230

```csharp
private async Task NotifyRolloutStatusUpdateAsync(
    Guid serviceId,
    RolloutStatusDto status,
    CancellationToken cancellationToken)
{
    // ═══════════════════════════════════════════════════════════
    // Group-Based Broadcasting (Multi-client support)
    // ═══════════════════════════════════════════════════════════
    var groupName = $"rollout-service-{serviceId}";
    
    await _hubContext.Clients.Group(groupName).SendAsync(
        "RolloutStatusUpdated",  // ← Event name
        new
        {
            ServiceId = serviceId,
            RolloutName = status.Name,
            Namespace = status.Namespace,
            Status = status.Status,              // "Healthy" | "Degraded" | "Progressing"
            Phase = status.Phase,                // "Running" | "Paused" | "Completed"
            Strategy = status.Strategy,          // "canary" | "blueGreen"
            RolloutStrategy = status.RolloutStrategy,  // 1=Canary, 2=BlueGreen, 3=AutoPromote
            
            // ═══════════════════════════════════
            // Replica Durumu
            // ═══════════════════════════════════
            Replicas = status.Replicas,
            ReadyReplicas = status.ReadyReplicas,
            UpdatedReplicas = status.UpdatedReplicas,
            StableReplicas = status.StableReplicas,
            CanaryReplicas = status.CanaryReplicas,
            
            // ═══════════════════════════════════
            // Canary-Specific
            // ═══════════════════════════════════
            CurrentWeight = status.CurrentWeight,   // Canary trafik yüzdesi (0-100)
            DesiredWeight = status.DesiredWeight,   // Hedef trafik yüzdesi
            CanaryStep = status.CanaryStep,         // Mevcut adım (0-based)
            CanarySteps = status.CanarySteps,       // Toplam adımlar [10, 25, 50, 75, 100]
            IsCanaryDeploymentActive = status.IsCanaryDeploymentActive,
            
            // ═══════════════════════════════════
            // BlueGreen-Specific
            // ═══════════════════════════════════
            ActiveRevision = status.ActiveRevision,   // Aktif revision (production)
            PreviewRevision = status.PreviewRevision, // Preview revision (staging)
            
            // ═══════════════════════════════════
            // AutoPromote-Specific
            // ═══════════════════════════════════
            AutoPromotionEnabled = status.AutoPromotionEnabled,
            
            // ═══════════════════════════════════
            // Image Bilgileri (ReplicaSet'lerden)
            // ═══════════════════════════════════
            Image = status.Image,             // Generic image
            StableImage = status.StableImage, // Stable (old) version image
            CanaryImage = status.CanaryImage, // Canary (new) version image
            
            // ═══════════════════════════════════
            // UI State (Frontend için hesaplanan)
            // ═══════════════════════════════════
            RolloutStrategyStatus = status.RolloutStrategyStatus,
            // "Pending" | "Progressing" | "Paused" | "Completed" | "Degraded"
            
            Message = status.Message,
            CurrentRevision = status.CurrentRevision,
            UpdatedAt = DateTime.UtcNow
        },
        cancellationToken
    );
}
```

**🔑 Anahtar Noktalar:**
- ✅ **Group-based broadcasting**: Sadece ilgili serviceId'yi dinleyen client'lar alır
- ✅ **Strateji-agnostic payload**: Hem Canary, hem BlueGreen, hem AutoPromote verileri gönderilir
- ✅ **Real-time**: Her `GetRolloutStatusAsync` çağrısında SignalR ile Angular'a push edilir

---

## 🔄 3. Real-Time Status Streaming (SignalR Hub)

### 3.1. RolloutStatusHub (Backend)

**Lokasyon:** `RolloutStatusHub.cs`, satır 17-202

```csharp
public class RolloutStatusHub : Hub
{
    // ═══════════════════════════════════════════════════════════
    // Server-Side Polling Mekanizması (Optional)
    // ═══════════════════════════════════════════════════════════
    public async Task StartRolloutStatusStream(
        string serviceId,
        string rolloutName,
        string @namespace,
        int intervalSeconds = 15)
    {
        // 1️⃣ Validate serviceId
        if (!Guid.TryParse(serviceId, out var serviceGuid))
            throw new HubException("Invalid serviceId");
        
        intervalSeconds = Math.Clamp(intervalSeconds, 5, 60);
        
        // 2️⃣ Background task: Poll RolloutAppService her X saniyede
        _ = Task.Run(async () =>
        {
            while (!cancellationToken.IsCancellationRequested)
            {
                try
                {
                    // Scope oluştur (dependency injection)
                    using var scope = _scopeFactory.CreateScope();
                    var rolloutAppService = scope.ServiceProvider
                        .GetRequiredService<IRolloutAppService>();
                    
                    // RolloutAppService'ten status çek
                    var status = await rolloutAppService
                        .GetRolloutStatusAsync(serviceGuid, rolloutName, @namespace);
                    
                    // 3️⃣ Client'a gönder (SignalR push)
                    await Clients.Caller.SendAsync(
                        "RolloutStatusUpdated",
                        BuildRolloutStatusPayload(serviceGuid, status)
                    );
                }
                catch { /* Ignore errors, retry next interval */ }
                
                // 4️⃣ Interval bekle
                await Task.Delay(TimeSpan.FromSeconds(intervalSeconds));
            }
        });
    }
    
    // ═══════════════════════════════════════════════════════════
    // Group Management (Service-specific subscriptions)
    // ═══════════════════════════════════════════════════════════
    public async Task JoinServiceGroup(string serviceId)
    {
        var groupName = $"rollout-service-{serviceId}";
        await Groups.AddToGroupAsync(Context.ConnectionId, groupName);
        await Clients.Caller.SendAsync("GroupJoined", groupName);
    }
    
    public async Task LeaveServiceGroup(string serviceId)
    {
        var groupName = $"rollout-service-{serviceId}";
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, groupName);
        await StopRolloutStatusStream(serviceId);
    }
}
```

**🎯 İki Farklı Yöntem:**

| Yöntem | Açıklama | Kullanım |
|--------|----------|----------|
| **Server-Side Polling** | Hub içinde background task çalışır, her X saniyede status çeker | `StartRolloutStatusStream()` |
| **Event-Based Push** | Backend'de bir şey değiştiğinde (promote, traffic update) `NotifyRolloutStatusUpdateAsync` çağrılır | Her `GetRolloutStatusAsync` çağrısında |

---

### 3.2. RolloutStatusSignalRService (Angular)

**Lokasyon:** `rollout-status-signalr.service.ts`, satır 58-301

```typescript
export class RolloutStatusSignalRService {
  private rolloutStatusConnection?: signalR.HubConnection;
  
  // ═══════════════════════════════════════════════════════════
  // Public Observables (Component'ler subscribe olur)
  // ═══════════════════════════════════════════════════════════
  public rolloutStatusUpdate$ = new Subject<RolloutStatusUpdate>();
  public autoPromoteStatusUpdate$ = new Subject<AutoPromoteStatusUpdate>();
  public connectionStatus$ = new BehaviorSubject<boolean>(false);
  
  // ═══════════════════════════════════════════════════════════
  // Bağlantı Kurma
  // ═══════════════════════════════════════════════════════════
  private async initializeConnection(): Promise<void> {
    const hubUrl = `${environment.apis.default.url}/rollout-status-hub`;
    
    this.rolloutStatusConnection = new signalR.HubConnectionBuilder()
      .withUrl(hubUrl, {
        accessTokenFactory: () => localStorage.getItem('access_token') || ''
      })
      .withAutomaticReconnect({
        nextRetryDelayInMilliseconds: (retryContext) => {
          // Exponential backoff: 0s, 2s, 10s, 30s
          if (retryContext.previousRetryCount === 0) return 0;
          if (retryContext.previousRetryCount === 1) return 2000;
          if (retryContext.previousRetryCount === 2) return 10000;
          return 30000;
        }
      })
      .build();
    
    // ═══════════════════════════════════════════════════════════
    // Event Listeners (Backend'den gelen mesajları dinle)
    // ═══════════════════════════════════════════════════════════
    this.rolloutStatusConnection.on(
      'RolloutStatusUpdated',
      (update: RolloutStatusUpdate) => {
        // Subject'e push et → Component'ler otomatik güncellenir
        this.rolloutStatusUpdate$.next(update);
      }
    );
    
    this.rolloutStatusConnection.on(
      'AutoPromoteStatusUpdate',
      (update: AutoPromoteStatusUpdate) => {
        this.autoPromoteStatusUpdate$.next(update);
      }
    );
    
    // Bağlantıyı başlat
    await this.rolloutStatusConnection.start();
  }
  
  // ═══════════════════════════════════════════════════════════
  // Service Group'a Katılma (Specific serviceId için)
  // ═══════════════════════════════════════════════════════════
  public async joinServiceGroup(serviceId: string): Promise<void> {
    if (!this.rolloutStatusConnection || 
        this.rolloutStatusConnection.state !== signalR.HubConnectionState.Connected) {
      await this.initializeConnection();
    }
    
    // Backend'e group join request gönder
    await this.rolloutStatusConnection!.invoke('JoinServiceGroup', serviceId);
    this.activeServiceGroups.add(serviceId);
  }
  
  // ═══════════════════════════════════════════════════════════
  // Server-Side Polling Başlat (Optional)
  // ═══════════════════════════════════════════════════════════
  public async startRolloutStatusStream(
    serviceId: string,
    rolloutName: string,
    namespace: string,
    intervalSeconds = 15
  ): Promise<void> {
    await this.rolloutStatusConnection!.invoke(
      'StartRolloutStatusStream',
      serviceId,
      rolloutName,
      namespace,
      intervalSeconds
    );
  }
}
```

---

### 3.3. Component'te Kullanım (service-dashboard.component.ts)

```typescript
export class ServiceDashboardComponent implements OnInit, OnDestroy {
  private rolloutStatusSubscription?: Subscription;
  
  constructor(
    private rolloutStatusSignalR: RolloutStatusSignalRService
  ) {}
  
  async ngOnInit() {
    // ═══════════════════════════════════════════════════════════
    // 1️⃣ SignalR Bağlantısını Başlat
    // ═══════════════════════════════════════════════════════════
    this.rolloutStatusSignalR.startConnection();
    
    // ═══════════════════════════════════════════════════════════
    // 2️⃣ Service Group'a Katıl (Bu service'i dinle)
    // ═══════════════════════════════════════════════════════════
    await this.rolloutStatusSignalR.joinServiceGroup(this.serviceId);
    
    // ═══════════════════════════════════════════════════════════
    // 3️⃣ Real-time Updates'i Dinle
    // ═══════════════════════════════════════════════════════════
    this.rolloutStatusSubscription = this.rolloutStatusSignalR
      .rolloutStatusUpdate$
      .pipe(
        filter(update => update.serviceId === this.serviceId)
      )
      .subscribe(update => {
        // ✅ Rollout status güncellendi → UI'ı güncelle
        this.rolloutStatus = {
          ...this.rolloutStatus,
          phase: update.phase,
          status: update.status,
          currentWeight: update.currentWeight,
          desiredWeight: update.desiredWeight,
          stableReplicas: update.stableReplicas,
          canaryReplicas: update.canaryReplicas,
          stableImage: update.stableImage,
          canaryImage: update.canaryImage,
          rolloutStrategy: update.rolloutStrategy,
          rolloutStrategyStatus: update.rolloutStrategyStatus,
          // ... diğer alanlar
        };
        
        // ✅ Change Detection tetikle
        this.cdr.detectChanges();
        
        // ✅ "Canlı Trafik Analizi" güncellenir (child component)
        // Pod-liveliness-cytoscape.component otomatik güncellenir
      });
  }
  
  ngOnDestroy() {
    // ═══════════════════════════════════════════════════════════
    // 4️⃣ Cleanup: Group'tan ayrıl
    // ═══════════════════════════════════════════════════════════
    this.rolloutStatusSignalR.leaveServiceGroup(this.serviceId);
    this.rolloutStatusSubscription?.unsubscribe();
  }
}
```

---

## 🚀 4. Rollout Süreci Aşamaları ve SignalR Bildirimleri

### 4.1. Yeni Deployment Başlatıldığında

```
┌──────────────────────────────────────────────────────────────────┐
│  YENİ DEPLOYMENT BAŞLATILDIĞINDA (Pipeline/Tekton)               │
└──────────────────────────────────────────────────────────────────┘

1️⃣ Pipeline: Docker Image Build → Push to Registry → Update Rollout
   │
   │ kubectl set image rollout/my-service app=my-image:v2.0
   │
   ▼
   
2️⃣ Kubernetes: Argo Rollouts Controller Güncellenir
   │
   │ Rollout Object Status Değişir:
   │ - Phase: "Progressing"
   │ - CurrentRevision: 2
   │ - CurrentWeight: 0 (Canary için)
   │
   ▼
   
3️⃣ Angular: HTTP Polling (her 5-15 saniyede)
   │
   │ GET /api/services/{id}/rollouts/status
   │
   ▼
   
4️⃣ Backend: RolloutAppService.GetRolloutStatusAsync()
   │
   │ - kubectl get rollout ... -o json
   │ - Parse JSON → RolloutStatusDto
   │ - Enrich with DB strategy
   │ - Calculate RolloutStrategyStatus
   │
   ▼
   
5️⃣ Backend: NotifyRolloutStatusUpdateAsync()
   │
   │ SignalR Push → Angular Clients
   │
   ▼
   
6️⃣ Angular: RolloutStatusSignalRService
   │
   │ rolloutStatusUpdate$.next(update)
   │
   ▼
   
7️⃣ UI: service-dashboard Component
   │
   │ - "Yayın Hazır" DOT güncellenir (Sarı → Progressing)
   │ - "Canlı Trafik Analizi" güncellenir
   │   * Canary: Trafik okları %0 → %10 animasyonu
   │   * BlueGreen: Preview hostname gösterilir
   │ - "PAUSED" badge gösterilir (eğer pause olduysa)
   │
   └─► KULLANICI EKRANDAKİ DEĞİŞİKLİĞİ ANINDA GÖRÜR
```

---

### 4.2. Canary Stratejisi: Trafik Artırıldığında

```
┌──────────────────────────────────────────────────────────────────┐
│  CANARY: TRAFİK ARTIRILDIĞINDA (Kullanıcı oklara tıkladı)       │
└──────────────────────────────────────────────────────────────────┘

1️⃣ Angular: Kullanıcı "Canlı Trafik Analizi" → Oklara tıklar
   │
   │ POST /api/services/{id}/rollouts/{name}/traffic
   │ Body: { "canaryWeight": 25, "currentWeight": 10 }
   │
   ▼
   
2️⃣ Backend: CanaryTrafficUpdateHandler.UpdateTrafficAsync()
   │
   │ kubectl argo rollouts promote {rolloutName} -n {namespace}
   │
   ▼
   
3️⃣ Kubernetes: Argo Rollouts Controller Traffici Değiştirir
   │
   │ HTTPRoute/Gateway API güncellenir:
   │ - Stable service: 90% → 75%
   │ - Canary service: 10% → 25%
   │
   ▼
   
4️⃣ Backend: Promote sonrası GetRolloutStatusAsync() çağrılır
   │
   │ - Status çekme
   │ - Parse
   │ - NotifyRolloutStatusUpdateAsync() → SignalR push
   │
   ▼
   
5️⃣ Angular: RolloutStatusSignalRService
   │
   │ rolloutStatusUpdate$.next({
   │   currentWeight: 25,
   │   desiredWeight: 25,
   │   canaryStep: 1,
   │   phase: "Progressing"
   │ })
   │
   ▼
   
6️⃣ UI: pod-liveliness-cytoscape Component
   │
   │ - Oklar yeniden çizilir (animasyon)
   │ - Trafik yüzdeleri güncellenir
   │   * Stable → Canary ok: 25%
   │   * Ingress → Stable ok: 75%
   │ - Pod sayıları güncellenir (eğer scale olduysa)
   │
   └─► KULLANICI TRAFİK DEĞİŞİKLİĞİNİ ANINDA GÖRÜR
```

---

### 4.3. BlueGreen/AutoPromote: Full Promote Yapıldığında

```
┌──────────────────────────────────────────────────────────────────┐
│  BLUEGREEN: FULL PROMOTE YAPILDIĞINDA                            │
└──────────────────────────────────────────────────────────────────┘

1️⃣ Angular: Kullanıcı "Full Promote" butonuna tıklar
   │
   │ POST /api/services/{id}/rollouts/{name}/promote
   │ Body: { "full": true }
   │
   ▼
   
2️⃣ Backend: BlueGreenStrategyOrchestrator.PromoteAsync()
   │
   │ kubectl argo rollouts promote {rolloutName} -n {namespace} --full
   │
   ▼
   
3️⃣ Kubernetes: Argo Rollouts Controller
   │
   │ - Preview → Active (anlık geçiş)
   │ - Service selector güncellenir
   │ - Eski Active → terminate edilir
   │
   ▼
   
4️⃣ Backend: Promote işlemi içinde polling loop çalışır
   │
   │ while (rollbackDetected == false && attempt < 10)
   │ {
   │     await Task.Delay(2000);
   │     var status = await GetRolloutStatusAsync(...);
   │     await NotifyRolloutStatusUpdateAsync(...); // ← SignalR push
   │     
   │     if (status.CurrentRevision == targetRevision)
   │         rollbackDetected = true;
   │ }
   │
   ▼
   
5️⃣ Angular: Her 2 saniyede SignalR update gelir
   │
   │ rolloutStatusUpdate$.next({
   │   phase: "Progressing" → "Completed",
   │   activeRevision: "2",
   │   previewRevision: null,
   │   rolloutStrategyStatus: "Completed"
   │ })
   │
   ▼
   
6️⃣ UI: service-dashboard Component
   │
   │ - "PAUSED" badge kaybolur
   │ - "Yayın Hazır" DOT yeşile döner
   │ - "Canlı Trafik Analizi" güncellenir:
   │   * Preview hostname kaybolur
   │   * Sadece Active hostname gösterilir
   │   * Tüm trafik Active'e gider (%100)
   │
   └─► KULLANICI PROMOTE İŞLEMİNİ ANINDA TAKİP EDER (2 sn aralıklarla)
```

---

## 📊 5. Durum Değişikliklerinin Ekranda Yansıması

### 5.1. UI Element'leri ve SignalR Etkisi

| UI Element | SignalR Field | Koşul | Görsel Değişiklik |
|------------|---------------|-------|-------------------|
| **"Yayın Hazır" DOT** | `rolloutStrategyStatus` | `"Pending"` | 🔴 Kırmızı |
|  |  | `"Progressing"` | 🟡 Sarı |
|  |  | `"Paused"` | 🟡 Sarı |
|  |  | `"Completed" && status=="Healthy"` | 🟢 Yeşil |
|  |  | `"Degraded"` | 🔴 Kırmızı |
| **"PAUSED" Badge** | `phase`, `rolloutStrategyStatus` | `phase == "Paused"` OR `rolloutStrategyStatus == "Paused"` | ⚠️ Sarı banner gösterilir |
| **Preview Hostname** | `previewRevision`, `activeRevision`, `phase` | BlueGreen: `previewRevision != activeRevision` | 🔗 Link gösterilir |
| **Trafik Okları** | `currentWeight`, `stableReplicas`, `canaryReplicas` | Canary: `currentWeight > 0` | ➡️ Animasyonlu oklar çizilir |
| **Pod Sayıları** | `stableReplicas`, `canaryReplicas`, `readyReplicas` | - | 📊 Pod counters güncellenir |
| **Revision Bilgisi** | `currentRevision`, `activeRevision`, `previewRevision` | - | 🔢 Revision numaraları gösterilir |
| **Image Tags** | `stableImage`, `canaryImage` | - | 🏷️ Docker image tag'leri gösterilir |

---

### 5.2. Örnek Senaryo: Canary Deployment (0% → 100%)

```typescript
// ═══════════════════════════════════════════════════════════
// T = 0s: Deployment Başladı
// ═══════════════════════════════════════════════════════════
{
  phase: "Progressing",
  rolloutStrategyStatus: "Progressing",
  currentWeight: 0,
  desiredWeight: 0,
  canaryStep: 0,
  stableReplicas: 3,
  canaryReplicas: 1,
  stableImage: "my-app:v1.0",
  canaryImage: "my-app:v2.0"
}
// UI: 🟡 Yayın Hazır (Sarı), Canary pod oluşturuldu, trafik yok

// ═══════════════════════════════════════════════════════════
// T = 30s: Kullanıcı %10 trafik verdi
// ═══════════════════════════════════════════════════════════
{
  phase: "Progressing",
  rolloutStrategyStatus: "Progressing",
  currentWeight: 10,
  desiredWeight: 10,
  canaryStep: 0,
  stableReplicas: 3,
  canaryReplicas: 1
}
// UI: 🟡 Yayın Hazır (Sarı), Oklar gösterildi: 10% → Canary, 90% → Stable

// ═══════════════════════════════════════════════════════════
// T = 60s: Kullanıcı %50 trafik verdi
// ═══════════════════════════════════════════════════════════
{
  phase: "Progressing",
  currentWeight: 50,
  desiredWeight: 50,
  canaryStep: 2,
  stableReplicas: 2,
  canaryReplicas: 2
}
// UI: 🟡 Oklar güncellendi: 50% → Canary, 50% → Stable, Pod sayıları eşitlendi

// ═══════════════════════════════════════════════════════════
// T = 120s: Kullanıcı %100 trafik verdi (Full Promote)
// ═══════════════════════════════════════════════════════════
{
  phase: "Progressing",
  currentWeight: 100,
  desiredWeight: 100,
  canaryStep: 4,
  stableReplicas: 0,
  canaryReplicas: 3
}
// UI: 🟡 Tüm trafik Canary'ye, Stable pod'lar terminate ediliyor

// ═══════════════════════════════════════════════════════════
// T = 130s: Rollout Tamamlandı
// ═══════════════════════════════════════════════════════════
{
  phase: "Completed",
  status: "Healthy",
  rolloutStrategyStatus: "Completed",
  currentWeight: null,
  stableReplicas: 3,
  canaryReplicas: 0,
  stableImage: "my-app:v2.0",
  canaryImage: null
}
// UI: 🟢 Yayın Hazır (Yeşil), Canary olmadığı için oklar kaldırıldı, v2.0 stable oldu
```

---

## ✅ 6. Özet: Sorulara Cevaplar

### ❓ Soru 1: Backend tarafı servisin sahip olduğu rollout stratejisini nasıl anlıyor?

**Cevap:**
1. **Veritabanından:** `ProjectRepoService` tablosunda `RolloutStrategy` enum'u saklanır (1=Canary, 2=BlueGreen, 3=AutoPromote)
2. **`GetRolloutStrategyAsync()` metodu:** ServiceId ile veritabanından stratejiyi çeker
3. **Kubernetes manifest formatına çevirir:** Enum → `"canary"` veya `"blueGreen"` string'i
4. **Her status sorgusunda eklenir:** `GetRolloutStatusAsync()` içinde `status.RolloutStrategy = (int)service.RolloutStrategy`

---

### ❓ Soru 2: Servisin Rollout durumunu doğru bir şekilde tespiti ve Angular tarafına iletme süreci nasıl işliyor?

**Cevap:**
1. **Tespit:**
   - `kubectl get rollout {name} -n {namespace} -o json` ile Kubernetes'ten Argo Rollout object'i çekilir
   - `RolloutStatusParser.ParseRolloutStatus()` ile JSON parse edilir
   - ReplicaSet'lerden gerçek pod image'ları alınır
   - Strateji veritabanından eklenir
   - `RolloutStrategyStatus` hesaplanır (Pending/Progressing/Paused/Completed/Degraded)

2. **Angular'a İletim:**
   - **SignalR Hub:** `NotifyRolloutStatusUpdateAsync()` her status update'inde çağrılır
   - **Group-based broadcasting:** `rollout-service-{serviceId}` grubuna push edilir
   - **Angular service:** `RolloutStatusSignalRService.rolloutStatusUpdate$` Observable'ına emit edilir
   - **Component:** `service-dashboard.component.ts` subscribe olur, UI otomatik güncellenir

---

### ❓ Soru 3: Yeni Rollout sürecinin tamamlanıp tamamlanmadığını, hangi aşamada olduğunu Angular tarafına SignalR ile durumu değiştiği anda bildiriyor mu?

**Cevap: ✅ EVET**

**3 Farklı Bildirim Mekanizması:**

1. **HTTP Polling + SignalR Push (Default):**
   - Angular her 5-15 saniyede HTTP request atar: `GET /api/services/{id}/rollouts/status`
   - Backend `GetRolloutStatusAsync()` çağrılır
   - Status hesaplandıktan sonra `NotifyRolloutStatusUpdateAsync()` otomatik çağrılır
   - SignalR ile tüm bağlı client'lara push edilir

2. **Server-Side Polling (Optional):**
   - `RolloutStatusHub.StartRolloutStatusStream()` çağrılırsa
   - Backend içinde background task başlatılır
   - Her X saniyede (default 15s) `GetRolloutStatusAsync()` çağrılır
   - Her update SignalR ile Angular'a push edilir

3. **Event-Driven Push (Promote/Rollback sırasında):**
   - Kullanıcı promote/rollback işlemi başlattığında
   - Backend içinde 2 saniyelik polling loop çalışır (max 10 attempt = 20 saniye)
   - Her denemede `NotifyRolloutStatusUpdateAsync()` çağrılır
   - Angular ekranında **2 saniyede bir** update görür

**Sonuç:** Rollout süreci boyunca her durum değişikliği (Progressing → Paused → Completed) Angular tarafına SignalR ile **gerçek zamanlı** olarak bildirilir. Kullanıcı ekranda:
- ✅ Trafik yüzde değişikliklerini
- ✅ Pod replica sayılarını
- ✅ Phase/Status değişikliklerini
- ✅ Revision geçişlerini
- ✅ Image tag güncellemelerini

**ANINDA GÖRÜR** 🚀

---

## 🔗 İlgili Dosyalar

### Backend:
- `RolloutAppService.cs` (satır 341-791): Status tespiti ve SignalR push
- `RolloutStatusHub.cs` (satır 17-202): SignalR hub ve server-side polling
- `RolloutStatusParser.cs`: JSON parsing helper
- `NotifyRolloutStatusUpdateAsync()` (satır 2181-2230): SignalR broadcast metodu

### Frontend:
- `rollout-status-signalr.service.ts`: SignalR connection management
- `service-dashboard.component.ts`: UI state management
- `pod-liveliness-cytoscape.component.ts`: "Canlı Trafik Analizi" visualization

---

## 📌 Notlar

1. ✅ **Multi-tenant support:** Her tenant kendi SignalR group'unda (`tenant-{tenantId}`)
2. ✅ **Reconnection:** Bağlantı koptuğunda otomatik yeniden bağlanır (exponential backoff)
3. ✅ **Group cleanup:** Component destroy olduğunda `leaveServiceGroup()` çağrılır
4. ✅ **Strategy isolation:** Yeni refactored v2 endpoints'lerde her strateji izole (Canary/BlueGreen/AutoPromote)

---

**Son Güncelleme:** 2026-02-09  
**Yazar:** AI Assistant (Cursor)
