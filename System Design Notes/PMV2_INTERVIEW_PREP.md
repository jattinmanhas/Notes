# PMV2 — Interview Prep (Backend-Heavy)

**Project:** Self-hosted, zero-knowledge family password manager
**Stack:** Go 1.24 (stdlib `net/http`, no framework) · PostgreSQL (raw `database/sql` + `lib/pq`) · React/TS SPA · Chrome MV3 extension
**Crypto:** Argon2id (KDF) · XChaCha20-Poly1305 (AEAD) · X25519 (ECDH for sharing)

> How to use this doc: every question has a **short answer** you say out loud, and where useful a
> **deeper dive** and a **trap** note. Questions marked 🔥 are the ones interviewers actually use to
> separate "built a CRUD app" from "understands the system". Questions marked ⚠️ are ones where
> **your codebase has a real weakness** — the winning move is to name it before they do.

---

## 0. The 60-Second Pitch (memorize this)

> "PMV2 is a self-hosted password manager built on a zero-knowledge model: the server never sees a
> master password, a vault key, or any plaintext secret. The Go backend is a layered
> `controller → service → repository → domain` API on stdlib `net/http` with Postgres. Its job is
> deliberately boring — it stores opaque encrypted blobs, enforces ownership and session auth, and
> orchestrates key delivery for sharing. All the interesting cryptography happens in the browser: a
> master password goes through Argon2id into a KEK, every vault item gets its own random DEK
> encrypted with XChaCha20-Poly1305, and the DEK is wrapped by the KEK. Sharing wraps that same DEK
> to a family member's X25519 public key, so the server relays ciphertext it can't read. Auth uses a
> client-derived verifier instead of the password itself, plus opaque server-side sessions, TOTP 2FA,
> and email-code recovery."

**Then immediately offer the honest caveat** (this scores enormously well):

> "It's a strong prototype, not production-hardened. I did a full security audit of it — the three
> things I'd fix before real users are: item metadata is still stored server-side in plaintext which
> partially breaks the zero-knowledge claim; the sharing code feeds the raw X25519 shared secret
> into the AEAD instead of running it through HKDF; and the browser extension's autofill has no
> origin binding, which is the one property a password manager exists to provide."

---

## 1. Architecture & Go Design Decisions

### Q1.1 🔥 Walk me through your backend architecture.

**Answer.** Four layers, dependencies point inward only:

```
HTTP  →  controller/   parse DTO, call service, map domain errors → HTTP status
         service/      business rules, authorization, orchestration, audit
         repository/   SQL against Postgres, implements domain interfaces
         domain/       models + repository interfaces + sentinel errors. Zero external deps.
```

Plus `dto/` (JSON shapes — deliberately *not* on domain structs), `router/` (wiring +
middleware composition), `middlewares/`, `util/` (crypto helpers, HTTP helpers, IP extraction),
`config/`, `database/`, `mailer/`, `logger/`.

The key idea: **`domain` declares the interfaces, `repository` implements them, `service` depends on
the interface.** That's the Go idiom of "define interfaces at the point of consumption," and it's
what makes the service layer unit-testable with hand-rolled mocks (which is exactly how
`auth_service_test.go` works — no mocking framework, just a struct of function fields).

Wiring is explicit constructor injection in `cmd/api/main.go`: build repos → build services → pass
services to the router. No DI container, no reflection, no magic.

**Trap follow-up: "Is your layering actually clean?"** ⚠️
Be honest — there are two violations and you should name them:
1. `service.AuditService` takes a `*repository.AuditRepository` (a *concrete* type from the
   repository package), so `service` imports `repository`. Every other repo goes through a
   `domain.XRepository` interface. Fix: add `domain.AuditRepository`.
2. `VaultRepository.GetVaultSaltForUser` reads `auth_credentials.salt` — the vault repository
   reaching into the auth aggregate. Fix: expose it through the auth repo, or accept it as a
   deliberate read-model shortcut and document it.

---

### Q1.2 🔥 Why no web framework? Why not Gin / Echo / Chi?

**Answer.** Go 1.22 added method-and-wildcard patterns to `http.ServeMux`
(`"POST /api/v1/vault/items/{item_id}"`, read back with `r.PathValue("item_id")`). That removed the
main historical reason to reach for a router library. What I lose is grouping and middleware
chaining, which is ~40 lines to write:

```go
type handlerMiddleware func(http.HandlerFunc) http.HandlerFunc

type routeGroup struct {
    mux         *http.ServeMux
    prefix      string
    middlewares []handlerMiddleware
}
```

`Group()` returns a new group with the prefix joined and middleware slice *copied and appended*, and
`Handle()` applies middleware in reverse so the first-registered runs outermost.

Trade-offs I'd state out loud:
- **Win:** zero dependency surface for the HTTP edge (matters a lot for a security product — every
  dep is supply-chain risk), stdlib stability, no framework upgrade treadmill.
- **Lose:** no built-in param binding/validation, no route listing/introspection, I hand-rolled
  the group abstraction, and `ServeMux` has no per-route 405 handling — an unmatched method falls
  through to my catch-all `/` handler and returns 404 instead of 405.

**Gotcha they may probe:** `routeGroup` is a *value* type and `Group()` does
`append([]handlerMiddleware(nil), ...)` — copying the slice. If it appended to the parent's backing
array directly, two sibling groups could clobber each other's middleware. That copy is load-bearing.

---

### Q1.3 Why raw `database/sql` instead of an ORM (GORM/ent) or `sqlc`?

**Answer.**
- Every query in this app is either a simple keyed lookup or something with a security-critical
  `WHERE owner_user_id = $1`. I want that predicate visible in the source, not generated. An ORM
  makes it easy to accidentally write a query that *isn't* scoped to the owner — the exact bug
  class (IDOR) I most need to avoid.
- `BYTEA` blobs and `JSONB` don't benefit from ORM mapping.
- The cost is real: manual `Scan` lists (my `scanVaultItem` takes 14 destinations), boilerplate, and
  no compile-time checking that the SELECT column order matches the Scan order. **If I rebuilt it
  I'd use `sqlc`** — it generates type-safe Go from the SQL I already wrote, which keeps the
  visibility of raw SQL and removes the scan-ordering footgun.

---

### Q1.4 How do errors flow from the database to the HTTP response?

**Answer.** Three-stage translation:

1. **Repository** converts driver errors to domain sentinels: `sql.ErrNoRows → domain.ErrNotFound`,
   `pq.Error` code `23505` (unique violation) → `domain.ErrEmailTaken` / `domain.ErrAlreadyShared` /
   `domain.ErrFamilyRequestAlreadySent`. Everything else is wrapped with
   `fmt.Errorf("context: %w", err)`.
2. **Service** adds business sentinels (`ErrNotItemOwner`, `ErrNotFamilyMember`,
   `ErrMFARateLimited`, …) and keeps wrapping with `%w` so the chain survives.
3. **Controller** does `errors.Is(err, domain.ErrX)` in a switch, maps to a status code + stable
   machine-readable error code, and — critically — the `default` branch logs the real error and
   returns a *generic* message. Internal errors never leak SQL text or table names to the client.

Why sentinels and not typed errors? Sentinels compose well with `%w` + `errors.Is`, and there's no
extra data to carry. If I needed the offending field name I'd switch to a custom type with
`errors.As`.

**Say this:** "The unique-violation-as-control-flow pattern (`23505`) is deliberate — checking
'does this email exist' then inserting is a TOCTOU race under concurrency. Letting the database's
unique index be the arbiter is the only correct way to do it."

---

### Q1.5 How is the app configured and how do you prevent an insecure production boot?

**Answer.** `config.Load()` reads env (with `godotenv` for local `.env`) into a single `Config`
struct with defaults. Then `cfg.ValidateForProduction()` runs *before anything else* in `main` and
`os.Exit(1)`s if `APP_ENV` is prod/staging and `AUTH_TOKEN_PEPPER` is either the shipped dev default
or shorter than 32 chars. Fail fast and loud beats booting insecurely.

**⚠️ Trap I should name first:** `mustInt` is a hand-rolled digit parser that returns **0** on any
non-digit input. So `KDF_MEMORY_KIB=64k` silently becomes `0`, and that 0 is then *served to every
client* from `/api/v1/vault/kdf-params` as the Argon2 memory parameter. That's a config typo turning
into a cryptographic downgrade. Fix: use `strconv.Atoi`, return an error, and clamp KDF params to a
sane floor (e.g. ≥ 19 MiB / ≥ 2 iterations per OWASP).

---

## 2. The Zero-Knowledge Crypto Model (the core of the interview)

### Q2.1 🔥 Explain the whole key hierarchy, end to end.

**Answer.** Draw this:

```
                    master password (never leaves the browser)
                        │
        ┌───────────────┴───────────────┐
        │                               │
  Argon2id(pw, authSalt)          Argon2id(pw, vaultSalt)
  authSalt = SHA256(              vaultSalt = random 32B,
    "pmv2-auth-v1:" + email)      stored server-side
        │                               │
        ▼                               ▼
   AUTH VERIFIER (32B, hex)          KEK  (32 bytes)
        │                               │
   sent to server                       ├── wraps per-item DEK   (AAD "pmv2:dek-wrap:v1")
        │                               └── wraps X25519 private key
        ▼                                     (AAD "pmv2:private-key-wrap:v1")
   SHA256(pepper ‖ verifier)
   stored in auth_credentials         DEK (random 32B per item)
                                          │
                                          └── XChaCha20-Poly1305 encrypts the item JSON
```

Server-side, a `vault_items` row is: `ciphertext`, `nonce` (24B), `dek_wrapped`, `wrap_nonce`,
`algo_version`, plus non-secret bookkeeping. The server can validate *shape* (non-empty, valid
base64, valid JSON metadata) and nothing else.

---

### Q2.2 🔥 Why derive **two** keys from one password? Why not just send a hash of the password?

**Answer.** This is the single most important design decision in the project.

If the value I send to the server were derived with the *same* salt as the vault KEK, then the
server (or anyone who breaches it) would hold a value one step away from the encryption key — the
zero-knowledge property collapses. So:

