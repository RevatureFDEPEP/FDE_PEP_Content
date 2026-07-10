# JWT Claim Verification And Role Enforcement

> *Day 18: Trainer Dashboard Slice (Backend) — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

The previous topic introduced the `require_trainer` dependency that gates the aggregate endpoints by checking `user.role == "trainer"`. That gate is only trustworthy if `user.role` itself is trustworthy — and the JWT claim is *only* trustworthy if you verify the token's signature and pull the role out of the *verified* payload, never out of the request body and never out of an unverified decode. This is the file where the cohort writes the actual JWT decode call. It is also the file where they internalize: a JWT without signature verification is a sticky note. With verification, it's a notarized affidavit. The difference is one line of code, and getting it wrong is the single most-cited backend security mistake in the industry.

## The Shape Of A JWT

A JWT is three base64url-encoded segments joined by dots:

```
<header>.<payload>.<signature>
```

- **Header** — JSON like `{"alg": "HS256", "typ": "JWT"}`. Identifies the signing algorithm.
- **Payload** — JSON of the claims, e.g. `{"sub": "u_42", "role": "trainer", "exp": 1747700000, "iat": 1747696400}`. Standard claims (`sub`, `exp`, `iat`, `aud`, `iss`) plus whatever custom claims the issuer added — `role` is custom but common.
- **Signature** — HMAC (for HS256) or RSA/ECDSA signature (for RS256/ES256) over `base64url(header) + "." + base64url(payload)`, using a secret (HS256) or private key (RS256).

The first two segments are *not encrypted* — base64 of JSON. Anyone with the token can read the role claim without the secret. What they *can't* do, without the secret/private key, is produce a valid signature for a *modified* payload. That's what verification checks.

## What "Verify" Actually Means

Three checks make up "verifying" a JWT:

