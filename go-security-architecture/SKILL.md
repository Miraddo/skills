---
name: go-security-architecture
description: Design security into Go (golang) systems from the start — threat modeling (STRIDE), trust boundaries, least privilege, defense in depth, zero trust, secure defaults / fail-closed, authN/authZ architecture (OAuth2/OIDC, mTLS, short-lived tokens), and secrets architecture (KMS/Vault). Use when designing or reviewing the security architecture of a service or system, before/while writing code.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Go Security Architecture (security-first design)

Design security **into the architecture before code exists**, so fixes are
cheap and layers reinforce each other. This is the *design* counterpart to
[[go-security]] (which covers secure coding, validation, and scanning). Apply
it when shaping a [[go-monolith]], [[go-modular-monolith]], or
[[go-microservices]] system. Sourced from secure-by-design and zero-trust
practice ([CISA/OWASP](https://owasp.org/www-project-proactive-controls/),
[Red Hat](https://www.redhat.com/en/blog/security-design-security-principles-and-threat-modeling)).

## Start with a threat model (STRIDE)
Threat-model **early** — at design time, when changes are a diagram edit, not
a refactor. For each component and data flow ask STRIDE:

| Threat | Question | Go-level control |
|--------|----------|------------------|
| **S**poofing | Can identity be faked? | Strong authN: OIDC for users, mTLS for services. |
| **T**ampering | Can data be altered in flight/at rest? | TLS, signed tokens, integrity checks, DB constraints. |
| **R**epudiation | Can an actor deny an action? | Audit logs with identity + trace IDs ([[go-observability]]). |
| **I**nfo disclosure | Can secrets/PII leak? | Encryption, least-privilege data access, no secrets in logs. |
| **D**enial of service | Can it be overwhelmed? | Timeouts, quotas, bulkheads (design-level; see [[go-http-api]]). |
| **E**levation of privilege | Can a low-priv actor gain more? | AuthZ on every action, least privilege, no confused deputy. |

Output: a short doc listing trust boundaries, assets, the STRIDE threats per
boundary, and the control that mitigates each. Keep it in the repo; revisit it
when the design changes.

## Core design principles
- **Least privilege** — every user, service, token, and DB role gets *only*
  the permissions it needs. Separate read/write DB users; scope tokens to one
  audience; one service account per service.
- **Defense in depth** — layer controls so one failure isn't total
  compromise: edge gateway → service authN/authZ → in-service input
  validation → data-layer constraints. Never rely on a single perimeter.
- **Zero trust** — the network is hostile; **authenticate and authorize every
  request**, including service-to-service and "internal" traffic. No implicit
  trust from being inside the VPC.
- **Secure defaults / fail closed** — the default state denies. On error, an
  auth check returns "deny," not "allow." A missing config disables a feature
  rather than opening it. Make the safe path the easy path.
- **Complete mediation** — check authorization on *every* access, not once at
  session start. Don't cache an allow decision past its validity.
- **Minimize attack surface** — fewer endpoints, fewer exposed ports, fewer
  dependencies, smaller container images (distroless/scratch, non-root).
- **Separate trust zones** — keep the internet-facing edge thin; push
  sensitive logic and data behind internal boundaries ([[go-modular-monolith]]
  facades, [[go-microservices]] data ownership).

## Map and defend trust boundaries
A **trust boundary** is any line where data crosses from less-trusted to
more-trusted (internet→edge, edge→service, service→service, service→DB,
app→third-party). At **every** boundary:
1. **Authenticate** the caller (who is this?).
2. **Authorize** the specific action (are they allowed *this*?).
3. **Validate** the input ([[go-security]], [[go-http-api]]) — never trust data
   just because it came from an "internal" caller.

> The backend validates everything. Client-side checks are UX, not security —
> a request from your own frontend is still untrusted input.

## Authentication & authorization architecture
- **Users**: OAuth 2.1 / OIDC with an identity provider; never roll your own
  password/session crypto. Short-lived access tokens + refresh rotation.
- **Service identity**: **mTLS** (often via a service mesh — Istio/Linkerd/
  Cilium) or signed, short-lived service tokens (SPIFFE/JWT). Each service
  proves who it is; peers verify.
- **Tokens**: short TTL, rotated, scoped to a single `aud`; verify signature,
  `exp`, `iss`, `aud`; reject `alg:none` ([[go-security]]).
- **AuthZ model**: pick one and enforce it centrally — RBAC for coarse roles,
  ABAC/policy engine (OPA) for fine-grained rules. Decide **where** decisions
  live: a gateway centralizes coarse policy (auth, quotas, schema checks);
  **in-service checks** enforce object-level authorization the gateway can't
  see (the user owns *this* record). Do both — gateway-only authZ misses
  broken-object-level-authorization, the top OWASP API risk.
- **Context-carry identity**: thread the authenticated principal through
  `context.Context` (a private key type), so services and repositories make
  authz decisions with the real caller, not an ambient assumption.

## Secrets architecture
- Secrets come from a **secret manager / KMS** (Vault, AWS/GCP/Azure), injected
  at runtime as env or mounted files — never baked into images or committed.
- **Rotate automatically**; use short-lived dynamic credentials where the
  manager supports it (e.g. Vault DB creds).
- Scope each secret to one consumer; a leaked credential should blast-radius to
  one service, not the fleet.
- Encrypt **in transit** (TLS 1.2+/1.3 everywhere, including internal hops) and
  **at rest** (DB/disk/object-store encryption); classify data (public /
  internal / PII / secret) and apply controls per class.

## Defense-in-depth layers (a Go service)
```
Internet
  │  TLS, WAF, rate limit, schema check
  ▼
API gateway ──── coarse authN + authZ, quotas, request shaping
  │  mTLS
  ▼
Service ──────── verify token, object-level authZ, input validation
  │              (handlers thin; logic in services — see go-monolith)
  ▼
Data layer ───── least-priv DB user, parameterized queries, row constraints,
                 encryption at rest
```
Each layer assumes the one in front of it may have failed. Compromising one
should not hand over the system.

## Designing for the failure case
- **Fail closed**: errors in authN/authZ deny access; a downstream outage
  doesn't bypass checks.
- **Blast-radius containment**: per-service credentials, network segmentation,
  separate data stores ([[go-microservices]]) so one breach is contained.
- **Auditability**: log security-relevant events (authn success/failure, authz
  denials, privilege use) with principal + trace ID — but never secrets/PII
  ([[go-observability]]). This covers the **R**epudiation leg of STRIDE.
- **Recoverability**: design key/credential rotation and revocation paths *now*,
  not after an incident.

## How this maps to the architecture skills
- **[[go-monolith]]** — one trust boundary at the edge; still enforce authZ per
  action and least-priv DB users. Don't conflate "one process" with "one trust
  level."
- **[[go-modular-monolith]]** — module facades are internal trust boundaries;
  validate at them and pass identity via context, so extraction to a service
  later keeps the checks.
- **[[go-microservices]]** — every hop is a boundary: mTLS service identity,
  per-service secrets/data, gateway + in-service authZ, zero implicit trust.

## Review checklist (design-level)
- [ ] Threat model exists (STRIDE per trust boundary) and is current.
- [ ] Every trust boundary authenticates + authorizes + validates.
- [ ] AuthZ enforced per action *and* per object; not gateway-only.
- [ ] Service identity via mTLS / signed short-lived tokens; zero implicit
      network trust.
- [ ] Secrets from a manager/KMS, scoped, rotated; none in images/repos/logs.
- [ ] TLS in transit (incl. internal) + encryption at rest; data classified.
- [ ] Least privilege for tokens, DB roles, service accounts, containers.
- [ ] Defaults deny; auth failures fail closed.
- [ ] Security events audited with identity + trace ID, no secrets/PII.
- [ ] Attack surface minimized (endpoints, ports, deps, image).

## Anti-patterns
- "Internal traffic is trusted" — the flat-network assumption zero trust kills.
- AuthZ only at the gateway → broken object-level authorization.
- Perimeter-only security (hard shell, soft center).
- Secrets in env files committed to the repo, or in container images.
- Fail-open auth (error path grants access).
- Designing security as a later "hardening pass" instead of up front.
- One God service account / one DB superuser shared by everything.

## References
- [OWASP Proactive Controls](https://owasp.org/www-project-proactive-controls/)
- [Security by design: principles and threat modeling — Red Hat](https://www.redhat.com/en/blog/security-design-security-principles-and-threat-modeling)
- [OWASP Threat Modeling (STRIDE)](https://owasp.org/www-community/Threat_Modeling)
- [Securing Microservices in Go: AuthN/AuthZ & Transport Security](https://moldstud.com/articles/p-securing-microservices-in-go-authentication-authorization-and-transport-security)
