# HuckVPN – TLS Model Decision (Final)

## Decision Summary

**HuckVPN will use self-signed TLS with certificate pinning (Option B).**

This decision is final and applies to the **client ↔ server registration channel** (HTTPS).  
WireGuard traffic security is separate and unaffected by this choice.

---

## 1. Problem Being Solved

When the HuckVPN client connects to an exit node for the **first time**, it must:

- Send its **WireGuard public key**
- Receive **server configuration parameters**
- Be absolutely sure it is talking to **the intended server**, not an attacker

This must hold even if:

- DNS is hijacked
- A Wi-Fi network is malicious
- A certificate authority is compromised

TLS alone is not sufficient unless **server identity verification** is clearly defined.

---

## 2. Chosen Solution: Self-Signed TLS + Certificate Pinning

### Definition

- Each HuckVPN server generates its **own self-signed TLS certificate**
- The HuckVPN client **pins** the server’s certificate fingerprint (SHA-256)
- The client **trusts only that exact certificate**
- No public Certificate Authorities (CAs) are used or trusted

This is conceptually similar to how SSH host key verification works.

---

## 3. Server-Side Behavior (HuckVPN Server)

### 3.1 TLS Certificate

On first setup, the server must:

- Generate a TLS private key
- Generate a self-signed TLS certificate
- Persist both securely on disk
- Use this certificate for its HTTPS registration endpoint

### 3.2 Server Secrets

The server has **three sensitive secrets**:

1. WireGuard private key
2. TLS private key
3. Bootstrap token

None of these are ever transmitted automatically to clients.

### 3.3 Registration Endpoint

- Listens on HTTPS only
- Presents the self-signed certificate
- Rejects plaintext HTTP
- Requires a valid bootstrap token for registration

---

## 4. Client-Side Behavior (HuckVPN Client)

### 4.1 Exit Node Definition (TLS-relevant fields)

Each exit node record **must include**:

- HTTPS endpoint (IP or hostname + port)
- TLS certificate fingerprint (SHA-256)
- Bootstrap token

### 4.2 Connection Process

When the client connects to an exit node:

1. Open HTTPS connection
2. **Disable default OS CA trust**
3. Extract server’s TLS certificate
4. Compute SHA-256 fingerprint
5. Compare against pinned fingerprint
6. If mismatch → **hard fail**
7. If match → proceed to registration

No registration data is sent until step 6 succeeds.

### 4.3 Failure Handling

If certificate verification fails:

- Connection must abort immediately
- User must be shown a clear error
- No retries unless user explicitly changes the exit node configuration

---

## 5. Security Properties

This model guarantees:

- Immunity to DNS hijacking
- Immunity to rogue or compromised CAs
- Strong protection against MITM attacks
- Explicit, user-controlled trust decisions
- Compatibility with raw IP addresses (no domain required)

An attacker would need:

- The server’s TLS private key **or**
- The pinned fingerprint

Both are under user control.

---

## 6. Operational Tradeoffs (Accepted)

### 6.1 Manual Trust Establishment

- User must copy the TLS fingerprint once when adding an exit node
- This is intentional and explicit

### 6.2 Certificate Rotation

- If the server regenerates its TLS certificate:
  - The client’s pinned fingerprint must be updated
- This is acceptable for self-hosted infrastructure

---

## 7. Comparison to Public CA TLS (Rejected)

| Aspect                  | Public CA TLS | Pinned TLS (Chosen)  |
| ----------------------- | ------------- | -------------------- |
| Domain required         | Yes           | No                   |
| CA dependency           | Yes           | No                   |
| Works with IP only      | No            | Yes                  |
| MITM resistance         | Strong        | Stronger             |
| Manual setup            | Minimal       | One-time fingerprint |
| Fit for self-hosted VPN | Moderate      | Excellent            |

Public CA TLS was rejected to avoid:

- External trust dependencies
- Domain and renewal requirements
- CA-based attack surface

---

## 8. Implementation Requirements (Mandatory)

### Client

- Must implement explicit certificate pinning
- Must not trust system CAs for registration
- Must never log certificates or fingerprints
- Must fail closed on verification errors

### Server

- Must generate and persist self-signed TLS cert
- Must expose fingerprint to user (e.g. via CLI output)
- Must never rotate cert silently

---

## 9. Status

- TLS model: **FINAL**
- No further architectural decisions required in this area
- Safe to proceed with:
  - Registration API specification
  - Server daemon implementation
  - Client connection logic

---

## 10. Next Recommended Artifact

**Client ↔ Server Registration Protocol Specification**, including:

- Exact HTTPS endpoints
- Request/response JSON schemas
- Error codes
- Peer lifecycle rules
- Live WireGuard updates

This document is now authoritative for TLS behavior in HuckVPN.
