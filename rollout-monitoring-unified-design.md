# 🚀 Rollout Monitoring: Unified & Intelligent Design

## 🔴 Mevcut Sorunlar

### 1. **3 Farklı Mekanizma = Karmaşıklık**
```
❌ HTTP Polling (Angular → Backend her 5-15s)
❌ Server-Side Polling (RolloutStatusHub içinde per-connection loop)
❌ Event-Driven Push (Promote/Rollback sırasında manuel push)
```

**Sorunlar:**
- 🔴 Aynı işi 3 farklı yöntemle yapıyor
- 🔴 Her Angular client ayrı HTTP request atıyor (N client = N x HTTP request)
- 🔴 Her SignalR connection ayrı polling loop başlatıyor
- 🔴 Kubernetes API'ye gereksiz yük (kubectl get çağrıları)
- 🔴 Multi-tenant isolation yok (her tenant aynı havuzda)
- 🔴 Gereksiz veritabanı query'leri (her poll'da service çekiliyor)
- 🔴 Memory leak riski (ActiveStreams cleanup eksik)

### 2. **Performans Sorunları**

```
100 kullanıcı, 10 servis izliyor = 1000 concurrent polling loop
Her loop 15 saniyede bir kubectl çalıştırıyor
= 1000 / 15 = 66 kubectl call/second
```

### 3. **Scalability Sorunları**

- Pod replica artırıldığında her pod kendi monitoring başlatıyor
- Redis backplane olsa bile gereksiz iş tekrarı
- Kubernetes API rate limit'e takılabilir

---

## ✅ Yeni Tasarım: Unified Event-Driven Architecture

### 🎯 Hedefler

1. ✅ **Tek kaynak:** Kubernetes Watch API (push-based)
2. ✅ **Zero polling:** Backend hiçbir yere HTTP request atmaz
3. ✅ **Multi-tenant isolation:** Her tenant ayrı monitoring context
4. ✅ **On-demand monitoring:** Sadece izlenen servisler için monitoring
5. ✅ **Resource efficient:** Tek Kubernetes watcher tüm rollout'ları izler
6. ✅ **Change detection:** Sadece değişen alanlar push edilir

