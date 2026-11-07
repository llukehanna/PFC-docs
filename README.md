# PFC — Personal Finance Coach (public engineering docs)

> This repo contains the **public documentation** for PFC, a single-user personal-finance system I run against my own accounts. The source is kept in a private repo with **no git remote at all** — it tracks the seed data (real balances, credit limits, card open dates, tradeline detail), and the only defensible way to version that is a written policy that it never leaves the machine. The goal here is the same as for [BT-docs](https://github.com/llukehanna/BT-docs): describe the engineering — architecture, data model, decision engine, safety, stack — with enough detail that it's verifiable as a real system, without publishing the financial state it holds.

## What it does

PFC holds the full state of my finances, verifies I'm executing a multi-year credit-card and investment plan correctly, tells me what to do next and when, and challenges whether the plan is still optimal as card terms change under me.

It is **not a budgeting app**. Budgeting is the easy, solved part. The hard part is the credit-card meta-game and the sequencing decisions that no existing tool covers alongside a real financial model:

- **Gates.** Chase's 5/24 rule, per-card utilization at statement close, sign-up-bonus (SUB) windows, annual-fee calendars, minimum account ages. Every action is checked against every gate before it happens.
- **Points.** Which card to use at which merchant, right now, given category caps that have already been partially consumed this quarter.
- **Opportunity cost.** Every dollar has a next-best use. Cash above the available rate, an employer match left unclaimed, an uninvested contribution, a fee that shouldn't exist — each is ranked in the same queue, in the same unit, as the card actions.
- **Trajectory.** A tax-aware five-year projection, frozen monthly and scored against what actually happens.

**Core value, and the one thing that cannot fail: never miss a gate.** The system must know, at any moment, whether an action I'm about to take breaks the plan — an unplanned account opening that busts 5/24, a SUB window about to close, a statement closing at high utilization weeks before a planned application. Everything else can degrade; this cannot.

## The three things that shape everything

1. **Plaid returns no MCC.** The merchant category code is what actually determines points earned, and no consumer-accessible aggregator exposes it — Plaid gives you a proprietary category taxonomy instead. So `merchant → bonus category` is a table the system **owns and improves**, seeded from in-house research and corrected from reconciliation drift. It is the single most valuable asset in the system; the bank connections are commodity.
2. **Fidelity is unreachable through Plaid.** It shares data only through Akoya, which Plaid declines to integrate. Fidelity arrives via SimpleFIN Bridge instead, whose *holdings* support is undocumented — I verified it empirically against my own account before building on it.
3. **Nothing needs to be on the public internet.** Poll on a timer instead of using webhooks, request read-only products and never Plaid's `auth` product, and delegate access control to Cloudflare Access at the edge. A total token compromise then reads history and cannot move money.

## How it decides

The v1 engine was a rules layer: derive the credit file, check the gates, project the points, alert. The v2 milestone ("The Wealth Model") replaced the objective function with **opportunity cost across every dollar**, evaluated over a versioned vector of assumptions.

### Assumptions are first-class, versioned, append-only data

Hurdle rate, expected returns, future marginal tax outlook, value of time, risk posture — declared as **ranges, not points**, stored append-only, and every derivation reruns when one changes. This is how the plan stays dynamic without a scraping loop or a chat history.

### Dominance over a declared range, not a confidence score

Every proposed action `A` is compared against the alternative use of the same dollar `B`, across the whole declared assumption range:

| Outcome | Behaviour |
|---|---|
| `V(A) − V(B)` has one sign across the entire range | Propose it |
| The sign flips inside the range | **Escalate** with both paths and the assumption that decides, naming the indifference point |
| `\|V(A) − V(B)\|` is under the materiality floor everywhere | Take the default silently, log `immaterial` |

**Never pick silently on a close call.** The comparator bisects the assumption axis to find the indifference point rather than sampling it, and a `narrowing` step reports how much of the range the verdict actually depends on.

### The rules that fire

Twenty rules, each a pure function over the derived state, each producing a ranked queue item in a common unit (dollars of opportunity cost per year). Grouped by family:

| Family | Rules |
|---|---|
| **A — Card plan** | SUB at risk · wrong card used · utilization before statement close · action breaks a gate · authorized-user tradeline closure |
| **B — Cash** | cash below the available rate · money-market fund vs invested |
| **C — Employer** | employer match unclaimed |
| **D — Investments** | expense-ratio alternative · uninvested contribution |
| **E — Leakage** | revolving interest · late fee · FX fee · maintenance fee · ATM / overdraft fee |
| **F — Anomaly** | outflow above baseline · new recurring charge · duplicate or unknown counterparty |
| **G — Model health** | assumption stale · tax-envelope breach |

The G-family exists because the model's own inputs decay: card terms change mid-plan without warning (one card in the plan was discontinued; another's annual fee has roughly doubled over its lifetime), and tax parameters are indexed annually. Re-verification is a scheduled job with a provenance trail, not a memory.

