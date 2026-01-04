# HuckVPN — AGENTS.md

This file instructs AI coding agents how to work in this repository.

## 1) Behavioral requirements for agents

### 1.1 Security-first, least privilege

- Never transmit or log private keys.
- Never write secret material to stdout/stderr.
- Treat all input as untrusted (UI text fields, config files, environment variables).
- Avoid introducing network-facing services unless explicitly specified.
- Prefer explicit user-driven actions over background automation.

### 1.2 No assumptions

- If a requirement is missing or ambiguous, do not invent behavior.
- Instead:
  1. implement a safe default that does not reduce security or privacy, and
  2. record the ambiguity in `docs/OPEN_QUESTIONS.md` with a proposed decision and tradeoffs.

### 1.3 Determinism and auditability

- All side effects (starting VPN, changing routes, editing configs) must be:
  - initiated by an explicit user action,
  - logged to an internal, user-viewable log panel (redacting secrets),
  - reversible (stop/disconnect should unwind changes).

### 1.4 Mac-first quality bar

- Primary output is a macOS `.app`.
- Code must follow macOS conventions (app bundle structure, Launch Services behavior).
- All platform-specific actions must be isolated in `internal/platform/macos`.

## 2) Languages, frameworks, and packaging

### 2.1 Implementation language

- Use **Go** for application logic and system integration.

### 2.2 UI framework

- Use **Wails (Go + WebView)** for the macOS desktop UI.
- UI is minimal and stable: start/stop, exit node selection, add/edit/delete nodes, status.

### 2.3 System integration

- WireGuard control must be performed via:
  - system WireGuard tools when available (e.g., `wg`, `wg-quick`) or
  - a dedicated implementation only if specified later.
- Privileged actions must be isolated behind a single internal interface:
  - `internal/platform/macos/privilege` (elevation strategy is a spec item; do not assume).

### 2.4 Repo layout (must follow)

- Monorepo is allowed, but **HuckVPN is a single standalone macOS app**.
- Required structure:

  /apps/huckvpn/
  cmd/huckvpn/ # main entrypoint
  internal/ # Go packages (no circular deps)
  ui/ # Wails frontend
  assets/ # icons, templates, etc.
  build/ # packaging scripts
  README.md

  /shared/ # optional shared libs (no secrets, no side effects)
  /docs/ # design docs, open questions, threat model

## 3) Workflow (step-by-step; do not implement everything at once)

Agents must follow this incremental workflow. Each step must compile and run before moving to the next.

### Step 0 — Repository bootstrap

- Create folder structure.
- Add Go module(s) for `/apps/huckvpn`.
- Add Wails scaffold for UI.
- Add `make dev` and `make build-macos` commands.

Deliverable: app launches and shows a placeholder window.

### Step 1 — Domain model + storage

- Implement exit node model:
  - ID, display name, bootstrap endpoint (host:port and scheme), isLocal flag.
- Implement local persistence:
  - a single config file in an OS-appropriate user directory.
- Implement CRUD operations:
  - add, edit, delete, list exit nodes.

Deliverable: UI can add/edit/delete exit nodes and persists them.

### Step 2 — WireGuard identity management

- Implement client identity:
  - generate one WireGuard keypair locally (private key never leaves device),
  - store securely (storage mechanism must be specified; if unclear, implement file storage with restrictive permissions and document the need for Keychain).
- Expose in UI:
  - "Initialize" or "Reset identity" actions (reset must warn about breaking server registrations).

Deliverable: identity exists and is stable across restarts.

### Step 3 — Connection orchestration (no server assumptions)

- Implement connection state machine:
  - DISCONNECTED → CONNECTING → CONNECTED → DISCONNECTING → DISCONNECTED
  - error transitions and retries are explicit.
- Implement `Start` and `Stop` functions that call a pluggable backend.
- Backend initially can be a stub (no real networking), but must preserve state and logs.

Deliverable: UI start/stop works with correct state transitions and logging.

### Step 4 — Real WireGuard bring-up (macOS)

- Implement actual WireGuard activation path for macOS.
- Ensure stop cleanly reverts changes.
- Add status checks (tunnel up, interface present, current endpoint).

Deliverable: real tunnel can be brought up given a complete config.

### Step 5 — Exit node first-connect bootstrap + registration

- Implement first-time connect behavior:
  - client contacts exit node bootstrap endpoint,
  - submits client public key and required auth material,
  - receives server-side parameters required for WireGuard config.
- Do not assume transport security; implement only as specified, otherwise mark as open question.

Deliverable: client can register and then connect using returned parameters.

### Step 6 — Tor integration

- Integrate Tor as specified (always-on or optional is a spec item).
- Implement leak protection and routing behavior only per specification.

Deliverable: Tor behavior is correct and testable.

### Step 7 — Packaging and distribution

- Produce a signed/notarized `.app` only if specified; otherwise produce an unsigned `.app` that runs on the developer machine.
- Provide reproducible builds: `make build-macos` outputs `dist/HuckVPN.app`.

Deliverable: `.app` artifact produced from clean repo checkout.

## 4) Coding standards

- Go:
  - `golangci-lint` configured; no lint suppression without justification.
  - Use context-aware functions for operations that can block.
  - Log redaction: never log keys, tokens, full configs.
- Frontend:
  - Keep UI minimal; avoid large frameworks unless Wails template uses one.
- Tests:
  - Unit tests for parsing, persistence, state machine.
  - Integration test harness stubs for platform calls.

## 5) Documentation obligations

Agents must update:

- `SPECIFICATIONS.md` if scope changes.
- `docs/OPEN_QUESTIONS.md` for unresolved decisions.
- `docs/THREAT_MODEL.md` whenever adding a new trust boundary (e.g., bootstrap API, privilege escalation, Tor routing).