---

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    UNIFIED MONITORING DESIGN                         │
└─────────────────────────────────────────────────────────────────────┘

 ┌──────────────────────────────────────────────────────────────────┐
 │  RolloutMonitoringBackgroundService (Singleton)                   │
 │  - IHostedService implementation                                  │
 │  - Startup'ta başlar, shutdown'da durur                           │
 │  - Multi-tenant aware                                             │
 │  └───────────────────────────────────────────────────────────────┘
      │
      │ StartAsync()
      ▼
 ┌──────────────────────────────────────────────────────────────────┐
 │  TenantRolloutMonitor (Per Tenant)                                │
 │  - Dictionary<TenantId, TenantRolloutMonitor>                     │
 │  - Her tenant için ayrı Kubernetes context                        │
 │  - Lazy initialization (ilk subscribe'da başlar)                  │
 │  └───────────────────────────────────────────────────────────────┘
      │
      │ AddSubscription(serviceId, rolloutName, namespace)
      ▼
 ┌──────────────────────────────────────────────────────────────────┐
 │  ServiceRolloutWatcher                                            │
 │  - Kubernetes Informer/Watch API                                  │
 │  - kubectl get rollout --watch yerine                             │
 │  - Event-driven: ADDED, MODIFIED, DELETED                         │
 │  - Cache: Son bilinen status (change detection için)              │
 │  └───────────────────────────────────────────────────────────────┘
      │
      │ OnRolloutModified(rollout)
      ▼
 ┌──────────────────────────────────────────────────────────────────┐
 │  ChangeDetector                                                   │
 │  - Diff calculator (old status vs new status)                    │
 │  - Sadece değişen alanları push eder                              │
 │  - Example: currentWeight 10 → 25 (sadece bu push edilir)        │
 │  └───────────────────────────────────────────────────────────────┘
      │
      │ if (hasChanges)
      ▼
 ┌──────────────────────────────────────────────────────────────────┐
 │  SignalR Hub (Push Only)                                          │
 │  - Group: "tenant-{tenantId}-service-{serviceId}"                 │
 │  - Broadcast: "RolloutStatusUpdated"                              │
 │  - Payload: Delta changes + full status                           │
 │  └───────────────────────────────────────────────────────────────┘
      │
      │ WebSocket Push
      ▼
 ┌──────────────────────────────────────────────────────────────────┐
 │  Angular Clients (Passive Receivers)                              │
 │  - NO HTTP polling                                                │
 │  - NO server-side stream request                                  │
 │  - Sadece SignalR connect + subscribe                             │
 │  - Kubernetes değişikliği → Anında bildirim                       │
 │  └───────────────────────────────────────────────────────────────┘
```

---

## 📝 Implementation Plan

### Phase 1: Background Service (Core)

```csharp
// File: RolloutMonitoringBackgroundService.cs
public class RolloutMonitoringBackgroundService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<RolloutMonitoringBackgroundService> _logger;
    private readonly ConcurrentDictionary<Guid, TenantRolloutMonitor> _tenantMonitors;
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("🚀 Rollout Monitoring Background Service started");
        
        // Keep running until app shutdown
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                // Cleanup idle monitors (no active subscriptions)
                await CleanupIdleMonitors();
                await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error in monitoring service");
            }
        }
    }
    
    public void AddSubscription(
        Guid tenantId, 
        Guid serviceId, 
        string rolloutName, 
        string @namespace,
        string kubeConfig)
    {
        // Get or create tenant monitor
        var monitor = _tenantMonitors.GetOrAdd(
            tenantId, 
            _ => new TenantRolloutMonitor(tenantId, _scopeFactory, _logger)
        );
        
        // Add service subscription to tenant monitor
        monitor.AddServiceSubscription(serviceId, rolloutName, @namespace, kubeConfig);
    }
    
    public void RemoveSubscription(Guid tenantId, Guid serviceId)
    {
        if (_tenantMonitors.TryGetValue(tenantId, out var monitor))
        {
            monitor.RemoveServiceSubscription(serviceId);
            
            // If no more subscriptions, remove tenant monitor
            if (monitor.SubscriptionCount == 0)
            {
                _tenantMonitors.TryRemove(tenantId, out _);
                monitor.Dispose();
            }
        }
    }
}
```

---

### Phase 2: Tenant-Level Monitor

```csharp
// File: TenantRolloutMonitor.cs
public class TenantRolloutMonitor : IDisposable
{
    private readonly Guid _tenantId;
    private readonly ConcurrentDictionary<Guid, ServiceRolloutWatcher> _serviceWatchers;
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger _logger;
    
    public int SubscriptionCount => _serviceWatchers.Count;
    
    public void AddServiceSubscription(
        Guid serviceId, 
        string rolloutName, 
        string @namespace,
        string kubeConfig)
    {
        // Idempotent: If already watching, do nothing
        if (_serviceWatchers.ContainsKey(serviceId))
        {
            _logger.LogDebug("Already watching service {ServiceId}", serviceId);
            return;
        }
        
        // Create new watcher
        var watcher = new ServiceRolloutWatcher(
            _tenantId,
            serviceId,
            rolloutName,
            @namespace,
            kubeConfig,
            _scopeFactory,
            _logger
        );
        
        _serviceWatchers.TryAdd(serviceId, watcher);
        
        // Start watching (Kubernetes Informer)
        _ = watcher.StartWatchingAsync();
        
        _logger.LogInformation(
            "🔍 Started monitoring: Tenant={TenantId}, Service={ServiceId}, Rollout={RolloutName}",
            _tenantId, serviceId, rolloutName
        );
    }
    
