# HuckVPN — SPECIFICATIONS.md

This document defines the build plan and functional specifications for HuckVPN, a standalone macOS VPN client delivered as a downloadable `.app`.

## 1) Product definition

### 1.1 Primary objective

Deliver a macOS desktop application (`HuckVPN.app`) that allows a user to:

- start a VPN connection,
- stop a VPN connection,
- select an exit node,
- add/edit/remove exit nodes,
- view connection status.

### 1.2 Non-goals (until explicitly added)

- No built-in cloud controller UI (e.g., `gcloud` wrapper) unless specified.
- No assumption that exit nodes exist at ship time.
- No automatic server provisioning.
- No hidden default exit nodes.

## 2) Supported platforms

- macOS is first-class.
- Any support for Windows/Linux is out of scope unless explicitly specified later.

## 3) Architecture overview

### 3.1 Components

- Single application: `HuckVPN.app`
- Internal modules:
  - UI (Wails)
  - Exit node registry (local persistence)
  - Identity manager (WireGuard keypair)
  - Connection orchestrator (state machine)
  - Platform adapter (macOS networking + privilege)
  - Optional: bootstrap/registration client (first-connect)

### 3.2 Trust boundaries

- Local machine (user space UI)
- Privileged operations (network stack changes)
- Remote exit node (bootstrap endpoint + WireGuard server)

Any addition across trust boundaries must be documented in `docs/THREAT_MODEL.md`.

## 4) Exit nodes

### 4.1 Exit node record (required fields)

Each exit node stored locally must include:

- `id`: stable identifier (string)
- `displayName`: shown in UI
- `bootstrapEndpoint`: scheme + host + port (e.g., `https://example.com:8443`)
- `isLocal`: boolean
- optional metadata fields may be added only if specified.

### 4.2 Shipping state

- Application ships with **no remote exit nodes**.
- Application includes a single built-in option:
  - "Local Machine" (`isLocal=true`)
  - bootstrap endpoint points to localhost (exact port is an open item if not specified elsewhere).

### 4.3 UI behavior

- Exit node dropdown includes:
  - "Local Machine"
  - user-added exit nodes
  - an "Add Exit Node…" affordance
- Selecting an exit node changes the target for future connection attempts.

## 5) Identity and keys (WireGuard)

### 5.1 Client identity

- The client must generate exactly one WireGuard keypair during initialization.
- The private key must never be transmitted off-device.
- Public key may be sent to exit nodes during registration.

### 5.2 Storage

- Storage mechanism (Keychain vs file) must be explicitly decided.
- If undecided, implement file storage with restrictive permissions and record the need for Keychain as an open question.

### 5.3 Reset behavior

- User can reset identity (if provided).
- Resetting identity invalidates prior server registrations and must warn the user.

## 6) Connection lifecycle

### 6.1 States

- DISCONNECTED
- CONNECTING
- CONNECTED
- DISCONNECTING
- ERROR (with reason and recovery guidance)

### 6.2 Start/Stop semantics

- Start:
  - validates selected exit node,
  - ensures identity exists,
  - performs registration if needed (if registration is part of the current spec),
  - brings WireGuard up,
  - applies required routing/DNS policies,
  - updates state and status view.
- Stop:
  - brings WireGuard down,
  - restores routing/DNS to pre-connection state as much as feasible,
  - updates state and status view.

## 7) First-time connect (bootstrap + registration)

### 7.1 Problem

If exit nodes are user-added at runtime, the client must know where to send its public key on first connect. This is solved by the `bootstrapEndpoint` stored for that exit node.

### 7.2 Registration data flow (conceptual)

- Client → bootstrap endpoint:
  - client public key
  - authentication material required to prevent anonymous registrations (token or other mechanism)
- Server → client response:
  - server public key
  - WireGuard endpoint (host:port)
  - allowed IPs
  - DNS settings
  - any additional required parameters

### 7.3 Security requirements

- Private keys are never transmitted.
- Anonymous registration must be prevented.
- Transport security requirements (TLS vs pinned cert vs SSH) must be explicitly specified; otherwise recorded as open question.

## 8) Tor integration

Tor integration was part of the earlier direction (VPN-over-Tor). For HuckVPN:

- Tor behavior must be explicitly specified:
  - always on vs optional,
  - client-side only vs additional server routing,
  - leak protection rules.

If not specified, Tor integration remains out of scope and must be recorded in `docs/OPEN_QUESTIONS.md`.

## 9) macOS privilege and networking

### 9.1 Privilege boundary

- Any operation requiring elevated privileges must be isolated in platform-specific code.
- Elevation strategy must be specified (helper tool vs per-action prompt).
- The app must not run the entire UI as root.

### 9.2 Cleanup requirements

- On stop or crash recovery, the app must attempt to restore networking state.

## 10) Logging and observability

### 10.1 User-facing logs

- App must provide a log view suitable for debugging.
- Logs must redact secrets:
  - private keys,
  - registration tokens,
  - full configs.

### 10.2 Diagnostics

- Status view should include:
  - selected exit node,
  - connection state,
  - tunnel/interface status,
  - last error (if any).

## 11) Build and packaging

### 11.1 Output

- `dist/HuckVPN.app` created via `make build-macos`.

### 11.2 Signing/notarization

- Whether the app must be signed/notarized must be explicitly specified.
- If not specified, the deliverable is an unsigned `.app` intended for local use/development.

## 12) Implementation plan (from scratch to fully developed)

### Phase 0 — Skeleton app

- Scaffold Wails + Go app.
- App launches, placeholder UI.

### Phase 1 — Exit node registry

- Local persistence
- CRUD UI
- Local Machine built-in entry

### Phase 2 — Identity

- WireGuard keypair generation
- Secure storage (per decided method)
- Public key display (optional; must be specified)

### Phase 3 — Connection orchestrator

- Implement state machine
- Start/stop flows, logging, error handling
- Backend stub for platform calls

### Phase 4 — macOS WireGuard integration

- Implement actual bring-up/tear-down on macOS
- Status checks

### Phase 5 — Registration (if required for MVP)

- Implement bootstrap call
- Prevent anonymous registrations
- Persist server parameters returned by registration

### Phase 6 — Tor integration (only if specified)

- Implement Tor routing and leak protection

### Phase 7 — Hardening + packaging

- Crash recovery and cleanup
- Build reproducibility
- `.app` packaging and (optional) signing/notarization

## 13) Open questions (must be answered to finalize MVP scope)

1. Does HuckVPN require Tor integration in MVP, or is it deferred?
2. What is the registration/auth model for first-time connect (bootstrap token, SSH, something else)?
3. Is TLS required for bootstrap endpoint? If yes, CA validation vs pinned cert?
4. What storage is required for the client private key on macOS (Keychain vs file)?
5. Must the `.app` be signed and notarized for distribution, or is developer/local distribution sufficient initially?
6. Should "Local Machine" be functional in MVP (i.e., assume a local server exists), or only a placeholder entry?
