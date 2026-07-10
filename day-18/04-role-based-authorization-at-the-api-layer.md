# Role-Based Authorization At The API Layer

> *Day 18: Trainer Dashboard Slice (Backend) — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

Every endpoint the cohort has shipped so far has been *authenticated* (the request carries a JWT identifying who made it) but not meaningfully *authorized* (any authenticated user can hit it). That's fine for candidate-scoped endpoints — the JWT subject is the user_id, the route reads "my own attempts," done. The trainer dashboard breaks that pattern. `GET /reports/aggregate` returns data *about other candidates* — pass rates, distributions, who's struggling. A candidate is not allowed to see that. So today the cohort installs the second half of the auth pipeline: **role-based authorization**, expressed as a FastAPI dependency that runs on every endpoint that needs it. The next topic (JWT claim verification) is what makes the role itself trustworthy; this topic is the *shape* of the gate.

## Authentication vs Authorization

A vocabulary the cohort will be tested on at interview:

- **Authentication (authN):** *Who* are you? Inherited PEP auth resolves this via JWT — the token's `sub` claim is the user_id.
- **Authorization (authZ):** *What* are you allowed to do? Role-based authorization decides this from a `role` claim plus the resource being accessed.

Authentication is a yes/no on identity. Authorization is a yes/no on the action *given* that identity. The two are conceptually separate; the implementation usually fuses them into one dependency chain.

## The Inherited Auth Surface

PEP's substrate already gives us:

- A user-service that issues JWTs at login. The claims include `sub` (user_id), `role` (`"candidate"` or `"trainer"`), `exp`, and `iat`.
- A `get_current_user` FastAPI dependency in each service that validates the JWT and returns a `User` model.

The cohort sees `get_current_user` already; today they add `require_trainer` on top.

## The Dependency Shape

The clean FastAPI pattern for "only trainers" is a layered dependency:

```python
# reporting-service/app/auth.py
from fastapi import Depends, HTTPException, status
from pydantic import BaseModel
from typing import Annotated, Literal

class User(BaseModel):
    id: str
    role: Literal["candidate", "trainer"]
    # ... whatever else is on the JWT

async def get_current_user(...) -> User:
    """JWT-validating dependency from D11; returns the authenticated user."""
    # JWT decoding + claim extraction lives in the next topic.
    ...

def require_trainer(user: Annotated[User, Depends(get_current_user)]) -> User:
    if user.role != "trainer":
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="trainer role required",
        )
    return user

TrainerDep = Annotated[User, Depends(require_trainer)]
```

Three things are doing useful work here:

1. **`require_trainer` *depends on* `get_current_user`.** Authentication happens first (resolves the user, or 401), then authorization checks the role (or 403). Same single chain on every protected route.
2. **The dependency returns the user.** It doesn't just gate; it provides the user object too, so the route handler doesn't need to call `get_current_user` separately.
3. **`TrainerDep` is a type alias.** Routes that need trainer-only access write `user: TrainerDep` as a parameter and that's the whole gate.

## Applying It To The Route

```python
# reporting-service/app/api/reports.py
from fastapi import APIRouter

router = APIRouter(prefix="/reports", tags=["reports"])

@router.get("/aggregate", response_model=AggregateReport)
async def aggregate(
    user: TrainerDep,
    params: AttemptListDep,
    db: DBSession,
):
    # `user` is guaranteed to be a trainer here; otherwise FastAPI already 403'd.
    return await reports_service.aggregate(db, params)

@router.get("/test/{test_id}", response_model=TestReport)
async def by_test(
    test_id: str,
    user: TrainerDep,
    db: DBSession,
):
    return await reports_service.for_test(db, test_id)
```

The route body assumes the user is a trainer — that assumption is correct because the dependency would have raised before the body runs. No `if user.role != "trainer"` inside the handler. The auth and the business logic are separated.

## 401 vs 403 — Get The Status Code Right

A surprisingly common bug: returning 401 when the right code is 403, or vice versa. The standard distinction:

- **401 Unauthorized** — the request was *not authenticated*. No token, expired token, malformed token. The client could retry after logging in.
- **403 Forbidden** — the request *was* authenticated, but the authenticated user is not allowed to do this. Logging in again won't help.

The HTTP RFCs are clear and the cohort should be too. A candidate hitting `/reports/aggregate` has a valid JWT — that's 403, not 401. The reverse: no JWT at all is 401, not 403. Mixing them confuses clients and breaks single-page-app patterns that automatically redirect to login on 401.

The `require_trainer` dependency above raises 403. The `get_current_user` dependency (next topic) raises 401 for missing/invalid tokens. Together the matrix is right.

## Granularity: Role, Permission, Or Resource?

PEP uses *role-based* authorization — the JWT carries a role string, the gate checks the role. This is simple and works for two-role systems (candidate, trainer). It's also one of three common patterns the cohort will encounter:

- **Role-based (RBAC).** Roles like `trainer`, `candidate`, `admin`. The user has one (or a small list of) roles; permissions are inherited from role. **PEP uses this.**
- **Permission-based.** The JWT carries a list of fine-grained permissions like `reports:read:aggregate`, `questions:write`. The gate checks the permission, not the role. More flexible; more setup.
- **Resource-based (ABAC).** "User X can edit resource Y if Y.owner_id == X.id". The gate depends on the *resource being accessed*, not just the user. PEP uses this implicitly for "candidate can read their own attempts but not others" — the SQL `WHERE user_id = current_user.id` enforces it.