    public void RemoveServiceSubscription(Guid serviceId)
    {
        if (_serviceWatchers.TryRemove(serviceId, out var watcher))
        {
            watcher.StopWatching();
            watcher.Dispose();
            
            _logger.LogInformation(
                "⏹️ Stopped monitoring: Tenant={TenantId}, Service={ServiceId}",
                _tenantId, serviceId
            );
        }
    }
    
    public void Dispose()
    {
        foreach (var watcher in _serviceWatchers.Values)
        {
            watcher.StopWatching();
            watcher.Dispose();
        }
        _serviceWatchers.Clear();
    }
}
```

---

### Phase 3: Kubernetes Watch (Event-Driven)

```csharp
// File: ServiceRolloutWatcher.cs
public class ServiceRolloutWatcher : IDisposable
{
    private readonly Guid _tenantId;
    private readonly Guid _serviceId;
    private readonly string _rolloutName;
    private readonly string _namespace;
    private readonly string _kubeConfig;
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger _logger;
    
    private Watcher<V1CustomResourceDefinition>? _watcher;
    private RolloutStatusDto? _lastKnownStatus; // Change detection için
    private CancellationTokenSource? _cts;
    
    public async Task StartWatchingAsync()
    {
        _cts = new CancellationTokenSource();
        
        try
        {
            // ═══════════════════════════════════════════════════════════
            // Option 1: Kubernetes Client Informer API (Recommended)
            // ═══════════════════════════════════════════════════════════
            var kubeClient = CreateKubernetesClient(_kubeConfig);
            
            // Watch Argo Rollout CRD
            _watcher = kubeClient.CustomObjects.ListNamespacedCustomObjectWithHttpMessagesAsync(
                group: "argoproj.io",
                version: "v1alpha1",
                namespaceParameter: _namespace,
                plural: "rollouts",
                fieldSelector: $"metadata.name={_rolloutName}",
                watch: true,
                cancellationToken: _cts.Token
            ).Watch<V1CustomResourceDefinition, V1CustomResourceDefinitionList>(
                onEvent: async (type, item) => await OnRolloutEvent(type, item),
                onError: (ex) => _logger.LogError(ex, "Watcher error"),
                onClosed: () => _logger.LogWarning("Watcher closed, reconnecting...")
            );
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to start watcher for {RolloutName}", _rolloutName);
        }
    }
    
    private async Task OnRolloutEvent(WatchEventType eventType, V1CustomResourceDefinition rollout)
    {
        try
        {
            _logger.LogDebug("📡 Rollout event: {EventType}, Rollout={RolloutName}", 
                eventType, _rolloutName);
            
            if (eventType == WatchEventType.Deleted)
            {
                await NotifyRolloutDeleted();
                return;
            }
            
            // ═══════════════════════════════════════════════════════════
            // Parse rollout status (reuse existing parser)
            // ═══════════════════════════════════════════════════════════
            using var scope = _scopeFactory.CreateScope();
            var parser = scope.ServiceProvider.GetRequiredService<RolloutStatusParser>();
            
            var rolloutJson = JsonSerializer.Serialize(rollout);
            var newStatus = parser.ParseRolloutStatus(rolloutJson);
            
            // ═══════════════════════════════════════════════════════════
            // Change Detection: Sadece değişen alanlar push et
            // ═══════════════════════════════════════════════════════════
            if (_lastKnownStatus != null)
            {
                var changes = DetectChanges(_lastKnownStatus, newStatus);
                if (!changes.Any())
                {
                    _logger.LogTrace("No changes detected for {RolloutName}", _rolloutName);
                    return; // ✅ Değişiklik yoksa push etme
                }
                
                _logger.LogDebug("🔄 Changes detected: {Changes}", 
                    string.Join(", ", changes.Select(c => c.Field)));
            }
            
            // Update cache
            _lastKnownStatus = newStatus;
            
            // ═══════════════════════════════════════════════════════════
            // Enrich with database info (strategy, etc.)
            // ═══════════════════════════════════════════════════════════
            await EnrichStatusAsync(newStatus, scope);
            
            // ═══════════════════════════════════════════════════════════
            // SignalR Push (Group-based, multi-tenant)
            // ═══════════════════════════════════════════════════════════
            await NotifyStatusUpdate(newStatus);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing rollout event");
        }
    }
    
