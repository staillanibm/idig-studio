# Task Management MCP Server — OAuth2-Secured (IDIG)

Governed MCP server exposing the Task Management API on IBM DataPower
Interact Gateway (IDIG) standalone, secured end-to-end with OAuth2
(Keycloak, realm `sttlab`).

**Security controls in this deployment:**

- 🔒 **Audience validation** — rejects tokens not intended for this server
- 🎯 **Per-tool scope enforcement** — `tasks:read` by default, `tasks:write` only for mutating tools
- 🔁 **Delegated token exchange (RFC 8693)** — no component holds more credential power than it needs
- 🚦 **Global quota + per-subscriber rate limiting** — the global quota protects the backend from abusive aggregate volume, the per-subscriber limit enforces fair usage across subscribers

Each policy below is documented with its exact configuration and **live
verification results**.

---

## 🧭 Assembly overview

```
1. ExtractIdentity (o1oK4)     OAuth2 bearer extraction
2. Authenticate    (gK5bK)     remote introspection + audience check
3. Switch          (dbs4W)     per-tool scope routing
     ├─ Authorize (write)      issuer check + tasks:write scope, for createtask / updatetask / deletetask
     └─ Authorize (read)       issuer check + tasks:read scope, everything else (default)
4. RateLimit                   per-subscriber
5. LuaScript                   RFC 8693 token exchange
6. Invoke                      real Task Management API
```

📄 `task-management-mcp-oauth-freeflowpolicysequence-s6hqi.yaml`

The global `Quota` isn't a step in this chain — it applies at the
`MCPServer` level, ahead of the whole assembly.

---

## 📐 Design rationale

The MCP Authorization spec expects servers to reject tokens not meant for
them, scope access to what each operation actually needs, and delegate
between services through proper token exchange rather than shared
credentials. This assembly maps directly onto those expectations:

| Best practice | Implementation | Verified |
|---|---|---|
| Reject tokens not intended for this server | `audClaim` on `Authenticate-gK5bK` | ✅ live negative test → `401` |
| Scope access per operation, not all-or-nothing | `Switch` + dual-`Authorize` | ✅ live read/write scope tests |
| Delegate via proper token exchange | RFC 8693, `LuaScript` package | ✅ full live flow, both scopes |
| Protect the backend from excessive aggregate volume | Global `Quota` (1000/min) | ✅ enforced at the MCPServer level |
| Enforce fair usage across subscribers | Per-subscriber `RateLimit` (50/min) | ✅ live rejection after threshold |

**Open by design** — flagged transparently rather than left implicit:

- **Delegation audit trail** — the token exchange does not currently populate an `act` (actor) claim on the exchanged token, so the backend cannot recover, from the JWT alone, which upstream caller a given request originated from. The IDIG assembly's OpenTelemetry traces (already emitted per request, see `Telemetry` policy) can reconstruct the full call chain independently of the JWT — this is a viable path to full delegation traceability without relying on the `act` claim
- **Client discovery** — no RFC 9728 protected-resource metadata (`/.well-known/oauth-protected-resource`) exposed yet, which some MCP clients use for auto-discovery

