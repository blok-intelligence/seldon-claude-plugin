---
name: seldon-experiments
description: Choose and run the right kind of Seldon experiment — QA baselines, persona usability experiments, or messaging/copy tests. Use when the user wants to test a flow with real personas, find usability friction, ask "why do users drop off", compare copy variants or headlines, or does not yet know which experiment type fits their question.
---

# Running Seldon experiments

Seldon runs three kinds of experiment. Picking the right one matters more than anything else you do —
the wrong kind answers a question nobody asked.

## Pick the kind first

| You want to know | Kind | Tools |
| --- | --- | --- |
| "Does this flow still work?" — a pass/fail regression check against known-good behaviour | **QA** | `run_seldon` (see the `seldon-qa` skill) |
| "Why is this hard to use?" — where real people get stuck, confused, or give up | **Classic (persona)** | `create_experiment` → `run_experiment` |
| "Which wording performs better?" — comparing copy variants | **Messaging** | `create_messaging_experiment` → `run_messaging_experiment` |

The distinction that catches people out: **QA tells you whether something is broken; a classic experiment
tells you whether something is bad.** A checkout flow can pass every QA case and still lose users at step
three. If the user's question contains "why", "confusing", "drop off", "friction", or "experience", they
want a classic experiment, not QA.

If the user has not said which, ask — or infer from the question and say which you picked and why.

---

## Classic (persona) experiments

Personas — simulated users with distinct backgrounds, goals and levels of patience — browse a real URL
trying to achieve a goal. You get findings about what got in their way.

### 1. Pick personas

`list_personas` — **start here.** `persona_ids` refers to these ids and there is no other way to discover
them. Returns each persona's name, id and description.

Choose personas whose profile matches who you actually care about. Three to five is a sensible default:
enough to see whether a problem is universal or specific to one kind of user, without a long run.

### 2. Create

`create_experiment` with:

| Argument | Notes |
| --- | --- |
| `url` | The live page they start on. |
| `goal` | What the persona is trying to accomplish. **This is the most important field.** |
| `persona_ids` | From `list_personas`. |
| `title` | For your own retrieval later. |

Creating does **not** start it.

**Write the goal as an outcome the persona wants, not a script.** The goal is the lever that decides how
much of the product gets explored.

- Narrow: `find the pricing for the team plan` — the persona goes to pricing and stops. Correct if that
  is your question.
- Broad: `decide whether this product is worth buying for a 10-person team` — the persona ranges across
  pricing, features, comparisons and social proof.

If a run explored less than you expected, the goal was too narrow. That is not a bug.

### 3. Run

`run_experiment` — dispatches it. `runs` is **per persona**, not total: 4 personas at `runs: 3` is 12
browser sessions. It defaults to 3 when omitted.

More runs mean more evidence that a finding is real rather than one persona having one bad session. When
you read the results, a finding that appeared in 3 of 3 runs is a different claim from one that appeared
in 1 of 3 — and the summary tells you which.

### 4. Poll

`get_experiment_status` — returns status, stage and completed/total runs, and tells you when it is
terminal. Do not poll tightly; these are real browser sessions and take minutes, not seconds.

Statuses you will see: `Draft` (created, not dispatched), `Running`, `Completed`, `Failed`, `Cancelled`,
`Timeout` (stalled and cleaned up), `Archived`.

Only `Completed` guarantees a result worth reading. On `Failed` or `Timeout`, still call `get_summary` —
partial evidence is often present, and the summary will tell you what coverage it had.

### 5. Read

`get_summary`. See the `seldon-results` skill — interpreting the output is a skill of its own.

---

## Messaging experiments

Copy variants scored against personas: which headline, subject line or CTA actually moves people.

1. `create_messaging_experiment` — pass the `variations` you want compared, a `goal`, and optionally
   `persona_ids`. Set `generate_variations_via_llm` if you want Seldon to propose variants alongside yours.
2. `run_messaging_experiment` — dispatch.
3. `get_messaging_status` — poll. Reports phase and iteration round.
4. `get_messaging_results` — the scoreboard.

Give it at least two genuinely different variants. Variants that differ by a word produce a scoreboard
you cannot act on.

---

## Finding what you already ran

`list_experiments` — the catalog, newest first. This is how you find the id of a run that already
happened; every read tool needs an `experiment_id` and nothing else produces one.

| Argument | Use |
| --- | --- |
| `search` | Free text over title, goal and findings. |
| `status` | e.g. `Completed`. |
| `persona_id` | Only experiments this persona took part in. |
| `limit` / `offset` | Page through history. |

Each row carries a `kind` of `classic` or `qa` — that tells you whether `get_report` applies as well as
`get_summary`.

## Practical notes

- **Runs are asynchronous and slow.** Create, dispatch, then poll. Do not block a conversation waiting.
- **Ids are tenant-scoped.** An id you do not own returns "not found", never a permission error.
- **Creating is not running.** `create_*` then `run_*`, always two steps.