- **Auth verifier** = `Argon2id(masterPassword, SHA256("pmv2-auth-v1:" || lowercase(email)))`
- **KEK** = `Argon2id(masterPassword, randomPerUserVaultSalt)`

Two different salts → two computationally unrelated 32-byte outputs. The server sees only the first
and can never reconstruct the second. Domain-separation by salt is doing the security work here.

**Why is the auth salt deterministic (email-derived) instead of random?** Because the client must
compute the verifier *before* it is authenticated — there's no session yet to fetch a random salt
with. Deriving it from the email is a chicken-and-egg fix. It's still per-user unique, so a single
rainbow table doesn't cover all users, and each guess still costs a full 64 MiB Argon2id.

**Follow-up trap: "So what happens if a user changes their email?"** ⚠️ Their auth salt changes, the
verifier changes, and login breaks. There's no email-change flow in the app today — which is
consistent, but if I added one it would have to re-derive and re-submit the verifier from the client
while the user is holding the master password.

---

### Q2.3 🔥 Why does the server hash the verifier with a *fast* SHA-256 instead of Argon2 again?

**Answer.** Because the expensive work already happened. The verifier is itself the output of
64 MiB / t=3 Argon2id over the master password. If the database leaks, an attacker holding
`SHA256(pepper ‖ verifier)` still has to guess master passwords and run the full client-side Argon2id
for each guess — the work factor is already baked into the input. Re-running Argon2 server-side
would multiply *my* CPU cost per login by a large constant and buy the attacker almost nothing.

There's a second reason: **the pepper.** The stored value is
`SHA256("pmv2:auth-verifier:" ‖ pepper ‖ verifier)` where the pepper is a 32+ char server secret held
in env, never in the database. An attacker with a *database-only* leak (SQL injection, stolen
backup, snapshot of a managed Postgres) can't even test guesses offline — they need the application
host's environment too. That's the point of a pepper as opposed to a salt.

**Follow-up: "Why not SRP or OPAQUE?"** — Correct answer: those are strictly better. An augmented
PAKE means the server never receives *anything* password-equivalent, even transiently, which
protects against a malicious/compromised server logging the verifier at the TLS edge. My scheme
protects the vault from a *passive* server compromise but not an *active* one; a server that starts
recording verifiers can build an offline cracking corpus. OPAQUE closes that. The reason I didn't:
implementation complexity and a much weaker library ecosystem in Go/browser at the time. Bitwarden
uses essentially the same verifier design I did.

---

### Q2.4 🔥 Why per-item DEKs (envelope encryption) instead of encrypting everything with the KEK?

**Answer.** Four reasons, and the sharing one is the big one:

1. **Sharing without exposing the vault.** To share one item I wrap *that item's DEK* to the
   recipient. If everything were encrypted directly under the KEK, sharing one password would mean
   handing over the key to the entire vault.
2. **Blast radius.** One leaked DEK compromises one item.
3. **Key rotation.** Changing the master password re-derives the KEK and only requires re-wrapping
   N small DEKs — not re-encrypting N potentially large ciphertexts.
4. **Nonce hygiene.** XChaCha20-Poly1305 has a 24-byte random nonce so collision risk is negligible
   anyway, but per-item keys mean the number of encryptions under any single key stays tiny.

The DEK wrap is itself authenticated with AAD `"pmv2:dek-wrap:v1"`, so a wrapped DEK can't be
pasted into a different context (e.g. a share wrap, which uses `"pmv2:share-dek-wrap:v1"`) and still
authenticate. That's **domain separation via AAD** and it's worth naming explicitly.

---

### Q2.5 🔥 Why XChaCha20-Poly1305 and not AES-256-GCM?

**Answer.**
- **Nonce size is the deciding factor.** GCM has a 96-bit nonce; with random nonces you hit the
  birthday bound uncomfortably early (~2³² messages per key), and nonce reuse in GCM is
  catastrophic — it leaks the authentication key, not just the plaintext. XChaCha20 has a 192-bit
  nonce, so `crypto.getRandomValues(24)` per message is safe forever with no counter state to
  manage across devices. In a system where **three independent clients (web, extension, future
  mobile) encrypt under the same key**, there is no safe way to coordinate a nonce counter, so a
  large random nonce isn't a nicety, it's a requirement.
- **Constant-time in software.** ChaCha20 is an ARX cipher — no lookup tables, so no cache-timing
  side channels — whereas AES without AES-NI (some mobile/older hardware) is either slow or
  table-based and leaky. The browser is not a place where I can guarantee hardware AES.
- **Trade-off honestly:** where AES-NI exists, AES-GCM is faster and it's what FIPS wants. I'm
  encrypting kilobytes of JSON, so throughput is irrelevant.

---

### Q2.6 🔥 Argon2id with 64 MiB, t=3, p=2 — defend those numbers.

**Answer.**
- **Argon2id** (not `i`, not `d`) because it's the hybrid: the first pass is data-independent
  (resists side-channel/cache-timing attacks against the memory access pattern) and later passes
  are data-dependent (resists GPU/ASIC time-memory trade-off attacks). It's the RFC 9106 and OWASP
  recommendation for password hashing.
- **64 MiB** is the real defense. GPU/ASIC cracking dies on memory bandwidth, not compute. A modern
  GPU has ~10–24 GB of VRAM; at 64 MiB per guess you can run a few hundred parallel guesses instead
  of the hundreds of thousands you'd get against bcrypt.
- **t=3, p=2** balances that against running inside a browser tab on a mid-range laptop or phone —
  I need unlock to feel like ~0.5–1s, not 5s.
- **It's server-configurable.** `KDF_MEMORY_KIB` / `KDF_ITERATIONS` / `KDF_PARALLELISM` are env
  vars, served to clients via a public `GET /api/v1/vault/kdf-params`, so I can raise the work factor
  as hardware improves without shipping a client.
- **It runs in a Web Worker** so allocating 64 MiB and grinding for a second doesn't freeze the UI
  thread.

**⚠️ THE TRAP — and you should raise it yourself:** if the *server* tells the *client* what KDF
parameters to use, and my whole threat model says the server is untrusted, then a malicious server
can serve `memory=1, iterations=1` and cause the client to derive a trivially crackable KEK. That's
a **KDF downgrade attack**. Two mitigations: (a) hard-code a client-side floor and refuse anything
weaker, (b) persist the params actually used at setup alongside the user's record and pin them.
Related real bug in my code: the unlock path (`crypto.worker.ts` `verify-vault-key`) ignores the
fetched params and uses compiled-in defaults — which accidentally blocks the downgrade but means
that **if I ever raise the server params, every existing user is permanently locked out.** Both
halves need the same fix: persist and pin the params used at setup.

---

### Q2.7 🔥 The server has no idea what the master password is. So how does the client know the user typed it right? Why not just "decryption failed"?

**Answer.** There's a **KEK verifier item**: a normal vault item whose plaintext is the fixed
sentinel string `"pmv2-kek-verifier-v1"`, with metadata `{kind: "...", salt: <base64 vault salt>}`.
On unlock the client derives the KEK from the typed password + that salt, tries to decrypt the
verifier item, and does a **constant-time compare** of the plaintext against the expected token.

Why this shape:
- Gives an instant, unambiguous "wrong password" without needing to fetch and attempt the whole
  vault.
- It carries the salt, so the salt travels with the thing it's needed for.
- The AEAD tag alone would already fail on a wrong key — the explicit token compare is belt-and-
  braces and gives a clean boolean instead of an exception path.
- Constant-time compare because a byte-by-byte `===` on a decrypted value is a (weak, but free to
  avoid) oracle.

**Trap:** "isn't the verifier item a known-plaintext attack surface?" — No. Known plaintext against
XChaCha20-Poly1305 with a 256-bit key doesn't help; the attacker's cost is still brute-forcing the
master password through Argon2id, which they could do against any item.

**Second trap:** there are now *two* sources of the vault salt — `GET /vault/salt` (returns
`auth_credentials.salt`) and the verifier item's metadata. That redundancy is a consistency bug
waiting to happen; one should be authoritative.

---

### Q2.8 What's the AAD for and where do you use it?

**Answer.** Associated Data is authenticated but not encrypted — it cryptographically binds a
ciphertext to its *context*. I use distinct constants per wrap purpose:

| Purpose | AAD |
|---|---|
| item DEK wrapped under KEK | `pmv2:dek-wrap:v1` |
| item DEK wrapped for a share (X25519) | `pmv2:share-dek-wrap:v1` |
| X25519 private key wrapped under KEK | `pmv2:private-key-wrap:v1` |
| TOTP secret at rest (server-side) | `pmv2:totp-secret:v1` |

So a malicious server can't take the blob from one field and splice it into another — the tag won't
verify. The `:v1` suffix is also my crypto-versioning hook.

**What I'd improve:** the *item* ciphertext's AAD is caller-supplied and in practice empty. Binding
each item's ciphertext to its `item_id` + `owner_user_id` as AAD would stop a malicious server from
swapping ciphertexts between two of your own items (a "confused deputy" / blob-shuffling attack) —
right now the client would happily decrypt a swapped blob because the key is the same.

---

### Q2.9 Where does key material live at runtime, and how do you clean it up?

**Answer.** The KEK lives in React state inside `VaultProvider` for the life of the tab; decrypted
items live in memory only. DEKs are wiped with `.fill(0)` in a `finally` block right after use, and
`encryptVaultItem` is careful to wipe *only* a DEK it generated — a caller-supplied DEK stays the
caller's responsibility.

**⚠️ Be honest about the limits:** JavaScript gives you no real control over memory. `fill(0)` zeroes
*that* `Uint8Array`, but the GC may already have copied it, and strings are immutable so any
password that touched a `string` is unrecoverable. It's defense-in-depth, not a guarantee. The
`beforeunload` KEK-wipe handler in my code is honestly theater — the page is being destroyed anyway.

**And the real gap I'd fix first:** ⚠️ there is **no idle auto-lock timer** in the web app. Once
unlocked, the KEK stays in memory until logout or a hard refresh. Every serious password manager
locks after N minutes of inactivity. The extension has a fixed 15-minute alarm set at unlock, but
it's not reset on activity and doesn't lock on browser close.

---

### Q2.10 ⚠️ Is your app *actually* zero-knowledge?

