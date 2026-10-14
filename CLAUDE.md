# CLAUDE.md

## Project

**tidrapp** drafts your weekly timesheet from evidence you already leave behind, and fills it into your time-reporting system for you to review and submit.

Consultants lose time every Friday reconstructing the week: which customer, which project, how many hours, and what to write. tidrapp collects the evidence (git commits across all your repos, events from one or more calendars, and what you have already reported) and builds a week proposal. The proposal respects your **allocation per customer** (e.g. 75 % customer A, 25 % customer B). You review and adjust it in conversation with Claude. tidrapp then fills the time system as a draft through the browser, and **you** submit it.

It ships as a **Claude Code plugin** (skill + MCP server) with a plain **CLI** for the same core logic.

Time systems are adapters. The first adapter targets a corporate time system that:
- has a **read-only** HTTP API with an API key
- can only be written to through its web UI

Anything specific to that system (base URL, endpoints, field names, selectors) lives in a private **site profile** on the user's machine, never in this repository.

## Tech stack

- **TypeScript on Node.js 24.** This is the native ecosystem for MCP servers and Claude plugins, and it can be installed with `npx`.
- **@modelcontextprotocol/sdk** for the MCP server the plugin exposes to Claude.
- **Playwright** for filling the time system's web UI. It uses a persistent browser profile, and the user signs in themselves.
- **node-ical** to read calendars from ICS URLs (Outlook "publish calendar", Google secret address).
- **zod** to validate config, site profiles and every external response at the boundary.
- **commander** for the CLI.
- **Vitest** for tests.
- Git is read by shelling out to `git log`. There is no git library and no GitHub/Azure DevOps token in the MVP.

## Domain

- **Evidence**: something that happened, with a time span or a timestamp. Commits, calendar events and already reported entries are all evidence.
- **Mapping rule**: turns evidence into a target, i.e. a customer + project. Examples: a repo path, a remote URL or a git author email → Customer A / Project X, or a calendar → Customer B.
- **Allocation**: the expected share per customer over a week, e.g. `{ "Customer A": 0.75, "Internal": 0.25 }`. It is configured, or derived from the last 4 reported weeks.
- **Proposal**: hours per day per target, rounded to 0.5 h, with a short description and the evidence behind each line.
- **Deviation**: where the proposal differs from the allocation by more than the configured tolerance. Deviations are flagged and never silently "fixed".
- **Site profile**: private, per-user JSON describing one time system (URLs, API paths, auth header name, form selectors). Loaded at runtime from the user's config directory.

## Architecture

- **`core/` is pure.** It does no I/O, no `Date.now()`, no env access and no network. It receives evidence + config + the week and returns a proposal. All allocation math, rounding, holidays and deviation checks live here and are unit tested.
- **All reading lives in `sources/`** (git, ICS, time-system API). **All writing lives in `targets/`** (browser fill). Sources and targets implement small interfaces defined in `core/`, so they can be swapped.
- **`core/` knows nothing about any specific time system.** Adapters are generic and driven by a site profile. Adding another time system means a new profile or adapter, never a change in `core/`.
- **Numbers are computed by code, never by the LLM.** Claude may write descriptions, ask the user, and classify evidence that no mapping rule covered. Hours and percentages always come from `core/`.
- **The MCP server and the CLI are thin.** They parse input, call `core/` and the adapters, and format output. No business logic lives there.
- Validate every external payload and site profile with zod at the boundary. Inside the system, types are trusted.
- Functions do one thing. No global mutable state; pass config and clock explicitly.
- Every external call has explicit error handling. A source that fails is reported in the proposal ("calendar X unreachable"), not swallowed.
- Tests are written together with the implementation. `core/` targets high coverage; adapters are tested against recorded, anonymised fixtures, not live services.

## Safety and privacy (non-negotiable)