## Architecture

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                          SOURCES  (all read-only)                              │
│                                                                                │
│   Plaid /transactions/sync        SimpleFIN Bridge        deterministic parsers│
│   + /liabilities + /investments   (Fidelity, incl.        — issuer statements  │
│   Hosted Link, no webhooks        holdings[])             — Equifax / Experian │
│                                                           — positions CSV      │
└──────────────┬───────────────────────────┬─────────────────────────┬───────────┘
               │                           │                         │
               ▼                           ▼                         ▼
      ┌────────────────────────────────────────────────────────────────────┐
      │                zod-validated ingest boundary                       │
      │   every payload validated before it can enter the ledger;          │
      │   decimal strings → integer cents, no intermediate float           │
      └───────────────────────────────┬────────────────────────────────────┘
                                      ▼
      ┌────────────────────────────────────────────────────────────────────┐
      │        APPEND-ONLY RAW LEDGER  (SQLite, WAL, better-sqlite3)       │
      │   raw_transaction · raw_holding · raw_account_balance · statement  │
      │   credit_file · assumption (versioned) · projection_snapshot       │
      │   manual observations (points balances, redemptions, offers)       │
      │                                                                    │
      │   RAISE(ABORT) triggers forbid UPDATE / DELETE on every raw table  │
      └───────────────────────────────┬────────────────────────────────────┘
                                      ▼
      ┌────────────────────────────────────────────────────────────────────┐
      │                recompute()  — the sole writer of derived state     │
      │                                                                    │
      │   credit file · 5/24 calendar · utilization vs close · SUBs ·      │
      │   benefits · caps · merchant map · allocation · net position ·     │
      │   comparator (20 rules × assumption range) · tax envelope ·        │
      │   projection · decision queue · staleness ladder                   │
      └───────────────────────────────┬────────────────────────────────────┘
                                      ▼
      ┌────────────────────────────────────────────────────────────────────┐
      │                DERIVED TABLES  (never hand-edited; not exported)   │
      └───────────────────────────────┬────────────────────────────────────┘
                                      │
          ┌───────────────────┬───────┴────────┬────────────────────┐
          ▼                   ▼                ▼                    ▼
   ┌─────────────┐   ┌─────────────────┐  ┌─────────────┐   ┌──────────────────┐
   │ MCP server  │   │ CLI  (`pfc`)    │  │ Next.js UI  │   │ notify           │
   │ Claude is   │   │ sync · import · │  │ standalone, │   │ daily digest +   │
   │ the primary │   │ recompute ·     │  │ mobile-first│   │ push to phone    │
   │ interface   │   │ export · backup │  │ read path   │   │ dead-man's switch│
   └──────┬──────┘   └─────────────────┘  └──────┬──────┘   └──────────────────┘
          │                                       │
          └──────────────┬────────────────────────┘
                         ▼
          ┌──────────────────────────────────────┐
          │  Cloudflare Tunnel + Cloudflare Access│
          │  zero open ports · JWT verified at    │
          │  origin with jose · one identity      │
          └──────────────────────────────────────┘
