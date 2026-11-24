# Security and fitness review: harvest-mcp-server v0.1.9

Review of this fork (HEAD `0cbcd45`, = upstream `ianaleck/harvest-mcp-server` master) before installing it with a
Harvest personal access token. Reviewed 2026-11-24; no real token used, nothing installed globally.

## Verdict: install after changes

The code is clean: one outbound host, no telemetry, token read from env and never logged or echoed. Before it gets a
real token:

| # | Change | Why | Effort |
|---|---|---|---|
| 1 | `npm audit fix` (axios -> >=1.20, follow-redirects -> >=1.16.2, MCP SDK -> >=1.32) and rebuild | Production deps carry 11 advisories, two of them in the code path that holds the token (see Dependencies) | Minutes |
| 2 | Gate the 30 write tools: Claude Code `ask` rules now; a `HARVEST_READ_ONLY` flag later | Michael wants free reads, approved writes; the server has no read-only mode | Minutes (rules) |
| 3 | Make `npm run build` work on Windows (drop `chmod` or make it non-fatal) | Build fails under npm's default `cmd.exe` shell | One line |
| 4 | Run over **stdio only**; never set `MCP_TRANSPORT=http` | HTTP mode listens on `0.0.0.0:3000` with no auth unless `MCP_API_KEY` is set | Config |

## 1. Network

| Host | Where | Notes |
|---|---|---|
| `api.harvestapp.com` (HTTPS) | `src/config/index.ts:6`, used at `src/client/base-client.ts:29` | The only outbound host |