**Answer — lead with this, don't get caught.** "Not fully, and I know exactly where it breaks."

`vault_items.metadata` is a JSONB column holding **cleartext** item name, type, and URL, and the API
echoes it back verbatim. So anyone with database access can enumerate every service each user has an
account with. That's a serious privacy leak even though no password is exposed — knowing that a
specific user has a Coinbase account is itself valuable to an attacker.

Why it exists: server-side metadata makes list rendering, search, and folder/type filtering trivial
without decrypting everything.

How I'd fix it: encrypt metadata client-side like everything else, and get search back with
(a) client-side search over the already-decrypted in-memory vault — fine up to thousands of items —
or (b) blind indexing: store `HMAC(searchKey, normalizedTerm)` so the server can do equality lookups
on tokens it can't read. Option (a) is right for this app's scale.

Related leak: `ListSentShares` currently sets `ItemTitle = string(metadata)` — it dumps the whole raw
metadata JSON blob into a field called "title." That's a code-quality bug on top of the design one.

---

## 3. Authentication & Session Management

### Q3.1 🔥 Opaque session tokens vs JWT — why did you choose opaque?

**Answer.** Opaque, and I'd make the same call again for this product.

The mechanics: on login I generate 32 bytes from `crypto/rand`, hex-encode them (64 chars), set them
in a cookie, and store `SHA256(pepper ‖ ":" ‖ token)` in `sessions.refresh_token_hash`. Every
authenticated request looks the token up by hash with `revoked_at IS NULL AND expires_at > NOW()`.

Why not JWT:
- **Revocation is the whole ballgame for a password manager.** "Sign me out of all devices" and
  "revoke everything after a password reset" must be *instant*. A stateless JWT can't be revoked
  without a denylist — at which point you're doing a database lookup per request anyway and have
  bought nothing but complexity and a signing-key rotation problem.
- I get free session *metadata*: device name, IP, user agent, created-at — which powers device
  management and the audit log.
- The cost is one indexed lookup per request. There's an index on `refresh_token_hash`. For a
  self-hosted family app that's nothing; at scale I'd put a short-TTL cache in front.

**Why hash the token at rest?** Because a session token is a bearer credential. If someone dumps the
`sessions` table they get hashes, not usable tokens. Same logic as password hashing. Fast SHA-256 is
fine here because the input is 256 bits of true entropy — there's nothing to brute-force. The pepper
makes even a rainbow-table-style precomputation impossible.

**⚠️ Nit worth owning:** `HashToken` is `SHA256(pepper + ":" + token)` with **no domain-separation
prefix**, unlike every other hash in the codebase (`"pmv2:auth-verifier:"`, `"pmv2:email-code:"`,
`"pmv2:recovery-code:"`). Inconsistent — should be `"pmv2:session:"`. Also the column is named
`refresh_token_hash` but there is no access/refresh split; it's just the session token. Naming debt.

---

### Q3.2 🔥 Walk me through your cookie flags and how you handle CSRF.

**Answer.** The session cookie is `HttpOnly`, `Secure` (in prod, driven by `APP_ENV`), `Path=/`,
`SameSite=Lax`, with both `Expires` and `MaxAge` set.

- `HttpOnly` — XSS can't read the token via `document.cookie`.
- `Secure` — never sent over plaintext HTTP.
- `SameSite=Lax` — the browser withholds the cookie from cross-site subrequests. Since I have **no
  state-changing GET endpoints** (Lax still sends cookies on top-level GET navigations), this
  neutralizes classic CSRF without a token.
- Plus CORS reflects **only** explicitly allow-listed origins when `Allow-Credentials: true`, and
  never combines `*` with credentials.

**🔥 The trap they will spring:** *"Your frontend is on Vercel and your API is on Render. Those are
different registrable domains. `SameSite=Lax` means the browser won't send that cookie on your XHRs
at all — how does your app work?"*

The right answer: **it doesn't, in that configuration.** `SameSite=Lax` requires the frontend and
API to be same-site — same registrable domain, subdomains are fine. That's why the deployment moved
to subdomains of a single domain (`app.jattin.in` / `api.jattin.in`). If I genuinely needed
cross-site, I'd have to use `SameSite=None; Secure`, which turns CSRF back on and forces me to add a
real anti-CSRF token (double-submit or the `Origin` header check). Being able to explain *why*
same-site deployment is a security decision and not a convenience is the point.

**Residual risk to mention:** SameSite is same-*site*, not same-*origin*. A compromised sibling
subdomain is "same-site" and can both send the cookie and, with a `Domain=` cookie, do cookie
tossing. So a CSRF token is still worth having as defense-in-depth. My audit flags it as a known
open item.

---

### Q3.3 🔥 You had a privilege-escalation bug in password reset. Tell me about it.

**Answer — this is a great story, tell it deliberately.**

The reset flow is: request code by email → verify the 6-digit code → get a short-lived reset token →
confirm the new password. The original implementation minted the reset token as an **ordinary row in
the `sessions` table**. But `AuthMiddleware` authenticates by looking up *any* active session by
token hash — so that reset token was accepted on **every authenticated endpoint** for 15 minutes.

Concretely: anyone who obtained a 6-digit email code could, without ever setting a password, read
the encrypted vault, call `POST /auth/totp/disable` to strip 2FA, and delete items. The email code
alone became a full account takeover.

**The fix:** I added a `purpose` column to `sessions` (`'auth'` | `'password_reset'`), made the
generic lookup a private helper parameterized by purpose, and exposed exactly two public methods:
`GetActiveSessionByTokenHash` (hard-filters `purpose='auth'`) and
`GetActiveResetSessionByTokenHash` (`purpose='password_reset'`, used only by the confirm handler).

**The lesson, which is the actual answer they want:** *reusing an authentication primitive for a
different trust level is a security bug, not a refactor.* A token's authority must be explicit in
the data model, not implied by which code path happened to create it. The general form is "capability
scoping" — and the reason I found it is that I sat down and wrote a real audit rather than trusting
that it looked fine.

---

### Q3.4 🔥 Password reset in a zero-knowledge system — what actually happens to the vault?

**Answer.** **It gets wiped, transactionally, and the user is warned before they start.**

This is the fundamental, unavoidable trade-off of zero knowledge: the vault is encrypted under a KEK
derived from the master password. If the user forgets the password, the KEK is gone and the server
*by construction* cannot recover the plaintext. So the reset flow does exactly what it must:

1. Verify the emailed code → issue a purpose-scoped reset token.
2. `UpdatePassword` with a fresh random 32-byte vault salt + the new verifier.
3. `WipeUserVault` — a single transaction deleting shares to the user, their items (cascading to
   versions/shares/attachments), folders, and `user_keys`.
4. `RevokeAllUserSessions`, then mint a clean session and log an audit event.

**What a *real* recovery mechanism looks like** (and this is the follow-up they want):
Store a second copy of the KEK, wrapped under a key derived from a high-entropy **recovery code**
generated at signup and shown to the user once. Then reset = unwrap KEK with the recovery code,
re-wrap KEK under the new password's KEK. **No data loss and still zero-knowledge, because the
server only ever holds two ciphertexts of the same key.** That is exactly what 1Password's Secret
Key and Bitwarden's emergency access do. The repo has scaffolding for this
(`GenerateRecoveryCodes`, `HashRecoveryKey`, and the README describes it), but the shipped path is
the wipe — and there's dead/misleading UI copy about recovery codes I should clean up.

**Also:** the reset preserves TOTP but wipes `user_keys`, so all previously received shares break —
correct, since the recipient's private key was wrapped under the old KEK.

---

### Q3.5 🔥 Walk me through your TOTP implementation.

**Answer.** Implemented from scratch (RFC 6238 on top of RFC 4226), not a library:

- Secret: 20 random bytes, Base32 (no padding), delivered as an `otpauth://` URI for QR scanning.
- Code: `HMAC-SHA1(secret, floor(unixTime/30))`, dynamic truncation using the low nibble of the last
  byte as an offset, mask the high bit, `mod 10^6`, zero-pad to 6.
- Verification accepts a **±1 step window** (~90s total) for clock skew, and compares with
  `subtle.ConstantTimeCompare`.
- **The secret is encrypted at rest**: XChaCha20-Poly1305 with key `SHA256("pmv2:totp-secret:" ‖
  pepper)`, random 24-byte nonce prepended to the ciphertext, AAD `"pmv2:totp-secret:v1"`. So a
  database-only leak doesn't yield working second factors.
- **Anti-brute-force is database-backed**, not in-memory: 5 failures in a 30s window → 5-minute
  lockout, tracked in `totp_failed_attempts` / `totp_window_started_at` / `totp_locked_until` and
  updated under `SELECT … FOR UPDATE` inside a transaction so concurrent attempts can't race past
  the counter. (Contrast with the login throttle, which *is* in-memory — see Q5.2.)
- Enabling is one-shot: `EnableTOTP` returns `ErrTOTPAlreadyEnabled` if already on, so the
  setup-secret can't be silently rotated by replaying the enable call.

