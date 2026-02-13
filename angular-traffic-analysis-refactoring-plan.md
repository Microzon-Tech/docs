# Angular Canlı Trafik Analizi - Strategy Pattern Refactoring Planı

## Tarih: 2026-02-14

## 🎯 Hedef

Backend'de yaptığımız gibi Angular'da da **Strategy Pattern** uygulayarak:
- Canary, BlueGreen, AutoPromote logic'ini izole et
- Kod tekrarını azalt
- Test edilebilirliği artır
- Maintainability'yi artır

---

## 📊 Mevcut Durum Analizi

### ArgoRolloutsManagementComponent (2034 satır)
**Sorunlar**:
- ✅ Monolitik component (2000+ satır)
- ✅ Strategy-specific logic karışık (if-else hell)
- ✅ Test edilemez
- ✅ Kod tekrarı yüksek

**Kullanılan Stratejiler**:
```typescript
getRolloutStrategy(): number {
  // 1: Canary, 2: BlueGreen, 3: AutoPromote
}

isCanaryStrategy(): boolean
isBlueGreenStrategy(): boolean
isAutoPromoteStrategy(): boolean
```

**Strategy-Specific Logic**:
- `getCanaryWeightForProgress()` - Canary için
- `getBlueGreenActiveImage()` - BlueGreen için
- `getBlueGreenPreviewImage()` - BlueGreen için
- `hasBlueGreenPreview()` - BlueGreen için
- `shouldShowBlueGreenPreviewCard()` - BlueGreen için
- `canPromoteFull()` - Tüm stratejiler için farklı logic
- `getPromoteTitle()` - Tüm stratejiler için farklı logic
- `getStatusExplanation()` - Tüm stratejiler için farklı logic
- `getTrafficPercentage(revision)` - Tüm stratejiler için farklı logic
- `isCanaryRevision(revision)` - Canary için
- `isCurrentRevision(revision)` - Tüm stratejiler için farklı logic

---

## 🏗️ Yeni Mimari (Strategy Pattern)

### 1. Strategy Interface
```typescript
// strategies/rollout-strategy.interface.ts
export interface IRolloutStrategy {
  // Traffic & Percentage
  getTrafficPercentage(revision: number, status: RolloutStatusDto, history: RolloutHistoryDto[]): number;
  getActiveWeight(): number;
  getCanaryWeight(): number;
  
  // Image Management
  getActiveImage(): string | null;
  getCanaryImage(): string | null;
  
  // Revision Checks
  isCurrentRevision(revision: number): boolean;
  isCanaryRevision(revision: number): boolean;
  
  // UI Display
  canPromoteFull(): boolean;
  getPromoteTitle(): string;
  getStatusExplanation(): string;
  shouldShowPreviewCard(): boolean;
  
  // Status Mapping
  getDisplayedStatus(): string;
}
```

### 2. Strategy Implementations

