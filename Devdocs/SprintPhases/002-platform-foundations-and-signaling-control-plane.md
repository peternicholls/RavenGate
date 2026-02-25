# Phase 002 — Platform Foundations & Signaling Control Plane

## Objective

Deliver stable platform foundations and a production-viable signaling control plane that supports both SSE and long-poll under PHP constraints.

## Scope

- Laravel control-plane baseline with authenticated share/session primitives.
- Signaling APIs and message envelope compliance.
- SSE lifecycle controls and transport fallback behavior.
- Core observability for signaling health and error diagnostics.

## In-Scope Workstreams

1. **Backend platform setup**
   - Provision Laravel modules for shares, participants, and signaling events.
   - Implement environment/config management for security-sensitive settings.
   - Add persistence model for room state, event IDs, and connection replacement markers.
2. **Signaling transport implementation**
   - Implement SSE stream with server-enforced lifetime cap and randomized closure window.
   - Enforce single active stream per user/share and explicit `replaced` event semantics.
   - Implement long-poll endpoint parity (same message contract as SSE).
3. **Protocol and input enforcement**
   - Implement canonical signaling envelope (`type`, `share_code`, `seq`, `ts`, `payload`).
   - Add strict payload limits for SDP and ICE; return deterministic validation errors.
   - Add replay and ordering guards using sequence and timestamp validation rules.
4. **Operational controls**
   - Add API-level rate limiting baseline for join and signaling endpoints.
   - Capture structured logs and metrics for stream churn, reconnect rates, and failures.
   - Produce runbook notes for PHP worker budgeting and stream lifecycle behavior.

## Out of Scope

- Data-channel file payload transfer.
- App-layer E2EE.
- UX polish beyond transport diagnostics needed for integration.

## Deliverables

- Signaling endpoints with SSE + long-poll functional parity.
- Envelope validator and payload-size guardrails.
- Audit and observability instrumentation for signaling operations.
- Control-plane integration contract for frontend Phase 003/004 work.

## Dependencies

- Phase 001 architecture and security decisions.
- Database/schema migration capability in target environments.

## Exit / Acceptance Criteria

- Sender and receiver can exchange all signaling message types through either transport.
- SSE lifecycle constraints are enforced and observable in logs/metrics.
- Oversized/invalid payloads are consistently rejected with documented error codes.
- Long-poll fallback works transparently when SSE cannot be maintained.

## Primary Risks

- PHP worker starvation from misconfigured stream limits.
- Message ordering/replay edge cases under reconnect pressure.
- Behavior drift between SSE and long-poll implementations.

## Definition of Done

- Transport conformance scenarios pass for normal, reconnect, and replacement flows.
- Operational limits are codified, tested, and documented.
- Backend is ready for trust-gate and WebRTC workflow integration.
