# 02 · TRD — Technical Requirements Document

> **Closed** decisions. The implementing agent must not choose alternatives without
> going through the "Decision log" first (section 11).
> **MVP principle:** a local CLI, zero services other than the OpenAI API (Langfuse optional),
> and the fewest possible npm dependencies.
> **Status:** Approved · 2026-10-15

---

## 1. Stack

| Layer | Choice | Version | Why |
|---|---|---|---|
| Runtime | Node.js | 24.x | Built-in SQLite and `util.parseArgs`, same as Radar |
| Language | TypeScript (strict) | 5.x | Same as Radar; imports Radar's types directly |
| CLI | `node:util` `parseArgs` | built-in | No CLI framework needed for a handful of commands |
| Storage | **SQLite via `node:sqlite`** | built-in | Zero dependencies; SQL makes run comparison and aggregation easy |
| Judge LLM | OpenAI Responses API via `openai` | 7.x | Owner decision (PRD §8); same SDK as Radar |
| Validation | Zod | 4.x | Datasets, config, judge output; built-in JSON Schema export |
| Tracing | OpenTelemetry JS (`@opentelemetry/api`, `@opentelemetry/sdk-trace-base`, `@opentelemetry/context-async-hooks`) | 2.x | Vendor-neutral standard; GenAI semantic conventions |
| Trace export (optional) | `@opentelemetry/exporter-trace-otlp-http` → Langfuse OTLP endpoint | — | Langfuse ingests OTLP, so no Langfuse SDK is needed |
| HTML report | Template literals + hand-written CSS + inline SVG charts | — | Static file, no framework, no chart library |
| System under test | Radar, as an **npm dependency from GitHub** (`github:marioarranzr/radar#<tag>`) | pinned tag | PRD §9: never copy Radar's code here (§3) |
| Tests | Vitest | 5.x | Same as Radar |
| Scripts runner | `tsx` | 4.x | Same as Radar |
| CI | GitHub Actions | — | Where the repo lives |

### Allowed dependencies (closed list)
- **Runtime:** `openai`, `zod`, `@opentelemetry/api`, `@opentelemetry/sdk-trace-base`,
  `@opentelemetry/context-async-hooks`, `@opentelemetry/exporter-trace-otlp-http`, `radar` (git dependency)
- **Development:** `typescript`, `@types/node`, `eslint` (+ `typescript-eslint`), `vitest`, `tsx`
- Anything else → goes through the Decision log.

## 2. Architecture

```
evals run ──▶ load dataset (Zod) ──▶ estimate cost, confirm ──▶ INSERT run
                                                    │
               for each case ───────────────────────┤
                 │  span "eval.case"                │
                 ├─▶ SUT adapter (Radar) ──▶ researchCompany()  ← emits its own OTel spans
                 │        returns output + usage + error
                 ├─▶ deterministic checks (pure functions)
                 ├─▶ LLM judge (OpenAI, rubric) ──▶ span "chat <judge-model>"
                 └─▶ INSERT case_result, check_results, judge_scores
                                                    │
                     spans ──▶ SqliteSpanExporter (always) + OTLP exporter (if Langfuse keys)
                                                    ▼
evals report / compare / gate ──▶ read SQLite ──▶ HTML file + Markdown summary + exit code
```

- Cases run **sequentially** in the MVP (simple, predictable cost). Concurrency → later.
- One process, no server. Everything a run produces is in `data/evals.db` (ignored by git).

## 3. System under test: Radar as a dependency

### 3.1 Change in Radar (done in the Radar repo, its own plan step and decision log entry)
Radar's agent already takes its dependencies by injection (`AgentDeps`), but it is coupled to its
SQLite store and its `@/` import alias. Radar gets a **library entry point**:

```ts
// radar/src/agent/research.ts (new) — no SQLite, no Next.js, no env reading
researchCompany(
  input: { query: string; context?: string },
  opts: { client: OpenAI; model: string; onStep?: (step: Step) => void; signal?: AbortSignal },
): Promise<ResearchResult>

type ResearchResult = {
  status: "done" | "failed";
  report: Report | null;        // Radar's ReportSchema type
  error: string | null;
  usage: { inputTokens: number; outputTokens: number; searches: number; pagesRead: number; requests: number };
  startedAt: string; finishedAt: string;
};
```
- `runResearchAgent(id)` becomes a thin wrapper that calls `researchCompany` and writes to SQLite,
  so the app keeps working unchanged.
