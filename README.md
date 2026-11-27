# CodeBench

**A workbench where an idea is proven before it becomes code.**

CodeBench is a Python agent that takes a software idea and carries it through six stations — problem research, user stories, prototype, technical design, implementation spec, evidence — until real, compiling code exists in a target you choose.

The core is **target-agnostic**. It does not know what Flutter is. It emits an *implementation specification*, and a **target profile** turns that spec into the code of a concrete stack.

```bash
codebench run --idea "a habit tracker for people who hate habit trackers" --target flutter
```

---

## Why this exists

Most AI coding tools start at the code. That is the cheapest place to start and the most expensive place to be wrong: you generate a thousand lines on top of a requirement nobody validated.

CodeBench starts where the money is lost — at the problem — and refuses to move forward until each station produces an artifact that survives its check.

| The usual flow | CodeBench |
|---|---|
| idea → prompt → code | idea → **problem** → **stories** → **prototype** → **design** → **spec** → code |
| "it works on my machine" | the target's own build and test commands decide |
| output is a chat transcript | output is a spec on disk, with a hash in a trace |
| locked to one stack | one core, N target profiles |

---

## The six stations

| # | Station | Reads | Writes | Done when |
|---|---|---|---|---|
| 1 | **Research** | one-line idea | interview script, notes, `problem.md` | the reformulated problem differs from the request, and the user agrees with it |
| 2 | **Stories** | `problem.md` | `stories/*.md` | every story has a *testable* acceptance criterion |
| 3 | **Prototype** | stories | `proto/` — screens, states, flow | the user points at what changed their mind |
| 4 | **Design** | stories + prototype | `design/` + ADRs | every decision carries the alternative it discarded |
| 5 | **Spec** | design | `impl_spec.json` | no word of any programming language in it |
| 6 | **Evidence** | everything | `traces/`, `eval/`, build logs | numbers per station — latency, cost, pass rate |

Each station's **output is the next station's input**, written to disk as a versioned artifact. The artifact is the contract; the prompt only carries data.

---

## Target profiles

The only thing that crosses the boundary is `impl_spec.json`: screens, states, entities, data contracts, dependencies. No language, no framework, no stack.

```python
class TargetProfile(Protocol):
    name: str

    def scaffold(self, spec: ImplSpec) -> list[FileArtifact]: ...
    def build_cmd(self) -> list[str]: ...
    def test_cmd(self) -> list[str]: ...
    def validate(self, root: Path) -> BuildResult: ...
```

| Profile | Build | Test | Status |
|---|---|---|---|
| `java` | `mvn -q compile` | `mvn -q test` | planned — week 5 |
| `flutter` | `flutter analyze` | `flutter test` | planned — week 5 |
| `web` | static | smoke test | planned |
| `game` | engine-dependent | — | contract only |

**Adding a target means writing a profile, not touching the core.**

> The proof isn't a claim: the same `impl_spec.json` must compile in two different profiles. That is an automated test, not a paragraph in a README.

---

## Hardening — the part that usually gets skipped

The product on day 50 is not day 1 plus features. It is day 1 plus seven layers of defence. Each layer ships with a test that **fails if the layer is removed**.

| Layer | What it guarantees | Test that proves it |
|---|---|---|
| **H0** | contract + trace — every run has a `run_id` and a `sha256` of its content | one run through all six stations, traced end to end |
| **H1** | structured output validated against schema; named failure | a malformed model response raises a named error, not a `KeyError` |
| **H2** | timeout, retry with backoff, model fallback | a fake provider fails twice and answers on the third |
| **H3** | input guardrail — a story without a testable criterion is rejected | 3 vague stories rejected, 3 good ones accepted |
| **H4** | idempotency, cache, measured cost per station | same input twice → same output hash, plus a cost table |
| **H5** | per-station eval + real build gate | `flutter analyze` passes or the pipeline fails |
| **H6** | full observability + cost ceiling + report | a report with latency, cost and accuracy per station |

No block advances while its hardening layer is red.

---

## Repository layout

```
codebench/
├── pyproject.toml              # python >= 3.11, pydantic
├── docs/adr/                   # architecture decisions, with discarded alternatives
├── src/codebench/
│   ├── contract.py             # H0 - schemas for the six stations
│   ├── orchestrator.py         # fixed workflow with gates
│   ├── stations/               # s1_research ... s6_evidence
│   ├── targets/
│   │   ├── base.py             # the TargetProfile protocol
│   │   ├── java.py
│   │   └── flutter.py
│   ├── llm/client.py           # timeouts, retries, validated returns
│   └── evidence/
│       ├── trace.py            # H0 - JSONL, one object per event
│       └── eval.py             # H5
└── tests/
```

---

## Status

🚧 **Built in public, day by day, 30 working days — 30 Nov 2026 → 18 Jan 2027.**

| | |
|---|---|
| **Today** | day 0 — contract and skeleton land on day 1 |
| Runnable | ❌ not yet — no station implemented |
| Stations | 0 / 6 |
| Hardening | 0 / 7 |
| Profiles | 0 / 4 |

This README is written before the code on purpose. It is the contract I am holding myself to, and it will be edited when reality disagrees — with the commit that shows it.

**What will NOT be here at day 50:** a GUI, multi-agent orchestration, deployment, auth, billing. One narrow thing that works and is measured beats four that look finished.

---

## Design decisions

- **Fixed six-stage workflow, not a graph.** Six fixed stages are measurable; a dynamic graph is not — the moment the flow is dynamic you cannot tell a wrong plan from a wrong step. A graph comes after the fixed version can be measured.
- **The artifact is the contract, not the prompt.** If the file shape lives in a prompt, every format change is a text change nobody reviews.
- **Telemetry on by default, one JSON object per line.** Telemetry that must be switched on is not on, and a `.jsonl` survives a crash because the event was already written.
- **Content hash, not path, in the state.** A path can point to a file that changed; a `sha256` cannot.
- **Deterministic where possible.** Same input, same output — or the eval measures noise.

---

## What I learned from the previous version

This is the third time I build a pipeline like this. The first one worked and could not be measured. The second one could be measured and was locked to one company's stack — 13k lines, real users, and eleven defects I catalogued afterwards:

1. No schema validated the agent's output — failures happened silently.
2. State was stored by reference: the file could change and the trace would not see it.
3. A refactor dropped the timeouts the first version had.
4. No `run_id` — two executions were indistinguishable.
5. Two sources of truth: the agent definition and the prompt said different things.
6. It depended on a local repository path, so it could not run in CI.
7. Context packing had no token budget and truncated silently.
8. Frontmatter parsed by hand instead of a YAML parser.
9. A malformed agent file was skipped without a word.
10. The menu pointed at a script that no longer existed.
11. Inconsistent flag names between entry points.

Every one of them is a hardening layer here. That is why H0–H2 come before any feature.

---

## Roadmap

| Week | Dates | Builds | Hardening |
|---|---|---|---|
| 1 | 30 Nov – 5 Dec | contract, trace, fixed workflow | H0 · H1 |
| 2 | 7 – 12 Dec | station 1, resilience | H2 |
| 3 | 14 – 18 Dec | station 2, guardrail | H3 |
| — | 19 Dec – 3 Jan | *offline* | — |
| 4 | 4 – 9 Jan | stations 3 and 4, idempotency, cost | H4 |
| 5 | 11 – 16 Jan | `impl_spec`, java + flutter profiles | H5 |
| 6 | 18 Jan | evidence report, this README finalised | H6 |

---

## License

[MIT](LICENSE) © 2026 Erick Costa Silva.
