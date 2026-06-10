---
name: go-security
description: Write secure Go (golang) code — govulncheck/gosec scanning, input validation, parameterized SQL (no injection), secrets via env/secret-managers, crypto/rand and strong algorithms, TLS and security headers, safe error handling, and OWASP API risks. Use when reviewing or writing security-sensitive Go code or hardening a service.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Go Security

Practical secure-coding for Go services, aligned with the official
[Security Best Practices](https://go.dev/doc/security/best-practices) and
OWASP. Pairs with [[go-http-api]], [[go-database]], and [[go-tooling-ci]]
(run the scanners in CI).

## Scan automatically (make CI fail on findings)
- **`govulncheck ./...`** — official vulnerability scanner. It uses **call-graph
  analysis**: it reports only vulns you actually *call*, not merely import, so
  signal is high. Run in CI and locally.
- **`gosec ./...`** — static security analyzer (hardcoded creds, weak crypto,
  unsafe patterns).
- **`golangci-lint`** — aggregate linter; include security-relevant linters.
- In 2025, CI must **reject builds with known vulnerabilities** — gate
  deploys on `govulncheck`.

```bash
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck ./...
```

## Validate & sanitize all input
User input (HTTP bodies/params, headers, files, env) is the primary attack
vector — SQL injection, XSS, path traversal, RCE.
- Validate **type, length, range, format** before use; reject unknown JSON
  fields (`DisallowUnknownFields`) — see [[go-http-api]].
- Bound body size (`http.MaxBytesReader`).
- For file paths, clean and confine to a base dir; reject `..` traversal.
- Escape output for the sink (HTML via `html/template`, which auto-escapes —
  never build HTML with `text/template` or string concat).

## SQL injection — parameterize, always
Use parameterized queries (`$1`/`?`); never interpolate input into SQL text.
See [[go-database]]. The same principle applies to any command/DSL built from
input (avoid `os/exec` with shell strings; pass args as a slice).

## Secrets management
- Load secrets from **environment** (`os.Getenv`) or a **secret manager**
  (Vault, AWS Secrets Manager, GCP/Azure equivalents). Never hardcode.
- Keep secrets out of git: `.gitignore`, secret-scanning, pre-commit hooks.
  If a secret was committed, rotate it — removal from history isn't enough.
- Don't log secrets/tokens/PII ([[go-observability]] redaction).

## Cryptography
- **Randomness for security → `crypto/rand`**, never `math/rand` (predictable).
- Use vetted algorithms: AES-256-GCM, RSA-2048+/ECDSA, SHA-256+. Don't roll
  your own crypto.
- Passwords: a slow KDF — `bcrypt`/`argon2`/`scrypt` — never plain SHA.
- Use constant-time comparison (`crypto/subtle.ConstantTimeCompare`) for
  secrets/MACs.
- **Never ignore errors from crypto or I/O** — a dropped error can mean
  unencrypted or unauthenticated data.

## Transport & headers
- Enforce **HTTPS**; redirect HTTP→HTTPS. Configure `tls.Config` with a
  modern `MinVersion` (TLS 1.2+, prefer 1.3); disable TLS 1.0/1.1.
- Set security headers: `Strict-Transport-Security`, `Content-Security-Policy`,
  `X-Content-Type-Options: nosniff`, `X-Frame-Options`.
- Set server timeouts (slowloris defense) — [[go-http-api]].

## AuthN / AuthZ
- Validate tokens (JWT: check signature, `exp`, `aud`, `iss`; reject `alg:none`).
- Enforce authorization on **every** protected handler — don't rely on the UI
  hiding actions (OWASP broken-object/-function-level authorization).
- Rate-limit auth endpoints; lock out / backoff on brute force.

## Safe error handling
- Return generic messages to clients; **log details server-side**. Don't leak
  stack traces, SQL, internal paths, or driver errors to users.
- Recover from panics in a middleware so a panic can't crash the process or
  spill internals ([[go-http-api]] Recover middleware).

## Dependency & build hygiene
- `go mod tidy` + commit `go.sum`; `go mod verify`. Pin/review new deps.
- Keep the Go toolchain current (security fixes land in releases).
- Minimal container images (distroless/scratch); run as non-root.

## Review checklist
- [ ] `govulncheck` + `gosec` run in CI and gate deploys.
- [ ] All input validated; JSON rejects unknown fields; body size bounded.
- [ ] All SQL parameterized; no shell-string `exec`.
- [ ] Secrets from env/manager, never committed; rotated if leaked.
- [ ] `crypto/rand` for tokens; bcrypt/argon2 for passwords; errors checked.
- [ ] TLS 1.2+, security headers, server timeouts set.
- [ ] AuthZ enforced per endpoint; client errors generic, details logged.

## References
- [Security Best Practices for Go Developers (official)](https://go.dev/doc/security/best-practices)
- [govulncheck](https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck)
- [How to Secure Go APIs Against OWASP Top 10](https://oneuptime.com/blog/post/2026-01-07-go-secure-apis-owasp-top-10/view)
- [Secure Go Error Handling — JetBrains](https://blog.jetbrains.com/go/2026/03/02/secure-go-error-handling-best-practices/)