- Radar instruments `researchCompany` with **`@opentelemetry/api` only** (a no-op unless the host
  registers a tracer provider — the standard pattern for libraries): one `invoke_agent radar` span,
  one `chat <model>` span per request, one `execute_tool <name>` span per tool call.
- Radar adds a `build` of the library to `dist/` (plain `tsc`, relative imports), a `prepare` script so
  npm builds it when installed from GitHub, and `exports` pointing at `dist/`. It is released as a
  git tag (e.g. `v0.2.0`).

### 3.2 Adapter here
```ts
interface SystemUnderTest<Input, Output> {
  name: string; version: string;           // e.g. "radar", "v0.2.0"
  run(input: Input, ctx: { signal: AbortSignal }): Promise<SutResult<Output>>;
}
type SutResult<O> = { output: O | null; error: string | null; usage: Usage; durationMs: number };
```
- The harness core (`src/core/`) never imports Radar types. Only `src/adapters/radar.ts` does.
- Replay mode (PRD F1.3) is an adapter that returns outputs stored in a previous run.

## 4. Scorers

### 4.1 Deterministic checks (Radar)
Pure functions `(case, output) → { pass: boolean; detail: string }`, versioned with the code:
`schema_valid`, `min_sources(3)`, `sources_are_urls`, `no_duplicate_sources`, `max_signals(5)`,
`max_pains(5)`, `questions_3_to_5`, `expected_facts` (case-insensitive substring match of the
case's `expected_facts`, only when present). A case **passes** when every check passes.

### 4.2 LLM-as-judge
- OpenAI Responses API, **structured output** (`text.format: json_schema`, strict) with one entry per
  rubric criterion: `{ criterion, score: 1..5, justification }` — validated again with Zod.
- Rubric lives in `rubrics/radar.v1.md` (versioned file; the version is stored with each score).
  MVP criteria: **groundedness** (claims supported by the listed sources), **relevance** (useful for
  preparing a first meeting), **specificity** (concrete, not generic), **uncertainty honesty**
  (unknowns marked unconfirmed, ambiguous company flagged).
- Judge input: case input + the output as JSON. The judge does **not** browse; groundedness is judged
  against source titles/URLs and internal consistency (limitation stated in the README).
- Config: `JUDGE_MODEL` (default `gpt-6.1-sol`), `reasoning.effort: "low"`, fixed instructions.
  Self-preference risk is measured by calibration (§4.3).
- One judge call per case; on invalid output, one retry, then the case is marked `judge_error`.

### 4.3 Calibration
- `calibration/radar.labels.json`: my 1–5 scores per criterion for N recorded outputs (N ≈ 20).
- `evals calibrate` runs the judge on those outputs and reports agreement per criterion
  (**agreement = |judge − human| ≤ 1**) and the exact-match rate. O1 target: ≥ 80 %.

## 5. Tracing
- `BasicTracerProvider` from `sdk-trace-base`, registered globally together with an
  `AsyncLocalStorageContextManager` (so Radar's spans nest under `eval.case` across `await`s),
  with `BatchSpanProcessor`s:
  - `SqliteSpanExporter` (ours): writes every span to the `spans` table.
  - OTLP HTTP exporter to `${LANGFUSE_BASE_URL}/api/public/otel` with Basic auth
    (`LANGFUSE_PUBLIC_KEY:LANGFUSE_SECRET_KEY`) **only if** those variables are set.
- Span tree per case: `eval.case` → (`invoke_agent radar` → `chat …`, `execute_tool …`) +
  `eval.judge` → `chat <judge-model>`. Attributes follow the OTel GenAI semantic conventions
  (`gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, …) plus
  `eval.run_id`, `eval.case_id`.
- **Prompt and output content are not put in span attributes** (they are already in SQLite); only
  metadata. This keeps exported traces small and free of research content.
- The provider is flushed and shut down at the end of every command.

## 6. Cost
- `config/prices.json`: USD per 1M input/output tokens per model and per 1,000 web searches
  (list prices, checked manually; the date is stored in the file).
- Cost per case = SUT cost + judge cost. Before a paid run, the CLI prints
  `cases × (mean SUT cost of the last run or the price-table estimate + judge estimate)` and asks
  `Continue? [y/N]` (`--yes` skips).
- `MAX_RUN_COST_USD` (default 5): after each case the accumulated cost is checked; if the next case
  could exceed it, the run stops with status `stopped_budget`.

## 7. Regression gate
- `evals.config.json` (committed): thresholds, e.g.
  `{ "minPassRate": 0.9, "maxJudgeDrop": 0.3, "maxCostIncreasePct": 20, "maxP50LatencyIncreasePct": 25 }`.
- Baseline = a run marked as baseline in SQLite, exported to `baselines/radar.json` (committed,
  reviewed before commit: outputs are briefs about public companies, no secrets).
- `evals gate --run <id>` compares with the baseline; prints a table of breached thresholds;
  exit code `1` if any, `0` otherwise.

### 7.1 CI (GitHub Actions)
- On every PR: `npm run lint`, `npm test`, and `evals gate` in **replay mode** against
  `baselines/radar.json` with deterministic checks only (no API key, $0).
- Live eval (Radar + judge) runs only when a PR has the label `run-evals`, using an
  `OPENAI_API_KEY` repo secret and `MAX_RUN_COST_USD`. The Markdown summary is posted as the job
  summary.
- O2 is demonstrated with a PR that bumps Radar to a tag with a deliberately degraded prompt.

## 8. CLI

| Command | Does |
|---|---|
| `evals run --dataset <file> [--label <name>] [--yes]` | Runs dataset → SUT → scorers; stores the run; writes its report |
| `evals replay --from <runId> [--label <name>]` | Re-scores the outputs of a run (no SUT calls) |
| `evals report <runId>` | (Re)writes `reports/<runId>/index.html` + `summary.md` |
| `evals compare <runA> <runB>` | Writes `reports/compare-<a>-<b>/index.html` + `summary.md` |
| `evals baseline set <runId>` / `export` | Marks the baseline / writes `baselines/radar.json` |
| `evals gate --run <runId>` | Exit code per §7 |
| `evals calibrate` | Judge vs human agreement (§4.3) |
| `evals runs` | Lists runs (id, label, date, pass rate, cost) |

Exposed as `npm run evals -- <command>` (via `tsx src/cli.ts`).

## 9. Environment variables (`.env.local`)

```
OPENAI_API_KEY=            # required for live runs and the judge
RADAR_MODEL=gpt-6.1-sol    # model the SUT runs with
JUDGE_MODEL=gpt-6.1-sol
MAX_RUN_COST_USD=5
DATABASE_PATH=./data/evals.db
LANGFUSE_PUBLIC_KEY=       # optional
LANGFUSE_SECRET_KEY=       # optional
LANGFUSE_BASE_URL=         # optional, e.g. https://cloud.langfuse.com
```

## 10. Non-functional requirements
- **Security:** keys only in `.env.local`/CI secrets; never logged, never in spans or reports.
  Every string in the HTML report is HTML-escaped (outputs come from the web and are untrusted);
  links get `rel="noopener noreferrer"`; no inline scripts from data. SQL always parameterized.
- **Reproducibility:** each run stores dataset hash, rubric version, scorer versions, SUT name and
  version, models and settings.
- **Errors:** a failing case never aborts the run; it is recorded with its error.
- **Accessibility:** the report meets WCAG AA contrast and works with keyboard and screen readers.
- **Language:** all code, docs, rubrics and reports in English.

### Testing
- Unit: dataset/config validation, every deterministic check, cost math, gate comparisons,
  agreement math, HTML escaping, `SqliteSpanExporter` (`:memory:`).
- Judge and SUT: tested with fakes (scripted responses); no test calls the real API.
- One end-to-end test of `run → report → gate` with a fake SUT and a fake judge.

## 11. Decision log (lightweight ADR)

| Date | Decision | Rejected alternatives | Reason |
|---|---|---|---|
| 2026-10-15 | **TypeScript** | Python | Same language as Radar; imports its types; one language in the portfolio |
| 2026-10-15 | **CLI + static HTML report** | Next.js dashboard, Langfuse UI only | Fewer dependencies, local-first, easy to publish from CI |
| 2026-10-15 | **OpenTelemetry, Langfuse optional via OTLP** | OTel only, Langfuse SDK | Vendor-neutral; Langfuse shown without lock-in |
| 2026-10-15 | **Judge: OpenAI** | Claude as an independent judge | Owner decision; self-preference measured by calibration |
| 2026-10-15 | **Radar as an npm git dependency with a library entry point** | HTTP to a running Radar, copying code, git submodule | Full traces inside the agent (O3); Radar code never copied here |
| 2026-10-15 | **SQLite via `node:sqlite`** | JSON/JSONL files | Zero dependencies; SQL for comparisons |
| 2026-10-15 | `parseArgs` instead of a CLI framework | commander, yargs | Built in; few commands |
| 2026-10-15 | Hand-written SVG charts | Chart.js, Vega-Lite | Static report, no JS dependency |
| 2026-10-15 | Sequential case execution | Concurrency pool | Predictable cost and simpler traces for the MVP |