- **A human always submits.** tidrapp only fills entries as a draft. It must never click any control that submits, locks, approves or marks time as done ("klarmarkera"), not even if the user asks it to in conversation. The adapter has no code path for it. Site profiles describe fill fields only, and a profile containing a submit selector is rejected.
- **Nothing is filled without approval.** The flow is always: propose, then the user reviews and edits, then the user approves the exact week shown, then fill as draft, then the user checks it in the UI and submits it themselves.
- **Never handle passwords or SSO.** The user signs in to the time system in the Playwright window themselves, and tidrapp reuses that profile.
- API keys and ICS URLs are secrets. Read them from env or the user config directory. Never log them and never commit them. Send API keys as headers, never in query strings.
- Evidence stays on the user's machine. Do not send commit messages or calendar contents anywhere except to the local Claude session.
- Only read the user's own time. Do not build features around other employees' data, even if an API allows it.

## Rules for the agent

- **Never put the name, URL, endpoints, selectors or screenshots of the real time system or employer in this repository.** That includes code, tests, fixtures, docs, commit messages and PR text. Use generic names (`timesystem`, `Customer A`, `example.invalid`).
- Do not add or swap dependencies without asking.
- Do not move logic into `mcp/` or `cli.ts`. If it needs a test, it belongs in `core/`.
- Do not let an LLM call decide hours, percentages or rounding.
- Do not call live time systems, calendars or remote git in tests. Use fixtures in `tests/fixtures/`.
- Do not change the CLI commands/flags or the MCP tool names/schemas without asking. They are the public interface.
- Do not commit directly to `main`. Work in feature branches with small PRs.
- When the time system behaves in a way not covered by the site profile, stop and ask instead of guessing selectors or endpoints.

## Project structure

```
/
├── CLAUDE.md
├── README.md
├── .claude-plugin/
│   └── plugin.json            # Plugin manifest: registers skill + MCP server
├── skills/
│   └── tidrapport/
│       └── SKILL.md           # Conversational flow: collect → propose → review → fill draft
├── src/
│   ├── core/
│   │   ├── types.ts           # Evidence, Target, Allocation, Proposal + adapter interfaces
│   │   ├── mapping.ts         # Evidence → target via rules
│   │   ├── allocate.ts        # Build proposal from evidence + allocation
│   │   ├── calendar-se.ts     # Working days, Swedish public holidays
│   │   └── deviations.ts      # Compare proposal vs allocation
│   ├── sources/
│   │   ├── git.ts             # git log over configured repo roots
│   │   ├── ics.ts             # One or more ICS calendars
│   │   └── timesystem-api.ts  # Read already reported time (read-only, profile-driven)
│   ├── targets/
│   │   └── browser-fill.ts    # Playwright: fill a confirmed proposal as draft (profile-driven)
│   ├── config.ts              # Load + validate user config and site profile (zod)
│   ├── mcp/
│   │   └── server.ts          # MCP tools: collect_evidence, propose_week, fill_week_draft
│   └── cli.ts                 # tidrapp propose --week 42 | tidrapp fill --week 42
├── tests/
│   ├── fixtures/
│   └── *.test.ts
├── examples/
│   ├── config.example.json
│   └── site-profile.example.json  # Fictional system on example.invalid
└── package.json
```

## Configuration

User config lives in `~/.config/tidrapp/` and is validated by `src/config.ts`.

`config.json` contains:
- the allocation per customer
- hours per day
- the rounding step
- the deviation tolerance
- repo root folders and git author emails
- calendar ICS URLs, each with an optional default target
- mapping rules
- which site profile to use

`sites/<name>.json` is the private site profile for a time system. It is never committed.

Secrets can be given as env vars instead: `TIDRAPP_TIMESYSTEM_API_KEY`, `TIDRAPP_ICS_<NAME>`.

## Setup

```bash
npm install
npx playwright install chromium
mkdir -p ~/.config/tidrapp/sites
cp examples/config.example.json ~/.config/tidrapp/config.json
```

## Run

```bash
npm run build
node dist/cli.js propose --week 42
```

To run it as a Claude Code plugin:

```bash
claude --plugin-dir .
```

## Test

```bash
npm test
npm run lint
npx tsc --noEmit
```

## Open questions (resolve before building the related part)

- Does the time system's web UI save drafts via form posts or JSON endpoints? If there is a JSON draft endpoint usable from the signed-in page, prefer it over clicking through the UI, but still never submit.
- Can a customer tenant's Outlook calendar be published as ICS, or is Microsoft Graph needed?
- Should the allocation default come from config or from reported history?
