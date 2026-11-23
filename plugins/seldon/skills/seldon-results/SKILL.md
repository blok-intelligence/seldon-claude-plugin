---
name: seldon-results
description: Read and act on Seldon experiment results — the narrative summary, findings and severities, per-persona breakdown, coverage caveats, QA verdicts, and messaging scoreboards. Use when an experiment has finished and the user wants to know what was learned, what to fix first, or wants evidence for a specific claim.
---

# Reading Seldon results

Getting a result is easy. Reading it correctly — knowing what a finding actually claims, and what the
report is *not* telling you — is the part that goes wrong.

## Which tool for which question

| Question | Tool |
| --- | --- |
| What did we learn? | `get_summary` |
| Which specific test case failed? | `get_report` (QA runs only) |
| What did the agent actually do? | `get_run_logs` |
| Did anyone mention X? | `search_knowledge` |
| Which copy variant won? | `get_messaging_results` |

`get_summary` and `get_report` are complementary, not alternatives. **The summary is what was learned;
the report is which case failed.** For a QA run worth understanding, read both.

`get_summary` works for both classic and QA runs — you do not need to know the kind before calling
it, and the response's `kind` says which you got. A study launched from a `<name>.usability.yml` is a
classic run afterwards, so it reads back exactly like one.

---

## The summary

### Availability — check this first

`get_summary` always answers; it does not error just because there is no summary yet. It reports an
availability state:

| State | Meaning |
| --- | --- |
| `available` | A summary exists. Read it. |
| `not_summarized_yet` | Still running, or it never produced one. **Poll, do not retry in a loop.** |
| `legacy_schema` / `unparseable` | Raw text is returned instead of structure. Read the raw text; do not treat missing fields as "no findings". |

Do not report "no issues found" when the state is anything other than `available`. That is the single
most damaging misreading of this API.

### Structure

- **Question** — what the run set out to answer.
- **Answer** — the direct answer, in one or two sentences. If someone wants a one-line takeaway, this is it.
- **Synthesis** — the reasoning connecting the evidence to the answer.
- **Key observations** — recurring patterns with how many personas hit each.
- **Per-persona breakdown** — runs and findings per persona. This is how you tell a universal problem from
  one persona's difficulty.
- **Findings** — the actionable items. See below.
- **Business insights** — impact framed around outcomes rather than UI.
- **Coverage** — how many personas and runs actually contributed usable evidence. See "Caveats".

### Findings

Each finding carries:

| Field | What it is |
| --- | --- |
| `severity` | `Critical`, `High`, `Medium`, `Low` |
| `title` / `summary` | What went wrong |
| `personas` | Who hit it |
| `context` | The situation it happened in |
| `root_cause` | Why, when it could be determined |
| `steps_to_reproduce` | How to see it yourself |
| `options` | Candidate fixes, each with pros and cons; one may be marked recommended |
| `citations` | Quotes tied to a persona, run and step |
| `screenshot_urls` | Visual evidence |

**Severity is about user impact, not engineering effort.** A `Critical` finding may be a one-line copy
change; a `Low` one may need a rebuild. Do not reorder by how hard something looks to fix.

### How to act on findings

1. **Sort by severity, then by how many personas hit it.** A `High` that three of four personas hit is a
   different problem from a `High` that one hit — the first is a product defect, the second may be an
   audience mismatch worth understanding before changing anything.
2. **Read the citations before acting.** The finding is a claim; the citations are the evidence. If a
   quote does not support the claim as strongly as the title suggests, say so rather than passing the
   title along.
3. **Use `options` as candidates, not instructions.** They come with pros and cons because the trade-off
   is real. Pick deliberately; the recommended one is a default, not a verdict.
4. **Reproduce before you fix.** `steps_to_reproduce` plus `screenshot_urls` should let you see it
   yourself. If you cannot reproduce it, that is worth reporting.
5. **Re-run after fixing.** Same personas, same goal. A finding that persists was not fixed.

The rendered summary is capped at the most significant findings, with the rest listed by title. Pass
`full` to expand everything — do that when you are working through a backlog rather than triaging.

