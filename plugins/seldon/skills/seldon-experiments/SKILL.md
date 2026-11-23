---
name: seldon-experiments
description: Choose and run the right kind of Seldon experiment — QA baselines, usability studies as code, ad-hoc persona experiments, or messaging/copy tests. Use when the user wants to test a flow with real personas, find usability friction, ask "why do users drop off", compare copy variants or headlines, or does not yet know which experiment type fits their question.
---

# Running Seldon experiments

Seldon answers three different questions. Picking the right one matters more than anything else you
do — the wrong kind answers a question nobody asked.

## Pick the kind first

| You want to know | Kind | Entry point |
| --- | --- | --- |
| "Does this flow still work?" — a pass/fail regression check against known-good behaviour | **QA** | `run_seldon` (see the `seldon-qa` skill) |
| "Why is this hard to use?" — where real people get stuck, confused, or give up | **Usability / persona** | `launch_usability_study`, or `create_experiment` → `run_experiment` |
| "Which wording performs better?" — comparing copy variants | **Messaging** | `create_messaging_experiment` → `run_messaging_experiment` |

The distinction that catches people out: **QA tells you whether something is broken; a usability run
tells you whether something is bad.** A checkout flow can pass every QA case and still lose users at
step three. If the user's question contains "why", "confusing", "drop off", "friction", or
"experience", they want a usability run, not QA.

If the user has not said which, ask — or infer from the question and say which you picked and why.

> Tools are offered according to the workspace's plan. If `launch_usability_study` or the messaging
> tools are not in your tool list, that workspace is not entitled to them — say so rather than
> improvising a substitute.

---

## Usability runs: two routes to the same thing

Personas — simulated users with distinct backgrounds, goals and patience — browse a real URL trying
to achieve a goal. You get findings about what got in their way. Both routes produce an ordinary
classic experiment, followed with `get_experiment_status` and `get_summary`.

| Route | Use it when |
| --- | --- |
| **A study file** (`<name>.usability.yml` → `validate_usability_study` → `launch_usability_study`) | The study should be committed, reviewed and re-run — it pins the hypothesis, the experiment type and the device frame. **Prefer this.** |
| **Ad-hoc** (`create_experiment` → `run_experiment`) | A one-off question, or the user wants to pick personas by id and nothing is being kept. |

### Route A — a study as code

`<name>.usability.yml`, schema v1, at the project root:

```yaml
version: 1
experiment:
  title: Trial signup — first-time visitors
  goal: Sign up and invite one teammate
  hypothesis: The pricing step causes hesitation
  type: Onboarding
target:
  device_type: Website
  frame_type: Desktop
  url: https://staging.example.com/signup
  credentials:
    username: qa@example.com
    password: hunter2
personas:
  - Budget-conscious founder
```

Every key above except `credentials` is required.

| Field | Rule |
| --- | --- |
| `experiment.goal` | **The most important field.** What each persona tries to do. See "Write the goal well". |
| `experiment.hypothesis` | What you expect to learn. Required — it is what the summary is written against. |
| `experiment.type` | One of `Onboarding`, `Task Completion`, `Churn`, `Conversion`, `Navigation Clarity`, `Feature Adoption`, `Trust Signals`. |
| `target.device_type` | `Website` is the only value today. |
| `target.frame_type` | `Desktop` or `Mobile`. |
| `target.url` | A full `http(s)` URL. |
| `target.credentials` | `username` + `password`, or the literal `none` to force a logged-out run. Omit it and a workspace link for that host may supply one — see the `seldon-qa` skill on links. |
| `personas` | Persona **names**, not ids — at least one. Get them from `list_personas`. |

Then:

1. `validate_usability_study` with the raw `study_yaml`. Free, and it catches shape errors before you
   spend a run. On failure it returns `error`, a `hint` and the annotated `schema` to fix against.
   It does **not** check that persona or type names exist — only the server can.
2. `launch_usability_study` with the same `study_yaml`. It re-validates, creates and dispatches in one
   call, and returns `experiment_id` and `app_url`. Pass `url` to retarget a preview deploy without
   editing the file.

If a persona or type name does not resolve, the launch is rejected whole: **nothing is created**, and
the error names what failed to resolve. Fix the name, do not retry blind.

**Local URLs are refused.** The personas run from Seldon's cloud and cannot reach `localhost` or
`127.0.0.1`. Tell the user to run it themselves from a terminal — `seldon run --file <study>` opens a
temporary tunnel for exactly this — or point the study at a deployed URL.

### Route B — ad-hoc

1. `list_personas` — **start here.** `persona_ids` refers to these ids and there is no other way to
   discover them. Three to five is a sensible default: enough to see whether a problem is universal
   or specific to one kind of user, without a long run.
