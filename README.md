# RavenGate

RavenGate is a browser-native, **direct peer-to-peer** file and folder transfer tool designed for large media workflows (rushes, raw dumps, multi-GB exports). 

It’s built around **explicit trust gates** (identity → approval → SAS verification) and **integrity verification**, with file bytes flowing **only between sender and receiver** — never through our servers.

**Key principles**
- **Direct only (no TURN):** if a direct WebRTC path can’t be established, the transfer won’t proceed (you’ll get clear diagnostics).
- **Trust before bytes:** authenticated users, sender approval, and a Short Authentication String (SAS) check.
- **Integrity first:** whole-file SHA-256 verification (app-layer E2EE is planned).

## Stack
- **Frontend:** SvelteKit + TypeScript (PWA-style UI, transfer engine in TS + Web Workers)
- **Backend (control plane):** Laravel (PHP) for OIDC auth, trusted list/policies, share lifecycle, signaling (SSE + long-poll fallback)
- **Data plane:** WebRTC data channels (direct P2P)

## Status
**Engineering draft / early build.** Expect breaking changes to protocol and storage behavior.

## Phases (realistic)
- **M0 — Prototype:** basic signaling + direct WebRTC connection; single-file transfer with chunking.
- **M1 — MVP:** OIDC auth, approval gate, SAS verification, multi-file, whole-file SHA-256, reconnect + resume *within session*, production-grade UX + diagnostics.
- **M1.5 — Hardening:** resume after refresh (IndexedDB journal where feasible), device binding + passkey step-up (high assurance), optional app-layer E2EE (AES-GCM) and receiver-side ZIP option.

## Why “RavenGate”?
In *Game of Thrones*, ravens are the trusted way to send messages from one place to another — direct, deliberate, and relied upon when it matters. That’s the feeling we want: a **message that arrives intact**, to the **right recipient**, without detours.

The “Gate” part reflects how RavenGate works in practice: transfers don’t start until you pass the **trust gates** (identity → approval → SAS verification). It’s a quiet nod to our early naming ideas too — *Arc* (a clean path between two points) and *Skein* (a strong thread joining endpoints). RavenGate is that same idea, but with a clearer story: a **trusted courier** and a **controlled handoff**.

## License
Apache License 2.0 — see [`LICENSE`](./LICENSE).