This design was informed in part by a review of IBM's published DataPower
Nano Gateway OAuth2/MCP guidance,
[*OAuth2 Concepts to Secure MCPServer*](https://community.ibm.com/community/user/blogs/kiruthiga-rajasekharan/2026/08/27/oauth-concepts-secure-mcpserver)
(K. Rajasekharan, IBM Community, 2026-08-27).

---

## 1️⃣ ExtractIdentity — `ExtractIdentity-o1oK4.yml`

Extracts the OAuth2 bearer token from the inbound `Authorization` header, so
downstream policies can act on it.

---

## 2️⃣ Authenticate — `Authenticate-gK5bK.yml`

```yaml
spec:
  operation:
    oauth2:
      providers:
        - sttlab-keycloak
      audClaim:
        - task-management-mcp-proxy
```

- Validates the token via **remote introspection** (RFC 7662) against
  Keycloak — every request triggers a live call to the authorization
  server; there is no local JWT/JWKS validation in this assembly
- Only tokens whose `aud` claim includes `task-management-mcp-proxy` are accepted
- A token that is otherwise valid — correctly signed, unexpired, same issuer — but minted for a different client is rejected with a bare `401`, before any other policy runs
- ⚠️ **`providers: [sttlab-keycloak]` is a reference to a pre-existing
  `OAuthProvider` custom resource on the gateway's cluster** — it is not
  created by this project or by any kind file here. It must already be
  provisioned out-of-band (by a cluster administrator, pointing at the
  Keycloak realm) before this policy can be published successfully. If you
  are replicating this setup on a different IDIG environment, confirm the
  correct provider name with whoever manages that cluster rather than
  reusing `sttlab-keycloak` as-is.

---

## 3️⃣ Switch — `Switch-dbs4W.yml`

A single `Authorize` policy checks one fixed set of scopes — it can't branch
per tool on its own. This `Switch` routes to one of two `Authorize` policies
before the request proceeds:

```yaml
spec:
  cases:
    - condition: >-
        $contains($telemetry('mcp_tool_name'), 'createtask-') or
        $contains($telemetry('mcp_tool_name'), 'updatetask-') or
        $contains($telemetry('mcp_tool_name'), 'deletetask-')
      execute:
        - $ref: task-management-mcp-oauth:Authorize-g15VO:1.0   # tasks:write
  otherwise:
    - $ref: task-management-mcp-oauth:Authorize-xjUvk:1.0       # tasks:read (default)
```

**Why `$telemetry('mcp_tool_name')` and not `$body('request')`:**

- At this point in the assembly, `$body('request')` resolves to just the tool's `arguments` (e.g. `{"pageSize":1}`) — the tool name is **not** in that body
- `$telemetry('mcp_tool_name')` reads the gateway-assigned tool name (e.g. `listtasks-7d5ac0a41481ae2e`) straight from the current span's OpenTelemetry attributes — stable, gateway-authoritative, independent of JSON-RPC body shape

---

## 4️⃣ Authorize (write / read) — `Authorize-g15VO.yml` / `Authorize-xjUvk.yml`

Both check the token's `iss` claim (issuer) against Keycloak, and a
required OAuth2 scope — `tasks:write` for the write-path policy,
`tasks:read` for the read-path (default) policy.

---

## 5️⃣ RateLimit + global Quota

Two independent, stacked limits — whichever is hit first applies:

| Limit | Policy | Threshold | Purpose |
|---|---|---|---|
| **Global quota** | `Quota` on `MCPServer.spec.quota` | 1000 requests / minute, across **all** subscribers | Protects the backend from excessive aggregate volume |
| **Per-subscriber rate limit** | `RateLimit` + `plan-limit` alias, ahead of the token exchange | 50 requests / minute, per subscriber | Fair usage — no single subscriber can starve the others |

- Rejection is reported as `isError: true` in the JSON-RPC `tools/call`
  result (`"Rate or count limit exceeded: 429 Too Many Requests"`), not as
  a transport-level HTTP 429 — check the response body, not just the
  status code, when testing this

---

## 6️⃣ LuaScript — token exchange (RFC 8693)

### Why

- The MCP client (an agentic layer upstream of IDIG) authenticates to Keycloak as its own client and gets a token scoped/audienced for *itself* — that audience is exactly what the MCP proxy's `Authenticate` policy checks for (`audClaim: task-management-mcp-proxy`, see above)
- That token is **not** directly usable against the backend, which expects audience `task-management-api` and scopes `tasks:read`/`tasks:write`
- A [RFC 8693 Token Exchange](https://datatracker.ietf.org/doc/html/rfc8693) call, authenticated as a separate broker client (`token-exchange-broker`), converts the inbound token into a backend-usable one
- The broker never sees the proxy's credentials; the proxy never holds a token valid against the backend directly
- A built-in OAuth2 token exchange policy is planned for DataPower Nano Gateway (targeted **Q4 2026**); until then, this project implements it as a self-contained, reusable `LuaScript` package

### Built-in policies used — kept to the bare minimum

| Policy | File | Role |
|---|---|---|
| `SetAuthorization` | `SetAuthorization-tokenexchange.yml` | Basic Auth for the *broker* client only, from secret `token-exchange-creds`. Targets the synthetic `exchange_request` message, never the real inbound request. |
| `Invoke` | `Invoke-tokenexchange.yml` | Single reusable POST to Keycloak's token endpoint. `input`/`output` are `exchange_request`/`exchange_response`, not `request`/`response`. |

Everything else — building the request body, driving both actions, parsing
the response, rewriting `Authorization` — lives entirely in the Lua
`source` of `LuaScript-tokenexchange.yml`. Nothing else in the assembly
needs to know a token exchange is happening.

### What the script does

1. Read `Authorization: Bearer <token>` from the inbound `request`
2. Extract the raw token
3. Build a form-urlencoded exchange request body:
   ```
   grant_type=urn:ietf:params:oauth:grant-type:token-exchange
   subject_token=<inbound token>
   subject_token_type=urn:ietf:params:oauth:token-type:access_token
   scope=task-management-api-audience tasks:read tasks:write
   ```
4. Write it to a new message, `exchange_request`
5. `context:run_action("set-auth-broker")` → Basic Auth for the broker
6. `context:run_action("invoke-keycloak")` → call Keycloak's token endpoint
7. Read `exchange_response`; fail loudly on non-200 or missing `access_token`
8. Overwrite `request`'s `Authorization` header with the exchanged token
9. Assembly continues to the real backend `Invoke`

---

## 7️⃣ Invoke — `invoke-s6hqi.yaml`

Calls the real Task Management API backend, now carrying the exchanged
token in `Authorization`.  

---

## 🔧 Keycloak configuration required

> Pre-existing infrastructure — nothing here is created by IDIG or this project.

| Object | Configuration |
|---|---|
| **`task-management-mcp-proxy`** *(client)* | Service account enabled · `standard.token.exchange.enabled=true` · client the agentic layer authenticates as |
| **`token-exchange-broker`** *(client)* | Service account enabled · `standard.token.exchange.enabled=true` (pre-existing) · client IDIG authenticates as |
| **`task-management-api`** | Not a Keycloak client — just an **audience value**, produced by a client scope mapper. The backend reads it from its own `JWT_AUDIENCE` config. |
| **Scope `token-exchange-broker-audience`** | `oidc-audience-mapper` → `token-exchange-broker`. **Optional** on the proxy. ⚠️ Must be explicitly requested at step 1 — omitting it makes Keycloak refuse the exchange outright, no matter what else is configured. |
| **Scope `task-management-api-audience`** | `oidc-audience-mapper` → `task-management-api`. **Default** on the broker. |
| **Scopes `tasks:read` / `tasks:write`** | **Optional** on the broker. ⚠️ Don't survive an exchange automatically — must be re-requested explicitly at step 2 (which the Lua script does). |

### The two-call flow

```
① Agentic layer → Keycloak   (client_credentials, as task-management-mcp-proxy)
   scope: tasks:read tasks:write token-exchange-broker-audience
   → token, aud includes task-management-mcp-proxy

② IDIG → Keycloak            (token-exchange, as token-exchange-broker)
   subject_token: <token from ①>
   scope: task-management-api-audience tasks:read tasks:write
   → token, aud = [task-management-api, account], scope = tasks:read tasks:write
   → used directly as Authorization: Bearer against the backend
```


### ⚠️ Common failure points

- Forgetting `token-exchange-broker-audience` at step ① → exchange refused outright
- Forgetting to re-request `tasks:read`/`tasks:write` at step ② → exchange succeeds, but the token has no usable scope against the backend