#### a. CanaryStrategy
```typescript
// strategies/canary-strategy.ts
export class CanaryStrategy implements IRolloutStrategy {
  constructor(
    private status: RolloutStatusDto,
    private history: RolloutHistoryDto[]
  ) {}
  
  getTrafficPercentage(revision: number): number {
    // Canary-specific logic
    const canaryWeight = this.status.canaryWeight || 0;
    const isCanaryActive = this.status.isCanaryDeploymentActive;
    
    if (!isCanaryActive) {
      return revision === this.status.currentRevision ? 100 : 0;
    }
    
    const stableImage = this.getActiveImage();
    const canaryImage = this.getCanaryImage();
    
    const stableRevision = this.history.find(h => h.image === stableImage)?.revision;
    const canaryRevision = this.history.find(h => h.image === canaryImage)?.revision;
    
    if (revision === stableRevision) return 100 - canaryWeight;
    if (revision === canaryRevision) return canaryWeight;
    
    return 0;
  }
  
  getActiveWeight(): number {
    return 100 - (this.status.canaryWeight || 0);
  }
  
  getCanaryWeight(): number {
    return this.status.canaryWeight || 0;
  }
  
  getActiveImage(): string | null {
    return this.status.stableImage || this.status.image || null;
  }
  
  getCanaryImage(): string | null {
    if (!this.status.isCanaryDeploymentActive) return null;
    if (this.status.canaryWeight === 0 || this.status.canaryWeight === null) return null;
    return this.status.canaryImage || null;
  }
  
  isCurrentRevision(revision: number): boolean {
    if (!this.status.isCanaryDeploymentActive) {
      return revision === this.status.currentRevision;
    }
    
    const stableImage = this.getActiveImage();
    const canaryImage = this.getCanaryImage();
    const stableRevision = this.history.find(h => h.image === stableImage)?.revision;
    const canaryRevision = this.history.find(h => h.image === canaryImage)?.revision;
    
    return revision === stableRevision || revision === canaryRevision;
  }
  
  isCanaryRevision(revision: number): boolean {
    if (!this.status.isCanaryDeploymentActive) return false;
    const canaryImage = this.getCanaryImage();
    if (!canaryImage) return false;
    
    const canaryRevision = this.history.find(h => h.image === canaryImage)?.revision;
    return revision === canaryRevision;
  }
  
  canPromoteFull(): boolean {
    return !!this.status.isCanaryDeploymentActive;
  }
  
  getPromoteTitle(): string {
    return this.status.isCanaryDeploymentActive
      ? 'Promote Canary to Stable'
      : 'No Active Canary Deployment';
  }
  
  getStatusExplanation(): string {
    if (this.status.aborted) return 'Canary rollout aborted';
    if (this.status.paused && this.status.isCanaryDeploymentActive) {
      return 'Canary deployment paused';
    }
    if (this.status.isCanaryDeploymentActive) {
      const canary = this.getCanaryWeight();
      const stable = this.getActiveWeight();
      return `Traffic split: ${canary}% canary, ${stable}% stable`;
    }
    return 'All traffic routed to stable version';
  }
  
  shouldShowPreviewCard(): boolean {
    return false; // Canary doesn't use preview card
  }
  
  getDisplayedStatus(): string {
    return this.status.paused ? 'Paused' : (this.status.phase || this.status.status);
  }
}
```

#### b. BlueGreenStrategy
```typescript
// strategies/bluegreen-strategy.ts
export class BlueGreenStrategy implements IRolloutStrategy {
  constructor(
    private status: RolloutStatusDto,
    private history: RolloutHistoryDto[]
  ) {}
  
  getTrafficPercentage(revision: number): number {
    // BlueGreen: Either 100% or 0%
    const activeRevision = this.getStableRevision();
    return revision === activeRevision ? 100 : 0;
  }
  
  getActiveWeight(): number {
    return 100; // BlueGreen always 100% to active
  }
  
  getCanaryWeight(): number {
    return 0; // BlueGreen doesn't use canary weight
  }
  
  getActiveImage(): string | null {
    return this.status.stableImage || this.status.image || null;
  }
  
  getCanaryImage(): string | null {
    // BlueGreen uses "preview" instead of "canary"
    // Only show preview if there's actually a preview deployment
    if (!this.hasPreview()) return null;
    return this.status.canaryImage || this.getPreviewImageFromHistory();
  }
  
  private hasPreview(): boolean {
    if (this.status.previewRevision) return true;
    if (this.status.stableImage && this.status.canaryImage && 
        this.status.stableImage !== this.status.canaryImage) return true;
    return !!this.getPreviewImageFromHistory();
  }
  
  private getPreviewImageFromHistory(): string | null {
    const activeImage = this.getActiveImage();
    const candidates = this.history.filter(h => h.image && h.image !== activeImage);
    if (candidates.length === 0) return null;
    
    const sorted = [...candidates].sort((a, b) => (b.revision || 0) - (a.revision || 0));
    return sorted[0]?.image || null;
  }
  
  private getStableRevision(): number | null {
    const stableImage = this.getActiveImage();
    if (!stableImage) return null;
    return this.history.find(h => h.image === stableImage)?.revision || null;
  }
  
  isCurrentRevision(revision: number): boolean {
    const stableRevision = this.getStableRevision();
    return revision === stableRevision;
  }
  
  isCanaryRevision(revision: number): boolean {
    return false; // BlueGreen uses preview, not canary
  }
  
  canPromoteFull(): boolean {
    return this.hasPreview();
  }
  
  getPromoteTitle(): string {
    return this.hasPreview()
      ? 'Promote Preview to Active'
      : 'No Preview Revision';
  }
  
  getStatusExplanation(): string {
    if (this.hasPreview()) {
      return 'Preview deployment available. Promote to make it active.';
    }
    return 'No preview deployment. Active version serving all traffic.';
  }
  
  shouldShowPreviewCard(): boolean {
    if (this.isHealthy()) return false;
    return !!this.getCanaryImage();
  }
  
  private isHealthy(): boolean {
    const status = (this.status.status || '').toLowerCase();
    const phase = (this.status.phase || '').toLowerCase();
    return status === 'completed' || status === 'healthy' || 
           phase === 'completed' || phase === 'healthy';
  }
  
  getDisplayedStatus(): string {
    if (this.hasPreview()) return 'Paused';
    return this.status.phase || this.status.status;
  }
}
```

