---
name: seldon-qa
description: Author, run and iterate on Seldon QA baselines — a seldon.yml of plain-English test cases run by AI agents against a live URL. Use when the user wants end-to-end QA, regression testing, "test my app", "does the signup flow still work", writing or fixing a seldon.yml, or reading a QA report to fix what failed.
---

# Seldon QA — baselines as code

A **baseline** is a QA test plan written as plain English in a `seldon.yml` file at the repo root. An AI
agent opens a real browser, performs each case's steps against a live URL, and a judge decides whether the
acceptance criterion was met. You get a per-case verdict with evidence.

The file is the source of truth. There is no separate authoring UI to keep in sync — you edit
`seldon.yml`, commit it, and run it.

## The loop

```
author seldon.yml → validate → run → read report → fix the product → re-run
```

Never skip validate. It is free and catches shape errors before you spend a run on them.

## 1. Author

`seldon.yml` at the repo root:

```yaml
version: 1
name: web-app                        # project identity when there's no git repo
run:
  url: https://staging.example.com   # default target for every case
testCases:
  - name: Login works
    id: login                        # optional but recommended — see "Case ids" below
    steps:
      - go to /login
      - submit valid credentials
    criteria: the user reaches the dashboard
  - name: Checkout completes
    steps:
      - add any product to the cart
      - proceed through checkout with test card 4242 4242 4242 4242
    criteria: an order confirmation with an order number is shown
```

Three fields do the work:

| Field | Rule |
| --- | --- |
| `name` | A plain-English case title. Required. |
| `steps` | Ordered plain-English instructions. Required. Write what a person would do, not selectors. |
| `criteria` | **One** acceptance criterion the verdict is judged against. Required. |

Optional per case: `id`, `url` (a leading `/` joins onto `run.url`; anything else is treated as a separate
site), `skip: true` to exclude it, `only: true` to run just the flagged cases.

### Writing steps and criteria well

This is where most QA runs succeed or fail. The agent reads these as instructions, and the judge reads
the criterion as the definition of done.

**Steps — describe intent, not implementation.**

- Good: `go to /login`, `submit valid credentials`, `open the account menu`
- Bad: `click #login-btn`, `wait 3000ms`, `assert response.status === 200`

CSS selectors, timing hacks and network assertions are not what this runs on. Describe what a person does.

**Criteria — one observable outcome, checkable from the screen.**

- Good: `an order confirmation with an order number is shown`
- Bad: `checkout works` (not observable), `the order is created and the email sends and the cart clears`
  (three criteria — split into three cases)

If you cannot tell from a screenshot whether the criterion was met, rewrite it.

**Give the agent what it cannot guess.** Test credentials, test card numbers, a specific product to pick.
The agent explores, but it cannot invent a valid login.

### Case ids

Every case resolves to an id: your explicit `id`, or the slugified `name` (`"Login works"` → `login-works`).
Verdict history is keyed to that id, so **set an explicit `id` on long-lived cases** — otherwise rewording
the title silently starts a new history.

## 2. Validate

`validate_seldon_yaml` — pass the raw file text. It parses and shape-checks, reporting precise errors.
Run it after every edit, before every run.

`create_baseline` scaffolds a starter `seldon.yml` if the repo has none. `add_test_case` appends one case
(name + steps + criteria) without you hand-editing YAML.

## 3. Run

`run_seldon` — ingests the file and dispatches the run. Returns an `experiment_id`.

| Argument | When to use it |
| --- | --- |
| `url` | Target this run at a preview/PR deploy. Overrides `run.url`. |
| `labels` | Tag the run (e.g. a PR number or commit sha) so you can find it later. |

Re-running an unchanged file is idempotent — no version churn. A new baseline version is recorded only
when case **content** changes (title, steps, criteria, per-case url). Toggling `skip`/`only` or changing
`run.url` does not count as a content change.

`cancel_baseline_run` stops a run mid-flight.

## 4. Read the report

Runs are asynchronous. Poll `get_experiment_status` until it reports terminal, then call `get_report`.

`get_report` returns the per-case ledger — one row per case with a verdict.

| Verdict | Meaning | What to do |
| --- | --- | --- |
| `pass` | The criterion was met. | Nothing. |
| `fail` | The criterion was not met. | A real defect, or a criterion that does not match intent. Read the evidence before assuming which. |
| `blocked` | The case could not be evaluated — an upstream blocker, missing credentials, a page erroring out. | **Not a product verdict.** Fix the blocker and re-run; do not report it as a bug. |
| `timed_out` | The run did not finish this case. | Re-run. Investigate only if it repeats. |

Failed cases carry a triage severity: `blocker`, `major`, `minor`, `cosmetic`. Fix in that order.

Useful arguments:

- `verdict` — narrow to failures. Do this first on a large baseline; a 60-case report is mostly noise
  when 3 cases fail.
- `full` — expand every detail block. Only after you have narrowed down which case you care about.

`get_run_logs` shows what the agent was thinking, step by step. Add `include_actions: true` to also get
what it actually *did* — each click and keystroke with a screenshot URL. This is how you tell "the button
is broken" apart from "the agent never found the button".

## 5. Act on it

Work failures in severity order. For each one:

1. Read the case's verdict reasoning and evidence.
2. Decide: **product defect** or **bad test**?
   - Product defect → fix the code.
   - The criterion did not describe what you actually wanted, or the steps were ambiguous → fix
     `seldon.yml`. This is a legitimate outcome, not a cop-out. Vague criteria produce meaningless verdicts.
3. Re-run. `only: true` on the case you are iterating on keeps the loop fast; remove it before committing.

Treat a `blocked` verdict as a task in its own right — usually credentials, a seeded fixture, or an
environment that is down. Blocked cases tell you nothing about product quality until unblocked.

## Finding earlier runs

- `list_baselines` — every baseline the key can see, across all sources.
- `list_baseline_runs` — runs for a baseline, newest first.
- `get_summary` — the narrative view of a QA run (see the `seldon-results` skill). Complementary to
  `get_report`: the summary is what was learned, the report is which case failed. For a QA run worth
  understanding, read both.