    private List<StatusChange> DetectChanges(RolloutStatusDto old, RolloutStatusDto @new)
    {
        var changes = new List<StatusChange>();
        
        if (old.Phase != @new.Phase)
            changes.Add(new StatusChange("Phase", old.Phase, @new.Phase));
        
        if (old.Status != @new.Status)
            changes.Add(new StatusChange("Status", old.Status, @new.Status));
        
        if (old.CurrentWeight != @new.CurrentWeight)
            changes.Add(new StatusChange("CurrentWeight", 
                old.CurrentWeight?.ToString(), @new.CurrentWeight?.ToString()));
        
        if (old.StableReplicas != @new.StableReplicas)
            changes.Add(new StatusChange("StableReplicas", 
                old.StableReplicas?.ToString(), @new.StableReplicas?.ToString()));
        
        if (old.CanaryReplicas != @new.CanaryReplicas)
            changes.Add(new StatusChange("CanaryReplicas", 
                old.CanaryReplicas?.ToString(), @new.CanaryReplicas?.ToString()));
        
        // ... diğer alanlar
        
        return changes;
    }
    
    private async Task NotifyStatusUpdate(RolloutStatusDto status)
    {
        using var scope = _scopeFactory.CreateScope();
        var hubContext = scope.ServiceProvider
            .GetRequiredService<IHubContext<RolloutStatusHub>>();
        
        // ✅ Multi-tenant group naming
        var groupName = $"tenant-{_tenantId}-service-{_serviceId}";
        
        await hubContext.Clients.Group(groupName).SendAsync(
            "RolloutStatusUpdated",
            BuildStatusPayload(status)
        );
        
        _logger.LogDebug("📤 Pushed status update to group: {GroupName}", groupName);
    }
    
    public void StopWatching()
    {
        _cts?.Cancel();
        _watcher?.Dispose();
    }
    
    public void Dispose()
    {
        StopWatching();
        _cts?.Dispose();
    }
}

public record StatusChange(string Field, string? OldValue, string? NewValue);
```

---

### Phase 4: SignalR Hub (Simplified)

```csharp
// File: RolloutStatusHub.cs (Refactored)
public class RolloutStatusHub : Hub
{
    private readonly ILogger<RolloutStatusHub> _logger;
    private readonly RolloutMonitoringBackgroundService _monitoringService;
    private readonly ICurrentTenant _currentTenant;
    private readonly IProjectRepoServiceRepository _serviceRepository;
    
    public override async Task OnConnectedAsync()
    {
        var tenantId = _currentTenant.Id ?? Guid.Empty;
        var connectionId = Context.ConnectionId;
        
        // Auto-join tenant group
        await Groups.AddToGroupAsync(connectionId, $"tenant-{tenantId}");
        
        _logger.LogInformation("✅ Client connected: {ConnectionId}, Tenant={TenantId}",
            connectionId, tenantId);
        
        await base.OnConnectedAsync();
    }
    
    /// <summary>
    /// Start monitoring a specific service
    /// ✅ Angular sadece bu metodu çağırır, başka hiçbir şey yapmaz
    /// </summary>
    public async Task SubscribeToService(string serviceId)
    {
        if (!Guid.TryParse(serviceId, out var serviceGuid))
            throw new HubException("Invalid serviceId");
        
        var tenantId = _currentTenant.Id ?? Guid.Empty;
        
        // Get service info from database (rolloutName, namespace, kubeConfig)
        var service = await _serviceRepository.GetProjectRepoServiceWithById(serviceGuid);
        if (service == null)
            throw new HubException("Service not found");
        
        // Join service-specific group
        var groupName = $"tenant-{tenantId}-service-{serviceGuid}";
        await Groups.AddToGroupAsync(Context.ConnectionId, groupName);
        
        // Register subscription to background service
        _monitoringService.AddSubscription(
            tenantId,
            serviceGuid,
            service.RolloutName,
            service.Namespacek8s,
            service.Cluster.KubeConfig
        );
        
        _logger.LogInformation(
            "📡 Client subscribed: Service={ServiceId}, Connection={ConnectionId}",
            serviceGuid, Context.ConnectionId
        );
        
        // Send initial status (cached, instant response)
        await SendInitialStatus(serviceGuid);
    }
    