```

### Write path — gated by provenance, not by surface

Three surfaces can propose a change (MCP tool, terminal, web UI). None of them writes derived state. Every mutation is an **intent** committed with an actor and a provenance record, and only six validated write tools exist on the MCP side — card attributes, SUB facts, an observation, a verification, a declared assumption, and resolving an escalation. Anything else is a terminal command, deliberately. `recompute` itself is never a tool call: a long-lived model session cannot trigger a full re-derivation, and a stale server process cannot silently write a false proposal.

Every MCP tool call is appended as a scrubbed JSONL line (args scrubbed, outcome recorded as row count or error class, never a result body), so "the tool said my 5/24 count was 4 on 3 Nov" is answerable later by joining back to the derivation run.

### Custody

- The system of record is a single SQLite file on an always-on ARM VM (Oracle Cloud Always Free), reached only through the tunnel. The Mac is dev-only.
- A startup path assertion refuses to open a database inside iCloud Drive, Dropbox, Google Drive or OneDrive. The failure mode there is silent — the database opens cleanly, loses recent commits, and passes `PRAGMA integrity_check`.
- Daily encrypted offsite blob to Backblaze B2 under Object Lock, written with a key **the host cannot use to delete**. A compromised host can stop backups; it cannot destroy them. A restore drill is a CLI command, not a runbook paragraph.
- Backups are the raw ledger as NDJSON. Derived tables are never exported because they are not data — restore is `import` + `recompute`, running the same code the fallback ingest path already exercises every day.
- Aggregator secrets are AES-256-GCM envelopes with the key outside the database; on the host they reach the process only through systemd `LoadCredential`.
- Every scheduled unit is `Persistent=true` with jitter, so a missed run coalesces on wake instead of silently skipping.

## Repo layout (private)

```
src/                      # ~71K LOC TypeScript, 231 files — runs directly under Node 24, no build step
  db/schema/              # Drizzle schema: raw · derived · seed · merchant · assumption · proposal · rules · tax · system
  domain/                 # money (integer cents), civil dates, observed values, card/earn-rule/merchant facts
  ingest/                 # issuer statement parsers, Equifax + Experian, Fidelity positions, provenance, reconcile
  plaid/  simplefin/      # connectors — Hosted Link, /transactions/sync cursor loop, update mode, holdings
  fred/                   # reference rates for the opportunity-cost baseline
  derive/                 # recompute(): credit, utilization, SUBs, caps, merchant map, allocation, net position, projection, queue
  compare/                # the comparator: axis, bisect, dominance, narrowing, verdict, sentence
    rules/                # the 20 rules, one file each (a1-sub-at-risk.ts … g3-tax-envelope-breach.ts)
  tax/                    # envelope, indexation, rounding
  project/                # frozen monthly projection grid + scoring against realized months
  intent/actors/          # mcp · terminal · web — the three provenance-bearing actors
  mcp/read/  mcp/write/   # named tools with outputSchema + readOnlyHint; six validated write tools
  backup/                 # NDJSON export, VACUUM INTO snapshot, B2 offsite, offsite marker, restore drill
  secrets/                # keychain + age-encrypted file store, AES-GCM envelope
  notify/                 # digest + push
  cli/commands/           # the `pfc` operator CLI (commander)
