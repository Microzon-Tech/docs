# DevOpsZon Rollout API v2 - Strategy-Isolated Endpoints

## Migration Strategy
- **v1 endpoints**: Existing endpoints remain UNCHANGED for backward compatibility
- **v2 endpoints**: New strategy-specific endpoints with isolated logic
- **Gradual transition**: Angular will migrate to v2 endpoints one strategy at a time

---

## v1 Endpoints (Unified - Backward Compatible)

### Status & History
- `GET /api/app/services/{serviceId}/rollouts/status`
  - **Returns**: `RolloutStatusDto`
  - **Strategy**: Unified (all strategies)
  - **Status**: ✅ Active, backward compatible
  
- `GET /api/app/services/{serviceId}/rollouts/history`
  - **Returns**: `List<RolloutHistoryDto>`
  - **Strategy**: Unified
  - **Status**: ✅ Active

- `GET /api/app/services/{serviceId}/rollouts/list`
  - **Returns**: `List<RolloutDto>`
  - **Strategy**: Unified
  - **Status**: ✅ Active

### Control Operations (Unified)
- `POST /api/app/services/{serviceId}/rollouts/{rolloutName}/promote`
  - **Body**: `RolloutPromoteRequest { ServiceId, RolloutName, Namespace, Full, InitiatedBy }`
  - **Returns**: `RolloutPromoteResponse`
  - **Strategy**: Unified (works for Canary, BlueGreen, AutoPromote)
  - **Status**: ✅ Active, backward compatible
  - **Note**: Behavior changes based on service's `RolloutStrategy` field

- `POST /api/app/services/{serviceId}/rollouts/{rolloutName}/abort`
  - **Returns**: `RolloutAbortResponse`
  - **Status**: ✅ Active

- `POST /api/app/services/{serviceId}/rollouts/{rolloutName}/pause`
  - **Returns**: `RolloutPauseResponse`
  - **Status**: ✅ Active

- `POST /api/app/services/{serviceId}/rollouts/{rolloutName}/resume`
  - **Returns**: `RolloutResumeResponse`
  - **Status**: ✅ Active

- `POST /api/app/services/{serviceId}/rollouts/{rolloutName}/retry`
  - **Returns**: `RolloutRetryResponse`
  - **Status**: ✅ Active

- `POST /api/app/services/{serviceId}/rollouts/{rolloutName}/rollback`
  - **Query**: `?revision={int}&namespace={string}`
  - **Returns**: `RolloutRollbackResponse`
  - **Status**: ✅ Active

---

## v2 Endpoints (Strategy-Isolated)

### 1️⃣ Canary Strategy

#### Status
- `GET /api/app/services/v2/{serviceId}/rollouts/canary/{rolloutName}/status`
  - **Query**: `?namespace={string}`
  - **Returns**: `RolloutStatusDto` (with canary-specific fields populated)
  - **Fields**: `CurrentWeight`, `DesiredWeight`, `CanaryStep`, `CanarySteps`, `StableReplicas`, `CanaryReplicas`
  - **Status**: 🔄 Contract ready, implementation in `backend-strategy-split`

#### Traffic Control
- `POST /api/app/services/v2/{serviceId}/rollouts/canary/{rolloutName}/traffic`
  - **Body**: `CanaryTrafficUpdateRequest { ServiceId, Namespace, RolloutName, CanaryWeight, BackendMapping? }`
  - **Returns**: `CanaryTrafficUpdateResponse { Success, Message, AppliedCanaryWeight, AppliedStableWeight }`
  - **Description**: Update canary traffic distribution (0-100% canary weight)
  - **Status**: 🔄 Contract ready, implementation in `backend-strategy-split`

#### Promotion
- `POST /api/app/services/v2/{serviceId}/rollouts/canary/{rolloutName}/promote`
  - **Query**: `?full={bool}&namespace={string}`
  - **Returns**: `RolloutPromoteResponse`
  - **Behavior**:
    - `full=false`: Promote one canary step (e.g., 20% → 40%)
    - `full=true`: Promote to 100% (complete rollout)
  - **Status**: 🔄 Contract ready, implementation in `backend-strategy-split`

---

### 2️⃣ BlueGreen Strategy

#### Status
- `GET /api/app/services/v2/{serviceId}/rollouts/bluegreen/{rolloutName}/status`
  - **Query**: `?namespace={string}`
  - **Returns**: `RolloutStatusDto` (with BlueGreen-specific fields populated)
  - **Fields**: `ActiveRevision`, `PreviewRevision`, `AutoPromotionEnabled=false`
  - **Status**: 🔄 Contract ready, implementation in `backend-strategy-split`