    /// <summary>
    /// Stop monitoring a specific service
    /// </summary>
    public async Task UnsubscribeFromService(string serviceId)
    {
        if (!Guid.TryParse(serviceId, out var serviceGuid))
            return;
        
        var tenantId = _currentTenant.Id ?? Guid.Empty;
        var groupName = $"tenant-{tenantId}-service-{serviceGuid}";
        
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, groupName);
        
        // Note: Don't remove from background service immediately
        // Wait for cleanup cycle to remove if no other clients are subscribed
        
        _logger.LogInformation(
            "📴 Client unsubscribed: Service={ServiceId}, Connection={ConnectionId}",
            serviceGuid, Context.ConnectionId
        );
    }
    
    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        // Cleanup handled by background service (periodic check)
        await base.OnDisconnectedAsync(exception);
    }
    
    // ❌ REMOVED: StartRolloutStatusStream (gereksiz, background service hallediyor)
    // ❌ REMOVED: StopRolloutStatusStream (gereksiz)
}
```

---

### Phase 5: Angular Integration (Ultra Simple)

```typescript
// File: rollout-status-signalr.service.ts (Refactored)
@Injectable({ providedIn: 'root' })
export class RolloutStatusSignalRService {
  private connection?: signalR.HubConnection;
  public rolloutStatusUpdate$ = new Subject<RolloutStatusUpdate>();
  
  async startConnection(): Promise<void> {
    if (this.connection?.state === signalR.HubConnectionState.Connected) {
      return;
    }
    
    this.connection = new signalR.HubConnectionBuilder()
      .withUrl(`${environment.apis.default.url}/rollout-status-hub`, {
        accessTokenFactory: () => localStorage.getItem('access_token') || ''
      })
      .withAutomaticReconnect()
      .build();
    
    // ✅ Single event listener (unified)
    this.connection.on('RolloutStatusUpdated', (update: RolloutStatusUpdate) => {
      this.rolloutStatusUpdate$.next(update);
    });
    
    await this.connection.start();
  }
  
  /**
   * ✅ Angular sadece bu metodu çağırır
   * Backend otomatik olarak monitoring başlatır
   */
  async subscribeToService(serviceId: string): Promise<void> {
    if (!this.connection) {
      await this.startConnection();
    }
    
    await this.connection!.invoke('SubscribeToService', serviceId);
  }
  
  async unsubscribeFromService(serviceId: string): Promise<void> {
    if (!this.connection) return;
    await this.connection.invoke('UnsubscribeFromService', serviceId);
  }
  
  // ❌ REMOVED: startRolloutStatusStream (gereksiz)
  // ❌ REMOVED: stopRolloutStatusStream (gereksiz)
  // ❌ REMOVED: joinServiceGroup (otomatik)
  // ❌ REMOVED: leaveServiceGroup (otomatik)
}
```

---

## 📊 Performance Comparison

### Eski Yaklaşım (Mevcut)

```
100 kullanıcı, 10 servis izliyor:

HTTP Polling:
- 100 user × 10 service × (1 request / 15s) = 66 request/s
- Her request: kubectl + DB query + JSON parse

Server-Side Polling:
- 100 connection × 10 service = 1000 background loop
- Her loop: kubectl + DB query + JSON parse
- Memory: ~1000 Task + 1000 CancellationTokenSource