The trainer dashboard is RBAC ("trainers only"). Resource-based authorization on top of RBAC ("trainers can only see their own assigned cohort") is a Phase 2 extension; for PEP, "any trainer sees everything" is acceptable.

## Multiple Roles, OR Vs AND

If three roles are added later (`candidate`, `trainer`, `admin`) and an endpoint allows trainers OR admins:

```python
def require_trainer_or_admin(user: Annotated[User, Depends(get_current_user)]) -> User:
    if user.role not in ("trainer", "admin"):
        raise HTTPException(403, "trainer or admin role required")
    return user
```

For a more reusable pattern, a factory function:

```python
def require_role(*allowed: str):
    def _checker(user: Annotated[User, Depends(get_current_user)]) -> User:
        if user.role not in allowed:
            raise HTTPException(403, f"one of {allowed} required, got {user.role!r}")
        return user
    return _checker

@router.get("/aggregate")
async def aggregate(
    user: Annotated[User, Depends(require_role("trainer", "admin"))],
    ...
):
    ...
```

This is the pattern most production FastAPI codebases settle into. PEP only needs `require_trainer` today, but flag the factory pattern as the next-step extension.

## Logging Authorization Decisions

When the dashboard goes live and a real audit happens, the trainer's manager will ask "who accessed the aggregate report yesterday?" If the access logs don't show user_id, the answer is "we don't know."

Log every authorization decision — success and failure — with `user_id` and `route`:

```python
import logging
log = logging.getLogger(__name__)

def require_trainer(user: Annotated[User, Depends(get_current_user)]) -> User:
    if user.role != "trainer":
        log.warning("authz_denied user=%s role=%s required=trainer", user.id, user.role)
        raise HTTPException(403, "trainer role required")
    log.info("authz_granted user=%s role=trainer", user.id)
    return user
```

The D10 distributed log correlation work makes this trivially aggregatable across services. The trainer's manager gets a clean answer; SREs get a signal when authorization denials spike (which is often the first symptom of someone trying to escalate privileges).

## Where The Gate Lives — Service-Level vs Route-Level

The cohort will ask "should I put the gate on every route, or at the router level?" Both work; the trade-off is:

- **Route-level (`Depends(require_trainer)` on each `@router.get`).** Verbose, but explicit. Each route's auth requirement is visible in its declaration. **Recommended for PEP.**
- **Router-level (`APIRouter(..., dependencies=[Depends(require_trainer)])`).** DRY, but the gate is invisible at the route. Someone adding a new route to the router gets the gate for free, which is good — until someone *removes* the gate at the router level and silently opens 14 routes.

A middle ground: route-level for sensitive endpoints, router-level for routers that are 100% trainer-only. The aggregate-reports router fits that case — every route under `/reports/test/{id}` and `/reports/aggregate` is trainer-only. PEP can use router-level for the aggregate router *only* and explicit route-level on mixed routers.

## Anti-Patterns

- **Checking the role inside the route body.** `if user.role != "trainer": raise HTTPException(...)` inside `async def aggregate(...)` works but spreads auth logic across the codebase, mixes it with business logic, and gets forgotten on new routes. Use a dependency.
- **Trusting a `role` field on the request body.** The client sending `{"role": "trainer"}` in the request body is not authoritative. The role lives on the *server-verified* JWT claim — next topic.
- **Using 401 for authorization failures.** Confuses clients; breaks SPA login-redirect logic. 401 = not authenticated; 403 = authenticated but forbidden.
- **Catching the 403 and returning 200 with an empty list.** Some cohorts do this to "be nice." Don't. Authorization failure is an error; the client needs to know.
- **Soft-deleting the gate during local dev (`if os.getenv("DEV"): return user`).** That env flag will absolutely ship to prod by accident. Use proper test fixtures (next topic) to bypass auth in tests; never an env flag in production code.
- **Forgetting to apply the gate to one route.** The most common bug in role-based auth is that the new endpoint added at 5pm Friday skipped the dependency. Tests (Topic 7) catch this; the router-level pattern catches it too.

## Key Takeaways

- Authentication identifies *who*; authorization decides *what they can do*. Two distinct gates in one dependency chain.
- The FastAPI pattern: `require_trainer = Depends(check_role(Depends(get_current_user)))` as a small dependency that returns the user.
- 401 for missing/invalid auth; 403 for valid auth without permission. Get this right.
- Default to route-level dependencies for explicitness; router-level only when *every* route in the router has the same gate.
- Log every authorization decision with user_id and route, so audits and anomaly detection have something to work with.
- The factory pattern (`require_role("trainer", "admin")`) generalizes when more roles arrive; PEP only needs the single role today.

---
*Prerequisites: [02-fastapi-routing-and-dependency-injection-patterns.md](../day-11/02-fastapi-routing-and-dependency-injection-patterns.md), [08-server-authoritative-state.md](../day-11/08-server-authoritative-state.md), [09-error-handling-and-http-status-code-discipline.md](../day-11/09-error-handling-and-http-status-code-discipline.md).*
