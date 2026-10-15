# Architecture

Eight diagrams, working from the outside in: what talks to what, how the workspace is
wired, which direction dependencies point inside the API, and what happens on the two
paths that are easy to get wrong — an authenticated request and a shutdown.

Prose lives elsewhere and is not repeated here. The reasoning behind the API layout is
[`adr/0001-layered-modules.md`](adr/0001-layered-modules.md); the error and logging
taxonomy is [`LOGGING_AND_ERROR_HANDLING.md`](LOGGING_AND_ERROR_HANDLING.md).

---

## 1. System context

Two deployables, one database, one hosted identity provider. Everything a browser sees
comes from the Next.js app; the API is never called from the browser directly, because
the token is fetched server-side.

```mermaid
flowchart LR
  browser["<b>Browser</b><br/>React 19 · Clerk widgets"]

  subgraph web["apps/web — Next.js 16"]
    direction TB
    proxy["proxy.ts<br/><i>route protection</i>"]
    rsc["Server Components<br/><i>app/dashboard</i>"]
    apiclient["lib/api-client.ts<br/><i>fetch + Zod parse</i>"]
    proxy --> rsc --> apiclient
  end

  subgraph api["apps/api — NestJS 11 · :5001"]
    direction TB
    guards["Global guards<br/><i>Throttler · Bearer · Roles</i>"]
    usersmod["modules/users"]
    hooks["POST /api/v1/webhooks/clerk"]
    guards --> usersmod
    hooks --> usersmod
  end

  pg[("PostgreSQL 17<br/>Drizzle · pg pool")]
  clerk{{"Clerk<br/><i>hosted identity</i>"}}

  browser -->|"HTTPS"| proxy
  apiclient -->|"GET /api/v1/users/me<br/>Authorization: Bearer"| guards
  usersmod -->|"SQL"| pg
  browser <-->|"sign-in · session cookie"| clerk
  apiclient -.->|"getToken()"| clerk
  guards -.->|"verifies session JWT"| clerk
  clerk ==>|"user.created / updated / deleted<br/><b>Svix-signed</b>"| hooks
```

`proxy.ts` is a redirect for humans, not the security boundary — the API authorizes every
request independently, so a matcher change in the proxy cannot silently expose data.

---

## 2. Workspace and build order

Solid edges are runtime dependencies, dashed edges are build and lint tooling. Turbo's
`build` task declares `dependsOn: ["^build"]`, so `packages/contracts` compiles before
either app — which is why `pnpm dev` must be run from the repo root and not with
`pnpm --filter api dev`.

```mermaid
flowchart TD
  web["<b>apps/web</b><br/>Next.js"]
  api["<b>apps/api</b><br/>NestJS"]

  contracts["<b>packages/contracts</b><br/>Zod schemas · framework-free"]
  ui["<b>packages/ui</b><br/>shadcn components"]

  eslintcfg["packages/eslint-config"]
  tscfg["packages/typescript-config"]

  web --> ui
  web -->|"UserSchema, parse responses"| contracts
  api -->|"createZodDto, validate requests"| contracts

  web -.-> eslintcfg
  web -.-> tscfg
  api -.-> eslintcfg
  api -.-> tscfg

  linkStyle 3,4,5,6 stroke-dasharray:4 4,opacity:0.55
```

`contracts` is deliberately framework-free — no Nest imports — because the browser bundle
imports it too. The API binds the same schemas as runtime DTOs through `nestjs-zod`; the
web client parses responses against them. Drift fails at the boundary rather than as an
`undefined` deep in a component.

---

## 3. Dependency direction inside `apps/api`

Three top-level areas, and imports only ever point downward.

```mermaid
flowchart TD
  subgraph modules["modules/ — business domains"]
    direction LR
    mu["users/"]
    mp["posts/ · billing/ · yours"]
  end

  subgraph core["core/ — app-wide infrastructure, no business rules"]
    direction LR
    cc["config/"]
    cd["database/"]
    ca["auth/"]
    co["observability/"]
    cw["webhooks/"]
  end

  subgraph shared["shared/ — domain-free helpers"]
    direction LR
    sf["filters/"]
    sh["http/"]
    st["types/"]
  end

  modules -->|"may import"| core
  core -->|"may import"| shared
  core -.->|"✗ core must not import @modules/*"| modules

  linkStyle 2 stroke:#b4442e,color:#b4442e
```

The forbidden edge is not a convention — it is a `no-restricted-imports` zone in
[`apps/api/eslint.config.mjs`](../apps/api/eslint.config.mjs), and
`pnpm --filter api lint` runs with `--max-warnings 0`, so a violation fails the build
rather than scrolling past.

When `core` genuinely needs something only a domain can provide, it declares a port and
the module implements it — the dependency still points downward. Figure 5 shows that
inversion for authentication.

---

## 4. Inside a module: ports and adapters

`modules/users` in full. The domain sits at the centre and depends on nothing; the
adapter on the right points _inward_ at the port, which is what makes the data source
swappable.