web/app/                  # Next.js 16 standalone UI — money · plan · spending · card/[id] · account/[id] · forward · which · ask · system
seed/                     # 15 CSVs — cards, earn rules, benefits, SUBs, allocation, credit file, tax parameters
drizzle/                  # 37 reviewed .sql migrations (triggers and views via --custom)
deploy/systemd/           # 13 units: mcp · sync · offsite · project-snapshot · web · recompute · path triggers · probe
tests/                    # 285 test files, ~104K LOC — node:test + Playwright across four viewports
.planning/                # 14 phases + 31 backlog items of discuss → plan → execute → verify, with verification reports
DECISIONS.md              # 30 locked decisions with rationale, two recorded reversals, and every amendment dated
```

## Stack + why

| Concern | Choice | Rationale |
|---|---|---|
| Runtime | **Node 24**, native TypeScript type-stripping | No `tsx`, no build step for the CLI, MCP server or timers. `tsc --noEmit` is the only thing that type-checks, and it runs in `npm run verify`. |
| Language | **TypeScript 6.0**, not 7 | TS 7 (the Go rewrite) has no stable programmatic API until 7.1 and `typescript-eslint` caps at `<6.1`. Pinned on evidence, revisited when the cap lifts. |
| Database | **SQLite** via `better-sqlite3` behind **Drizzle** | Single file, WAL, synchronous driver, `.backup()` and `VACUUM INTO` for snapshots. Drizzle emits plain reviewable `.sql` migrations and keeps a Postgres escape hatch if this ever moves to a bigger box. |
| Money | **Integer cents**, everywhere | Aggregators return decimal strings. They are parsed to integer cents with no intermediate float, and a rounding module owns the one place fractions are allowed (tax). |
| Validation | **zod 4** | Every Plaid / SimpleFIN payload is validated *before* it can enter the append-only ledger. Doubles as the MCP tool schema. |
| Banks | **Plaid** (`transactions`, `liabilities`, `investments`) — never `auth` | Plaid's Trial plan is the only route to Chase / Amex / Capital One OAuth without a business registration. 10 lifetime Items, so broken connections are repaired in Link update mode — re-linking burns a slot forever. |
| Fidelity | **SimpleFIN Bridge**, hand-written client (~120 lines) | Only consumer-reachable path to Fidelity. No maintained TS client worth depending on; the protocol is small. |
| Statements / credit reports | **`unpdf`** + deterministic parsers | Four files today and ~12 a year forever. Financial totals must reconcile exactly; an LLM that transposes a digit is worse than a parser that throws. Every parse is reproducible. |
| AI interface | **MCP** (`@modelcontextprotocol/server` 2.x) over stdio locally, HTTP through the tunnel on the host | Claude is the product's primary interface (locked decision). Named tools with `outputSchema` and `readOnlyHint`, one per derivation — never a generic `run_sql`. |
| Scheduling | **systemd timers** (host) / **launchd** (Mac) | Process supervision belongs to the OS. Both coalesce missed runs on wake; `node-cron` silently skips them. |
| Access | **Cloudflare Tunnel + Access**, `jose` at origin | Zero open ports. The app authenticates the *person* by verifying the Access JWT, not the path. No self-written auth, ever. |
| Web UI | **Next.js 16** standalone, React 19, CSS Modules | Self-hosted on the same VM, binds loopback only, holds no credential. Read path only — the privileged half of "refresh" is a systemd `.path` unit watching a sentinel file. |
| Push | **ntfy-style push** from the daily unit | Alerts reach the phone; a web dead-man's switch fires if the UI stops answering. |
| Secrets | macOS Keychain (dev) · **age**-encrypted file + `LoadCredential` (host) | The database never holds a plaintext token; the process only sees them at start. |
| Testing | **`node:test`** + **Playwright** | Zero-dep unit runner. A UI behaviour claim requires a committed, passing Playwright test (locked decision), across four viewports. |
| Lint / types | **eslint 10 + typescript-eslint**, strict | `npm run verify` = typecheck (app + web) + lint + unit tests. Must pass before every commit. |

### What's explicitly NOT used

| Tech | Why not |
|---|---|
| SQLite inside iCloud Drive (or any synced folder) | Documented corruption modes: separated `-wal`, synced `-shm`, inode swap on open files. Enforced by a startup assertion, not discipline. |
| Plaid `auth` product | Account and routing numbers turn a privacy incident into a financial one. Unit-tested out of the products array. |
| Plaid webhooks | Needs a public inbound endpoint. Daily poll + `/link/token/get` polling instead. |
| Re-linking a broken Plaid Item | Burns one of ten lifetime slots. Update mode with the existing token, always. |
| A generic `run_sql` MCP tool | One prompt injection exfiltrates the ledger; output is unvalidatable. Named tools only. |
| LLM extraction for statements | Non-reproducible and can transpose digits. Deterministic parsers, and they throw on drift. |
| Floating-point money | Integer cents, with a single rounding module for the tax engine. |
| `node-cron` / `crontab` | Both skip runs while the machine sleeps — the exact scenario. |
| Postgres / Docker, libSQL / Turso, Vercel | Each puts financial history in someone else's account or adds a daemon to a laptop. Local SQLite; Drizzle makes the dialect swap cheap if it's ever needed. |
| `keytar` | Archived. `security(1)` or `@napi-rs/keyring`. |
| A git remote | The repo tracks the seed. See the top of this file. |

## Safety

This is the part I spent the most time getting right, because the failure mode is a financial mistake I can't undo.

- **Read-only everywhere.** No product that can move money is ever requested. A total credential compromise reads history.
- **Append-only ledger, enforced in the database.** `RAISE(ABORT)` triggers on every raw table; derived tables are recomputed by pure functions and are never exported, so a hand-edit is invisible in a backup and erased on the next `recompute()`.
- **Every action is checked against every gate before it happens.** `check_action()` is the single entry point, and "am I about to break 5/24 / a SUB window / a utilization threshold" is answered from derived state, never from memory.
- **No recommendation without its alternative evaluated across the declared range.** The comparator escalates rather than picking on a close call.
- **Writes are gated by provenance, not by surface.** Every mutation carries an actor; a model session can propose, never recompute or apply.
- **Custody assumes the host is hostile.** The offsite blob is written with a key the host cannot delete with; the restore drill is exercised, not documented.
- **Staleness is surfaced, not hidden.** Every response carries `as_of` and the last successful sync; a data-freshness ladder (fresh → stale → missing) fires per source, and the dead-man's switch fires when the timers stop.
- **Seed decay is a scheduled job.** Card terms and tax parameters are re-verified on a cadence with a proposal → apply loop that records what was found and where.

## What I learned

- **Verification, not construction, is the binding constraint.** The first five phases took six days. A 5/24-window bug then survived eleven plans and 446 passing tests, because every test encoded the same wrong assumption. Every phase now has to answer *"how would I know this is wrong?"* before it is planned, and the tax and projection engines are tested against a workbook built independently as a ground-truth oracle.
- **Patch rounds don't converge; redesigns do.** One five-branch function needed four gap-closure rounds, and each round's fix caused the next round's regression. What closed it was a single consolidated pass with a total classification over the input space and a 125-cell enumerated acceptance table. That became a repo hook: a third unpatched repair round on a phase is hard-blocked at plan-write time.
- **"Missing capability" is usually a missing reader.** Six of ten backlog items in one burn-down diagnosed a capability that already existed — a ranked queue every reader discarded, a column no one read, a balance table with a writer and no reader. The defect was one layer up every time.
- **A host that "times out" is not a host that's down.** Five records across two phases declared the VM unreachable. It was a firewall rule pinned to an ISP address that had rotated, under a DROP policy that sends nothing back — and the "100% packet loss" cited as proof proved nothing, because ICMP echo was never allowed. Diagnose from the host's own firewall, and try the tunnel path first.
- **A long-lived MCP server does not hot-reload.** A process that started before a fix kept serving the old comparator while every fresh CLI invocation had the fix. Restart the connection before any re-verification session that follows a code change; check for the field the fix added.
- **The undocumented field is the one you build on.** SimpleFIN's `holdings[]` does not exist in the protocol spec. It was verified against a real account, the observed payload was recorded as the authority, and the ingest boundary validates it on every sync because the shape can change without notice.
- **Decisions get relitigated unless they're written down with their rejected alternatives.** Thirty locked decisions, two formal reversals with triggers named, and the rule that a superseded claim is annotated, never deleted — the claim that was made is the lesson.

## Status

- **Milestone v1.0** (Aug 2026): foundation, credit derivation and gates, MCP server, statement/credit-report backfill, merchant map and allocation, live Plaid feed — shipped; two phases carry open verification residue rather than being marked done.
- **Milestone v2.0 — The Wealth Model** (opened 13 Aug 2026): wealth ontology, versioned assumptions and the comparator, the twenty rules, tax engine and projection — shipped. Custody and the surfaces are live; the projection's monthly scoring cannot be evaluated until snapshotted months elapse.
- Three Plaid Production Items and one SimpleFIN connection syncing daily on the host; frozen projection snapshot #1 is in the production ledger with the monthly timer live.
- Web UI live behind Cloudflare Access; push alerts and both dead-man's switches (sync and web) firing.
- As of 8 Oct 2026: 1,567 commits since 6 Aug 2026 · 43 migrations · 327 test files · 3,229 unit tests, plus Playwright checks across four viewports.
- Since these docs were first written: "which card" answers by merchant; a guard test reads every screen for machine vocabulary; a read-only MCP server for claude.ai verifies the Cloudflare Access token at the origin and refuses to start ungated; a card's product change keeps its account id.
- Deferred, not cancelled: monthly points reconciliation against hand-typed balances. Its payoff needs the merchant map to converge, which needs a year of my own spend flowing through it.

## What's in the private repo

Full source (~84K LOC TypeScript across app and UI, ~104K LOC of tests), the seed tables with real figures, the statement and credit-report corpus, calibrated assumption ranges, and the complete `.planning/` trail — 14 phases and 31 backlog items of discuss → plan → execute → verify, each with its verification report.

Happy to walk through it on request.

---

**Author:** [Luke Hanna](https://github.com/llukehanna) · [lllukehanna@gmail.com](mailto:lllukehanna@gmail.com) · Los Angeles
