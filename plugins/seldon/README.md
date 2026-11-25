# Seldon

Run AI personas against your live product to find what breaks and why — from inside your agent.

Seldon drives real browser sessions against a URL you own. It answers three different questions, and
picking the right one matters more than anything else: **QA baselines** tell you whether a flow is
broken, **usability studies** tell you where people get stuck, and **messaging experiments** tell you
which copy performs better.

This plugin is configuration and documentation only. It ships no executables, no hooks and no local
server. Everything it does happens over one hosted MCP server.

## What's in the plugin

| Component | What it is |
| --- | --- |
| `.mcp.json` | One remote MCP server, `seldon`, over HTTPS |
| `skills/seldon-qa` | Authoring, running and iterating on a `seldon.yml` QA baseline |
| `skills/seldon-experiments` | Choosing an experiment kind; persona, usability and messaging runs |
| `skills/seldon-results` | Reading summaries, findings, verdicts and scoreboards without over-claiming |

## What it sends, and where

The plugin adds a single HTTP MCP server pointed at `https://mcp.seldon.com/mcp` and nothing else. It
runs no code on your machine.

- **Your API key** is collected by Claude Code's configuration prompt as a `sensitive` option, stored
  in your OS keychain (macOS) or `~/.claude/.credentials.json`, and sent to that endpoint only, as an
  `X-API-Key` header. It is never written into this repository or a settings file.
- **What a tool call sends** is whatever that tool takes: the text of your `seldon.yml` or
  `<name>.usability.yml`, a target URL, goals and persona selections, and the ids of runs you ask to
  read back. Seldon's browser agents then load the URL you gave them from Seldon's cloud.
- **Credentials for the site under test** are yours to place. Either commit them to `seldon.yml` under
  `run:` (anything committed there is as exposed as the repo is — use a throwaway test account) or
  register them once with
  `create_link`, which stores them encrypted server-side and returns only a masked preview afterwards.
- **The endpoint is fixed.** The plugin talks to `https://mcp.seldon.com/mcp` and nowhere else; there
  is no option that can repoint it. To reach a non-production Seldon backend, add that server yourself
  with `claude mcp add` rather than through the plugin.

Seldon's [privacy policy](https://www.seldon.com/privacy) and [terms](https://www.seldon.com/terms)
cover what the service does with a run's data.

## Install

You need a Seldon API key — **Settings → API keys** in the [Seldon app](https://app.seldon.com). The
value is shown once.

```text
/plugin marketplace add blok-intelligence/seldon-claude-plugin
/plugin install seldon@seldon
```

The install prompts for the key with masked input. Run `/plugin configure seldon@seldon` any time to
change it. `/mcp` should then list `seldon` as connected.

| Option | Required | Default |
| --- | --- | --- |
| `api_key` | Yes | — |

## Tools

Twenty-seven tools over the hosted MCP server. Your workspace's plan decides which are offered, so a
QA-only workspace will not see the persona and messaging tools.

*QA baselines as code* — `validate_seldon_yaml`, `create_baseline`, `add_test_case`, `run_seldon`,
`list_baselines`, `list_baseline_runs`, `get_report`, `get_run_logs`, `cancel_baseline_run`

*Usability studies as code* — `validate_usability_study`, `launch_usability_study`

*Persona experiments* — `list_personas`, `list_experiments`, `create_experiment`, `run_experiment`,
`rerun_experiment`, `get_experiment_status`

*Messaging experiments* — `create_messaging_experiment`, `run_messaging_experiment`,
`get_messaging_status`, `get_messaging_results`

*Reading a run* — `get_summary`, `search_knowledge`

*Logins* — `list_links`, `create_link`, `update_link`, `delete_link`

## Try it

```text
Write me a seldon.yml covering login and checkout for https://staging.example.com, then run it
```

```text
Run a usability study on our pricing page — can a technical buyer figure out which plan they need?
```

```text
What did experiment <id> find, and what should I fix first?
```

## If tools fail with "Invalid or revoked API key"

`claude mcp list` showing **✔ Connected is not proof your key is valid.** The server accepts the
connection whenever a key header is present and only checks it when a tool actually runs, so a wrong
key looks connected and then fails on every call. Re-enter it with `/plugin configure seldon@seldon`
and check the key has not been revoked under **Settings → API keys**.

Ids are tenant-scoped: an id you do not own returns "not found" rather than a permission error.

## Licence

[MIT](./LICENSE). © Blok Intelligence Inc.
