# Phase 006 — Security Hardening: App-Layer E2EE (v1.5)

## Objective

Implement staged v1.5 app-layer end-to-end encryption with AES-256-GCM per chunk, correct nonce/key derivation, and authenticated metadata handling.

## Scope

- ECDH-derived session key establishment tied to trust-gated session context.
- Per-chunk AES-GCM encryption/authentication.
- Nonce derivation implementation per Appendix C.
- AAD enforcement and validation for protocol headers.

## In-Scope Workstreams

1. **Cryptographic key workflow**
   - Implement key agreement flow consistent with PRD constraints and SAS gating.
   - Derive encryption keys with explicit context binding to session identifiers.
   - Prevent key reuse across sessions and enforce secure key lifecycle cleanup.
2. **Chunk encryption/authentication**
   - Encrypt payload chunks with AES-256-GCM and validate authentication tags.
   - Bind AAD to required header metadata (file/session/chunk context).
   - Enforce fail-closed behavior on any tag or metadata validation failure.
3. **Nonce and file identifier compliance**
   - Implement `file_id` uint64 counter semantics.
   - Implement nonce derivation using session salt + file/chunk dimensions per Appendix C.
   - Add guardrails and tests to prove nonce uniqueness constraints.
4. **Operational and UX safety controls**
   - Surface meaningful but non-sensitive user errors for crypto failures.
   - Ensure logging redacts keys, plaintext, and sensitive crypto internals.
   - Document upgrade path and compatibility boundaries between v1 and v1.5 peers.

## Out of Scope

- Alternative cipher suite support.
- Key escrow, server-side key access, or relay decryption pathways.

## Deliverables

- E2EE-enabled transfer mode with authenticated chunk payloads.
- Nonce/key derivation implementation aligned with cryptographic specification.
- Backward-compatibility behavior definition and enforcement.

## Dependencies

- Stable transfer/reconnect baseline from Phases 004–005.
- Finalized security review criteria from Phase 001 threat model.

## Exit / Acceptance Criteria

- Encrypted transfer succeeds and decrypts correctly across supported browsers.
- Tampered chunk payloads or headers are rejected deterministically.
- Nonce uniqueness properties hold across files, chunks, and sessions.
- Sensitive cryptographic material is never exposed in logs or signaling.

## Primary Risks

- Subtle nonce/AAD bugs causing catastrophic cryptographic weakness.
- Interoperability issues across browser crypto APIs.
- Performance regressions from encryption overhead on large transfers.

## Definition of Done

- Security validation checklist is complete and signed off.
- E2EE behavior is test-covered for success, tamper, and compatibility paths.
- Release is ready for final UX/policy execution and launch hardening.