```mermaid
flowchart LR
  subgraph presentation["presentation/ — changes when the transport changes"]
    direction TB
    ctrl["UsersController<br/><i>@Roles, Swagger</i>"]
    sync["ClerkUserSyncController"]
    dto["dto/ — createZodDto wrappers"]
  end

  subgraph application["application/ — changes when a rule changes"]
    direction TB
    svc["UsersService<br/><i>use cases, entity → DTO</i>"]
    res["UserPrincipalResolver"]
  end

  subgraph domain["domain/ — plain types and ports"]
    direction TB
    ent["User · UserRole<br/><i>plain interface</i>"]
    port["UsersRepository<br/><b>abstract class = DI token</b>"]
  end

  subgraph infrastructure["infrastructure/ — changes when the database changes"]
    direction TB
    repo["DrizzleUsersRepository"]
    mapper["user.mapper.ts<br/><i>the only file that knows a column name</i>"]
  end

  schema["core/database/schema/users.ts"]
  pg[("PostgreSQL")]
  barrel["index.ts<br/><i>the module's only public surface</i>"]
  other["another module"]

  ctrl --> svc
  sync --> svc
  svc -->|"depends on the port"| port
  repo -.->|"implements"| port
  repo --> mapper --> schema --> pg
  other -->|"imports"| barrel
  barrel -.->|"exports UsersModule · UsersService · User types"| svc

  linkStyle 3 stroke:#1f6f5c,color:#1f6f5c
```

Bound in `users.module.ts`, and this line is the whole of the swap:

```ts
{ provide: UsersRepository, useClass: DrizzleUsersRepository }
```

Ports are **abstract classes, not interfaces**: an interface is erased at compile time and
cannot be a Nest DI token. For the same reason, never write `import type` on a port — it
erases the runtime value and DI fails at boot while typecheck stays green.

To place a new file, ask what would force it to change. A different database is
`infrastructure`; a different transport is `presentation`; a different rule is
`domain`/`application`. Two answers means it needs splitting.

---

## 5. Authentication: a guard that knows no provider

`BearerAuthGuard` contains no Clerk imports. It delegates to two ports, and both are
supplied from the composition root in `app.module.ts`. This is the inversion Figure 3
refers to: `core/auth` defines `PrincipalResolver`, and `modules/users` implements it, so
`core` still imports nothing from `modules`.

```mermaid
flowchart TB
  root["<b>app.module.ts</b> — composition root<br/><i>AuthModule.register({ verifier, resolver })</i>"]

  subgraph coreauth["core/auth"]
    direction TB
    guard["BearerAuthGuard<br/><i>APP_GUARD, fails closed</i>"]
    roles["RolesGuard<br/><i>@Roles('admin')</i>"]
    tv["TokenVerifier<br/><b>port</b>"]
    pr["PrincipalResolver<br/><b>port</b>"]
    guard --> tv
    guard --> pr
    guard -->|"runs after · reads request.user"| roles
  end

  subgraph strategies["core/auth/strategies"]
    direction TB
    composite["CompositeTokenVerifier<br/><i>dispatches on supports()</i>"]
    clerkv["ClerkTokenVerifier<br/><i>scheme: Bearer</i>"]
    apikey["ApiKeyTokenVerifier<br/><i>stub, worked example</i>"]
    composite --> clerkv
    composite --> apikey
  end

  upr["modules/users<br/>UserPrincipalResolver"]

  root -.->|"binds"| tv
  root -.->|"binds"| pr
  clerkv -.->|"implements — bound today"| tv
  composite -.->|"implements — when several coexist"| tv
  upr -.->|"implements"| pr

  pub["@Public() — the explicit opt-out"] -.-> guard

  linkStyle 7,8,9 stroke:#1f6f5c,color:#1f6f5c
```

A route is protected unless it says otherwise, so a newly added endpoint is safe by
default. Moving to Auth0 is a new `TokenVerifier` and a changed binding, not a rewrite.
Supporting session tokens _and_ machine API keys at once is `CompositeTokenVerifier`; the
guard still does not change.

---

## 6. An authenticated request, end to end

`GET /users/me` from the dashboard. The correlation id is the thread to follow — it is
minted or honoured on the way in, attached to every log line, echoed on the response, and
placed in the error body, so a user's screenshot maps to one request in the logs.

