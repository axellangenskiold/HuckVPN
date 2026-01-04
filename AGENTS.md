# HuckVPN — AGENTS.md

This repository contains two separate applications:

1. HuckVPN Client (macOS `.app`)
2. HuckVPN Server (headless background service)

Agents must follow this document when writing code.

## 1. Agent behavior

### 1.1 Security and privacy are non-negotiable

- Never transmit, print, or log private keys.
- Never log bootstrap tokens.
- Treat all user inputs (UI fields, config files, env vars) as untrusted.
- Minimize privileged operations; isolate them; make them explicit.
- Prefer deterministic, auditable behavior over clever automation.

### 1.2 No assumptions

- If a requirement is unclear, do not guess.
- Implement the safest minimal behavior that does not reduce privacy/security.
- Record the ambiguity in `docs/OPEN_QUESTIONS.md` and keep code structured so the decision can be swapped later.

### 1.3 Incremental delivery only (no “big bang” implementation)

Agents must implement in the step order below. Each step must:

- build successfully,
- run successfully,
- include basic tests where applicable,
  before moving to the next step.

## 2. Languages, frameworks, and tooling

### 2.1 Client

- Language: Go
- UI: Wails (Go backend + WebView frontend)
- macOS secret storage: Keychain (required)
- Tor: required for MVP
- WireGuard: controlled via macOS-compatible tooling (exact mechanism implemented behind interfaces)

### 2.2 Server

- Server is headless and must run on a server OS (Linux preferred) and optionally on a local machine.
- Implementation may be:
  - a Go binary plus a minimal install script, OR
  - a shell-first script that installs/runs a Go binary.
- Server must expose an HTTPS endpoint for registration and run WireGuard.

### 2.3 Repository layout (must follow)

/apps
/apps/client
/apps/client/cmd
/apps/client/cmd/huckvpn-client
/apps/client/internal
/apps/client/ui
/apps/client/assets
/apps/client/build

/apps/server
/apps/server/cmd
/apps/server/cmd/huckvpn-server
/apps/server/internal
/apps/server/scripts

/shared
/docs
/AGENTS.md
/SPECIFICATIONS.md

### 2.4 Standards

- Go:
  - Use `context.Context` for all operations that may block (network calls, process exec).
  - Use structured errors; do not hide failures.
  - Keep platform-specific code under `internal/platform/<os>`.
- UI:
  - Keep minimal: Start, Stop, Exit Node selector, Add Exit Node, Status, Logs.
- Logging:
  - Must be user-visible (in-app log panel) and redacted.
  - Never log secrets (private keys, tokens, full configs).

## 3. Step-by-step workflow

### Step 0 — Scaffolding

- Create repo structure.
- Scaffold Wails client app; client launches with placeholder UI.
- Scaffold server Go module; server builds and prints version/help.
  Deliverable: both apps build; client window opens.

### Step 1 — Shared domain model

- Define shared types (non-secret):
  - ExitNode record
  - ConnectionState enum
- Place in `/shared/model`.
  Deliverable: both client and server can import shared model without cycles.

### Step 2 — Client: exit node registry (local persistence)

- Implement local registry:
  - list/add/edit/delete exit nodes
  - persist to a config file in a user directory
- Include built-in entry: “Local Machine” (non-removable).
  Deliverable: UI can manage exit nodes and persists across restarts.

### Step 3 — Client: WireGuard identity with macOS Keychain

- Implement identity lifecycle:
  - generate one WireGuard keypair
  - store private key in macOS Keychain (required)
  - store public key in non-secret local storage for display/requests
- Provide UI actions:
  - Initialize identity
  - Reset identity (with warnings)
    Deliverable: identity persists; private key never appears in logs.

### Step 4 — Client: Tor lifecycle (required)

- Implement Tor management:
  - start Tor when connecting
  - confirm readiness (bootstrap completed)
  - stop Tor on disconnect (as specified in specs)
- No “fallback to clearnet.”
  Deliverable: client can start/stop Tor deterministically and detect readiness.

### Step 5 — Client: connection orchestrator (state machine)

- Implement connection state machine with explicit transitions and errors:
  - DISCONNECTED → CONNECTING → CONNECTED → DISCONNECTING → DISCONNECTED
- Implement Start/Stop wiring to state machine.
  Deliverable: UI behaves correctly even under failure.

### Step 6 — Client ↔ Server: registration over HTTPS + TLS + bootstrap token

- Implement:
  - `RegisterPeer(exitNode, clientPublicKey, bootstrapToken)` over HTTPS
  - Server validates token and registers peer
  - Server returns parameters required for WireGuard client config
- TLS must be enforced (no plaintext).
  Deliverable: first-time connect performs registration and stores server response.

### Step 7 — Client: WireGuard bring-up and leak protection

- Use the server-returned parameters to create a runtime WireGuard configuration.
- Bring up tunnel and apply:
  - DNS settings (as specified)
  - routing rules
  - leak prevention rules
- Ensure Stop cleans up.
  Deliverable: connect/disconnect functions reliably.

### Step 8 — Server: headless daemon + installer script

- Server responsibilities:
  - run WireGuard interface
  - expose HTTPS registration endpoint
  - validate bootstrap token
  - add/remove peers (live update)
  - firewall baseline (as specified)
- Provide scripts:
  - install dependencies
  - install service
  - start/stop/status
    Deliverable: server can run on a VPS and accept registrations securely.

### Step 9 — Packaging

- Build client `.app` to `dist/HuckVPN.app` (dev/local distribution).
- Document how signing/notarization can be added later (no implementation required for MVP unless specified).
  Deliverable: downloadable `.app` artifact.

## 4. Documentation obligations

Agents must maintain:

- `SPECIFICATIONS.md` (truth source of requirements)
- `docs/OPEN_QUESTIONS.md` (unresolved decisions)
- `docs/THREAT_MODEL.md` (trust boundaries, attack surfaces, mitigations)
