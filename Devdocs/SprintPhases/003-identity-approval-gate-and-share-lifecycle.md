# Phase 003 — Identity, Approval Gate, and Share Lifecycle

## Objective

Implement trust gates up to transfer eligibility: OIDC authentication, explicit sender approval, and SAS confirmation workflow readiness.

## Scope

- OIDC-based authentication for sender and receiver.
- Share code creation, join flow, and approval/rejection lifecycle.
- SAS display/confirmation flow binding to active share session.
- Session-state transitions and policy guardrails before byte transfer begins.

## In-Scope Workstreams

1. **Identity and authorization integration**
   - Implement OIDC login and verified identity claims ingestion.
   - Enforce authenticated-only share creation and join requests.
   - Bind authorization checks to share ownership and participant role.
2. **Share lifecycle and approval gate**
   - Build sender create-share workflow and receiver join-request flow.
   - Surface receiver identity to sender for explicit allow/deny action.
   - Implement lifecycle states: created, pending approval, approved, rejected, expired.
3. **SAS trust checkpoint**
   - Implement deterministic SAS generation and display pipeline.
   - Require bilateral SAS confirmation before transfer state can progress.
   - Block data-plane start when SAS confirmation is missing or mismatched.
4. **Abuse and audit controls**
   - Apply join-attempt and SAS-confirmation throttles from PRD limits.
   - Add event audit logs for approval, rejection, and confirmation actions.
   - Add protective messaging for suspicious or repeated invalid attempts.

## Out of Scope

- Full chunk transfer protocol implementation.
- App-layer encryption payload wrapping.

## Deliverables

- End-to-end authenticated share flow through approval and SAS gate completion.
- Policy engine for share expiration and invalid-state rejection.
- Audit trail and rate-limit enforcement for trust-gate interactions.

## Dependencies

- Phase 002 signaling transport and envelope enforcement.
- Identity provider configuration and claim mapping.

## Exit / Acceptance Criteria

- Unauthenticated users cannot create or join shares.
- Sender approval is mandatory and correctly gates progression.
- SAS mismatch or non-confirmation prevents transfer initialization.
- Lifecycle transitions are deterministic and recoverable after reconnect.

## Primary Risks

- Identity claim mismatches across providers.
- UX friction in SAS verification reducing successful completions.
- State desynchronization between peers during reconnect/replacement events.

## Definition of Done

- Trust-gate sequence is enforceable end-to-end in integration scenarios.
- Rejection and expiry paths are explicit, test-covered, and user-visible.
- Phase 004 can consume approved-and-verified session state directly.