#### c. AutoPromoteStrategy
```typescript
// strategies/autopromote-strategy.ts
export class AutoPromoteStrategy extends BlueGreenStrategy {
  // AutoPromote uses BlueGreen logic but with automatic promotion
  // Inherit all BlueGreen logic, just override necessary methods
  
  getStatusExplanation(): string {
    if (this.hasPreview()) {
      return 'Auto-promotion enabled. Preview will be promoted automatically when ready.';
    }
    return 'No preview deployment. Active version serving all traffic.';
  }
  
  getPromoteTitle(): string {
    return 'Auto-Promote Enabled';
  }
}
```

### 3. Strategy Factory
```typescript
// strategies/rollout-strategy.factory.ts
export class RolloutStrategyFactory {
  static create(
    strategyType: number,
    status: RolloutStatusDto,
    history: RolloutHistoryDto[]
  ): IRolloutStrategy {
    switch (strategyType) {
      case 1:
        return new CanaryStrategy(status, history);
      case 2:
        return new BlueGreenStrategy(status, history);
      case 3:
        return new AutoPromoteStrategy(status, history);
      default:
        return new CanaryStrategy(status, history); // Default fallback
    }
  }
}
```

### 4. Component Refactoring
```typescript
// argo-rollouts-management.component.ts
export class ArgoRolloutsManagementComponent {
  private strategyInstance: IRolloutStrategy | null = null;
  
  // Update strategy instance when status changes
  private updateStrategyInstance(): void {
    if (!this.rolloutStatus) return;
    
    const strategyType = this.getRolloutStrategy();
    this.strategyInstance = RolloutStrategyFactory.create(
      strategyType,
      this.rolloutStatus,
      this.rolloutHistory
    );
  }
  
  // Refactored methods (delegate to strategy)
  getTrafficPercentage(revision: number): number {
    return this.strategyInstance?.getTrafficPercentage(revision) || 0;
  }
  
  getCanaryWeightForProgress(): number {
    return this.strategyInstance?.getCanaryWeight() || 0;
  }
  
  getStableImage(): string | null {
    return this.strategyInstance?.getActiveImage() || null;
  }
  
  getCanaryImage(): string | null {
    return this.strategyInstance?.getCanaryImage() || null;
  }
  
  isCurrentRevision(revision: number): boolean {
    return this.strategyInstance?.isCurrentRevision(revision) || false;
  }
  
  isCanaryRevision(revision: number): boolean {
    return this.strategyInstance?.isCanaryRevision(revision) || false;
  }
  
  canPromoteFull(): boolean {
    if (this.operating || this.rolloutStatus?.aborted) return false;
    return this.strategyInstance?.canPromoteFull() || false;
  }
  
  getPromoteTitle(): string {
    if (this.rolloutStatus?.aborted) {
      return this.localizationService.instant('::CannotPromoteAbortedRollout');
    }
    return this.strategyInstance?.getPromoteTitle() || 'Promote';
  }
  
  getStatusExplanation(): string | null {
    return this.strategyInstance?.getStatusExplanation() || null;
  }
  
  shouldShowBlueGreenPreviewCard(): boolean {
    if (!this.isBlueGreenStrategy()) return false;
    return this.strategyInstance?.shouldShowPreviewCard() || false;
  }
  
  getDisplayedStatus(): string {
    return this.strategyInstance?.getDisplayedStatus() || '';
  }
}
```

