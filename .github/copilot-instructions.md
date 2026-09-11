# Copilot instructions for Clodds

## Project overview

Clodds is a Node.js 22+ TypeScript application that exposes an AI trading terminal through a local HTTP/WebSocket gateway, WebChat, and many messaging adapters. It integrates prediction-market feeds, crypto/futures exchanges, Solana/EVM services, trading/risk controls, automation, and persistent conversation/trading state.

The production entry point is `src/index.ts`. It loads environment configuration, validates startup requirements, creates the gateway, and starts the services. `src/gateway/index.ts` is the main composition root: it wires the database and migrations, feeds, sessions, agents, channels, memory/embeddings, market index, automation, trading services, and HTTP/API routes. Follow that wiring when adding a service so lifecycle management and shutdown behavior remain consistent.

The main request path is:

`channel adapter or WebSocket -> authentication/pairing and rate limits -> session/context loading -> agent/tool routing -> response or trade execution -> channel delivery and persistence`

Market data is normalized behind `src/feeds/` and emitted through the feed manager. Trading requests pass through the execution/trading layers, where orderbook/slippage checks, risk and circuit-breaker guards, routing/MEV protections, platform signing, fill tracking, portfolio updates, and ledger/database persistence are coordinated. Do not bypass the execution or risk layers for a new trading capability.

Mutable runtime state is stored under `~/.clodds` by default (SQLite database, credentials, logs, backups, and config); `CLODDS_STATE_DIR`, `CLODDS_CONFIG_PATH`, and `CLODDS_WORKSPACE` can override the relevant paths. Most integrations are opt-in through configuration/environment gates. Optional channel/feed initialization is isolated so one unavailable integration should not prevent the rest of the gateway from starting.

There is also a Rust crate at `rust/fast-broadcast/`. It is a separately testable performance helper; the normal TypeScript build does not compile it.

## Commands

Install dependencies with Node.js 22:

```bash
npm install
```

The CI-equivalent install is:

```bash
npm ci
```

Common development commands:

```bash
npm run dev          # TypeScript entry point with tsx watch mode
npm run typecheck    # tsc --noEmit
npm run build        # TypeScript build plus bundled SKILL.md copying into dist
npm start            # Run dist/index.js after building
npm run gateway      # Run the gateway directly with tsx
npm run onboard      # Run the interactive setup wizard
npm run doctor       # Use the CLI doctor command after building/installing as appropriate
```

Run the full TypeScript test suite:

```bash
npm test
```

Run one test file directly (the repository test setup must be imported):

```bash
node --test --import tsx --import ./tests/helpers/test-setup.ts tests/unit/risk-guards.test.ts
```

Replace the final path with any test file under `tests/`. The integration tests named `market-index-live*.test.ts` call real external APIs and may need credentials/network access.

Run the complete local CI sequence:

```bash
npm run ci
```

Security checks used by CI:

```bash
npm audit
npx --yes audit-ci --config audit-ci.jsonc --moderate --production
```

For the Rust helper:

```bash
cd rust/fast-broadcast
cargo test
cargo test broadcast_race_test
```

There is no standalone lint script configured in `package.json`; use `npm run typecheck` and the CI commands above for the repository’s configured static validation.

## Architecture and extension points

### Channels and agents

Channel implementations live under `src/channels/<platform>/` and are created by `src/channels/index.ts`. Adapters implement the common lifecycle/message contract (`start`, `stop`, `sendMessage`, with optional edit/delete/reaction/poll/streaming support). Register a new adapter in the channel manager and pass the shared message, pairing, and command callbacks through it. Channel failures are intentionally isolated during initialization.

The agent manager in `src/agents/` owns Claude/provider interaction, tool registration, context compaction, memory access, session history, command/skill dispatch, and message handling. Keep new agent capabilities behind the existing tool or skill registries rather than embedding platform-specific behavior in the gateway.

### Feeds and execution

Feed integrations live under `src/feeds/<platform>/` and are assembled in `src/feeds/index.ts`. A feed should provide the common market lookup/orderbook/subscription behavior and emit normalized price/orderbook events. Add a feed to the manager and registry rather than having callers import a concrete platform implementation.

Execution and trading code is split across `src/execution/`, `src/trading/`, `src/risk/`, `src/portfolio/`, `src/ledger/`, and exchange/chain adapters. The expected trade flow is intent resolution, market and price discovery, risk validation, smart routing, signed order submission, fill confirmation, persistence, and monitoring. Preserve dry-run, max-order-size, slippage, circuit-breaker, and kill-switch behavior when extending order execution.

### Skills

There are two compatible skill mechanisms:

- Prompt skills use `src/skills/**/SKILL.md` with optional YAML frontmatter and are discovered by the skill loader. Workspace and user-managed skill directories override bundled skills.
- CLI handler skills live in `src/skills/bundled/<name>/index.ts`, export a default handler object, and are lazy-loaded. For a new bundled handler, add its directory to the alphabetized `SKILL_MANIFEST` in `src/skills/executor.ts` and add its command category in `src/commands/registry.ts`.

Handlers receive the text after the command prefix and return a `Promise<string>`. Use dynamic imports for optional backing modules so an unavailable dependency disables only that skill. If a skill requires environment variables, declare them through its existing `requires.env` contract.

### Persistence and configuration

Use the database abstractions in `src/db/` and parameterized SQL rather than creating ad hoc state files or raw database connections. Configuration is resolved through `src/utils/config`/`src/config`, supports environment variables and JSON5 config, and should be extended with typed defaults and validation rather than read directly throughout the application.

When adding a gateway service, wire its creation, start/stop lifecycle, configuration reload behavior if applicable, and shutdown cleanup through the gateway composition root. Respect the default loopback gateway and authentication/rate-limiting setup.

## Repository conventions

- TypeScript is compiled in strict mode with CommonJS output, declaration files, source maps, and `src` as the compiler root. Keep production imports compatible with this configuration.
- Prefer `async`/`await` and the existing typed interfaces (`Feed`, channel adapter, service interfaces, and shared types in `src/types.ts`) over platform-specific coupling.
- Optional integrations are commonly guarded by `enabled` flags and required credentials. Follow that pattern and report initialization failures through the existing logger; do not turn missing optional configuration into a process-wide startup failure.
- Skills and integrations may be loaded lazily because the project supports a broad dependency set. Avoid top-level imports of optional modules when a dynamic import and an isolated failure path are appropriate.
- Security-sensitive code must preserve the repository’s hardened patterns: use `execFile`/`execFileSync` with argument arrays instead of shell-interpolated commands, validate user-controlled paths and payloads, use parameterized SQL, and keep unsafe evaluation features disabled unless explicitly opted in.
- Keep credentials and mutable state out of the repository. Use `.env.example` for documenting configuration names and the credential manager/encryption settings for stored secrets.
- Tests use Node’s built-in test runner with `tsx`; `tests/helpers/test-setup.ts` sets quiet logging, and `tests/helpers/state.ts` provides temporary state directories for isolated database/state tests. Prefer deterministic unit tests and clearly isolate tests that call live APIs.
- When changing a skill, feed, channel, command, or service, update every relevant registry/manifest and its lifecycle wiring; a new directory alone is not necessarily discoverable.
- Keep changes focused and update the relevant docs when changing public commands, configuration, skill contracts, or API behavior. The most relevant references are `CONTRIBUTING.md`, `docs/ARCHITECTURE.md`, `docs/SKILLS.md`, `docs/USER_GUIDE.md`, and `docs/SECURITY_AUDIT.md`.