**Why SHA-1?** Because RFC 6238's de-facto interop profile is SHA-1/6/30 and that's what Google
Authenticator, Authy, and 1Password actually implement. HMAC-SHA1 is not broken (SHA-1's collision
weakness doesn't transfer to HMAC), and interoperability wins.

**Traps to pre-empt:**
- ⚠️ **No replay protection.** A code stays valid for its whole ~90s window and can be used more
  than once. Real implementations store the last-used counter per user and reject `counter <= last`.
- ⚠️ `ParseStoredTOTPSecret` falls back to interpreting the blob as *plaintext Base32* if decryption
  fails — a migration compatibility shim for secrets stored before encryption existed. It's a
  downgrade path that should be removed after a backfill.
- ⚠️ `DisableTOTP` requires only a valid session — **no password or TOTP re-verification.** Anyone
  with a stolen session cookie can strip 2FA. Real fix: require step-up auth for security-sensitive
  mutations.

---

### Q3.6 How do the emailed 6-digit codes work, and why are they safe enough?

**Answer.** Two purposes share one mechanism (`email_verification_codes`, keyed by
`(user_id, purpose)`): `password_reset` and `mfa_recovery`.

- 6 random digits, 10-minute TTL, max 5 attempts, stored as `SHA256("pmv2:email-code:" ‖ pepper ‖
  code)` and compared with `ConstantTimeCompare`.
- Issuing a new code **invalidates all prior unconsumed codes** for that purpose (marks them
  consumed) — so you can't accumulate a pool of valid guesses by spamming "resend."
- Consumption marks `consumed_at`; expiry or attempt-exhaustion invalidates the whole set.
- `mfa_recovery` codes are only issued if the caller **also proves knowledge of the password** and
  the account actually has TOTP enabled — so it's a genuine second factor swap, not a bypass.

**Why 6 digits (a 10⁶ space) is defensible here:** the code is single-use, expires in 10 minutes,
and dies after 5 wrong guesses; an attacker gets 5 shots at 1-in-a-million per issued code.

**⚠️ Two traps:**
1. `NewNumericCode` does `'0' + (b % 10)` on a random byte. 256 isn't divisible by 10, so digits
   0–5 are ~1.5% more likely than 6–9. Negligible in practice but it's textbook **modulo bias** and
   the correct fix is rejection sampling (`if b >= 250 { redraw }`) or `rand.Int(rand.Reader,
   big.NewInt(10))`. Naming this unprompted signals real crypto literacy.
2. The verify step has a per-*code* attempt cap but no per-*account* throttle beyond the shared
   per-IP limiter — a distributed attacker rotating IPs gets 5 guesses per issued code with no
   account-level backoff.

---

### Q3.7 How do you prevent account enumeration?

**Answer.** Several layers, and one place I fail:

- **Login:** on unknown email I still run `HashAuthVerifier` and a `ConstantTimeCompare` against a
  zero buffer before returning the *same* `invalid_credentials` error, so an unknown email costs
  roughly the same as a wrong password. And I record the failure in the throttle either way.
- **Password reset request:** always returns `200 {"status":"reset_email_sent"}` regardless of
  whether the account exists; failures are logged server-side only.
- **MFA email code request:** always returns `200`, even on wrong password or no TOTP.

**⚠️ Where I leak anyway (say it first):**
- `POST /auth/register` returns **409 `email_taken`** — a direct enumeration oracle. Unavoidable
  UX-wise unless you move to email-confirmation-first signup, which is the real fix.
- `GET /users/keys/lookup?email=` and `POST /family/request` return distinguishable responses for
  known vs unknown emails (authenticated, so lower severity, but still an oracle for any user).
- The timing flattening is partial: I skip the *database round trip* on unknown emails, and that's
  probably a bigger timing signal than the hash. A more rigorous version would keep the code path
  shape identical.

---

### Q3.8 Session lifecycle: expiry, revocation, cleanup?

**Answer.**
- TTL from `SESSION_TTL`, default **720h (30 days)** — chosen for a family app where re-typing a
  master password constantly is the thing that makes people quit.
- Logout revokes by setting `revoked_at` (soft), which keeps the row for audit.
- Password reset calls `RevokeAllUserSessions`.
- A background goroutine started in `main` ticks **hourly** and hard-deletes rows where
  `expires_at < NOW() OR revoked_at IS NOT NULL`, so the table doesn't grow unbounded.
- Graceful shutdown: `SIGINT`/`SIGTERM` → `srv.Shutdown(ctx)` with a 10s drain.

**⚠️ Weaknesses to own:** 30 days is long for a password manager and tokens are **never rotated on
use** — a stolen cookie is good for a month. There's no per-user concurrent-session cap and no
session-listing/revocation UI. I'd add sliding expiry with rotation, a max absolute lifetime, and a
device-management screen.

**Also:** the cleanup goroutine has no context/cancellation and isn't stopped on shutdown, and it
ignores the parent context — fine in practice because the process is exiting, but sloppy. It also
runs in *every* replica, so at multi-instance scale N replicas do the same DELETE hourly (harmless,
but should be a leader-elected or DB-scheduled job).

---

## 4. Data Modeling & PostgreSQL

### Q4.1 🔥 Walk me through the schema.

**Answer.** 13 tables. Core spine:

| Table | Purpose | Notes |
|---|---|---|
| `users` | identity | email UNIQUE, `email_verified` |
| `auth_credentials` | 1:1 with users | `salt`, `password_hash`, Argon2 `params` JSONB, TOTP state + lockout counters |
| `user_keys` | 1:1 | X25519 public key + KEK-encrypted private key + nonce |
| `vault_items` | the vault | `ciphertext`, `nonce`, `dek_wrapped`, `wrap_nonce`, `algo_version`, `metadata` JSONB, `version`, `deleted_at` |
| `vault_item_versions` | append-only history | full snapshot per prior version |
| `vault_folders` | folders | name is a *ciphertext* column, not text |
| `vault_shares` | PK `(item_id, user_id)` | per-recipient wrapped DEK + permissions |
| `sessions` | opaque sessions | token **hash**, `purpose`, device/IP/UA, `expires_at`, `revoked_at` |
| `audit_events` | activity log | `event_type` + `event_data` JSONB |
| `email_verification_codes` | reset/MFA codes | code hash, attempts, expiry, consumed_at |
| `family_memberships` | PK `(user_id, friend_id)` | `status` CHECK ('pending','accepted'), `CHECK (user_id != friend_id)` |

Design points worth calling out:
- **Secrets are `BYTEA`, not `TEXT`.** No base64 tax in storage; base64 only at the JSON boundary.
- **Auth is split from identity** (`auth_credentials` separate from `users`) so adding WebAuthn or
  OAuth later is a new table, not a widened one — and so a `SELECT * FROM users` for display never
  drags credential material along.
- **Constraints in the database, not just the app**: unique email, the `status` CHECK, the
  self-friend CHECK, composite PKs that make "shared twice with the same person" and "duplicate
  friendship" structurally impossible.
- **`ON DELETE CASCADE` vs `SET NULL` is chosen per relationship**: deleting a user cascades their
  vault away, but `audit_events.user_id` and `vault_shares.shared_by_user_id` are `SET NULL` so
  history survives the actor.

---

### Q4.2 🔥 Why soft delete, and what's the catch?

**Answer.** `deleted_at TIMESTAMPTZ` gives a trash/restore UX, which for a password manager is a
genuine safety feature — accidentally deleting the only copy of a credential is unrecoverable.
Delete is `UPDATE … SET deleted_at = NOW() WHERE id=$1 AND owner_user_id=$2 AND deleted_at IS NULL`;
restore is the inverse with `deleted_at IS NOT NULL`. Both use `RowsAffected() > 0` so a
double-delete is correctly a 404 rather than a false success.

There's a partial-ish index `(owner_user_id, deleted_at)` supporting both the live-list and
trash-list queries.

**Catches to name:**
- ⚠️ **No purge job.** Nothing ever hard-deletes trashed items, so "deleted" ciphertext lives
  forever. For a privacy product that's wrong — trash should auto-purge after 30 days.
- ⚠️ **`ListSharesByRecipient` doesn't filter `deleted_at`.** If I trash an item I shared with you,
  it vanishes from my vault but **still appears in your "shared with me" list.** That's a real bug —
  a `WHERE vi.deleted_at IS NULL` is missing from that JOIN.
- Every future query has to remember the predicate. A `WHERE deleted_at IS NULL` view, or Postgres
  RLS, would make it structural rather than a discipline problem.

---

### Q4.3 🔥 How does item version history work? Walk me through the concurrency.

**Answer.** `UpdateVaultItemForOwner` runs one transaction:

```sql
BEGIN;
SELECT … FROM vault_items
 WHERE id=$1 AND owner_user_id=$2 AND deleted_at IS NULL
 FOR UPDATE;                          -- row lock
INSERT INTO vault_item_versions (…);  -- snapshot the CURRENT state
UPDATE vault_items SET …, version = current+1, updated_at = NOW()
 WHERE id=$1 AND owner_user_id=$2
RETURNING …;
COMMIT;
```

`FOR UPDATE` is the important line. Without it, two concurrent edits could both read `version = 4`,
both write `version = 5`, and one snapshot would be lost or duplicated. The row lock serializes
them: the second transaction blocks on the `SELECT` until the first commits, then reads `version=5`
and correctly writes `6`. `defer tx.Rollback()` is a no-op after a successful `Commit` (it returns
`ErrTxDone`, deliberately ignored), so it's a safe unconditional cleanup.

Note this is **pessimistic** locking. The alternative is optimistic concurrency: have the client
send the version it read and do `UPDATE … WHERE version = $expected`, returning 409 on zero rows
affected. Optimistic is better for a multi-device app because it can actually *tell the user* their
edit conflicted, instead of silently last-write-wins after a lock wait. That's a change I'd make.

**⚠️ Also:** history is unbounded — every edit appends a full ciphertext snapshot forever. Needs a
retention cap (keep last N, or N days).

---

### Q4.4 What's the transaction story elsewhere?

**Answer.** Transactions where multi-statement atomicity actually matters:
- `CreateUserWithCredentials`: `users` + `auth_credentials` insert together (a user without
  credentials is an unloggable-in zombie).
- `CreateVaultItemsBulk`: one tx + a **prepared statement** reused across rows — one parse/plan for
  N inserts. Capped at 500 items per request.
- `UpdateVaultItemForOwner`: snapshot + update (above).
- `RecordTOTPFailure`: `SELECT … FOR UPDATE` + counter update.
- `WipeUserVault`: four DELETEs, all-or-nothing.

Everything else is a single statement and therefore already atomic — I don't wrap single statements
in transactions for ceremony.

**Inconsistency worth owning:** ⚠️ most methods use `defer tx.Rollback()`, but
`CreateUserWithCredentials` does manual `_ = tx.Rollback()` on each error branch. Same effect, but
the defer form is harder to get wrong on a later edit. Standardize.

---

### Q4.5 🔥 Talk me through your indexes and any query you'd expect to be slow.

**Answer.** Indexes: `vault_items(owner_user_id)`, `vault_items(owner_user_id, deleted_at)`,
`vault_item_versions(item_id)`, `sessions(user_id)`, `sessions(refresh_token_hash)`,
`vault_shares(user_id)`, `audit_events(user_id)`, plus the folder/attachment/code FKs.

The **`sessions(refresh_token_hash)`** one is the hot path — it's hit on literally every
authenticated request.

**Problems I'd fix:**
1. ⚠️ **`is_shared` is a correlated subquery per row.**
   ```sql
   (SELECT COUNT(*) > 0 FROM vault_shares vs WHERE vs.item_id = vi.id) AS is_shared
   ```
   That runs once per returned row — the classic N+1, just pushed into SQL. Two fixes: use
   `EXISTS(...)` instead of `COUNT(*) > 0` so Postgres can short-circuit on the first match, or
   `LEFT JOIN LATERAL` / a grouped join. `EXISTS` alone is the one-line win.
2. ⚠️ **`GET /vault/items` has no pagination at all.** It returns every item with full ciphertext.
   A user with 5,000 items gets a multi-megabyte response every page load. Needs keyset pagination —
   and keyset (`WHERE (updated_at, id) < (…)`) rather than OFFSET, because OFFSET degrades linearly.
3. ⚠️ **Audit log:** `ORDER BY created_at DESC` filtered by `user_id`, but the index is only on
   `user_id` — so Postgres filters then sorts. Wants a composite `(user_id, created_at DESC)`. It
   also runs a `COUNT(*)` over the filtered set on *every* page for `total`, and uses
   `LIMIT/OFFSET`. And the `search` filter does `event_data::text ILIKE '%…%'` — a leading-wildcard
   cast, unindexable, full scan. Fix with a GIN index on `event_data` and jsonb containment, or
   `pg_trgm`.
4. ⚠️ **No connection pool tuning.** `sql.Open` is never followed by `SetMaxOpenConns` /
   `SetMaxIdleConns` / `SetConnMaxLifetime`. Go's default is *unlimited* open connections, which
   against a managed Postgres (Neon/Supabase have low connection ceilings) means a traffic spike
   produces `too many connections` instead of graceful queueing. This is a one-line fix and a very
   common interview probe.

---

### Q4.6 How do migrations work, and what's wrong with that?

**Answer.** `MigrateUp` runs a single idempotent `CREATE TABLE IF NOT EXISTS …` script plus a few
`ALTER TABLE … ADD COLUMN IF NOT EXISTS` patches, executed automatically at startup, with a
`cmd/migrate` CLI for manual up/down/drop.

Why it's fine *for this project*: greenfield, self-hosted, single instance, zero-friction deploy.

**⚠️ Why it's wrong at scale, and I should say so:**
- **No version tracking.** There's no `schema_migrations` table, so I can't tell what's applied or
  write a migration that isn't expressible as `IF NOT EXISTS`. Column *type* changes, data
  backfills, and constraint additions have no home.
- **`MigrateDown` is `DROP TABLE … CASCADE`** — a "rollback" that destroys all data. That is a
  loaded gun pointed at production.
- **No advisory lock.** If two replicas boot simultaneously they race on DDL.
- **Migrations at startup couples deploy to schema change**, so a bad migration takes the app down
  rather than failing a separate step.

Fix: `golang-migrate` or `goose` with versioned up/down files, `pg_advisory_lock` around the run,
and migrations as a distinct deploy step (an init container / release phase).

---

### Q4.7 How do you know you're safe from SQL injection?

**Answer.** Every value is a `$N` placeholder passed as an argument — the driver sends them
out-of-band, so they're never parsed as SQL. There is exactly **one** place I build SQL with
`fmt.Sprintf`, in `AuditRepository.ListEvents`, and what it interpolates is **placeholder indices
and a fixed predicate string**, never user data:

```go
where += fmt.Sprintf(" AND event_type LIKE $%d", argIdx)
args = append(args, filter.Category+"%")
```

The filter *value* still goes through `args`. Same for the `listVaultItemsByOwner` helper, which
`Sprintf`s only a hardcoded `IS NULL` / `IS NOT NULL` and an `ORDER BY` chosen from two literals —
never from request input. That's the rule: **dynamic SQL may only ever be assembled from a
closed set of constants, never from input.**

---

## 5. Concurrency, Rate Limiting & Scaling

### Q5.1 🔥 Where is there shared mutable state, and how is it protected?

**Answer.** Go's `net/http` runs every request in its own goroutine, so anything shared is a data
race by default. In this codebase:

- **`RateLimiter.clients` map** — guarded by a `sync.Mutex`. Note the lock is held only long enough
  to fetch/insert the `*rate.Limiter` and update `lastSeen`; the actual `limiter.Allow()` call is
  made **after** unlocking, because `rate.Limiter` is itself goroutine-safe. Holding my mutex across
  it would serialize every request through one lock.
- **`loginThrottle.entries` map** — same pattern, `sync.Mutex`.
- **Both have background sweeper goroutines** (rate limiter: every 1 min, evicting entries idle >3
  min; sessions: hourly DB delete) that also take the lock. Without the sweeper the map is an
  unbounded memory leak keyed by attacker-controlled IP — a slow-burn DoS.
- **`util.trustedProxyCount`** is a package-level global set once at startup by
  `ConfigureTrustedProxies`. ⚠️ It's written before any request is served so it's safe in practice,
  but it's a mutable global — untestable in parallel tests and racy if anything ever set it at
  runtime. It should be a field on a struct.
- Everything else — repositories, services — holds no per-request state; `*sql.DB` is itself a
  concurrency-safe pool.

---

### Q5.2 🔥 You have two different rate limiters. Why? Explain the layering.

**Answer.** They defend different things and neither is sufficient alone:

| | Per-IP limiter (middleware) | Per-account login throttle (service) |
|---|---|---|
| Key | client IP | normalized email |
| Config | `rate.Limit(5)/sec`, burst 15 | 10 failures / 15 min window → 15 min lock |
| Counts | *all* requests to auth routes | *failed password attempts only* |
| Stops | one noisy source hammering the API | **distributed** guessing at one account from many IPs |
| State | in-memory map | in-memory map |

Third layer: TOTP lockout, which is **database-backed** (5 failures / 30s → 5 min lock).

The insight to state: an IP limiter alone loses to a botnet — 10,000 IPs each making 4 requests/sec
never trip it while collectively hammering one account. An account limiter alone lets one IP
enumerate a million *different* accounts. You need both, on different keys.

**⚠️ The scaling flaw, which I'd volunteer:** two of the three are `map[string]*x` in process memory.
That means (a) they **reset on restart** — an attacker who can trigger a deploy or crash clears the
lockout, and (b) they're **per-replica** — with 4 instances behind a load balancer an attacker gets
4× the budget, and lockout state doesn't follow the user. The self-hosted single-instance target
makes it acceptable, and the code says so in a comment, but the correct answer for multi-replica is a
shared store: Redis with `INCR`+`EXPIRE`, or a sliding-window/token-bucket in Redis (or just move
the login throttle into Postgres like the TOTP one already is — that path is already proven in this
codebase).

---

### Q5.3 🔥 How do you get the client's real IP, and why is that security-critical?

**Answer.** Because the IP is the rate-limiter *key*, letting a client spoof it means letting them
mint unlimited limiter buckets and bypass rate limiting entirely. `X-Forwarded-For` is
client-appendable, so naïvely taking the first entry is a vulnerability.

My implementation is `TRUSTED_PROXY_COUNT`-driven:
- `0` (default) → **ignore XFF entirely**, use `RemoteAddr`. Correct when there's no proxy.
- `N > 0` → take the **Nth entry from the right**. Each trusted proxy appends its immediate peer on
  the right, so the rightmost N entries are the ones my own infrastructure wrote. Anything a client
  injected sits to the *left* and is ignored.

Then `NormalizeIP` parses and strips the port so the value is safe for the Postgres `INET` column.

**Why "count from the right" is the right mental model:** you can only trust as many hops as you
actually operate. Behind one Render/Cloudflare LB, that's `1`. Getting this wrong in either
direction is a bug: too high and you trust attacker input; too low and you rate-limit your own load
balancer's IP, which throttles *everyone*.

---

### Q5.4 The audit log writes on the request path. Is that OK?

**Answer.** Today `AuditService.LogEvent` is a **synchronous** INSERT inside the request, but
**best-effort** — it never returns an error to the caller; failures are logged via `slog`. The
reasoning: an audit write failing shouldn't fail a successful login.

**⚠️ Problems I'd name:**
- It adds a DB round trip to the latency of every mutating endpoint.
- It uses `r.Context()`, so if the client disconnects mid-request the context is cancelled and the
  audit write is **dropped** — precisely for the requests an attacker might abort deliberately.
  Should use `context.WithoutCancel(ctx)` (Go 1.21+) or a detached context with its own timeout.
- Swallowing errors means security events can silently vanish. For a real audit trail you want
  either (a) the write in the *same transaction* as the action so they're atomic, or (b) a durable
  queue / outbox with retry. "Fire and forget" and "audit log" are in tension.
- Sensitive-data hygiene: `event_data` currently stores IP and device name — fine — but the pattern
  makes it easy for someone to later log something they shouldn't.

Also: `GetSecuritySummary` fetches 50 events and scans them **in Go** to decide "secure"/"warning".
That's a `SELECT EXISTS(… WHERE event_type IN (…) AND created_at > now() - interval '24 hours')`
that belongs in SQL.

---

### Q5.5 How would you scale this to a million users?

**Answer.** Structured as: what's already fine, what breaks first, what I'd change.

**Already fine.** The API is stateless apart from the two in-memory limiters, so it scales
horizontally. Payloads are opaque blobs — no server-side crypto on the hot path, so CPU is
essentially free (all the Argon2 cost is on clients, which is an *architectural* scaling win worth
naming: my most expensive operation is paid for by the user's device).

**Breaks first, in order:**
1. **Connection exhaustion** — no pool limits (Q4.5). Fix first: `SetMaxOpenConns`, plus PgBouncer
   in transaction pooling mode.
2. **Rate limiters** become per-replica and useless → move to Redis.
3. **`GET /vault/items` unbounded** → keyset pagination + `updated_at`-based delta sync, so a
   returning client fetches only changes.
4. **Session lookup per request** → short-TTL cache (Redis, or an in-process LRU with a few seconds
   TTL) keyed by token hash; accept a few seconds of revocation lag or publish revocations.
5. **`audit_events` becomes the biggest table** → time-partition by month (`PARTITION BY RANGE
   (created_at)`), drop old partitions instead of `DELETE`.
6. **Reads** → `vault_items` reads are user-scoped and cache-poisonous (they're secrets), so I'd
   scale with read replicas for audit/history and keep vault reads on the primary.

**Then:** structured request IDs + OpenTelemetry tracing, `/healthz` that actually pings the DB
(right now it returns "ok" while Postgres is down — a real bug, since a load balancer will keep
routing to a dead instance), and separate readiness vs liveness probes.

---

## 6. Sharing & Asymmetric Crypto

### Q6.1 🔥 Walk me through end-to-end encrypted sharing.

**Answer.**
1. On first unlock, each user generates an **X25519 key pair** client-side. The public key goes to
   the server; the private key is encrypted under that user's KEK (AAD
   `"pmv2:private-key-wrap:v1"`) and stored as an opaque blob in `user_keys`. So the server holds
   the private key but can never decrypt it.
2. To share item X with Bob, Alice: fetches Bob's public key by email → unwraps X's DEK with her own
   KEK → computes `sharedSecret = X25519(alicePrivate, bobPublic)` → encrypts the DEK under that
   secret with a fresh nonce and AAD `"pmv2:share-dek-wrap:v1"` → `POST
   /vault/items/{id}/shares`.
3. The server stores a `vault_shares` row: `(item_id, user_id, shared_by_user_id, dek_wrapped,
   wrap_nonce, permissions)`. **It cannot read the DEK.**
4. Bob lists shared items, computes `X25519(bobPrivate, alicePublic)` — the *same* shared secret by
   the commutativity of ECDH — unwraps the DEK, and decrypts the item ciphertext.

Server-side authorization on the share call: the caller must **own** the item
(`GetVaultItemByIDForOwner`), can't share with themselves, and the recipient must be an **accepted
family member**. The composite PK `(item_id, user_id)` makes duplicate shares a `23505` mapped to
`ErrAlreadyShared`.

The elegant part: the server is a pure **relay for ciphertext it cannot read** — it enforces *who
may receive* without knowing *what is received*.

---

### Q6.2 🔥⚠️ Critique your own sharing crypto.

**Answer — volunteer all of this; it's the strongest thing you can do in a crypto interview.**

**Flaw 1 — raw ECDH output used directly as an AEAD key (no KDF).**
`mod.sharedKey(priv, pub)` returns the raw X25519 shared point. That is **not** a uniformly random
32-byte string — it's an element of a group with algebraic structure and some bias. Every standard
(RFC 7748, NIST SP 800-56C, the Noise Protocol) says: run it through a KDF. The fix is
`HKDF-SHA256(ikm = sharedSecret, salt = <fixed or random>, info = "pmv2:share:v1" ‖ sortedPubkeys)`.
Binding both public keys into `info` also gives **key-compromise-impersonation and identity-binding**
protection — the derived key is provably tied to *this specific pair*.

**Flaw 2 — the shared secret is static per user pair.**
X25519 with long-term keys yields the *same* secret for every share between Alice and Bob forever.
Only the nonce varies. That's a lot of ciphertext under one key with no **forward secrecy**: if
Bob's private key ever leaks, every share he ever received is retroactively decryptable. The real
fix is an **ephemeral-static (HPKE / sealed-box) construction**: generate a fresh ephemeral key pair
per share, `ECDH(ephemeralPriv, bobPub)`, ship the ephemeral public key alongside. That's exactly
what NaCl's `crypto_box_seal` and RFC 9180 HPKE do — and I'd just use HPKE rather than hand-roll it.

**Flaw 3 — no sender authentication of the share.**
Nothing signs the share. A malicious *server* can't read the DEK, but it can forge a
`vault_shares` row claiming Carol shared an item, or swap `shared_by_user_id` so Bob computes ECDH
against the wrong public key (which would just fail to decrypt — so it's a DoS, not a forgery — but
the general lack of an Ed25519 signature over `(item_id, recipient, wrappedDek)` means share
provenance is server-asserted). Note the schema *has* a `public_key_ed25519` column that's never
populated — that was the plan.

**Flaw 4 — revocation is not cryptographic.** See next question.

---

### Q6.3 🔥 What actually happens when you revoke a share?

**Answer.** `DELETE FROM vault_shares WHERE item_id=$1 AND user_id=$2`, plus an audit event. And
removing a family member revokes all shares between the two users.

**That is access-control revocation, not cryptographic revocation** — and the distinction is the
whole answer. Bob already downloaded and decrypted the DEK; deleting a row doesn't reach into his
browser. He can decrypt any copy of the ciphertext he retained, forever.

**Worse, and this is the subtle one:** `encryptVaultItem` has a `dek?` parameter used specifically so
that **editing a shared item reuses the existing DEK** — that's deliberate, because re-generating
the DEK would invalidate every outstanding share and force a re-wrap. But it means a *revoked*
recipient who cached the DEK can still decrypt **all future versions** of the item. Revocation
doesn't even protect going forward.

**Correct design:** revocation must trigger **DEK rotation** — generate a fresh DEK, re-encrypt the
item, re-wrap for the KEK and every *remaining* recipient, and (critically) tell the user to change
the underlying password on the actual service, because the secret itself is what leaked. Every
serious manager surfaces this: "this password was shared; rotate it." Bitwarden and 1Password both
say the same thing.

**Also decorative today:** the `permissions` column stores `'read'`/`'write'`, but **no endpoint
lets a recipient write**, so the field is never enforced. Either implement it or drop it — an
unenforced permission field is worse than none because it implies a guarantee.

---

### Q6.4 Explain the family/friendship model and its race condition.

**Answer.** `family_memberships` is a single row per relationship — `(user_id = initiator,
friend_id = recipient)` with `status ∈ {pending, accepted}` and `initiated_by`. Composite PK plus
`CHECK (user_id != friend_id)`. All the read queries handle bidirectionality with
`WHERE user_id = $1 OR friend_id = $1` and a `CASE` in the JOIN to resolve "the other person."

Why one row instead of two mirrored rows: no dual-write consistency problem, and accept is a single
`UPDATE … WHERE user_id=$sender AND friend_id=$me AND status='pending'` — the predicate itself
enforces that only the *recipient* can accept. That's authorization expressed as a WHERE clause,
which I like: it can't be bypassed by a code path forgetting a check.

**⚠️ The race:** `SendRequest` does `GetMembership` (checks both directions) and then `CreateRequest`
— a check-then-act. If A and B send each other a request at the same moment, both reads return "no
relationship" and both inserts succeed, because the PK `(A,B)` and `(B,A)` are *different* rows. Now
there are two rows for one relationship and `DeleteMembership` (which deletes both directions) will
clean up inconsistently.

Fix: normalize the pair before insert — store `(LEAST(a,b), GREATEST(a,b))` so there is exactly one
possible row per pair and the PK makes the race impossible. That's the general lesson: **make the
invariant structural, don't defend it with a read.**

---

## 7. Security Deep Dive (the trap section)

### Q7.1 🔥 Do you have IDOR anywhere? Prove it.

**Answer.** No — and the reason is structural, not incidental.

The user ID **never comes from the request body or a path parameter.** `AuthMiddleware` validates the
session and passes a `domain.Session` value directly into the handler signature:

```go
func (m *AuthMiddleware) WithSession(next func(http.ResponseWriter, *http.Request, domain.Session)) http.HandlerFunc
```

That's the key design choice: an authenticated handler *cannot compile* without receiving the
session. Then every single vault/folder/version/audit query carries
`WHERE … owner_user_id = $ownerFromSession`, so an attacker guessing another user's `item_id` gets
`sql.ErrNoRows` → `domain.ErrNotFound` → **404, not 403**, which also avoids confirming the resource
exists. Sharing and family additionally check ownership + membership in the service layer before
mutating.

**How I'd make it even stronger:** Postgres **Row-Level Security** with the user ID in a session
GUC, so the database refuses cross-tenant reads even if application code forgets a predicate.
Defense that doesn't depend on every future developer remembering.

**⚠️ And what's missing:** there is **no authorization test suite**. The controls are right today,
but nothing stops a future refactor from silently dropping a predicate. The single highest-value
test I could add is a table-driven "user B cannot touch user A's resource" test across every
endpoint.

---

### Q7.2 🔥 Your `ReadJSON` has no body size limit. Walk me through the attack.

**Answer.** `util.ReadJSON` wraps `json.NewDecoder(r.Body)` with `DisallowUnknownFields()` but never
wraps the body in `http.MaxBytesReader`. So any endpoint accepts an unbounded body, and the decoder
allocates as it goes.

Concretely: `POST /vault/items/bulk` accepts up to 500 items each containing base64 ciphertext with
**no per-field length cap**. An authenticated attacker sends a few hundred MB and the server
allocates it in Go heap; a handful of concurrent requests OOM the process. And because there's **no
panic-recovery middleware**, a panic (OOM-adjacent, or any nil deref) takes down the whole server
process, not just the request.

Fixes, in order:
1. `r.Body = http.MaxBytesReader(w, r.Body, 1<<20)` in `ReadJSON`, with a larger cap on the bulk
   route specifically.
2. Per-field length validation on ciphertext/nonce/DEK — a nonce is *exactly* 24 bytes and a wrapped
   DEK is exactly 32+16; today `validateVaultPayload` only checks non-empty. **Length checks on
   crypto fields are free and should be exact.**
3. A `recover()` middleware returning a clean 500 and logging the stack.
4. `Server.MaxHeaderBytes` and `ReadHeaderTimeout` (I set Read/Write/Idle timeouts but not
   `ReadHeaderTimeout`, which is the Slowloris defense).

**Bonus point:** `DisallowUnknownFields` is a nice strictness default (catches client typos, blocks
mass-assignment-style surprises) but it makes the API **brittle to client/server version skew** — an
older server rejects a newer client's extra field outright. That's a conscious trade-off worth
stating.

---

### Q7.3 What security headers do you set, and what's missing?

**Answer.** Set: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`,
`Referrer-Policy: same-origin`.

⚠️ Missing and I'd add: **`Strict-Transport-Security`** (with `includeSubDomains; preload` — for a
credential app, downgrade protection is non-negotiable), a **Content-Security-Policy** (the API
serves JSON so a strict `default-src 'none'` is trivially safe; the SPA host needs a real one), and
`Cross-Origin-Opener-Policy` / `Cross-Origin-Embedder-Policy` if I wanted `SharedArrayBuffer` for
faster Argon2.

For a password manager, CSP on the *web app* is the highest-value one: XSS in a vault UI means
reading the decrypted vault out of memory, which no amount of server-side crypto prevents.

---

### Q7.4 🔥 XSS in your web app — how bad, and what saves you?

**Answer.** **Catastrophic, and that's the honest answer.** Zero-knowledge encryption protects
against a compromised *server*; it does nothing against compromised *client code*. Injected JS in
the vault tab can read the KEK out of React state and exfiltrate the whole decrypted vault. There is
no cryptographic mitigation — the defense is entirely "don't have XSS."

What I have: React escapes by default, and there is **no `dangerouslySetInnerHTML` anywhere** in the
codebase. Item URLs render as copyable text rather than anchors, so there's no `javascript:` sink.
`HttpOnly` protects the session cookie specifically.

What I'd add: strict CSP with nonces and no `unsafe-inline`, Subresource Integrity on any CDN
assets, a dependency audit in CI (`npm audit` / Dependabot — supply-chain XSS via a compromised npm
package is the realistic vector here), and idle auto-lock to shrink the window in which a KEK is
even in memory.

---

### Q7.5 🔥 The pepper is used for four different things. What happens if you rotate it?

**Answer — this is a great question and the answer is "everything breaks," which is itself the
finding.**

`AUTH_TOKEN_PEPPER` feeds:
1. `HashAuthVerifier` → stored password hashes
2. `HashToken` → session token hashes
3. `HashEmailCode` → reset/MFA codes
4. `DeriveTOTPEncryptionKey` → the key encrypting every TOTP secret at rest

Rotate it and: **every user's login breaks** (stored hash no longer matches), **every session is
invalidated**, every pending email code dies, and — worst — **every TOTP secret becomes permanently
undecryptable**, locking out every 2FA user with no recovery path. So the pepper is effectively
un-rotatable, which is a serious operational flaw for a secret that lives in an env var (and which
`render.yaml` even has `generateValue: true` for, meaning a Render service recreation could silently
generate a *new one* and brick the deployment).

**How it should be designed:**
- **Versioned peppers**: store a `pepper_version` alongside each hash, keep the old peppers around,
  and re-hash lazily on next successful login. That makes rotation a gradual, safe operation.
- **Separate the TOTP encryption key from the auth pepper entirely.** Mixing a
  *verification* secret with an *encryption* key is bad key management — an encryption key needs
  envelope-style rotation (re-encrypt secrets under a new key), a hashing pepper needs versioning.
  They have different lifecycles and shouldn't be the same value.
- Ideally both live in a KMS / secrets manager, not an env var.

I'd also note the deliberate good part: the code *does* domain-separate the four uses with distinct
string prefixes, so the derived values are unrelated even though they share an input.

---

### Q7.6 What's the threat model? Who are you defending against?

**Answer.** Explicitly:

| Adversary | Protected? | Why |
|---|---|---|
| Passive DB breach / stolen backup | ✅ mostly | Only ciphertext, peppered hashes, encrypted TOTP secrets. **But** plaintext metadata leaks which services you use. |
| Malicious/curious server operator (passive) | ✅ mostly | Never sees password, KEK, or plaintext. |
| Malicious server (**active**) | ❌ | Can serve weakened KDF params, ship malicious JS, or record verifiers. Mitigations: pinned client-side KDF floor, and — the real answer — signed/reproducible clients or a native app. Web crypto always trusts the code-server. |
| Network attacker | ✅ | TLS + HSTS(todo) + Secure cookies; payload is E2E encrypted anyway. |
| Online password guessing | ✅ | Three-layer throttling (IP / account / TOTP) + Argon2 cost on the client. |
| Offline cracking after DB breach | ✅ strong | Argon2id 64 MiB + per-user salt + server-held pepper. |
| XSS in the web client | ❌ | Fatal. Defense is CSP + no injection sinks + auto-lock. |
| Malicious browser extension / compromised endpoint | ❌ | Out of scope for any password manager. |
| Phishing site + our own extension autofill | ❌ ⚠️ | **Real gap** — see Q7.7. |

Being able to say clearly what you *don't* defend against is more convincing than claiming you
defend against everything.

---

### Q7.7 ⚠️ Tell me the worst bug in the whole project.

**Answer.** The browser extension's autofill has **no origin binding.**

`fillCredentials` in the content script writes whatever the popup hands it into the current page's
fields. The domain-matching function in the popup UI is only a *visual grouping* hint — every
credential is fillable via its Fill button on *any* open page. So a user sitting on `evil.com` can
be induced to fill their real GitHub password into an attacker's form.

Why that's the worst one: **phishing resistance is the single property a password manager exists to
provide that a human with a good memory doesn't already have.** A human can be fooled by
`github.co`; a manager that binds credentials to origins cannot. Breaking that inverts the product's
core value.

The fix: the **content script** — which is the only component that authoritatively knows the page
origin — must enforce a hard origin match against the stored URL as a *precondition* for fill, and
must refuse to fill passwords into non-secure (`http:`) contexts (excepting localhost). It's a
precondition, not a UI hint.

Adjacent extension issues in the same audit: `host_permissions: https://*/*` plus the `cookies`
permission means the extension could read cookies from every HTTPS site (wildly over-scoped); the
service worker's `vault-status` message returns the **raw KEK** to any in-extension caller instead
of keeping it and decrypting internally; and message handlers never validate `sender`. All of those
are "least privilege" failures.

---

### Q7.8 Any other traps in the codebase you'd flag?

**Answer — a rapid list; naming these unprompted is very strong:**

- **CSV export has no formula-injection neutralization.** Cells beginning `= + - @` execute when the
  exported file is opened in Excel/Sheets. Prefix with `'`. (A password-manager export containing a
  formula that exfiltrates the sheet is a real, published attack class.)
- **Clipboard is never auto-cleared** after copying a password. Competitors clear after 10–30s.
- **Dev mailer logs the full email body**, including the verification code, when `RESEND_API_KEY` is
  unset — fine locally, a credential leak into logs if it ever ran that way in prod.
- **`ValidatePasswordStrength` is dead code** on the ZK path — the server never sees the password, so
  strength enforcement is necessarily client-side only. Worth stating plainly: **in a
  zero-knowledge system the server cannot enforce password policy**, which is a genuine trade-off,
  not an oversight.
- **`/healthz` never pings the database** — reports healthy while Postgres is down, so orchestrators
  won't restart or drain a broken instance.
- **Logs are written to a rotating local file** (`lumberjack`) — on Render that disk is ephemeral, so
  the file sink is effectively write-only. Should be stdout-only + a log aggregator in prod.
- **No panic recovery middleware.**
- **`vault_attachments` and `backups_registry` tables exist but have no code paths** — schema
  written ahead of implementation. Dead schema is a maintenance smell.

---

## 8. Testing, Observability, Operations

### Q8.1 🔥 What's your test coverage and what would you write next?

**Answer — lead with the honest number.** Three test files, **auth only**:
`auth_crypto_test.go` (Argon2 round-trip, TOTP verify, TOTP secret encrypt/decrypt, token/UUID
helpers, otpauth URL), `auth_service_test.go` (register/login/logout/authenticate against a
hand-rolled mock repo), `auth_controller_test.go` (HTTP status + error-code mapping via
`httptest`). **Nothing** for vault, folders, sharing, family, audit, authorization, rate limiting, or
the SQL layer. Zero frontend tests.

The mocking approach is worth defending: mocks are plain structs with function fields satisfying
`domain.AuthRepository` — no gomock/testify codegen. That's idiomatic Go, keeps the test readable,
and only works *because* the service depends on an interface declared in `domain`. It's the payoff
of the architecture decision from Q1.1.

**What I'd write, in priority order:**
1. **Authorization/IDOR matrix** — table-driven: for every resource endpoint, user B gets 404/403 on
   user A's resource. Highest value per line of code in the whole project, because it locks in the
   one property that must never regress.
2. **Repository tests against a real Postgres** via `testcontainers-go` — my SQL is hand-written, so
   mocking the DB tests nothing. Especially the `FOR UPDATE` version-history path, which needs a
   *concurrent* test (two goroutines updating the same item, assert versions 2 and 3 with no lost
   snapshot).
3. **Crypto round-trip property tests** across web and extension, asserting a blob encrypted by one
   decrypts in the other — the two crypto modules are duplicated and *must* stay in sync (the audit
   already found a divergence: the extension lacks `unwrapVaultItemDek` and the DEK-reuse parameter,
   so editing a shared item from the extension would silently break shares).
4. **Rate limiter / throttle tests** with an injected clock — `loginThrottle` already takes a
   `now func() time.Time` specifically for this, so the seam exists and is unused.
5. Fuzz `ReadJSON` / base64 parsing.

---

### Q8.2 What does observability look like?

**Answer.** `log/slog` with a custom `multiHandler` fanning out to two sinks: stdout (text in dev,
JSON in prod) and a `lumberjack`-rotated JSON file (100 MB, 10 backups, 28 days, compressed). One
structured access-log line per request from `RequestLogger` middleware — method, path, client IP,
status (captured via a `responseRecorder` wrapping `http.ResponseWriter`), duration, user agent.
Application errors log with `slog.Any("error", err)` and the wrapped chain.

**⚠️ Gaps:** no request/correlation ID (so I can't tie an error to its access-log line), no metrics
at all (no Prometheus — I can't answer "what's my p99?" or "what's the login failure rate?"), no
tracing, and the `responseRecorder` doesn't implement `http.Flusher`/`http.Hijacker` so it would
break SSE or websockets if I added them. And logging to a local file on ephemeral container disk is
pointless.

Two things I'd add first: a request-ID middleware threaded through `slog` via context, and a
Prometheus counter on `auth_login_failed` — because a spike in failed logins is the single most
useful security alert this app could have.

---

### Q8.3 How is it deployed?

**Answer.** Multi-stage Docker: `golang:1.24-alpine` builder → `CGO_ENABLED=0` static binary →
`alpine:3.20` runtime as a **non-root user** (`adduser -D -u 1000 appuser`). Small image, small
attack surface. Locally, `docker-compose` brings up Postgres (with a `pg_isready` healthcheck and
`depends_on: condition: service_healthy`) plus the API. Prod is `render.yaml` (Docker service, env
vars, `PORT` handled by falling back `APP_PORT → PORT → 8080`), frontend on Vercel, Postgres managed.

**What I'd improve:** `FROM scratch` or `gcr.io/distroless/static` instead of Alpine — a static Go
binary needs no shell, and shipping one is free RCE-escalation surface. Pin the base image by
digest. Add a `HEALTHCHECK`. Move migrations out of app startup into a release step (Q4.6). And
`AUTH_TOKEN_PEPPER: generateValue: true` in `render.yaml` is dangerous given Q7.5 — it should be
set once, manually, and stored in a secret manager.

---

## 9. Go-Specific Questions They Might Drill

**Q: Why `defer tx.Rollback()` after you've already committed?**
`Rollback` on a committed transaction returns `sql.ErrTxDone`, which I discard. It's the idiomatic
"unconditional cleanup" pattern — you can't forget a rollback on an early-return error path.

**Q: What does `%w` do that `%v` doesn't?**
`%w` wraps, preserving the chain so `errors.Is` / `errors.As` can walk it. `%v` flattens to a
string and destroys the ability to match sentinels. My whole error-mapping strategy depends on `%w`
surviving three layers.

**Q: Why does `subtle.ConstantTimeCompare` matter, and what's its gotcha?**
`==` and `bytes.Equal` short-circuit on the first differing byte, leaking a prefix-match oracle via
timing. `ConstantTimeCompare` always reads both fully. **Gotcha:** it returns `0` immediately if the
lengths differ — so it leaks *length*, and you must never rely on it to compare secrets of variable
length. All my uses are fixed-size hashes/codes, which is why it's fine.

**Q: `context.Context` — where does it come from and what does it do here?**
It originates in `r.Context()` and threads all the way to `db.QueryContext`. When the client
disconnects, the context cancels and in-flight queries are cancelled too — that's real backpressure,
not decoration. It's also the correct place for deadlines. ⚠️ And it's exactly why the audit-log
write on a cancelled request gets dropped (Q5.4) — the same mechanism that helps you can bite you.

**Q: Interfaces — why are they in `domain` and not next to the implementation?**
Go idiom: **define interfaces where they're consumed, not where they're implemented.** The service
layer owns the contract it needs; the repository satisfies it implicitly (no `implements` keyword).
That inverts the dependency so `domain` has zero imports and everything points inward.

**Q: Pointers vs values — why do repositories return `domain.VaultItem` by value but services are
`*AuthService`?**
Data structs by value (immutable-ish, no nil checks, safe to copy across goroutines). Services by
pointer because they hold state (mutexes, config) and must be shared, not copied — copying a struct
containing a `sync.Mutex` is a bug `go vet` will flag.

**Q: How do you know `*sql.DB` is safe to share across goroutines?**
It's documented as safe for concurrent use — it's a pool, not a connection. That's why it's created
once in `main` and injected everywhere.

**Q: What's the risk in the `for … range` inside `CreateVaultItemsBulk`?**
None as written (Go 1.22+ gives per-iteration loop variables anyway), but note the loop reuses one
prepared statement and does an individual round trip per row. For 500 rows that's 500 round trips —
a `COPY` or a multi-row `INSERT … VALUES (…),(…)` would be dramatically faster. Correct, but not
fast.

---

## 10. "What Would You Do Differently?" — have three ready

**1. Encrypt the metadata and rebuild search client-side.** It's the one flaw that undercuts the
project's headline claim. Client-side fuzzy search over the decrypted in-memory vault covers this
app's scale; blind indexing (`HMAC(searchKey, term)`) covers it at scale.

**2. Replace hand-rolled sharing crypto with HPKE (RFC 9180).** Fixes the missing HKDF, gives
forward secrecy through ephemeral keys, and gives sender authentication — three of my four sharing
flaws in one library swap. General lesson: *use the standard construction; hand-rolling the
composition of primitives is where crypto bugs live, even when each primitive is correct.*

**3. Ship real recovery instead of vault-wipe.** Recovery-code-wrapped KEK — no data loss, still
zero-knowledge. It's the difference between a demo and something a family would actually trust.

Runners-up: authorization test matrix + `testcontainers` repo tests; Redis-backed rate limiting;
versioned migrations with advisory locking; RLS in Postgres; keyset pagination + delta sync.

---

## 11. Rapid-Fire

| Q | A |
|---|---|
| Why `BYTEA` not `TEXT` for ciphertext? | Binary is binary; base64 costs 33% storage and a conversion. Base64 only at the JSON boundary. |
| Why UUIDs not serial IDs? | Non-enumerable (no "how many users do you have?" oracle), client-generatable, merge-safe. Cost: 16 bytes and worse index locality — UUIDv7 would fix the locality. |
| Where are UUIDs generated? | Both: `util.NewUUID` hand-rolls v4 from `crypto/rand` with the version/variant bits set; `google/uuid` is used in the audit path. ⚠️ Inconsistent — pick one. |
| Why `TIMESTAMPTZ` everywhere? | Stores an absolute instant; `TIMESTAMP` silently drops the offset and is a bug factory. All app-side times are `.UTC()`. |
| Why is `/vault/kdf-params` public? | The client needs KDF params *before* it can authenticate. See Q2.6 for why that's also a risk. |
| Why 32-byte session tokens? | 256 bits of entropy — unguessable, and a fast hash at rest is fine because there's nothing to brute-force. |
| Why hex-encode tokens instead of base64? | URL/cookie-safe with no padding or `+/` escaping concerns. Costs 2× vs base64's 1.33×; irrelevant at 32 bytes. |
| Why `metadata JSONB` not `JSON`? | JSONB is parsed, deduplicated, and indexable (GIN). `JSON` is a text blob. |
| What's `algo_version` for? | Crypto agility — lets me migrate ciphertext to a new scheme by branching on the stored version rather than a flag day. |
| Biggest single-file complexity? | `auth_service.go` at ~670 lines. It's doing auth, MFA, email codes, and recovery. I'd split it into `auth`, `mfa`, and `recovery` services. |
| Why is `Register` not auto-login? | Keeps registration and session issuance separate; the client must derive the vault KEK anyway. Also a smaller blast radius if registration is abused. |
| Line count / scale? | ~8.2k lines of Go across 52 files, 13 tables, ~46 routes. |

---

## 12. Numbers Cheat Sheet

| Thing | Value |
|---|---|
| Argon2id | 64 MiB (`65536` KiB), t=3, p=2, 32-byte output |
| Vault salt | 32 bytes, random, per user |
| Auth salt | `SHA256("pmv2-auth-v1:" + lowercase(email))` |
| Auth verifier | 32 bytes → 64 hex chars |
| AEAD | XChaCha20-Poly1305, 32-byte key, 24-byte nonce |
| Session token | 32 bytes → 64 hex chars, TTL 720h (30d) |
| Session cleanup | hourly goroutine |
| TOTP | HMAC-SHA1, 20-byte secret, 6 digits, 30s, ±1 window |
| TOTP lockout | 5 fails / 30s window → 5 min (DB-backed) |
| Login lockout | 10 fails / 15 min window → 15 min (in-memory, per email) |
| IP rate limit | 5 req/s, burst 15, auth routes only; idle entries evicted after 3 min |
| Email codes | 6 digits, 10 min TTL, 5 attempts |
| Reset token | 15 min TTL, `purpose='password_reset'` |
| Bulk import cap | 500 items |
| HTTP timeouts | read 10s, write 15s, idle 60s, shutdown drain 10s |
| Audit page size | default 20, max 100 |

---

## 13. Questions to Ask *Them* (shows seniority)

- "How do you handle secret management and key rotation — KMS, Vault, env vars?"
- "What's your approach to authorization testing? Do you have a regression suite for tenant
  isolation, or is it reviewed per-PR?"
- "How much of your schema evolution is online migrations vs. maintenance windows?"
- "Where do you draw the line between defense-in-depth and complexity you have to maintain?"

---

## Appendix — Endpoint Map (for quick recall)

```
GET    /healthz

POST   /api/v1/auth/register                 (rate-limited)
POST   /api/v1/auth/login                    (rate-limited)
POST   /api/v1/auth/password-reset/request   (rate-limited)
POST   /api/v1/auth/password-reset/verify    (rate-limited)
POST   /api/v1/auth/password-reset/confirm   (rate-limited)
POST   /api/v1/auth/mfa/email/request        (rate-limited)
GET    /api/v1/auth/me                       ·  POST /auth/logout  ·  PUT /auth/profile
POST   /api/v1/auth/totp/{setup,enable,verify,disable}

GET    /api/v1/vault/kdf-params              (PUBLIC — no auth)
GET    /api/v1/vault/salt
POST   /api/v1/vault/items    ·  POST /vault/items/bulk
GET    /api/v1/vault/items    ·  GET  /vault/items/trash
GET    /api/v1/vault/items/{id}              ·  GET /vault/items/{id}/history
PUT    /api/v1/vault/items/{id}              ·  DELETE /vault/items/{id}
POST   /api/v1/vault/items/{id}/restore

GET    /api/v1/vault/shared   ·  GET /vault/shared/sent
POST   /api/v1/vault/items/{id}/shares       ·  GET /vault/items/{id}/shares
DELETE /api/v1/vault/items/{id}/shares/{user_id}

PUT    /api/v1/users/keys  ·  GET /users/keys  ·  GET /users/keys/lookup?email=
POST   /api/v1/folders     ·  GET/PUT/DELETE  /folders[/{id}]
POST   /api/v1/family/request[/accept|/reject]  ·  GET /family  ·  GET /family/requests
DELETE /api/v1/family/{user_id}
GET    /api/v1/audit  ·  GET /audit/summary  ·  DELETE /audit
```