```mermaid
sequenceDiagram
  autonumber
  participant B as Browser
  participant P as proxy.ts
  participant S as Dashboard<br/>(Server Component)
  participant C as api-client
  participant N as Nest pipeline
  participant G as BearerAuthGuard
  participant V as ClerkTokenVerifier
  participant R as UserPrincipalResolver
  participant U as UsersService
  participant D as DrizzleUsersRepository
  participant PG as PostgreSQL

  B->>P: GET /dashboard
  P->>P: auth.protect() — redirect if signed out
  P->>S: render
  S->>C: getCurrentUser()
  C->>C: Clerk auth().getToken()
  C->>N: GET /api/v1/users/me<br/>Authorization: Bearer …

  N->>N: genReqId — honour or mint x-request-id
  N->>N: ThrottlerGuard
  N->>G: canActivate
  G->>G: @Public()? no
  G->>V: verify(credential)
  V-->>G: AuthClaims {subject, email, role}
  G->>R: resolve(claims)
  R->>U: ensureJitUser(...)
  U->>D: findByExternalId
  D->>PG: SELECT … WHERE clerk_id = $1 AND deleted_at IS NULL
  PG-->>D: row
  D-->>U: User (domain entity, never a Drizzle row)
  U-->>R: UserDto
  R-->>G: Principal
  G->>G: request.user = principal
  N->>U: UsersController.findMe(user.id)
  U-->>N: UserDto
  N-->>C: 200 · x-request-id
  C->>C: UserSchema.parse(body)
  C-->>S: UserDto
  S-->>B: rendered profile

  rect rgba(180, 68, 46, 0.08)
    Note over N,C: On failure, ProblemDetailsFilter returns<br/>application/problem+json carrying the same requestId.<br/>Internal messages never reach the client — they go to the log.
  end
```

Log level follows the response status rather than the exception class, and each failure
logs exactly once.

---

## 7. Two paths keep the local `users` row in sync

Clerk owns identity; a local `users` table exists so other tables have a foreign key to
point at. Two independent paths write it, and both converge on the same idempotent
operation, so neither is a single point of failure.

```mermaid
flowchart TB
  clerk{{"Clerk"}}

  subgraph webhookpath["Path A — webhook, eventually consistent"]
    direction TB
    svix["WebhookVerifier<br/><i>Svix signature over the raw body</i>"]
    ctrl["ClerkUserSyncController<br/><i>@Public()</i>"]
    map["clerk-event.mapper<br/><i>payload → JIT payload</i>"]
    svix --> ctrl --> map
  end

  subgraph jitpath["Path B — just-in-time, on the request itself"]
    direction TB
    guard["BearerAuthGuard"]
    resolver["UserPrincipalResolver"]
    guard --> resolver
  end

  jit["UsersService.ensureJitUser()<br/><i>idempotent — skips the write when nothing drifted</i>"]
  del["UsersService.removeByExternalId()<br/><i>soft delete</i>"]
  pg[("users table")]

  clerk ==>|"user.created / user.updated"| svix
  clerk ==>|"user.deleted"| svix
  clerk -.->|"session JWT on a normal request"| guard

  map --> jit
  map --> del
  resolver --> jit
  jit --> pg
  del --> pg
```

Two details that are load-bearing:

- The signature is verified over the **raw bytes**, which is why `main.ts` bootstraps with
  `rawBody: true`. Verifying a re-serialised body silently passes on some payloads and
  fails on others.
- The webhook path keeps a fallback for a missing email, while the request path
  **fails closed** and returns `401`. Clerk retries a non-2xx webhook response, so an
  emailless `user.created` that rejected would retry forever.

---

## 8. Graceful shutdown

The ordering here is the whole point, and it is not the order you would write by
accident. Nest runs its lifecycle hooks in a fixed sequence; putting a step in the wrong
hook fails every request that arrives during the drain.

```mermaid
sequenceDiagram
  autonumber
  participant K as Orchestrator
  participant A as Nest app
  participant H as HealthService
  participant S as HTTP server
  participant DB as pg pool

  K->>A: SIGTERM
  Note over A: 1 · onModuleDestroy()<br/>runs BEFORE the drain — nothing<br/>in-flight requests need may close here

  A->>H: 2 · beforeApplicationShutdown → markDraining()
  H-->>K: GET /health/ready now 503
  Note over A: hard deadline armed (SHUTDOWN_TIMEOUT_MS, 30s)<br/>force-exits if anything hangs

  A->>A: wait DRAIN_INTERVAL_MS (5s)<br/>orchestrator stops routing new traffic
  A->>S: closeIdleConnections()<br/>hangs up idle keep-alive sockets only

  A->>S: 3 · dispose() → server.close()
  S-->>A: in-flight requests finished
  A->>DB: 4 · onApplicationShutdown → pool.end()
  DB-->>A: closed
  A->>A: flush pino, then exit
```

- **Readiness fails first, liveness never does.** `/health/live` deliberately checks
  nothing but the process — if it pinged the database, a database blip would make
  Kubernetes restart every healthy pod.
- **The pool closes in step 4, not step 1.** `onModuleDestroy` runs before the drain
  window; closing the pool there would tear it down seconds before the server stops
  accepting traffic.
- **Idle sockets are hung up before `close()`.** `server.close()` waits for sockets to go
  idle on their own, and a load balancer holds keep-alive sockets open continuously — so
  without this, every ordinary deploy would end at the hard deadline with `exit(1)`.
- **`useProcessExit: true`.** Nest's default re-raises the signal via `process.kill()`,
  discarding whatever pino still holds in its transport worker — in practice the entire
  tail of the shutdown log, which is the part you need when a deploy misbehaves.
