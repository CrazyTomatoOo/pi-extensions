# pi-extensions

Personal collection of extensions for **pi** (`@earendil-works/pi-coding-agent`,
v0.83.0 installed). Each top-level directory is a **standalone extension** with
its own `package.json` — there is no root manifest, no workspace, no root build.

**The root IS a git repo** (`github.com/CrazyTomatoOo/pi-extensions`). Every
plugin directory is a **git submodule** — each has its own repo, own `package.json`,
own `origin`, and its own `AGENTS.md` where present. Work inside a plugin happens
in that submodule (commit & push there), then bump the superproject's submodule
pointer. There is no root manifest, no workspace, no root build. Clone recursively
with `git clone --recursive https://github.com/CrazyTomatoOo/pi-extensions.git`
(HTTPS URLs in `.gitmodules` — no SSH key needed).

## Directory Structure

| Dir | What it is |
| --- | --- |
| `pi-bailian-provider/` | 阿里云百炼 (DashScope) custom model provider — OpenAI-compatible. Registers `qwen3.7-max`, `qwen3.7-plus`, `glm-5.2` under provider id `bailian`. |
| `evolver-pi-plugin/` | Evolver self-evolving engine port — **local-only edition** (network layer removed). **Submodule** (`github.com/CrazyTomatoOo/evolver-pi-plugin`), own `AGENTS.md` (read it first when working there). `src/` (7 modules: `index`/`recall`/`signals`/`record`/`memory`/`filter`/`paths`), `scripts/self-check.ts`, `dogfood/` (Docker integration test), `skills/capability-evolver/`, `docs/agents/` + `docs/research/`. |
| `pi-powerline-status/` | Powerline-style status bar + Claude-Code-style startup header. One self-contained 30 KB `index.ts` (ported from pi-powerline-footer + pi-claude-code-tui; does NOT depend on them). |
| `pi-init/` | `/init` + `/init-deep` commands that scan a project and write/improve `AGENTS.md`. Prompt-only — the extension crafts the prompt, pi's agent does the scanning. |
| `pi-skill-manager/` | `/skills` TUI command to enable/disable skills via checkbox list; persists a disabled-list. |
| `pi-usage-query/` | `/usage` slash command that queries coding-plan usage across 百炼/Kimi/智谱/DeepSeek/GPT-Plus in one panel (`setWidget` below editor, auto-dismiss on next input). Env-var credentials, fails open. |
| `pi-dynamic-ctx-prune/` | **Fork of `@pi-vault/pi-dcp` v0.5.0** with forced compression (formerly `pi-dcp-enforcer`). Own `package.json`/`tsconfig`/`biome.json`/`vitest` (originally cloned verbatim — now its own git submodule: `github.com/CrazyTomatoOo/pi-dynamic-ctx-prune`). Unified `limits: { mode, min, max, block }` block — one set of thresholds for BOTH tiers (soft nudge reads min/max, hard `tool_call` block reads block), all read through the single `mode` (`"percent"` default → % of window, `"absolute"` → token count). No silent precedence (fixes the upstream bug where `DEFAULT_CONFIG`'s absolute `200000` shadowed `maxContextPercent`). `enforcer.escapeValve` (default 5) dead-loop guard. Removed dead `compress.nudgeForce`; folded the old scattered fields (`limitMode`/`compress.maxContextPercent`/`maxContextLimit`/`modelMaxLimits`/`enforcer.blockPercent`/`blockTokens`) into `limits`. Verify with `npm install && npx tsc --noEmit && npx vitest run`. |
| `.evolver/` | Just `workspace-id` (32-hex) — the evolver workspace id for THIS folder. Gitignored (machine-local, not committed). |

Every extension follows the same shape: `export default function (pi: ExtensionAPI)`
declared under `pi.extensions` in `package.json` (or implied by a single `index.ts`).

## Build / Test / Lint / Format

There is no build anywhere — **pi loads TypeScript directly via jiti**. No lint or
formatter is configured; match existing style by hand.

```bash
# Load any extension for a manual check (run from its directory or with a path):
pi -e ./pi-init/index.ts
pi -e .                        # evolver-pi-plugin (entry via package.json pi.extensions)

# evolver-pi-plugin ONLY (it has a tsconfig and a self-check; run in this order):
cd evolver-pi-plugin
npm install                    # typebox is a RUNTIME dep
npx tsc --noEmit               # type check — there is no compile step
bun scripts/self-check.ts      # logic self-check in a temp sandbox (or: npx tsx ...)

# Full integration test (real pi in Docker vs mock model, no network/keys):
docker build -f dogfood/Dockerfile -t evolver-dogfood . && docker run --rm evolver-dogfood
```

The other four extensions have no tsconfig and no tests — verification = load in pi
and exercise the command/UI.

## Architecture & Key Abstractions

All extensions wire into pi's extension API (`import type { ExtensionAPI } from
"@earendil-works/pi-coding-agent"`). Patterns used across the repo:

| `pi.registerCommand` | pi-init (`init`, `init-deep`), pi-skill-manager (`skills`), pi-usage-query (`usage`) |
| `pi.registerProvider` | pi-bailian-provider |
| `pi.on("session_start")` | evolver (recall injection), pi-powerline-status (header), pi-skill-manager |
| `pi.on("tool_result")` | evolver (signal detection on `write`/`edit`/`replace`) |
| `pi.on("session_shutdown")` | evolver (record git-diff outcome), pi-powerline-status |
| `pi.on("before_agent_start" / "input")` | pi-skill-manager (filter disabled skills), pi-powerline-status, pi-usage-query (dismiss usage panel on next non-`/usage` input) |
Evolver internals (see `evolver-pi-plugin/AGENTS.md` for the full map):
`src/index.ts` wires events → `recall.ts` / `signals.ts` / `record.ts`;
`paths.ts` (forge-resistant `.evolver/workspace-id`), `memory.ts` + `filter.ts`
(JSONL graph I/O). **Local-only** — no Proxy/tools/commands; everything
fails open — handlers never throw.

## Configuration & Environment

| Variable | Extension | Purpose |
| --- | --- | --- |
| `DASHSCOPE_API_KEY` | pi-bailian-provider | Bearer key for DashScope (or `/login bailian`). Models are auth-gated — no key, no models in `/model`. |
| `EVOLVER_WORKSPACE_ID` | evolver | Override workspace scoping id. |
| `EVOLVER_SESSION_STATE_DIR` | evolver | Throttle/dedupe state (default `~/.evolver`). |
| `EVOLVER_HOOK_LOG_DIR` | evolver | Evolution breadcrumb log (default `~/.evolver/logs`). |
| `POWERLINE_NERD_FONTS` | pi-powerline-status | `1`/`0` forces Nerd Font icons on/off (auto-detects Ghostty / `TERM_PROGRAM` otherwise). |
| `PI_CODING_AGENT_DIR` | pi-powerline-status, pi-skill-manager | pi agent dir override (default `~/.pi/agent`). |
| `BAILIAN_COOKIE` / `KIMI_API_KEY` / `ZHIPU_API_KEY` / `DEEPSEEK_API_KEY` / `CODEX_OAUTH_TOKEN` | pi-usage-query | Optional env-var overrides. Credentials also storable via `/usage login <provider>` into `credentials.json` (chmod 600). 智谱 key sent **without** Bearer prefix. Optional `*_BASE_URL` overrides.
| `CHATGPT_ACCOUNT_ID` | pi-usage-query | Sent as `ChatGPT-Account-Id` for Codex `wham/usage`. If unset, read from `~/.codex/auth.json` along with the OAuth access token (auto — no paste).
| `PI_USAGE_QUERY_NO_OPEN` | pi-usage-query | `1` disables browser-opening in `/usage login` (headless/test). ChatGPT calls go through pi's global proxy dispatcher when `http(s)_proxy` is set.

## Conventions & Gotchas

- **Style: tabs for indentation, double quotes** (matches pi's own codebase).
  Keep imports **extensionless** and node built-ins via `node:*` — jiti resolves
  them; explicit `.ts` extensions and bare built-ins break loading.
- **`pi-bailian-provider/` requires its own `npm install`** and reaches into
  `./node_modules/@earendil-works/pi-ai/dist/api/openai-completions.lazy.js`
  directly — pi's jiti can't resolve the `pi-ai/api/*` subpath export, and
  `openAICompletionsApi` isn't re-exported from the package root. `pi-ai` is
  pinned to pi's bundled version (0.83.0); bump both together.
- **Evolver's `memory_graph.jsonl` record shape is a hard external contract** —
  the `@evomap/evolver` engine and Claude/Cursor sibling plugins read it
  byte-compatibly. Never rename fields. Signal keywords, the recall filter, and
  workspace-id forging are ported **verbatim** from
  `EvoMap/evolver-claude-code-plugin` — match the reference, don't "improve" it.
- **`pi-skill-manager/state.json` in this repo is not the live state.** The real one
  is `$PI_CODING_AGENT_DIR/extensions/pi-skill-manager/state.json`, shaped
  `{ "disabled": string[] }`, keyed by skill path.
- `// ponytail:` comments mark deliberate simplifications with a named ceiling
  (e.g. approximate pricing in `pi-bailian-provider/index.ts`) — keep them when editing.
- DashScope quirks handled in `pi-bailian-provider/index.ts`: rejects OpenAI's `developer`
  role (`supportsDeveloperRole: false`); Qwen uses top-level `enable_thinking`;
  GLM-5.2 `reasoning_effort` only accepts `high`/`max` (lower levels map up).

## CI/CD & Deployment

No CI anywhere in this repo. The closest thing is evolver's dogfood Docker test
(above) — run it before publishing. Extensions are distributed via
`pi install github:<owner>/<repo>` (evolver) or manual copy to
`~/.pi/agent/extensions/<name>/` then `/reload`.

## References

- `evolver-pi-plugin/AGENTS.md` — authoritative for that subproject (architecture table, record-shape contract, dogfood). Read it before touching anything under `evolver-pi-plugin/`.
- `evolver-pi-plugin/README.md` — install, the three automatic behaviors, env vars.
- `evolver-pi-plugin/docs/agents/` — issue-tracker (`gh` CLI), triage-labels, domain-doc conventions for that repo.
- `evolver-pi-plugin/docs/research/` — port research: `reference-internals.md` (upstream plugin internals), `pi-api-mapping.md` (pi API mapping).
- `pi-init/README.md` — install + usage of `/init` and `/init-deep`.
- `pi-powerline-status/README.md` / `pi-bailian-provider/README.md` / `pi-skill-manager/README.md` — install, usage, env vars per plugin.
- Header comments of `pi-bailian-provider/index.ts` and `pi-powerline-status/index.ts` — auth/region notes and port provenance respectively.

## Agent skills

### Issue tracker

Issues for this collection live as markdown files under `.scratch/`. See `docs/agents/issue-tracker.md`.

### Triage labels

The five canonical roles map to the default label strings (`needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`). See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: `CONTEXT.md` + `docs/adr/` at the repo root. See `docs/agents/domain.md`.
