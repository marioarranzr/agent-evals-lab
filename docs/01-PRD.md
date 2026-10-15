# 01 · PRD — Product Requirements Document

> **Project:** agent-evals-lab — evals & observability for LLM agents
> **Version:** 0.1 (local MVP, personal use) · **Date:** 2026-10-15 · **Status:** Approved

---

## 1. Summary

agent-evals-lab is a command-line harness that **measures an LLM agent the way a test suite
measures code**. You point it at an agent (the "system under test"), it runs the agent over a
**golden dataset** of cases, scores every output with **deterministic checks** and a
**rubric-based LLM-as-judge**, records an **OpenTelemetry trace** of every run, and writes a
**static HTML report** with quality, cost and latency — per case and against a baseline.

The same command runs in **CI as a regression gate**: if a prompt or model change makes quality
drop (or cost/latency rise) beyond a threshold, the check fails and the pull request is blocked.

The first system under test is **Radar** (https://github.com/marioarranzr/radar), the company
research agent. The harness talks to it through a small adapter interface, so other agents can be
plugged in later.

**MVP = a local CLI.** No server, no login, no hosted dashboard. Results live in local files.
The only external services are the model APIs used by the system under test and by the judge.
Exporting traces to Langfuse is optional.

## 2. Problem

Teams ship agents by "trying a few inputs and eyeballing the output". When they change a prompt,
a model or a tool, they cannot tell whether quality went up or down, nor what it cost.
Tracing alone shows *what happened* in one run but not *whether it was good*; most teams trace,
few run real evaluations. Radar is a concrete example: its eval script reports validity, sources,
time and cost, but "is the brief useful?" is still a manual column, and nothing stops a regression
from being merged.

## 3. Target user

- **MVP: me**, an engineer who builds agents and wants to change them without fear.
- **Future:** any team shipping an LLM agent who wants evals in their PR workflow.
- **Portfolio reader:** a recruiter or hiring manager for AI engineering / AI evals / LLMOps roles,
  who should understand the project from the README and one report.

## 4. MVP goals

| # | Goal | Success metric |
|---|---|---|
| O1 | The judge agrees with a human | On a hand-labelled calibration set, judge vs my rating agree on ≥ 80 % of criteria |
| O2 | Catches regressions | A deliberately degraded Radar prompt makes the CI gate fail; the unchanged prompt passes |
| O3 | Full visibility per run | 100 % of runs have a trace with model calls, tool calls, tokens, cost and latency |
| O4 | Comparable runs | Two runs (e.g. two models or two settings) can be compared side by side in one report |
| O5 | Cheap to run | A full eval of the golden dataset (incl. judge) costs ≤ $3 |
| O6 | Learning / portfolio | Documented and reusable; README explains the method to a non-expert |

## 5. Features (exact scope)

### F1 · System under test (adapter)
- F1.1 A typed adapter interface: given a case input, run the agent and return its output plus
  usage (tokens, tool calls, duration, cost) and any error.
- F1.2 A **Radar adapter** that runs Radar's agent on a case.
- F1.3 A **replay mode**: re-score outputs recorded in a previous run without calling the agent
  again (so judge/checks can be iterated cheaply).

### F2 · Golden dataset
- F2.1 Dataset files versioned in the repo (cases with an id, input, optional context and
  optional expected facts / reference notes).
- F2.2 A starter Radar dataset (initially the 10 companies already used in Radar's eval, extended
  with harder cases: ambiguous names, rebrands, tiny companies, non-existent companies).
- F2.3 Dataset validation: the command fails with a clear message if a case is malformed.

### F3 · Scorers
- F3.1 **Deterministic checks** (no model): schema valid, required sections present, minimum
  number of sources, sources are well-formed URLs, no duplicate sources, limits respected
  (e.g. max 5 signals), expected facts present when given.
- F3.2 **LLM-as-judge** with a written rubric (e.g. groundedness, relevance, specificity,
  honesty about uncertainty). Each criterion returns a score **and a short justification**.
- F3.3 **Judge calibration**: a small hand-labelled set and a command that reports judge-vs-human
  agreement per criterion.
- F3.4 Every scorer has a name and version, and its version is stored with the result.

### F4 · Tracing
- F4.1 Every case run produces an OpenTelemetry trace (agent run → model calls → tool calls),
  using the GenAI semantic conventions where they apply.
- F4.2 Traces are stored locally and linked from the report.
- F4.3 Optional: if Langfuse credentials are configured, traces are also exported to Langfuse.

### F5 · Runs and reports
- F5.1 `eval run`: runs a dataset against the system under test with a named configuration
  (model, settings) and stores the run (outputs, scores, usage, trace ids).
- F5.2 A **static HTML report** per run: summary (pass rate, mean scores, p50/p95 latency,
  total and mean cost), a table per case, and a detail view per case (output, check results,
  judge scores with justifications, trace link).
- F5.3 **Comparison**: a report that puts two runs side by side (deltas per metric and per case).
- F5.4 A short **Markdown summary** of a run or comparison, suitable for a PR comment.

### F6 · Regression gate (CI)
- F6.1 Thresholds in a config file (e.g. min pass rate, max drop in a judge score vs baseline,
  max cost / latency increase).
- F6.2 `eval gate`: compares a run with the stored baseline and exits non-zero if a threshold
  is breached, printing which one and why.
- F6.3 A GitHub Actions workflow that runs the gate on pull requests (in replay mode by default, so
  CI does not spend money unless explicitly enabled).

### F7 · Cost safety
- F7.1 Before a run that calls paid APIs, the CLI prints an estimated cost and asks for
  confirmation (skippable with a flag for CI).
- F7.2 A hard per-run spending cap; the run stops when it is reached.

## 6. Out of scope (NOT built in the MVP)

- A web app or hosted dashboard (the HTML report is static).
- Users, login, multi-tenant anything.
- Online / production evals on live traffic (MVP is offline evals on a dataset).
- Prompt-injection red-teaming (planned as a later module — candidate project #2).
- Human labelling UI (labels are edited as files).
- Auto-tuning prompts or models from the results.
- Adapters for agents other than Radar (the interface allows them; none are built).
- Python SDK or support for other languages.

## 7. Assumptions and risks

| Risk | Mitigation |
|---|---|
| The judge is biased or inconsistent | Written rubric, justification per criterion, calibration against my labels (F3.3), fixed judge model + temperature, judge version stored |
| The judge favours its own model family (judge and Radar are both OpenAI) | Calibration against my labels (F3.3) measures it; judge model is configurable |
| Eval runs cost too much | Replay mode, estimated cost + confirmation, per-run cap, CI in replay mode by default |
| Web results change over time, so outputs are not reproducible | Store every output; compare against a baseline, not absolute truth; checks avoid volatile facts |
| Coupling to Radar's internals | Thin adapter; harness core does not import Radar types |
| Dataset too small to be meaningful | Start with ~20 cases, report per-case deltas, not only averages |

## 8. Decisions taken

- **Judge provider: OpenAI** (owner decision, 2026-10-15). The judge model is configurable.

## 9. Open questions

- **How exactly is Radar consumed?** Decided: Radar's code is **never copied into this repo**; it
  is consumed as a dependency. The mechanism (e.g. npm dependency from GitHub) → decide in the TRD.
- Should Radar's own `npm run agent:eval` be replaced by this harness once it works?
