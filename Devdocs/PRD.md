# Peer-to-Peer File Transfer — Product Requirements Document

**Version:** 3.0  
**Status:** Engineering Draft  
**Last revised:** 2026-02-25  
**Change summary:** Staged E2EE (v1 transport-only → v1.5 app-layer AES-GCM); corrected SAS/key derivation (no DTLS EKM); AAD requirement added to chunk frames; file_id standardised to uint64 counter; ACK payload de-duplicated; share-code semantics clarified; Tailnet “no relay” language tightened; battery toggle marked best-effort.

-----

## Table of Contents

1. [Product Overview](#1-product-overview)
1. [Goals and Non-Goals](#2-goals-and-non-goals)
1. [Trust Model & Security Architecture](#3-trust-model--security-architecture)
1. [Signaling Plane](#4-signaling-plane)
1. [Data Plane — WebRTC Transfer](#5-data-plane--webrtc-transfer)
1. [Tailnet Mode (Revised)](#6-tailnet-mode-revised)
1. [Encryption](#7-encryption)
1. [Resumability & Storage](#8-resumability--storage)
1. [Browser Support Policy](#9-browser-support-policy)
1. [Rate Limits & Abuse Controls](#10-rate-limits--abuse-controls)
1. [UX & User Education](#11-ux--user-education)
1. [Video-Specific UX Knobs](#12-video-specific-ux-knobs)
1. [SvelteKit Add-on](#13-sveltekit-add-on)
1. [Open Questions](#14-open-questions)
1. [Appendix A — Canonical State Machines](#appendix-a--canonical-state-machines)
1. [Appendix B — Protocol Envelope & Chunk Format](#appendix-b--protocol-envelope--chunk-format)
1. [Appendix C — Crypto Nonce Specification](#appendix-c--crypto-nonce-specification)
1. [Appendix D — Browser Support Matrix](#appendix-d--browser-support-matrix)

-----

## 1. Product Overview

A browser-native, direct peer-to-peer file transfer system targeting large media files (video rushes, raw camera dumps, multi-gigabyte project exports). The transfer path is strictly user-to-user: no relay server ever touches file bytes. A lightweight control plane (signaling + approval) is the only server-side component.

**Core promise:** file bytes travel only between the sender’s browser and the receiver’s browser (or, in Tailnet Mode, between their devices via a user-controlled network — see §6 for scope and caveats).

-----

## 2. Goals and Non-Goals

### Goals

- Direct WebRTC data-channel transfer; zero file-byte relay via our infrastructure.
- Authenticated share links with OIDC-based identity for both parties.
- Explicit receiver-approval gate before any transfer begins.
- Short Authentication String (SAS) confirmation to prevent MITM on the signaling channel.
- Resumable transfers: survive browser refresh, network hiccup, and tab suspension.
- Chunk-level integrity (staged):
  - **v1:** whole-file SHA-256 verification; chunk correctness ensured by WebRTC reliable transport + protocol sequencing. Resume maps and retransmission serve reconnect/resume, not in-flight corruption.
  - **v1.5+:** per-chunk AES-GCM authentication tag (app-layer E2EE) plus whole-file SHA-256.
- PHP-compatible signaling server (shared hosting deployable).
- SvelteKit UI add-on.

### Non-Goals

- **No TURN relay.** If a direct ICE path cannot be established, the transfer does not proceed. Users are shown a clear diagnostic.
- **No server-side file storage** at any point.
- **No companion desktop agent in v1/v1.5.** Tailnet Direct (local-agent mode) is deferred — see §6.
- **No mobile-native apps** (web only; mobile browsers are best-effort).

-----

## 3. Trust Model & Security Architecture

```
OIDC login
    │
    ▼
Share created (sender) ──── share_code ────► Receiver joins
                                                    │
                                            Sender sees request
                                                    │
                                            Sender APPROVES
                                                    │
                                       SAS displayed both sides
                                                    │
                                        Both confirm SAS match
                                                    │
                                        WebRTC data channel open
                                                    │
                                       Encrypted chunks flow P2P
```

**Trust gates (in order):**

1. **OIDC authentication** — both sender and receiver must be authenticated.
1. **Explicit approval** — sender sees receiver’s identity claim and approves the request.
1. **SAS verification** — a short human-readable string derived from the DTLS fingerprint exchange; both parties read it aloud or compare visually before transfer begins.
1. **Whole-file SHA-256** — sender provides hash in manifest; receiver verifies after completion (all versions).
1. **App-layer E2EE (v1.5+)** — per-chunk AES-GCM encryption/authentication with explicit ECDH-derived key; still gated behind SAS confirmation. See §7.

-----

## 4. Signaling Plane

The signaling server brokers SDP and ICE candidates only. It never sees file bytes or decryption keys.

### 4.1 Transport options

|Mode    |Mechanism               |Notes                           |
|--------|------------------------|--------------------------------|
|Primary |Server-Sent Events (SSE)|Low-latency; stateful connection|
|Fallback|Long-poll               |Always available; PHP-safe      |

**Implementation must support both.** Client negotiates at join time.

### 4.2 SSE operational constraints (PHP hosting)

SSE holds an open PHP worker for the duration of the stream. Without guardrails this exhausts the worker pool under modest concurrency. The following are **hard requirements**, not suggestions:

- **Stream lifetime cap:** each SSE stream MUST be closed server-side after 25–55 seconds (randomised to avoid thundering-herd reconnects). The client reconnects automatically using `Last-Event-ID`.
- **Max one SSE stream per user/share:** opening a second stream from the same user+share invalidates the first. The server sends a `{type:"replaced"}` event before closing the old stream.
- **Strict payload size limits:** SDP offer/answer ≤ 8 KB. Each ICE candidate message ≤ 512 bytes. Batch-send at most 20 ICE candidates per message. Reject oversized messages with HTTP 413.
- **PHP worker budget:** document the expected max concurrent shares per worker and set PHP `max_execution_time` accordingly (suggest 60 s).
- **Long-poll always available:** any client that cannot maintain SSE (e.g., proxy strips keep-alive) must be able to fall back transparently. The API surface is identical; only the transport changes.

### 4.3 Signaling message types

```
share_created      → sender → server (creates share room)
share_join         → receiver → server
share_request      → server → sender (receiver wants in)
share_approved     → sender → server
share_rejected     → sender → server
sdp_offer          → sender → server → receiver
sdp_answer         → receiver → server → sender
ice_candidate      → either → server → other
sas_confirm        → either → server → other
transfer_complete  → either → server → other
ping / pong        → either → server (keepalive)
```

All messages are JSON. Envelope defined in Appendix B §B.1.

### 4.4 Signaling security

- All signaling over HTTPS/WSS only.
- **Share codes are join tokens, not transfer tokens:**
  - A share code may be used multiple times to request access until expiry; only one receiver can be approved at a time. Approving a new receiver invalidates any pending request.
  - Share codes expire after 24 hours or when the share is manually closed.
  - A share configured as **single-transfer** (default for production): after `transfer_verified`, the share is closed and the code is invalidated.
- See §10 for brute-force protection.

-----

## 5. Data Plane — WebRTC Transfer

### 5.1 ICE / STUN configuration

- **STUN only.** No TURN servers configured. If all candidate pairs fail, the transfer fails with a user-facing diagnostic.
- ICE candidate gathering: collect all host and server-reflexive candidates. Do not filter by IP prefix at the browser layer — mDNS obfuscation makes IP-based filtering unreliable (see §6 for the Tailnet Mode correction).
- Set `iceTransportPolicy: "all"` for normal mode. Tailnet Mode does not change this value from the browser side (see §6).

### 5.2 Data channel configuration

One data channel per transfer session.

```javascript
{
  label: "transfer",
  ordered: true,      // reliable + ordered (simplest; no head-of-line concern at this layer
                      // because we implement our own flow control above it)
  maxRetransmits: undefined,  // null = reliable delivery
  protocol: "p2p-transfer-v1"
}
```

**Rationale for ordered+reliable:** we implement our own ACK/NACK and window-based flow control. Relying on in-order delivery from the data channel simplifies reassembly and resume-map bookkeeping. If profiling shows head-of-line blocking at high throughput on lossy links, revisit unordered+reliable in v2.

### 5.3 Transfer protocol

See Appendix B for the full wire format.

**Phase 1 — Manifest exchange**

Sender sends `transfer_manifest`:

```json
{
  "type": "transfer_manifest",
  "version": 1,
  "msg_id": "<uuid>",
  "timestamp": 1700000000000,
  "files": [
    {
      "file_id": "<uint64 as decimal, e.g. 1>",
      "name": "B_roll_day2_0047.mov",
      "size": 4294967296,
      "sha256": "<hex>",
      "chunk_count": 4096,
      "chunk_size": 1048576
    }
  ]
}
```

Receiver replies `manifest_ack` (or `manifest_reject` with reason).

**Phase 2 — Chunk transfer**

- Default chunk size: 1 MiB.
- Sender maintains a sliding window (default: 32 chunks in-flight, configurable).
- Each chunk: see Appendix B §B.2 for wire format.
- **v1:** chunks are sent as plaintext; integrity is enforced solely by whole-file SHA-256 at completion. WebRTC DTLS provides transport confidentiality and integrity.
- **v1.5+ (E2EE enabled):** each chunk is AES-GCM encrypted with AAD over the frame header; see §7 and Appendix C.
- Receiver sends `chunk_ack` messages as sparse ranges: `{ received: [[0,31],[33,50]] }`. This keeps ACK messages small even for large files.
- On NACK or gap detection, sender retransmits missing chunks.
- Keepalive `ping`/`pong` every 15 seconds during transfer.

**Phase 3 — Completion**

Sender sends `transfer_done` with whole-file SHA-256. Receiver verifies and replies `transfer_verified` or `transfer_hash_mismatch`.

-----

## 6. Tailnet Mode (Revised)

> **This section was substantially revised from v1.** The previous version claimed the ability to prefer or filter ICE candidates by IP prefix (e.g., 100.64/10). That is not reliably achievable in modern browsers. This section describes what Tailnet Mode actually can and cannot do.

### 6.1 What Tailnet Mode is

Tailnet Mode is a **user-guided connectivity enhancer** for teams that are already using Tailscale. It does not change the browser’s ICE candidate selection algorithm. What it does:

- Displays a pre-flight checklist: “Are both you and your recipient on the same Tailnet right now?”
- Shows Tailscale IP addresses from the user’s profile/config (if the user pastes them in) to help confirm mutual reachability.
- Provides a guided troubleshooting flow when direct WebRTC fails.
- Optionally, enables the **Tailnet Direct fallback** (see §6.3 — out of scope for v1).

### 6.2 What Tailnet Mode cannot do (browser constraints)

- **Cannot deterministically route WebRTC traffic through the Tailnet.** The browser’s ICE agent selects candidate pairs based on its own priority algorithm. When a Tailscale interface is present, its host candidates may be offered, but the browser may obfuscate IPs via mDNS and may prefer a different pair.
- **Cannot guarantee “no relay.”** Even with Tailscale active, Tailscale may route traffic via DERP when it cannot establish a direct WireGuard path. The browser cannot reliably detect whether the underlying tailnet path is direct vs DERP. Tailnet Mode therefore carries this disclosure and remains strictly opt-in.

**Required disclosure to users in Tailnet Mode UI:**

> “Tailnet Mode improves the chance of direct connection when both parties are on Tailscale, but cannot guarantee it. If Tailscale falls back to a DERP relay, your file bytes may transit Tailscale infrastructure. For guaranteed direct transfer with no relays, both parties must be on the same LAN or have open network paths.”

### 6.3 Tailnet Direct fallback (out of scope for v1)

The previously described “send data over HTTPS/WebSocket through the Tailnet to a small sender-side service” requires a **companion agent** running locally on the sender’s machine. This is a separate deliverable with its own distribution, update, and security surface. It is deferred.

**If/when a companion agent is scoped:**

- Define it as “Peer2PeerShare Agent” — a separate versioned component.
- Requires explicit installer, auto-update mechanism, and security review.
- Must not run as root; must bind only to loopback.
- Tailnet Direct only activates when the agent reports a healthy Tailscale direct (non-DERP) connection.

### 6.4 Scope statement

|Feature                               |v1              |v1.5|v2+|
|--------------------------------------|----------------|----|---|
|Tailnet Mode UI / pre-flight checklist|✅               |✅   |✅  |
|Tailnet IP display / guidance         |✅               |✅   |✅  |
|ICE candidate filtering by prefix     |❌ (not feasible)|❌   |❌  |
|Companion agent / Tailnet Direct      |❌               |TBD |TBD|

-----

## 7. Encryption

### 7.1 Baseline (v1): WebRTC transport encryption

All WebRTC data channels are encrypted in transit via DTLS. In **v1**, this is the only encryption layer.

**v1 security posture:**

- Confidentiality and integrity in transit via WebRTC (DTLS + SRTP).
- End-to-end integrity via **whole-file SHA-256** verified by the receiver after transfer completes.
- SAS verification prevents the most common signaling/MITM mixups before any file bytes are sent.
- No app-layer keys; no custom crypto code in v1.

### 7.2 App-layer End-to-End Encryption (v1.5+): AES-256-GCM per chunk

In **v1.5+**, an optional **application-layer** E2EE mode adds per-chunk encryption on top of the DTLS transport layer.

**Key agreement:**

- Both peers generate an ephemeral ECDH key pair (P-256) using WebCrypto (`crypto.subtle.generateKey`).
- Public keys are exchanged over the already-established WebRTC data channel (after SAS confirmation).
- ECDH shared secret is derived via `crypto.subtle.deriveBits`.

**Key derivation (HKDF-SHA-256):**

```
app_key = HKDF-SHA-256(
  ikm  = ecdh_shared_secret,
  salt = <empty>,
  info = "p2pt-e2ee-v1" || share_id || fp_sender || fp_receiver
)
```

Where `fp_sender` and `fp_receiver` are the hex-encoded SHA-256 DTLS certificate fingerprints, obtained from `RTCDtlsTransport.getRemoteCertificates()` after the DTLS handshake completes. This binds the app key to the verified connection identity — the same certificates that underpin the SAS.

> **Note:** browsers do not expose DTLS Exported Keying Material (EKM) directly; the fingerprints are the accessible and correct binding mechanism here.

The server never sees `app_key` or either ECDH public key.

### 7.3 SAS binding (required for E2EE)

SAS is derived from the DTLS fingerprints of both peers. If E2EE is enabled, SAS confirmation MUST complete before the ECDH key exchange frame is sent. This prevents an active MITM from substituting their own ECDH public key.

**SAS derivation:**

```
sas_input = SHA-256(fp_sender || fp_receiver)   // concatenated hex strings, lexically sorted by role
sas_string = base32(sas_input[0:5])              // 8-character base32 string shown to both peers
```

Both peers compute and display independently; the out-of-band comparison is the verification step.

### 7.4 Authenticated headers (AAD, v1.5+)

For E2EE chunk frames, AES-GCM MUST authenticate not only the ciphertext but also the header fields that define placement and identity. This prevents an attacker from transplanting a valid ciphertext to a different file or offset.

- **AAD** = bytes 0–51 of the chunk frame (common header + `payload_len` + `crc32c` + `nonce`). See Appendix B §B.2.
- Ciphertext = payload bytes.
- Tag = `auth_tag` field.

Decryption MUST fail (and the chunk MUST be discarded) if the tag does not verify.

### 7.5 Checksums in E2EE mode

When E2EE is enabled, the AES-GCM tag is authoritative for chunk integrity. The `crc32c` field is optional and used only for pre-decryption diagnostics (e.g., early detection of gross frame corruption). It MUST be covered by AAD if present.

-----

## 8. Resumability & Storage

### 8.1 What is persisted

|Item                         |Storage                                 |Notes                      |
|-----------------------------|----------------------------------------|---------------------------|
|Transfer manifest            |IndexedDB                               |Needed to resume           |
|Received chunk range map     |IndexedDB                               |Sparse range list, small   |
|Per-file SHA-256             |IndexedDB                               |For completion verification|
|File bytes (received)        |File System Access API (where available)|Written incrementally      |
|File bytes (in-flight buffer)|Memory only                             |Not persisted              |

**File bytes are never stored in IndexedDB.** Only the range map and manifest go there.

### 8.2 Graceful degradation

- **If IndexedDB is unavailable or quota is exceeded:** resume after refresh degrades to “resume within session only.” The transfer continues in memory but a page refresh starts from zero. The user is informed.
- **If File System Access API is unavailable** (Firefox, most mobile): received bytes are accumulated in memory and offered as a single download on completion. For very large files this may exceed available RAM; warn the user before transfer begins if estimated file size > 500 MB and FSAPI is unavailable.
- **Mobile browsers:** storage may be purged under memory pressure. Treat mobile as “best effort for resume”; do not promise resume across browser kills.

### 8.3 Storage quota check

Before starting a transfer, check `navigator.storage.estimate()`. If available quota is less than 110% of the manifest total size, warn the user.

-----

## 9. Browser Support Policy

See Appendix D for the full matrix.

**Summary:**

- **Production-recommended:** Chrome/Edge latest two versions.
- **Supported:** Firefox latest two versions (no FSAPI; resume = session only).
- **Best-effort:** Safari latest (significant limitations: no folder picking, no FSAPI, backgrounding kills transfer).
- **Not supported:** IE, legacy Edge, Opera Mini.

Safari-specific known issues must be documented in the user-facing help center, not silently papered over.

-----

## 10. Rate Limits & Abuse Controls

### Share creation

- Max 10 active shares per authenticated user.
- Max 50 share creations per user per hour.

### Join attempts

- Max 5 join attempts per share code per IP per hour.
- After 5 failures: exponential backoff, starting at 30 seconds.
- Share code brute-force: lock share (require re-auth) after 20 failed join attempts across all IPs within 1 hour.

### SAS confirmation

- **Lock share after 3 failed SAS confirmations.** Require the sender to explicitly unlock or re-generate the share.
- Log SAS failures for abuse monitoring (no file content, just event + timestamp + share_id hash).

### ICE / signaling

- Max ICE candidates relayed per session: 200.
- Max concurrent sessions per user: 3.
- Signaling message rate limit: 60 messages/minute per session.

-----

## 11. UX & User Education

### 11.1 Pre-flight network test

Provide a `/test-network` page that:

1. Attempts to gather ICE candidates (host + srflx).
1. Reports whether the browser can reach the STUN server.
1. Displays a plain-language result: “Your network supports direct transfer” / “Direct transfer may fail on your network.”
1. Explains what a symmetric NAT is and what users can do (use mobile hotspot, contact IT, try Tailscale).

### 11.2 Transfer failure UX

When ICE fails:

- Do not show a raw error code.
- Show: “Direct connection could not be established. Your network may be blocking peer-to-peer connections.”
- Offer: “Test your network” link, “Try again” button, and “What can I do?” help article link.
- Do not offer a relay fallback (it would violate the product promise).

### 11.3 No-TURN policy disclosure

On the share creation page, one-line disclosure: “Transfers are direct between devices. This app does not relay your files through any server. Some corporate or restrictive networks may block direct connections.”

-----

## 12. Video-Specific UX Knobs

These are first-class requirements for the target use case (large video file transfers), not stretch goals.

|Feature                           |Requirement                                                                                                                                                                                                                                                                                |
|----------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|**Throttle slider**               |Sender can cap upload bandwidth (e.g., 10/25/50/100 Mbps or unlimited). Implemented as window-size + inter-chunk delay control at application layer.                                                                                                                                       |
|**Pause on battery (best-effort)**|Optional toggle; enabled only when the browser exposes battery state via the Battery Status API. If the API is unavailable (Safari, most iOS, Firefox in some configurations), hide the toggle entirely or show “Not supported on this browser.” Do not promise this feature on Safari/iOS.|
|**Background hash worker**        |SHA-256 computation and chunk-hash verification run in a dedicated Web Worker with `requestIdleCallback`-style yielding so the UI thread stays responsive during 100 GB+ transfers.                                                                                                        |
|**Per-file progress**             |Show individual file progress bars in multi-file transfers, not just aggregate.                                                                                                                                                                                                            |
|**ETA display**                   |Rolling 30-second average throughput → estimated time remaining.                                                                                                                                                                                                                           |
|**Transfer log**                  |Downloadable JSON log of transfer events (start, chunk acks, pauses, completion) for post-transfer diagnostics.                                                                                                                                                                            |

-----

## 13. SvelteKit Add-on

The SvelteKit integration exposes the core transfer engine as a set of Svelte stores and components.

### 13.1 Package structure

```
@p2ptransfer/sveltekit
  src/
    lib/
      stores/
        transfer.ts      // writable store: TransferSession
        signaling.ts     // derived store: SignalingState
      components/
        ShareCreate.svelte
        ShareJoin.svelte
        TransferProgress.svelte
        SASConfirm.svelte
        NetworkTest.svelte
      server/
        signaling.ts     // SvelteKit server-side signaling handler
```

### 13.2 Server-side signaling integration

The SvelteKit server handler uses `+server.ts` endpoints. It supports both SSE and long-poll transports. The SSE implementation must respect the constraints in §4.2: stream lifetime cap, one stream per user/share, payload limits.

```typescript
// src/routes/api/transfer/events/+server.ts
export const GET: RequestHandler = async ({ url, locals }) => {
  // validate auth, get share_id
  // return SSE stream, capped at MAX_SSE_LIFETIME_MS
};
```

### 13.3 No tracking parameters

All documentation links and references use clean URLs. No `utm_source`, `oai_citation`, or tracking fragments anywhere in source code, comments, or docs.

-----

## 14. Open Questions

### Staging summary

|Capability                                    |v1           |v1.5|v2+|
|----------------------------------------------|-------------|----|---|
|WebRTC direct transfer + OIDC + approval + SAS|✅            |✅   |✅  |
|Whole-file SHA-256 integrity                  |✅            |✅   |✅  |
|Resume within session                         |✅            |✅   |✅  |
|Resume across refresh (IndexedDB + FSAPI)     |⚠️ best-effort|✅   |✅  |
|App-layer E2EE (ECDH + AES-GCM per chunk)     |❌            |✅   |✅  |
|AAD-authenticated chunk headers               |❌            |✅   |✅  |
|Tailnet Mode UI + pre-flight                  |✅            |✅   |✅  |
|Companion agent / Tailnet Direct              |❌            |❌   |TBD|

**v1 is a valid production release.** DTLS provides transport encryption; SHA-256 guarantees end-to-end integrity; SAS prevents identity mixups. This is “secure enough for production” for most teams. E2EE in v1.5 adds defence-in-depth against a compromised WebRTC implementation or future DTLS issues — worth building, but not a v1 blocker.

### Open questions

### Open questions

1. **Self-hosted DERP for Tailnet Mode?** If we officially support self-hosted DERP as part of Tailnet Mode, we need to document the configuration path and update the “no relay” disclosure.
1. **Companion agent scope for v2?** If scoped, define the IPC protocol between the agent and the browser extension or page.
1. **Multi-party transfer?** Current design is strictly 1-sender:1-receiver. Many-to-one (e.g., multiple camera operators sending to one editor) requires a different approval/SAS flow.
1. **iOS Safari transfer size limit?** Needs empirical testing. Suspected cap around 2–4 GB due to memory constraints.
1. **Ordered vs unordered data channel:** revisit after performance profiling with representative 100 GB+ video file transfers on lossy links.
1. **ECDH public key exchange message type:** should the ECDH key exchange be a new signaling message type (pre-data-channel) or a data-channel control frame (post-connection)? Current spec puts it on the data channel; confirm this is acceptable given that the data channel is not open until after ICE completes.

-----

## Appendix A — Canonical State Machines

### A.1 Sender states

```
DRAFT
  │  (user fills share form)
  ▼
SHARE_CREATED
  │  (share code issued; waiting for receiver)
  ▼
WAITING_REQUEST
  │  (receiver joins with share code)
  ▼
APPROVING          ◄──── (sender reviews receiver identity)
  │  (sender clicks Approve)
  ▼
WAITING_SAS        ◄──── (SDP/ICE exchange happening in background)
  │  (sender confirms SAS match)
  ▼
TRANSFERRING
  │  (all chunks ACKed, SHA-256 verified by receiver)
  ▼
COMPLETE
  
  (from any state except DRAFT)
  ▼
FAILED             (ICE failure, SAS mismatch, network drop with no resume, timeout)
  
  (from TRANSFERRING on reconnect)
  ┌─► RECONNECTING
  │     │  (ICE restart negotiated)
  │     ▼
  │   RESYNC          (exchange resume maps)
  │     │  (resume map agreed)
  └─────┴─► TRANSFERRING
```

**State transitions that must be persisted** (survive page refresh): `SHARE_CREATED`, `WAITING_REQUEST`, `TRANSFERRING` (with resume map), `COMPLETE`.

### A.2 Receiver states

```
IDLE
  │  (user opens share link)
  ▼
JOINING
  │  (authenticated; join request sent)
  ▼
WAITING_APPROVAL
  │  (sender approves)
  ▼
WAITING_SAS
  │  (receiver confirms SAS match)
  ▼
TRANSFERRING
  │  (transfer complete, SHA-256 verified)
  ▼
COMPLETE

  (from WAITING_APPROVAL if sender rejects)
  ▼
REJECTED

  (from any active state)
  ▼
FAILED

  (from TRANSFERRING on reconnect)
  ┌─► RECONNECTING
  │     │
  │     ▼
  │   RESYNC
  │     │
  └─────┴─► TRANSFERRING
```

### A.3 Reconnect sub-machine (shared)

```
TRANSFERRING ──(connection lost)──► RECONNECTING
                                        │
                                    (ICE restart)
                                        │
                                    RESYNC
                                        │  (exchange: sender sends resume_offer
                                        │   with last-acked-range; receiver replies
                                        │   resume_accept with its range map)
                                        │
                                    TRANSFERRING   (continues from gap)
                                        
                                    (if reconnect fails after 3 attempts + 30s total)
                                        │
                                    FAILED
```

-----

## Appendix B — Protocol Envelope & Chunk Format

### B.1 Signaling message envelope

All signaling messages (over SSE or long-poll) are JSON with this envelope:

```typescript
interface SignalingMessage {
  type: string;          // message type (see §4.3)
  version: 1;            // protocol version; reject messages with unknown version
  msg_id: string;        // UUIDv4; used for deduplication on reconnect
  timestamp: number;     // Unix milliseconds (sender wall clock; not trusted for ordering)
  share_id: string;      // identifies the share room
  payload: unknown;      // type-specific payload
}
```

Max envelope size (signaling): 8 KB. Server rejects with 413 if exceeded.

### B.2 Data channel frames

Binary frames over the WebRTC data channel. All integers are big-endian.

#### Common header (32 bytes, all frame types)

```
Offset  Size  Field
------  ----  -----
0       1     frame_type     (0x01=chunk_data, 0x02=chunk_ack, 0x03=ping, 0x04=pong,
                              0x05=manifest_json, 0x06=manifest_ack_json,
                              0x07=transfer_done_json, 0x08=transfer_verified_json,
                              0x09=resume_offer_json, 0x0A=resume_accept_json)
1       1     protocol_ver   (0x01)
2       2     flags          (bit0: e2ee_enabled; bit1: reserved; rest reserved)
4       4     msg_seq        (uint32; monotonic per session; for dedup/diagnostics)
8       8     file_id        (uint64; session-scoped counter starting at 1; 0 if N/A)
16      8     chunk_index    (uint64; 0-based; 0 if N/A)
24      8     byte_offset    (uint64; absolute byte offset in file; 0 if N/A)
```

#### Chunk frame extension (only for frame_type = 0x01)

```
Offset  Size  Field
------  ----  -----
32      4     payload_len    (uint32; byte count of payload field)
36      4     crc32c         (uint32; CRC-32C of plaintext chunk bytes; 0 if unused)
40      12    nonce          (12 bytes; required if e2ee_enabled, else all-zero)
52      16    auth_tag       (16 bytes; AES-GCM tag; required if e2ee_enabled, else all-zero)
68      N     payload        (N bytes; ciphertext if e2ee_enabled, else plaintext)
```

Total chunk frame header: 68 bytes. Maximum frame size: 1 MiB + 68 bytes.

**AAD requirement (E2EE only):** AES-GCM AAD MUST cover bytes 0–51 of the chunk frame:

- Common header (0–31)
- `payload_len` (32–35)
- `crc32c` (36–39)
- `nonce` (40–51)

This authenticates all placement and identity fields. Decryption MUST fail and the chunk MUST be discarded if the tag does not verify.

#### JSON-control frames (non-chunk frame types)

For all non-chunk frames, the chunk extension is replaced with a minimal JSON envelope:

```
Offset  Size  Field
------  ----  -----
32      4     payload_len    (uint32)
36      N     json_payload   (UTF-8 JSON)
```

Control frames carry no file bytes and remain plaintext in both v1 and v1.5.

### B.3 Chunk ACK JSON payload (frame_type 0x02)

The ACK frame’s common header carries `file_id`. The JSON payload MUST NOT repeat it.

```json
{
  "received": [[0, 31], [33, 100]],
  "missing":  [32]
}
```

Ranges are `[start_inclusive, end_inclusive]`. `missing` is a flat list of individual chunk indices. Max ACK payload: 4 KB. If the range map would exceed 4 KB, split across multiple ACK frames (each with the same `file_id` in the header).

### B.4 Window and flow control defaults

|Parameter                       |Default               |Range|
|--------------------------------|----------------------|-----|
|Initial window size             |32 chunks             |4–256|
|Max in-flight                   |32 chunks             |—    |
|Window increase on clean ACK run|+2 chunks/RTT         |—    |
|Window decrease on NACK         |halve                 |—    |
|Keepalive ping interval         |15 s                  |—    |
|Reconnect timeout               |30 s total, 3 attempts|—    |

-----

## Appendix C — Crypto Nonce Specification

> **Applies to v1.5+ only.** In v1, there is no app-layer encryption and therefore no nonce to specify. Do not implement this appendix until the E2EE milestone.

### C.1 Threat model

AES-GCM nonce reuse with the same key is catastrophic: it allows full plaintext recovery and breaks authentication. The nonce scheme must guarantee uniqueness across all chunks of all files in a session, even if the implementation delivers chunks out of order or retransmits them.

### C.2 file_id definition

`file_id` is a **uint64 session-scoped counter**, starting at 1, assigned by the sender to each file in the manifest. It is encoded as an unsigned 64-bit big-endian integer in frame headers and as a decimal number in JSON payloads.

Using a counter (not a random value) makes uniqueness provable by inspection and avoids any birthday-problem risk.

### C.3 Session salt

- Generated by the sender at session start using `crypto.getRandomValues`.
- Length: **128 bits (16 bytes)**.
- Transmitted to the receiver as part of E2EE session setup, over the established data channel (after SAS confirmation and ECDH key exchange).
- Fixed for the entire transfer session. A session restart (new share, or reconnect after `FAILED`) generates a new salt.

### C.4 Nonce derivation

```
nonce = HMAC-SHA256(session_salt, domain_sep || file_id_be64 || chunk_index_be64)
        truncated to the leftmost 96 bits (12 bytes)
```

Where:

- `session_salt`: 16-byte random value (§C.3).
- `domain_sep`: the ASCII string `"p2pt-chunk-nonce-v1"` (19 bytes), hardcoded constant.
- `file_id_be64`: `file_id` as an 8-byte big-endian unsigned integer.
- `chunk_index_be64`: `chunk_index` as an 8-byte big-endian unsigned integer.
- `||`: byte concatenation.
- Truncation: take bytes `[0:12]` of the 32-byte HMAC output.

### C.5 Uniqueness proof

For a fixed `session_salt`, the HMAC input is unique as long as `(file_id, chunk_index)` pairs are unique across the session. `file_id` is a monotonic counter — uniqueness is guaranteed by construction. `chunk_index` is 0-based within each file. Therefore no two chunks in a session share a nonce.

### C.6 Bounds

|Parameter            |Value                           |
|---------------------|--------------------------------|
|`file_id` type       |uint64 counter, starting at 1   |
|Max files per session|2^64 − 1 (effectively unlimited)|
|Max file size        |1 TiB                           |
|Max chunk size       |4 MiB                           |
|Min chunk size       |64 KiB                          |
|Max chunks per file  |16,777,216 (1 TiB / 64 KiB)     |
|Session salt length  |128 bits                        |
|Nonce length         |96 bits (12 bytes)              |

### C.7 Implementation checklist

- [ ] This appendix is only implemented as part of the v1.5 E2EE milestone; it is not shipped in v1.
- [ ] `session_salt` is generated with `crypto.getRandomValues`, not `Math.random`.
- [ ] `session_salt` is 16 bytes (128 bits).
- [ ] `file_id` is a uint64 counter starting at 1; it matches the value used in chunk frame headers and the manifest JSON.
- [ ] `file_id_be64` and `chunk_index_be64` are each encoded as **unsigned** big-endian 64-bit integers.
- [ ] HMAC output is truncated by taking bytes `[0:12]`; no other truncation method is used.
- [ ] `domain_sep` is the exact ASCII string `"p2pt-chunk-nonce-v1"`, hardcoded as a constant.
- [ ] A session restart (new share or reconnect after FAILED) regenerates `session_salt`.
- [ ] Unit test: `(file_id=1, chunk_index=0)` and `(file_id=1, chunk_index=1)` produce different nonces.
- [ ] Unit test: `(file_id=1, chunk_index=0)` with `salt_A` ≠ `(file_id=1, chunk_index=0)` with `salt_B`.
- [ ] Unit test: `(file_id=1, chunk_index=0)` and `(file_id=2, chunk_index=0)` produce different nonces.
- [ ] Integration test: receiver rejects a chunk with a tampered `chunk_index` in the header (AAD mismatch).

-----

## Appendix D — Browser Support Matrix

|Browser         |Version |Transfer|Resume (session)|Resume (refresh)|Folder pick|Notes                                                                                          |
|----------------|--------|--------|----------------|----------------|-----------|-----------------------------------------------------------------------------------------------|
|Chrome          |Latest 2|✅       |✅               |✅               |✅          |Production recommended                                                                         |
|Edge            |Latest 2|✅       |✅               |✅               |✅          |Production recommended                                                                         |
|Firefox         |Latest 2|✅       |✅               |⚠️ partial       |❌          |No FSAPI; resume after refresh = session only; no folder pick                                  |
|Safari          |Latest 1|⚠️       |⚠️               |❌               |❌          |Backgrounding kills transfer; no FSAPI; max tested file ~2 GB; data channel quirks on older iOS|
|Chrome Android  |Latest  |⚠️       |⚠️               |❌               |❌          |Mobile storage purge risk; no FSAPI                                                            |
|Safari iOS      |Latest  |⚠️       |⚠️               |❌               |❌          |Same as Safari desktop plus tighter memory limits                                              |
|Firefox Android |Latest  |⚠️       |⚠️               |❌               |❌          |Similar to Firefox desktop                                                                     |
|IE / Legacy Edge|Any     |❌       |❌               |❌               |❌          |Not supported                                                                                  |

**Key:**

- ✅ Supported and tested
- ⚠️ Best-effort; known limitations documented
- ❌ Not supported

Safari-specific limitations must be surfaced to the user before transfer begins (not discovered mid-transfer).