Toplam:
- ~66-100 kubectl call/second
- ~66-100 DB query/second
- High memory usage (per-connection state)
```

### Yeni Yaklaşım (Unified)

```
100 kullanıcı, 10 servis izliyor:

Kubernetes Watch:
- 1 tenant × 10 unique service = 10 Kubernetes watcher
- Her watcher: Event-driven (sadece değişiklik olduğunda trigger)
- Memory: ~10 Watcher instance

Change Detection:
- Sadece değişen alanlar push edilir
- Duplicate event'ler filtrelenir

Toplam:
- 0 kubectl call (Watch API kullanıyor)
- 0 DB query (cache'den servis info çekilir)
- 10x - 100x daha az resource kullanımı
```

---

## 🎯 Benefits Summary

| Feature | Old Approach | New Approach |
|---------|--------------|--------------|
| **HTTP Requests** | 66/second | 0 |
| **Kubectl Calls** | 66/second | 0 (Watch API) |
| **DB Queries** | 66/second | 0 (cached) |
| **Memory Usage** | High (1000 loops) | Low (10 watchers) |
| **Latency** | 0-15 seconds | <1 second |
| **Multi-tenant** | ❌ Shared pool | ✅ Isolated |
| **Scalability** | Poor | Excellent |
| **Complexity** | 3 mechanisms | 1 mechanism |

---

## 🚀 Migration Strategy

### Step 1: Add Background Service (Non-breaking)
```csharp
// Startup.cs
services.AddSingleton<RolloutMonitoringBackgroundService>();
services.AddHostedService(sp => sp.GetRequiredService<RolloutMonitoringBackgroundService>());
```

### Step 2: Feature Flag (A/B Testing)
```csharp
// appsettings.json
{
  "RolloutMonitoring": {
    "UseUnifiedApproach": true,  // ← Toggle
    "FallbackToPolling": false   // ← Fallback for safety
  }
}
```

### Step 3: Gradual Rollout
```
Week 1: Internal testing (10% traffic)
Week 2: Beta users (25% traffic)
Week 3: Production (50% traffic)
Week 4: Full rollout (100% traffic)
Week 5: Remove old code
```

### Step 4: Monitoring & Metrics
```csharp
// Metrics to track:
- Active watchers count
- Events processed/second
- SignalR push latency
- Memory usage
- Kubernetes API errors
```

---

## 🔧 Code Files to Create

### Backend (C#)
1. `RolloutMonitoringBackgroundService.cs` (Core service)
2. `TenantRolloutMonitor.cs` (Per-tenant monitor)
3. `ServiceRolloutWatcher.cs` (Kubernetes watcher)
4. `RolloutChangeDetector.cs` (Diff calculator)
5. `RolloutStatusHub.cs` (Refactored, simplified)

### Frontend (Angular)
6. `rollout-status-signalr.service.ts` (Refactored, simplified)

### Configuration
7. `appsettings.json` (Feature flags)

### Tests
8. `RolloutMonitoringBackgroundServiceTests.cs`
9. `ServiceRolloutWatcherTests.cs`
10. `RolloutChangeDetectorTests.cs`

---

## ✅ Checklist

- [ ] Implement `RolloutMonitoringBackgroundService`
- [ ] Implement `TenantRolloutMonitor`
- [ ] Implement `ServiceRolloutWatcher` (Kubernetes client integration)
- [ ] Implement `RolloutChangeDetector`
- [ ] Refactor `RolloutStatusHub` (remove polling methods)
- [ ] Refactor Angular SignalR service
- [ ] Add feature flag configuration
- [ ] Add comprehensive tests
- [ ] Performance testing (load test with 100+ concurrent users)
- [ ] Documentation update
- [ ] Gradual rollout plan
- [ ] Remove old code after migration

---

**Sonuç:** 3 farklı mekanizmadan **tek, zeki, event-driven mekanizmaya** geçiş. 10x-100x daha performanslı, multi-tenant aware, scalable! 🚀