2. `create_experiment` with `url`, `goal`, `persona_ids`, and optionally `title`, `hypothesis` and
   `experiment_type_id`. **Creating does not start it.**
3. `run_experiment` with `experiment_id` and optionally `runs`.

`runs` is **per persona**, not total: 4 personas at `runs: 3` is 12 browser sessions. It defaults to 3.
More runs mean more evidence that a finding is real rather than one persona having one bad session —
a finding in 3 of 3 runs is a different claim from one in 1 of 3, and the summary tells you which.

### Write the goal well

**State an outcome the persona wants, not a script.** The goal is the lever that decides how much of
the product gets explored.

- Narrow: `find the pricing for the team plan` — the persona goes to pricing and stops. Correct if
  that is your question.
- Broad: `decide whether this product is worth buying for a 10-person team` — the persona ranges
  across pricing, features, comparisons and social proof.

If a run explored less than you expected, the goal was too narrow. That is not a bug.

### Re-running

**`run_experiment` on an experiment that already finished overwrites its stored results**, permanently
losing the original run. That is almost never what someone means by "run it again".

Use **`rerun_experiment`** instead. Despite the name it does not re-dispatch the id you give it: it
reads that experiment's config, creates an **entirely new experiment with a new id**, and dispatches
that one. The id you passed in is only ever read — its status, results and summary are untouched.

So: it returns `new_experiment_id`, and **that** is the id to poll with `get_experiment_status` and
read with `get_summary`. Polling the original will show you the old run, finished, forever.

Two consequences of it being a copy:

- **Only `title`, `url`, `goal`, `hypothesis`, `personas` and the experiment type are carried over.**
  Nothing else is. In particular `runs` is **not** inherited — the new run uses the server default of
  3 per persona unless you pass `runs` yourself. If the original ran 5 per persona and you want to
  compare like with like, pass `runs: 5`.
- **The copy keeps the original's title**, so the catalog will show two rows with the same name. Tell
  the user which id is the new one, or they will read the wrong run.

`rerun_experiment` only works on classic experiments. To re-run a QA baseline, call `run_seldon`
again; it always runs the committed `seldon.yml` fresh as a new run.

### Poll, then read

`get_experiment_status` returns status, stage and completed/total runs, and `isTerminal` tells you
when to stop. When a run looks stuck it also checks the workflow and says what it found. Do not poll
tightly — these are real browser sessions and take minutes, not seconds.

Statuses: `Draft` (created, not dispatched), `Running`, `Completed`, `Failed`, `Cancelled`, `Timeout`
(stalled and cleaned up), `Archived`.

Only `Completed` guarantees a result worth reading. On `Failed` or `Timeout`, still call `get_summary`
— partial evidence is often present, and the summary says what coverage it had. Interpreting the
output is a skill of its own: see `seldon-results`.

---

## Messaging experiments

Copy variants scored against personas, who are simulated reading them and predicted to act or bounce.

1. `create_messaging_experiment` — pass `variations` (the variants as plain strings), a `goal`, and
   optionally `title`, `content_type` (e.g. email, push, landing headline) and `persona_ids`. Set
   `generate_variations_via_llm` to have starting variants written for you.
2. `run_messaging_experiment` — dispatch. It scores your variants, then iteratively generates and
   scores better ones until they beat the originals.
3. `get_messaging_status` — poll. Reports phase and iteration round.
4. `get_messaging_results` — the scoreboard. Final round by default; `full` for every round.

Give it at least two genuinely different variants. Variants that differ by a word produce a
scoreboard you cannot act on.

---

## Finding what you already ran

`list_experiments` — the catalog, newest first. This is how you find the id of a run that already
happened; every read tool needs an `experiment_id` and nothing else produces one.

| Argument | Use |
| --- | --- |
| `search` | Free text over title, goal, hypothesis, url and findings. |
| `status` | e.g. `Completed`. |
| `persona_id` | Only experiments this persona took part in. |
| `created_by_me` | Only what this API key's own user created. |
| `limit` / `offset` / `sort` | Page through history; `sort` is `newest` (default) or `oldest`. |
| `full` | Every row rather than the first 40. |

Each row carries a `kind` of `classic` or `qa` — that tells you whether `get_report` applies as well
as `get_summary`. When you passed a filter, an empty result means "no match", not "no data".

## Practical notes

- **Runs are asynchronous and slow.** Create, dispatch, then poll. Do not block a conversation waiting.
- **Ids are tenant-scoped.** An id you do not own returns "not found", never a permission error.
- **Creating is not running**, except for `launch_usability_study`, which does both.
- **Write tools take an `idempotency_key`.** Pass one when retrying a dispatch you are not sure landed;
  without it a retry can start a second unattended run.