### Caveats you must carry forward

- **Coverage.** When present, it says how many of the selected personas contributed usable evidence. If
  coverage is partial, the report is partial — say so when you report the result. Silence here reads as
  "we checked everything" when you did not.
- **Personas are simulated.** Findings are strong signal about where a product is confusing. They are not
  a substitute for a claim about what real users measurably did.
- **A run that explored less than expected usually had a narrow goal.** Check the goal before concluding
  a section of the product is fine — nobody may have visited it.

---

## QA reports

`get_report` returns one row per test case.

| Verdict | Meaning |
| --- | --- |
| `pass` | The criterion was met. |
| `fail` | It was not. A real defect, or a criterion that did not match intent. |
| `blocked` | The case could not be evaluated at all — credentials, an upstream error, an environment problem. **Not a product verdict.** |
| `timed_out` | The run did not finish this case. Re-run before investigating. |

Failed cases carry triage severity: `blocker`, `major`, `minor`, `cosmetic`.

Never count `blocked` or `timed_out` as failures when reporting a pass rate. They are absent data, not
bad results, and conflating them turns an environment problem into a fake quality regression.

Use `verdict` to narrow to failures first, then `full` on the ones you care about.

---

## Messaging results

`get_messaging_results` returns per-variant scores across rounds:

| Metric | Meaning |
| --- | --- |
| `goalCompletionRate` | How often the persona completed the goal |
| `actionRate` | How often they took the intended action |
| `bounceRate` | How often they left |
| `avgGoalLikelihood` | Modelled likelihood of converting |

Plus a recommended variant, the reasoning behind it, and top improvements.

Read the **spread**, not just the winner. If variants score within noise of each other, the honest answer
is "the copy is not the bottleneck" — report that rather than crowning a winner. A recommendation always
comes back; it does not mean the difference is meaningful.

`get_messaging_results` shows the final round by default; pass `full` for every iteration round. The
per-persona prediction matrix is omitted either way — it is very large and rarely what you want.

---

## Digging into evidence

`search_knowledge` — semantic search over what personas did and said across runs. Use it when you have a
specific question the summary did not answer: "did anyone mention pricing?", "what happened at checkout?"

| Argument | Use |
| --- | --- |
| `query` | Natural language. It is semantic, so phrase it as a question. |
| `experiment_id` | Scope to one run. |
| `persona_id` | Scope to one persona's experience. |
| `persona_knowledge_only` | **Set this whenever `persona_id` is set.** Without it, experiment-level sources that aggregate across every persona come back too, and you will attribute a group observation to one persona. |
| `run_number` | Scope to a single simulation run index. |
| `top_k` | How many results (default 10, server max 100). |
| `similarity_threshold` | Minimum cosine similarity, 0-1. Raise it when the results are loosely related. |

It returns ranked excerpts, not an answer — there is no model in the loop. That is the point: it is
evidence, not another opinion. Quote what it returns rather than paraphrasing it into a stronger claim.

`get_run_logs` shows the agent's step-by-step reasoning. Add `include_actions: true` for what it actually
did — clicks and typing, each with a screenshot. This is how you distinguish "the feature is broken" from
"the agent never found the feature", which look identical in a summary.

It returns a bounded page in server order, not a live stream: to advance, call again with `since` set to
the last entry's timestamp. Narrow with `type` (`thinking` is the default; also `action`, `analysis`,
`status`, `recommendation`, `notes`, or `all`) and `source` (e.g. `UI Agent`, `Persona: {name}`). A
`type: action` filter returns nothing unless `include_actions` is set — only thinking survives
consolidation.

---

## Reporting results to a human

- Lead with the answer, not the method.
- Give severity and persona counts together — "High, hit by 3 of 4 personas" is decision-ready in a way
  that "High" alone is not.
- State coverage caveats explicitly. If it was partial, say so in the same breath as the finding.
- Distinguish what the run **observed** from what you **infer**. The summary already does this; preserve it.
- Link the evidence. Screenshots and quotes are what make a finding actionable rather than arguable.
