# Phase 005 — Resumability, Storage, and Reliability Hardening

## Objective

Make transfers resilient to disconnects, refreshes, and browser lifecycle interruptions while preserving integrity and predictable recovery behavior.

## Scope

- Resume metadata model and persistence strategy.
- IndexedDB-based journals with graceful degradation paths.
- Reconnect sub-machine implementation and timeout/retry policy.
- Storage quota awareness and user-visible preflight checks.

## In-Scope Workstreams

1. **Resume model and state persistence**
   - Persist chunk completion maps and transfer manifest metadata.
   - Persist minimal session context required for reconnect continuation.
   - Validate consistency between persisted state and live peer session metadata.
2. **Reconnect and recovery orchestration**
   - Implement reconnect state machine from Appendix A (shared sub-machine).
   - Support bounded retries with backoff and explicit terminal conditions.
   - Reconcile partially acknowledged chunks and avoid duplicate write corruption.
3. **Storage management and degradation**
   - Implement quota checks using browser storage estimates.
   - Warn and fail safely when required persistence capacity is unavailable.
   - Provide fallback behavior when persistent storage is blocked/unsupported.
4. **Reliability observability**
   - Emit reconnect/recovery metrics and recovery success rate telemetry.
   - Track resume latency, duplicate chunk incidence, and abort causes.
   - Capture structured error events for support diagnostics.

## Out of Scope

- New cryptographic primitives beyond v1 baseline.
- Non-browser resumability agents.

## Deliverables

- Resume-after-refresh capability where storage support permits.
- Robust reconnect behavior under transient network loss.
- Storage quota and degradation UX aligned to PRD requirements.

## Dependencies

- Stable transfer protocol baseline from Phase 004.
- Browser storage APIs and compatibility constraints from support policy.

## Exit / Acceptance Criteria

- Interrupted transfers can resume without restarting already completed chunks.
- Recovery behavior is deterministic across refresh, temporary disconnect, and tab suspension.
- Storage-constrained scenarios fail gracefully with clear user messaging.
- Reliability metrics are available for acceptance and ongoing tuning.

## Primary Risks

- Browser-specific storage eviction and quota variability.
- Resume-map corruption under abrupt browser termination.
- State divergence between peers during prolonged reconnect windows.

## Definition of Done

- Reliability acceptance scenarios pass in supported browser matrix.
- Recovery and fallback behaviors are documented and supportable.
- Baseline is ready for v1.5 security hardening in Phase 006.