---

## 📁 Dosya Yapısı

```
src/app/service-management/
├── strategies/
│   ├── rollout-strategy.interface.ts      (Interface)
│   ├── rollout-strategy.factory.ts        (Factory)
│   ├── canary-strategy.ts                 (Canary implementation)
│   ├── bluegreen-strategy.ts              (BlueGreen implementation)
│   └── autopromote-strategy.ts            (AutoPromote implementation)
├── components/
│   ├── argo-rollouts-management.component.ts  (Refactored, uses strategies)
│   └── traffic-flow-cytoscape.component.ts    (Unchanged)
└── services/
    └── rollout-status-signalr.service.ts      (Unchanged)
```

---

## 🎯 Refactoring Adımları

### Phase 1: Strategy Interface & Factory
1. ✅ Create `rollout-strategy.interface.ts`
2. ✅ Create `rollout-strategy.factory.ts`

### Phase 2: Strategy Implementations
3. ✅ Create `canary-strategy.ts`
4. ✅ Create `bluegreen-strategy.ts`
5. ✅ Create `autopromote-strategy.ts`

### Phase 3: Component Refactoring
6. ✅ Inject RolloutStrategyFactory in component
7. ✅ Replace strategy-specific methods with strategy delegation
8. ✅ Update `loadRolloutStatus()` to create strategy instance
9. ✅ Update SignalR subscription to create strategy instance

### Phase 4: Testing & Cleanup
10. ✅ Test Canary flow
11. ✅ Test BlueGreen flow
12. ✅ Test AutoPromote flow
13. ✅ Remove unused code

---

## 💡 Kazanımlar

### 1. Strategy Isolation ✅
- Her strateji kendi logic'ini yönetir
- Canary ↔ BlueGreen ↔ AutoPromote arası bağımlılık yok

### 2. Single Responsibility ✅
- Component sadece koordinasyon yapar
- Strategy sadece calculation yapar
- Service sadece SignalR yönetir

### 3. Testability ✅
- Her strategy unit test edilebilir
- Mock edilebilir
- Isolated test cases

### 4. Maintainability ✅
- 2034 satır → Component ~800 satır + 3 strategy (~200 satır each)
- Her dosya tek bir şey yapar
- Kod tekrarı %0

### 5. Scalability ✅
- Yeni strateji eklemek çok kolay
- Interface-driven design
- Factory pattern

---

## 📊 Metrik Tahmini

| Metrik | Önce | Sonra | İyileşme |
|--------|------|-------|----------|
| Component Size | 2034 satır | ~800 satır | %60 azalma |
| Strategy Logic | Mixed | Isolated | %100 izolasyon |
| Code Duplication | High | None | %100 azalma |
| Testability | Hard | Easy | %1000 iyileşme |
| Maintainability | Hard | Easy | %1000 iyileşme |

---

## 🚀 Implementation Plan

**Estimated Time**: 2-3 hours

**Priority**: HIGH

**Status**: READY TO START

---

## 📝 Notes

- Backend refactoring tamamlandı, aynı pattern'i Angular'da uygulayacağız
- SignalR service değişmeyecek (zaten iyi tasarlanmış)
- TrafficFlowCytoscapeComponent değişmeyecek (visualization only)
- Sadece ArgoRolloutsManagementComponent refactor edilecek

**Başlayalım!** 🚀