#### Promotion
- `POST /api/app/services/v2/{serviceId}/rollouts/bluegreen/{rolloutName}/promote`
  - **Body**: `BlueGreenPromoteRequest { ServiceId, Namespace, RolloutName, FullPromote, InitiatedBy }`
  - **Returns**: `BlueGreenPromoteResponse { Success, Message, ActiveRevision, PreviousActiveRevision, AlreadyPromoted }`
  - **Behavior**: Switch preview → active (instant cutover)
  - **Status**: 🔄 Contract ready, implementation in `backend-strategy-split`

---

### 3️⃣ AutoPromote Strategy

#### Status
- `GET /api/app/services/v2/{serviceId}/rollouts/autopromote/{rolloutName}/status`
  - **Query**: `?namespace={string}`
  - **Returns**: `AutoPromoteStatusDto { RolloutName, Namespace, Phase, PreviewPodsReady, AutoPromotionEnabled, ActiveRevision, PreviewRevision, LastPromoteCheckAt }`
  - **Fields**:
    - `Phase`: Pending | Progressing | Completed
    - `PreviewPodsReady`: bool (pod readiness check result)
    - `AutoPromotionEnabled`: true
  - **Status**: 🔄 Contract ready, implementation in `backend-strategy-split`

---

## Migration Path

### Phase 1: v2 Contract (✅ Complete)
- [x] Define v2 endpoint contracts in `ServicesController.cs`
- [x] Create strategy-specific DTOs (`CanaryTrafficUpdateRequest`, `BlueGreenPromoteResponse`, `AutoPromoteStatusDto`)
- [x] Create `IStrategyOrchestrator` interface
- [x] Create `CanaryStrategyOrchestrator` skeleton
- [x] Add v2 routes to controller (returning `NotImplementedException`)

### Phase 2: Backend Strategy Split (Next)
- [ ] Implement `BlueGreenStrategyOrchestrator`
- [ ] Implement `AutoPromoteStrategyOrchestrator`
- [ ] Refactor `UpdateHttpRouteTrafficAppService` into strategy-specific handlers
- [ ] Refactor `ServiceManagementAppService` into strategy-specific handlers
- [ ] Wire up v2 endpoints to use `StrategyOrchestratorFactory`

### Phase 3: Angular Migration (After Backend)
- [ ] Create `StrategyAdapter` interface in Angular
- [ ] Implement `CanaryAdapter`, `BlueGreenAdapter`, `AutoPromoteAdapter`
- [ ] Refactor `pod-liveliness-cytoscape` to use adapters
- [ ] Update `service-dashboard` to inject correct adapter based on strategy

---

## Key Differences: v1 vs v2

| Feature | v1 (Unified) | v2 (Isolated) |
|---------|--------------|---------------|
| **Endpoints** | Single endpoints for all strategies | Separate endpoints per strategy |
| **Request DTOs** | Generic `RolloutPromoteRequest` | Strategy-specific (`CanaryTrafficUpdateRequest`) |
| **Response DTOs** | Generic `RolloutPromoteResponse` | Strategy-specific (`BlueGreenPromoteResponse`) |
| **Business Logic** | Shared with if/switch statements | Isolated in strategy orchestrators |
| **Traffic Update** | Generic `UpdateHttpRouteTraffic` | `CanaryTrafficUpdate` (BlueGreen doesn't need it) |
| **Backward Compat** | ✅ Maintained indefinitely | ➖ Not needed (new endpoints) |

---

## Error Handling

All v2 endpoints follow consistent error responses:

- **400 Bad Request**: Invalid input (e.g., canary weight > 100)
- **404 Not Found**: Rollout or service not found
- **409 Conflict**: Operation not allowed in current state (e.g., promote when already promoted)
- **500 Internal Server Error**: Unexpected errors

Example error response:
```json
{
  "error": {
    "code": "CANARY_WEIGHT_INVALID",
    "message": "Canary weight must be between 0 and 100",
    "details": "Provided: 150"
  }
}
```

---

## Testing Strategy

### v1 Endpoints
- ✅ Keep existing tests unchanged
- ✅ No regression testing needed (endpoints remain identical)

### v2 Endpoints
- [ ] Unit tests for each orchestrator
- [ ] Integration tests for each strategy endpoint
- [ ] E2E tests for Angular-to-v2 migration
- [ ] Load tests for concurrent strategy operations

---

## Notes

1. **v1 endpoints will NOT be deprecated** - they remain as a unified interface for tools/scripts
2. **v2 endpoints are opt-in** - Angular migrates incrementally, other clients can stay on v1
3. **Strategy detection**: v2 endpoints are strategy-aware via URL path (`/canary/`, `/bluegreen/`, `/autopromote/`)
4. **Versioning**: `v2` is in the URL path, future changes will be `v3`, etc.
