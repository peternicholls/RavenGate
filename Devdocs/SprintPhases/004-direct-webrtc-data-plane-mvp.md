# Phase 004 — Direct WebRTC Data Plane MVP

## Objective

Deliver direct browser-to-browser file transfer over WebRTC data channels with protocol-compliant framing, sequencing, and whole-file integrity verification.

## Scope

- Peer connection and ICE handling under no-TURN policy.
- Data channel configuration and reliability controls.
- Manifest exchange, chunk transfer, acknowledgements, and completion signaling.
- Whole-file SHA-256 verification at receiver.

## In-Scope Workstreams

1. **Connection establishment**
   - Implement SDP offer/answer flow tied to signaling control plane.
   - Configure ICE with STUN-only policy and explicit failure path if no direct route exists.
   - Expose connection state and candidate diagnostics for UX and logging.
2. **Transfer protocol implementation**
   - Implement manifest and chunk-stream workflow aligned to Appendix B framing.
   - Support chunk IDs, sequencing, and windowed flow control defaults.
   - Implement ACK handling and retransmission logic for reconnect/resume contexts.
3. **Integrity and completion checks**
   - Compute sender-provided whole-file SHA-256 and verify post-reassembly.
   - Implement transfer-complete signaling semantics and success/failure terminal states.
   - Handle partial-transfer cancellation and deterministic cleanup.
4. **Large-file performance baseline**
   - Tune chunk sizing and send-window limits for multi-GB file behavior.
   - Capture throughput, stall, and retry telemetry for bottleneck analysis.
   - Validate memory behavior during sustained transfers.

## Out of Scope

- App-layer E2EE encryption (handled in Phase 006).
- Refresh-persistent resumability (handled in Phase 005).

## Deliverables

- End-to-end direct transfer for single and multi-file sessions.
- Protocol-conformant frame encoder/decoder implementation.
- SHA-256 verification and error-handling paths for mismatch/failure.

## Dependencies

- Phase 002 signaling plane and Phase 003 trust-gated session readiness.
- Browser APIs for WebRTC DataChannel and Streams availability.

## Exit / Acceptance Criteria

- Transfer succeeds over direct P2P path with no server file-byte relay.
- No-TURN failure path produces clear and actionable user diagnostics.
- Protocol state machine reaches deterministic terminal states across success/failure.
- Whole-file hash mismatch is detected and surfaced with safe failure handling.

## Primary Risks

- NAT edge cases reducing successful direct connection rates.
- Browser-specific channel backpressure behavior affecting throughput.
- Reassembly and memory pressure for very large file sets.

## Definition of Done

- MVP data-plane path is stable for representative large media transfer cases.
- Protocol and state transitions align with PRD appendices.
- Phase 005 reliability enhancements can build on this baseline without redesign.
