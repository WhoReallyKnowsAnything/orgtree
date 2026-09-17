# External Integrations

**Analysis Date:** 2026-09-17

## APIs & External Services

**AI Coding Agent Providers (core product surface):**
- Claude Code (Anthropic) - invoked as a local CLI subprocess, not a direct HTTP API call; provider id `"claude"`, `engine/backend/orgtree/providers.py` (`claude_tiers`, `PROVIDER_LABEL`)
- OpenAI Codex - invoked as local CLI subprocess (`codex`/`codex.exe`/`codex.cmd`), provider id `"openai"`; resolution logic `engine/backend/orgtree/providers.py` (`codex_path`, `_codex_pin`, `codex_argv`), execution in `engine/backend/orgtree/codexrun.py`
- Google Antigravity (Gemini-based agent) - invoked as local CLI subprocess, provider id `"google"`; `engine/backend/orgtree/antigravityrun.py`, `antigravity_hook.py`, `antigravity_limits.py`, `antigravity_session.py`, `antigravity_provenance.py`
- OpenRouter - additional provider integration, `engine/backend/orgtree/openrouter.py`, `openrouter_limits.py`

Orgtree acts as an orchestrator/wrapper around these CLI tools (auth, rate-limit/quota tracking, account pooling, fallback routing) rather than calling their APIs directly for most flows. `apikey_accounts.py` handles API-key-based account auth as an alternative to CLI OAuth sessions. `agentauth.py`, `bridgeauth.py` manage auth flows.

**Rate limit / quota / account management:**
- `engine/backend/orgtree/account_fallback.py`, `accounts.py`, `accountusage.py`, `codex_limits.py`, `codex_route.py`, `limits.py`, `registry.py` - track per-provider/per-account usage pools ("reserve", "plan") and route work across multiple logged-in accounts per provider.

## Data Storage

**Databases:**
- No conventional SQL/NoSQL database detected. Engine persists state via its own store/schema modules: `engine/backend/orgtree/store.py`, `schema.py`, `registry.py`, `registry_migration.py` (likely local file-based/SQLite-style persistence — file format not confirmed by manifest scan, verify by reading `store.py` before building on it).

**File Storage:**
- Local filesystem only. `engine/backend/orgtree/artifact_downloads.py`, `filedelivery.py`, `disk.py` manage local file handling; `desktop_import.py`/`desktop_import_jobs.py` handle importing external data into the app.

**Caching:**
- Local in-process/on-disk caching: `engine/backend/orgtree/cachecontinuity.py`, `cachedeny.py`, `staffcache.py`, `warmpool.py` (warm process pool for agent sessions).

## Authentication & Identity

**Auth Provider:**
- No third-party identity provider (no OAuth-to-external-SSO, no Auth0/Clerk/Firebase Auth found). Authentication is per-AI-provider CLI login/session handoff, managed by `agentauth.py`, `bridgeauth.py`, `apikey_accounts.py`, `providerlogin.ts` (`apps/desktop/main/providerlogin.ts`).

## Monitoring & Observability

**Error Tracking:**
- No external error-tracking SaaS (Sentry, Bugsnag, etc.) detected. Internal crash/failure handling via `engine/backend/orgtree/crashreports.py`, `failclass.py`, `failfix.py`, and `apps/desktop/main/process-failure.ts`.

**Logs:**
- Local log/turn history recorded internally: `engine/backend/orgtree/turnlog.py`, `turnread.py`, `turnusage.py`, `history.py`, `transcript_ingest.py`, `transcript_records.py`.

## CI/CD & Deployment

**Hosting:**
- Distributed as a downloadable Windows desktop installer; no server hosting component detected for the app itself.

**Release/Update Channel:**
- GitHub Releases via `electron-updater`, configured in `package.json` (`build.publish`: provider `github`, owner `Maurdekye`, repo `orgtree`).
- Windows-specific release scripts: `tools/release-windows.mjs`, `tools/release-verification.mjs`, `tools/installer-upgrade.ps1`.
- No CI pipeline config (`.github/workflows`) found in the scanned tree — verify directly if CI matters for the port, this scan did not check `.github/`.

## Environment Configuration

**Required env vars:**
- None found hardcoded as required at repo root (no `.env*` present). Provider CLI credentials are stored/read through each CLI tool's own session storage (Claude Code/Codex/Antigravity native auth), surfaced to Orgtree via `agentauth.py`/`bridgeauth.py` rather than Orgtree-owned env vars.

**Secrets location:**
- Delegated to underlying provider CLIs' own credential stores; Orgtree's `apikey_accounts.py` manages any user-supplied API keys, storage location not confirmed in this scan (verify before handling secrets).

## Webhooks & Callbacks

**Incoming:**
- None detected (no public webhook receiver route found).

**Outgoing:**
- None beyond the electron-updater's polling checks against GitHub Releases and any outbound calls made by the provider CLIs themselves (which Orgtree does not directly initiate).

## macOS Port Notes (integrations-specific)

- All three primary "integrations" (Claude Code, Codex, Antigravity) are consumed by **spawning their CLIs as OS subprocesses** — the port risk is in subprocess spawning/path-resolution code (`providers.py` executable resolution, Windows `cmd.exe`/`.exe`/`.cmd` assumptions — see STACK.md Platform Requirements section), not in API contract differences. These CLIs themselves are cross-platform, so the integration logic should port cleanly once the subprocess-spawning layer is fixed for macOS (`os.name`, executable suffixes, `cmd.exe` wrapping in `providers.py`, `antigravityrun.py`).
- Mailhub-related docs exist (`docs/mailhub-migration.md`, `docs/mailhub-sync.md`, `engine/mailhub_runtime.py`, `engine/mailhub/`) suggesting an internal or external mail-delivery integration (`engine/backend/orgtree/maildrain.py`) — not fully traced in this pass; read `engine/mailhub_runtime.py` and `docs/mail-delivery-boundaries.md` directly if mail behavior is in scope for the port.

---

*Integration audit: 2026-09-17*
