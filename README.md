# Seldon for Claude Code

Run AI personas against your live product to find what breaks and why — from inside your agent.

This repo is a [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces).
It contains connection config and skills only; the Seldon MCP server itself is hosted, so there is
nothing to build, compile or run locally. The plugin ships no executables and no hooks.

## Install

You need a Seldon API key — **Settings → API keys** in the [Seldon app](https://app.seldon.com). The
value is shown once.

In Claude Code:

```text
/plugin marketplace add blok-intelligence/seldon-claude-plugin
/plugin install seldon@seldon
```

The install prompts for your API key with masked input. Nothing to export and nothing to edit: Claude
Code stores it in your OS keychain (macOS) or `~/.claude/.credentials.json` (elsewhere), never in a
settings file and never in this repository. Run `/plugin configure seldon@seldon` any time to change it.

To script the install instead:

```bash
claude plugin install seldon@seldon --config api_key=seldon_your_api_key
```

`/mcp` should then list `seldon` as connected.

### Options

| Option | Required | Default |
| --- | --- | --- |
| `api_key` | Yes | — |

The endpoint is fixed at `https://mcp.seldon.com/mcp`. To point a session at a
non-production backend, add the server directly with `claude mcp add` instead of using the plugin.

### Just the MCP server, without the plugin

If you only want the tools and not the skills:

```bash
claude mcp add --transport http seldon https://mcp.seldon.com/mcp \
  --header "X-API-Key: seldon_your_api_key"
```

Add `--scope user` to make it available in every project rather than just the current one.

## What you get

**Three skills**, so the agent knows how to use the tools rather than just having them:

| Skill | Covers |
| --- | --- |
| `seldon-qa` | Authoring, running and iterating on a `seldon.yml` QA baseline; storing the logins runs sign in with; reading verdicts and fixing what failed |
| `seldon-experiments` | Choosing the right experiment kind, running usability studies, persona and messaging experiments, finding past runs |
| `seldon-results` | Interpreting summaries, findings, severities, coverage caveats, QA verdicts and messaging scoreboards |

**Twenty-seven tools** over the hosted MCP server. Which ones are offered depends on your workspace's
plan — a QA-only workspace does not see the persona and messaging tools.

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

```
Write me a seldon.yml covering login and checkout for https://staging.example.com, then run it
```

```
Run a usability study on our pricing page — I want to know whether a technical buyer can
figure out which plan they need
```

```
What did experiment <id> find, and what should I fix first?
```

## What leaves your machine

The plugin configures one remote MCP server at `https://mcp.seldon.com/mcp` and nothing else. Your API
key is sent to that endpoint as an `X-API-Key` header; a tool call sends whatever that tool takes —
your `seldon.yml` or study text, a target URL, goals, persona selections and run ids. Seldon's browser
agents then load the URL you gave them from Seldon's cloud. Seldon's
[privacy policy](https://www.seldon.com/privacy) and [terms](https://www.seldon.com/terms) cover what
the service does with a run's data.

Credentials for the site under test are yours to place: commit them to `seldon.yml` under `run:` (treat
that like a `.env` value — use a throwaway account) or register them once with `create_link`, which
stores them encrypted server-side and only ever returns a masked preview afterwards.

## If tools fail with "Invalid or revoked API key"

`claude mcp list` showing **✔ Connected is not proof your key is valid.** The server accepts the
connection whenever a key header is present and only checks it when a tool actually runs, so a wrong
key looks connected and then fails on every call.

Re-enter it with `/plugin configure seldon@seldon`, and check the key has not been revoked in
**Settings → API keys**.

Keys are tenant-scoped: an id you do not own returns "not found" rather than a permission error, so it
cannot be used to confirm that someone else's id exists.

## Links

- [Seldon](https://www.seldon.com)
- [Get an API key](https://app.seldon.com)
- [Privacy policy](https://www.seldon.com/privacy) · [Terms](https://www.seldon.com/terms)

## Licence

[MIT](./LICENSE). © Blok Intelligence Inc.
