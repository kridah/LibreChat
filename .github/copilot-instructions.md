# LibreChat Copilot Instructions

## Architecture
- Full-stack app with Express backend (`api/server/index.js`) and Vite/React frontend (`client/src/App.jsx`) plus shared packages under `packages/` (`api`, `client`, `data-provider`, `data-schemas`).
- Backend bootstraps `@librechat/api` + `@librechat/data-schemas` helpers and uses `module-alias` so `~/*` maps to `api/*`; keep that import style when moving code.
- App configuration flows from `librechat.yaml` (or `CONFIG_PATH`) via `api/server/services/Config`; edits must stay schema-compliant with `librechat-data-provider`'s `configSchema`.
- LLM adapters + tools live under `api/app/clients`; reuse `BaseClient` for new providers and expose curated tools from `api/app/clients/tools/structured` or MCP definitions.

## Backend (`api/`)
- Routes live in `api/server/routes/*`, wired via `routes/index.js` with `requireJwtAuth` + config/limiters; keep persistence/business logic split (`~/models`, `~/server/services/*`, `packages/api`) and remember schemas + RBAC seeds live in `@librechat/data-schemas`.
- `getAppConfig()` merges YAML, structured tools, and MCP servers, then caches via Keyv (`cache/getLogStores`); refresh with `clearAppConfigCache` after config-changing migrations.
- Search endpoints rely on Meilisearch (`Message.meiliSearch`); ensure `MEILI_HOST` + index sync run (`indexSync()` in `db/index.js`) before querying.
- File uploads route through `api/server/routes/files`, `createMulterInstance`, and limiter middleware; update both the strategy (`initializeFileStorage`) and `librechat.yaml -> fileStrategy/fileConfig` when supporting new file types.
- Auth uses Passport strategies in `api/strategies` and social login wiring in `server/socialLogins`; set `ALLOW_SOCIAL_LOGIN`/LDAP envs instead of editing auth flows directly.

## Frontend (`client/`)
- Entry `client/src/App.jsx` composes `QueryClientProvider`, `RecoilRoot`, `ThemeProvider` (from `@librechat/client`), `LiveAnnouncer`, and `RouterProvider`; wrap new providers here if they must apply globally.
- Routing is centralized in `client/src/routes/index.tsx` (Auth layouts, chat `/c/:conversationId`, `agents`, `share/:id`); honor the `<base>` tag handling when adding routes.
- State: React Query hooks in `client/src/data-provider/*` manage server fetches (`librechat-data-provider` + `QueryKeys`), while Recoil/context atoms (`client/src/store`, `client/src/Providers`) hold UI state—reuse the provided persistence helpers.
- Styling uses Tailwind classes backed by CSS vars; source colors from `packages/client/src/theme` / `client/src/style.css` so runtime theming works, and keep components accessible (ARIA hooks exist in `client/src/a11y`).
- Localization pulls from `client/src/locales`; only edit `en/translation.json` manually and use `config/translations` scripts + Locize for other languages.

## Shared Packages & Config
- `packages/data-provider` defines types, config schemas, QueryKeys, `Time` constants, and REST helpers; update it whenever API shapes or config options change, then rebuild via `npm run build:data-provider`.
- `packages/api` centralizes MCP, agents, tool execution, storage backends, caching, and OAuth reconnect helpers—backend code should import from here instead of reimplementing.
- `packages/data-schemas` hosts mongoose schemas, logger, AppService builders, and seeding routines; schema changes belong here to stay reusable across services.
- `packages/client` exposes reusable UI primitives (ThemeProvider, Toast, etc.) used by the app; align new UI patterns with this package before duplicating components in `client/`.
- `librechat.example.yaml` shows every configurable feature (endpoints, agents, limits, speech, files); keep comments synced with actual behavior whenever you add config knobs.

## Data Stores & Infra
- MongoDB (`MONGO_URI`) stores conversations/presets (indexes via `db/indexSync`), while Meilisearch powers search (`MEILI_HOST`/`MEILI_MASTER_KEY` in `docker-compose.yml`); Meili downtime simply disables search.
- Optional Redis caches/limiters are configured through `redis-config/` (cluster, TLS, plain); toggle with `USE_REDIS`, `REDIS_URI`, and rely on `@librechat/api` cache factories.
- RAG features talk to an external Postgres/pgvector service defined in `rag.yml` + env `RAG_API_URL`; backend hits it through `packages/api` endpoints, so mirror env defaults when deploying.
- File storage is pluggable (local/S3/Firebase) per type via `librechat.yaml -> fileStrategy`; ensure buckets/keys exist before switching strategies to avoid runtime failures.

## Workflows & Tooling
- Typical local dev: `npm run backend:dev` (nodemon Express) + `npm run frontend:dev` (Vite) after running `npm run build:data-provider && npm run build:data-schemas && npm run build:api` once; or run `npm run frontend` to build every package plus client.
- Docker path uses `docker compose up -d` (see `docker-compose.yml`) to launch API, Mongo, Meilisearch, pgvector RAG API, and binds `.env`/`librechat.yaml`; override via `docker-compose.override.yml` for custom configs.
- Upgrade/reset helper scripts sit under `config/` (`update.js`, `deployed-update.js`, `create-user.js`, migrations); they expect `.env` populated, Bun `b:*` commands mirror the Node ones, and logging should stay on Winston/`@librechat/data-schemas`.

- API tests run with `npm run test:api` (Jest + mongodb-memory-server), frontend with `npm run test:client` (Jest + RTL), packages via their `jest.config.*`, and Playwright E2E suites (`npm run e2e`, `e2e:a11y`, `e2e:headed`) expect the stack on `http://localhost:3080`; lint/format via `npm run lint`, `lint:fix`, `format`.
- When adding features, include regression tests where possible (RTK Query hooks, React components, or API services) and ensure QueryKeys invalidation is covered.

## Conventions & Gotchas
- Always add new routes to `api/server/routes/index.js` and expose matching client hooks/mutations; remember to update RBAC/permissions via `@librechat/data-schemas` if the endpoint is role-restricted.
- Use `Time` constants, Keyv caches (`getLogStores(CacheKeys.X)`), and helper middlewares (`createFileLimiters`, `uaParser`, `checkBan`) instead of rolling custom timers/guards.
- Respect base-path-aware URLs: server rewrites `<base>` when `DOMAIN_CLIENT` is set, and router uses it—avoid hard-coded `/` links in client components.
- Agents/Assistants + MCP features span backend (`api/server/routes/agents|assistants`, `packages/api/src/agents`, MCP registry) and frontend contexts/components (`client/src/Providers/AgentsContext`, `client/src/components/Agents`, `client/src/data-provider/mcp.ts`); keep both sides and `librechat.yaml -> mcpServers` in sync.
- File uploads and speech endpoints enforce strict limits; adjust both middleware and config if you raise limits, and note that `/api/files/speech` bypasses some limiters and must stay earliest in the stack.