1. **Signature check.** Recompute the signature using the server's secret/public key; compare against the token's signature segment. If they don't match, the token was tampered with (or signed by someone else); reject.
2. **Expiry check.** The `exp` claim is a unix timestamp; if `now > exp`, reject.
3. **Algorithm check.** The token's header declares the alg; verify it's one of the algorithms you accept. (The classic vulnerability: a library that trusts the header's `alg: "none"` and skips signature verification.)

PyJWT's `decode(token, key, algorithms=[...])` does all three. The `algorithms=[...]` argument is the **single most important parameter** — without it, PyJWT will trust the header's declared algorithm, including `"none"`, opening a critical vulnerability. *Always* pass `algorithms=["HS256"]` (or whatever the issuer actually uses).

## The Decode Call

PEP's user-service signs tokens with HS256 and a shared secret (the simpler PEP setup; production usually uses RS256 with public-key distribution). The reporting service decodes:

```python
# reporting-service/app/auth.py
import os
import jwt  # PyJWT
from fastapi import HTTPException, status, Depends
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from pydantic import BaseModel
from typing import Annotated, Literal

JWT_SECRET = os.environ["JWT_SECRET"]
JWT_ALGORITHM = "HS256"
JWT_ISSUER = "user-service"

bearer = HTTPBearer(auto_error=False)

class User(BaseModel):
    id: str
    role: Literal["candidate", "trainer"]

async def get_current_user(
    creds: Annotated[HTTPAuthorizationCredentials | None, Depends(bearer)],
) -> User:
    if creds is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="authentication required",
            headers={"WWW-Authenticate": "Bearer"},
        )

    token = creds.credentials
    try:
        payload = jwt.decode(
            token,
            JWT_SECRET,
            algorithms=[JWT_ALGORITHM],
            issuer=JWT_ISSUER,
            options={"require": ["exp", "iat", "sub", "role"]},
        )
    except jwt.ExpiredSignatureError:
        raise HTTPException(401, "token expired", headers={"WWW-Authenticate": "Bearer"})
    except jwt.InvalidTokenError as e:
        raise HTTPException(401, f"invalid token: {e}", headers={"WWW-Authenticate": "Bearer"})

    role = payload.get("role")
    if role not in ("candidate", "trainer"):
        # Token was valid but role is something unexpected. Treat as invalid.
        raise HTTPException(401, "invalid role claim")

    return User(id=payload["sub"], role=role)
```

Five details to drill on:

- **`algorithms=[JWT_ALGORITHM]` is mandatory.** Without it, PyJWT trusts the token's header — `alg: "none"` would bypass signature verification entirely. This is the alg-confusion attack; never make the library guess.
- **`HTTPBearer(auto_error=False)`** lets the dependency return `None` when no `Authorization: Bearer ...` header is present, so we can raise 401 explicitly with a useful `WWW-Authenticate` header. With `auto_error=True`, FastAPI raises a generic 403 (wrong code).
- **`options={"require": [...]}`** asserts that specific claims must exist in the payload. If `role` is missing, decode raises. Defense in depth: even if the issuer changes, the decoder fails fast.
- **`issuer=JWT_ISSUER`** validates the `iss` claim. Tokens signed by some other service won't decode here even if they happen to share the secret. (PEP signs all tokens from the user-service today; tomorrow when a second issuer exists, this catch matters.)
- **`role` validation after decode.** A signed token with `"role": "admin"` is a *valid token* — but our `User` model only permits two values. Reject explicitly with 401 rather than letting Pydantic raise a 500.

## What `jwt.decode` Catches For Free

PyJWT raises `InvalidTokenError` (or a more specific subclass) for any of:

- Malformed token (not three segments, not valid base64)
- Wrong signature
- Expired (`exp` in the past) — `ExpiredSignatureError`
- Not yet valid (`nbf` in the future) — `ImmatureSignatureError`
- Wrong issuer (when `issuer=...` is passed) — `InvalidIssuerError`
- Wrong audience (when `audience=...` is passed)
- Algorithm not in `algorithms` list — `InvalidAlgorithmError`

Catching the generic `InvalidTokenError` covers all of these. The cohort doesn't need to enumerate every subclass; one `except InvalidTokenError` is the right pattern unless a specific case needs different handling (e.g., expired might prompt the frontend to refresh, while invalid signature might log a security alert).

## Where The Secret Comes From

`JWT_SECRET` is loaded from an environment variable. In Compose, the `.env` file holds it; in ECS (PEP's AWS substrate), it's a secret stored in Secrets Manager and injected into the task environment. **Never** hardcode the secret in source; **never** commit it to git; **never** log the token in plaintext beyond debug builds.

A small but important property: the same `JWT_SECRET` value must be used by the user-service (signer) and reporting-service (verifier). In PEP's Compose stack today, both services share the env var — fine for a local cohort. In production this is brittle (rotate the secret and both services must redeploy together); the more grown-up answer is RS256 with the user-service holding the private key and other services downloading the public key from a JWKS endpoint. PEP defers JWKS to Phase 2; the cohort should know it exists.

## Plugging Into `require_trainer`

The previous topic's `require_trainer` calls `get_current_user`:

```python
def require_trainer(user: Annotated[User, Depends(get_current_user)]) -> User:
    if user.role != "trainer":
        raise HTTPException(403, "trainer role required")
    return user
```

That's the whole chain:

1. Request arrives with `Authorization: Bearer <jwt>`.
2. `HTTPBearer` extracts the credentials.
3. `get_current_user` decodes and verifies, raising 401 on failure.
4. `require_trainer` checks the role, raising 403 on failure.
5. Route body runs with a guaranteed-trainer `user`.

Every protected route reuses the same chain. That's the property of FastAPI dependencies that makes them worth using: one declaration on the route, one source of truth for the gate.

## The Reverse-Proxy Consideration

D6 set up an api-gateway reverse proxy in front of the services. The proxy doesn't decode JWTs (PEP's design); it passes the `Authorization` header through to the downstream service. So the reporting service performs the decode. Two architectural alternatives the cohort should be aware of:

- **Proxy decodes once, passes user_id and role as trusted headers** (e.g., `X-User-Id`, `X-User-Role`). Faster (one decode per request, not one per service hop), but downstream services must trust the proxy implicitly. Anyone who can hit the service directly (bypassing the proxy) can forge headers. Requires network-level lockdown.
- **Every service decodes independently.** Slower, but each service is self-contained — if the proxy is bypassed, the service still enforces auth. **PEP uses this.** Defense-in-depth (topic 5).

## Common Vulnerabilities To Avoid

This is one of two topics in PEP where I want every cohort member to walk away with a small mental checklist. The checklist:

- **`algorithms=[...]` is always passed.** No exceptions. The "alg: none" attack is real and ancient.
- **No `verify=False` ever in production code.** PyJWT supports it for debugging; it's a footgun.
- **Don't trust client-provided "role" outside the JWT.** Not in query strings, not in JSON bodies, not in custom headers.
- **Token is a Bearer credential.** Treat it like a password — never log it, never embed it in URLs (it leaks in referer headers and access logs).
- **Short expiry (15-60 min) + refresh token pattern.** PEP's user-service issues 60-min access tokens. Shorter means stolen tokens are less useful.
- **Rotate the signing secret on incident.** If the secret is ever exposed, every issued token is forgeable until it expires; rotate and force re-login.
- **Use HTTPS in production.** Bearer tokens over HTTP are passively interceptable. Compose dev is HTTP-only and that's fine; ECS termination is TLS.

## Test Fixtures (Preview Of Topic 7)

The tests in Topic 7 need a way to mint valid JWTs as different roles. The reverse of decode is sign:

```python
# tests/conftest.py
import jwt
from datetime import datetime, timezone, timedelta

JWT_SECRET = "test-secret"  # overrides app's secret in test env

def make_token(user_id: str, role: str, *, expired: bool = False) -> str:
    now = datetime.now(timezone.utc)
    exp = now - timedelta(minutes=1) if expired else now + timedelta(minutes=30)
    payload = {
        "sub": user_id,
        "role": role,
        "iss": "user-service",
        "iat": int(now.timestamp()),
        "exp": int(exp.timestamp()),
    }
    return jwt.encode(payload, JWT_SECRET, algorithm="HS256")
```

This is the helper Topic 7 builds the test matrix on top of.

## Anti-Patterns

- **`jwt.decode(token, options={"verify_signature": False})`** — convenience for poking at claims in dev; deadly in production. Cohort members who copy a Stack Overflow snippet will end up with this; teach them to recognize it.
- **`jwt.decode(token)` with no `algorithms`.** Trusts the header. Alg-confusion attack vector.
- **Parsing the role from a custom request header.** The header is client-controllable. The role *must* come from the verified payload.
- **Storing the JWT in `localStorage` and embedding the token in URLs.** Frontend concern but worth a note — JWTs belong in `Authorization` headers, ideally with an HttpOnly cookie if the threat model includes XSS.
- **Catching `Exception` around decode.** Hides real bugs (network errors, programming mistakes). Catch `InvalidTokenError` specifically.
- **Logging the full token on errors.** Tokens are credentials; they end up in log aggregators, dashboards, and accidentally in screenshots. Log the user_id (post-decode) or a token *hash* if you need correlation.
- **Skipping `exp` validation in a hotfix.** "We'll re-enable it next sprint" never happens.

## Key Takeaways

- A JWT is base64'd JSON plus a signature; the payload is not secret, the signature is what makes the payload trustworthy.
- "Verify" = signature + expiry + algorithm. PyJWT's `decode(token, key, algorithms=[...])` does all three; the `algorithms` list is non-negotiable.
- Always check `issuer=` (when applicable) and require the claims you depend on via `options={"require": [...]}`.
- 401 for missing/invalid/expired token (`WWW-Authenticate` header set); 403 only after a valid token is rejected on role.
- The secret comes from the environment, never from source. Defense in depth: every service decodes; never trust an upstream proxy alone.
- In tests, sign tokens with a known test secret in `conftest.py`; never disable verification.

---
*Prerequisites: [02-fastapi-routing-and-dependency-injection-patterns.md](../day-11/02-fastapi-routing-and-dependency-injection-patterns.md), [07-opaque-token-generation-and-session-identifiers.md](../day-11/07-opaque-token-generation-and-session-identifiers.md), [04-role-based-authorization-at-the-api-layer.md](04-role-based-authorization-at-the-api-layer.md).*