- No telemetry, analytics, update checks or other hosts anywhere in `src/` (grep for URLs, `fetch`, `http`/`https`/`net`).
- No OAuth; `id.getharvest.com` is never contacted.
- `HARVEST_API_BASE_URL` is parsed (`src/config/index.ts:20`) but **never used**: `src/server.ts:42-45` passes only
  token + account ID, so `base-client.ts:29` always falls back to the constant. Good: the env var cannot redirect the
  token elsewhere. (README and `.env.example` imply it works; it doesn't.)
- axios honours `HTTPS_PROXY`/`HTTP_PROXY` env vars (verified: a proxy var rerouted requests). Keep those unset in the
  server's environment, or the bearer token goes through the proxy.
- Inbound: only in HTTP mode (`src/server.ts:187-236`): Express on `0.0.0.0`, port 3000, API key optional and compared
  non-constant-time (`:199`). stdio mode (the default, `src/index.ts:26-33`) opens no ports.

## 2. Token handling

| Aspect | Finding | Ref |
|---|---|---|
| Source | Env vars `HARVEST_ACCESS_TOKEN`, `HARVEST_ACCOUNT_ID`, validated by zod at module load | `src/config/index.ts:11-12,36` |
| `.env` files | Not read. `dotenv` is a dependency but never imported (dead dep) | `package.json`, `src/config/index.ts:3` |
| Sent to | Only `api.harvestapp.com`, as `Authorization: Bearer` + `Harvest-Account-Id` headers | `src/client/base-client.ts:31-36` |
| Logged | No. Logs go to stderr (winston console); request logs print method + path only | `src/utils/logger.ts:15-22`, `base-client.ts:47,59` |
| Errors / tool output | No. Interceptor rethrows fresh `Error`s carrying status text only, so the axios config (with headers) never reaches tool output | `base-client.ts:62-101`, `src/utils/errors.ts:91-109` |
| Written to disk | No `fs` use in `src/` | grep |

- **Canary test (verified):** ran with `HARVEST_ACCESS_TOKEN=SECRETCANARY123`, `LOG_LEVEL=debug` and a dead local proxy
  to force errors on `get_company` and `list_time_entries`. The canary appeared 0 times in stdout (tool results) and
  0 times in stderr.
- Residual risk: `base-client.ts:51` logs a raw request-setup error object. It is unreachable in practice (no request
  interceptor can fail before it), but if hit, winston could serialise the axios config including the header.
- stderr is captured by the MCP host (Claude Code keeps MCP server logs on disk). It holds Harvest response bodies on
  errors (`base-client.ts:68-72`): business data, not the token.
- A Harvest PAT carries the full rights of its user; there are no scoped tokens. The tools can do whatever Michael's
  Harvest role allows.

## 3. Dependencies

| Runtime dep | Locked | Used for | Note |
|---|---|---|---|
| `@modelcontextprotocol/sdk` | 1.29.0 | MCP server | 6 advisories (<1.32.0); mostly HTTP transport / client-side OAuth, not stdio |
| `axios` | 1.18.1 | Harvest HTTP client | 12 advisories (<1.20.0): prototype-pollution gadgets, header injection |
| `follow-redirects` (via axios) | 1.16.0 | Redirects | 3 advisories: **credential headers can survive a cross-origin redirect**; Harvest would have to redirect off-host to trigger it |
| `express` | 5.2.1 | HTTP transport only | proxy-addr (critical), body-parser, qs advisories; irrelevant in stdio mode |
| `winston` | 3.19.0 | Logging | Fine |
| `zod` | 3.25.76 | Validation | 1 advisory, fix is zod 4 (breaking); low risk, inputs come from Claude |
| `dotenv` | 17.4.2 | Nothing | Unused; remove |

- `npm ci` (Node 26.13.0, npm 11.20.0): 31 advisories total (2 critical, 10 high, 19 moderate); **11 in production deps**
  (1 critical, 7 high, 3 moderate). All but zod fix non-breaking with `npm audit fix`. The lockfile is from the
  2026-07 "resolve security vulnerabilities" commit; advisories published since.
- Lifecycle scripts: none in this package. Two in devDeps: `esbuild` (postinstall downloads its platform binary) and
  `unrs-resolver` (postinstall). npm 11's `allowScripts` gate blocked both; the build and tests don't need them
  (`build.js`, the only esbuild user, is not wired to any script).
- All 607 lockfile packages resolve from `registry.npmjs.org`. Dev deps are mainstream (TypeScript, jest, eslint,
  prettier, ts-node). Nothing abandoned or unusual.

## 4. Write surface

54 tools: 24 read, **30 write**. No tool declares MCP `annotations` (`readOnlyHint`/`destructiveHint`). No read-only
flag or env var exists.

| Area | Write tools | File |
|---|---|---|
| Time entries | `create_time_entry`, `update_time_entry`, `delete_time_entry`, `start_timer`, `stop_timer`, `restart_timer` | `src/tools/time-entries.ts:214-335` |
| Projects | `create_project`, `update_project`, `delete_project`, `create_project_task_assignment`, `update_project_task_assignment`, `delete_project_task_assignment` | `src/tools/projects.ts:241-383` |
| Tasks | `create_task`, `update_task`, `delete_task` | `src/tools/tasks.ts:150-189` |
| Clients | `create_client`, `update_client`, `delete_client` | `src/tools/clients.ts:150-187` |
| Users | `create_user`, `update_user`, `delete_user` | `src/tools/users.ts:179-240` |
| Invoices | `create_invoice`, `update_invoice`, `delete_invoice` | `src/tools/invoices.ts:154-205` |
| Expenses | `create_expense`, `update_expense`, `delete_expense` | `src/tools/expenses.ts:175-220` |
| Estimates | `create_estimate`, `update_estimate`, `delete_estimate` | `src/tools/estimates.ts:153-200` |

Read tools: `get_company`, `list_*`/`get_*` per area, `get_current_user`, `list_expense_categories`, and the four
reports (`get_time_report`, `get_expense_report`, `get_project_budget_report`, `get_uninvoiced_report`).

IDs are validated as positive integers before being put in URL paths (e.g. `time-entries.ts:46`), so no path injection.

### Recommendations (not implemented; Michael to choose)

| Option | How | Pros | Cons |
|---|---|---|---|
| A. Claude Code `ask` rules (now) | `permissions.ask` in settings: one entry per write tool, e.g. `mcp__harvest__create_time_entry` (30 names); leave reads unlisted or in `allow` | No code change; prompts on every write with the arguments shown | Claude-specific; a new write tool from upstream is not covered until added |
| B. `HARVEST_READ_ONLY=true` flag (fork) | In `src/server.ts:89-94`, skip registering tools whose name starts `create_`/`update_`/`delete_`/`start_`/`stop_`/`restart_` | Hard guarantee, vendor-neutral, ~10 lines; good upstream PR | All-or-nothing: approved writes need a second server entry without the flag |
| C. MCP annotations (fork) | Add `annotations: { readOnlyHint, destructiveHint }` to each tool | Lets any MCP host apply its own read/write policy; good upstream PR | Advisory only; hosts decide what to do with it |

Suggested: A immediately; B + C as a fork change worth offering upstream. With A, list write tools by name rather than
using `mcp__harvest__*` in `ask`, so reads stay prompt-free.

## 5. Build and run

Node 26.13.0 via mise; added repo-local `mise.toml` pinning it (`engines` says `>=18`).

| Step | Result |
|---|---|
| `npm ci` | OK (scripts gated, see above) |
| `npx tsc --noEmit --skipLibCheck` | OK, no errors |
| `npm run build` from PowerShell | **Fails**: `tsc` succeeds and writes `dist/`, then `chmod +x` is not found (npm runs scripts via `cmd.exe`) -> exit 1 |
| `npm run build` from Git Bash | OK |
| `npm test` | 17 suites, 183 tests pass (mocked; no network) |
| Start over stdio, no token | Exits 1 at module load: `Environment validation failed: HARVEST_ACCESS_TOKEN: Required ...` on stderr, uncaught-throw stack trace, 0 bytes on stdout. Safe; not pretty |
| Start over stdio, dummy token | `initialize` OK (`serverInfo` reports version `0.1.0`, stale), `tools/list` returns 54 tools, no network traffic at startup |

Fix for the build: `"build": "tsc --skipLibCheck"` (the shebang in `src/index.ts:1` is enough on Windows; `chmod` only
matters for a global npm bin on Unix), or `node -e "fs.chmodSync('dist/index.js',0o755)"`.

## 6. Upstream health

| Item | State |
|---|---|
| Repo | `ianaleck/harvest-mcp-server`, MIT, 8 stars, 3 forks, not archived |
| Activity | 36 commits 2025-09 -> 2026-07; last push 2026-07-09 (v0.1.9); fork is level with upstream |
| Issues | None, open or closed |
| PRs | #3 open since 2026-10-12, unreviewed: adds invoice payment listing/recording tools (more write surface). #1, #2 merged |

Low-traffic single-maintainer project; expect to carry our own fixes. Worth contributing back:

- Dependency bump (`npm audit fix`).
- Read-only flag (B) and tool annotations (C).
- Windows-safe build script.
- Either wire `HARVEST_API_BASE_URL` through or remove it from the schema, README and `.env.example`.
- Drop unused `dotenv`; report the real version in `serverInfo` and the User-Agent (both say `0.1.0`).
