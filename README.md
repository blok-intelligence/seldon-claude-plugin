# Seldon for Claude Code

Run AI personas against your live product to find what breaks and why — from inside your agent.

This repo is a [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces).
It contains connection config and skills only; the Seldon MCP server itself is hosted, so there is
nothing to build or install locally.

## Install

You need a Seldon API key — **Settings → API keys** in the [Seldon app](https://app.seldon.com). The
value is shown once.

```bash
export SELDON_API_KEY=seldon_your_api_key    # add to your shell profile to persist
```

Then, in Claude Code:

```
/plugin marketplace add blok-intelligence/seldon-claude-plugin
/plugin install seldon@seldon
```

Restart, and `/mcp` should list `seldon` as connected.

### Just the MCP server, without the plugin

If you only want the tools and not the skills:

```bash
claude mcp add --transport http seldon https://app.seldon.com/mcp \
  --header "X-API-Key: seldon_your_api_key"
```

Add `--scope user` to make it available in every project rather than just the current one.

## What you get

**Three skills**, so the agent knows how to use the tools rather than just having them:

| Skill | Covers |
| --- | --- |
| `seldon-qa` | Authoring, running and iterating on a `seldon.yml` QA baseline; reading verdicts and fixing what failed |
| `seldon-experiments` | Choosing the right experiment kind, running persona and messaging experiments, finding past runs |
| `seldon-results` | Interpreting summaries, findings, severities, coverage caveats, QA verdicts and messaging scoreboards |

**Twenty tools** over the hosted MCP server:

*QA baselines as code* — `validate_seldon_yaml`, `create_baseline`, `add_test_case`, `run_seldon`,
`list_baselines`, `list_baseline_runs`, `get_report`, `get_run_logs`, `cancel_baseline_run`

*Persona experiments* — `list_personas`, `list_experiments`, `create_experiment`, `run_experiment`,
`get_experiment_status`

*Messaging experiments* — `create_messaging_experiment`, `run_messaging_experiment`,
`get_messaging_status`, `get_messaging_results`

*Reading a run* — `get_summary`, `search_knowledge`

## Try it

```
Write me a seldon.yml covering login and checkout for https://staging.example.com, then run it
```

```
Run a persona experiment on our pricing page — I want to know whether a technical buyer can
figure out which plan they need
```

```
What did experiment <id> find, and what should I fix first?
```

## Configuration

| Variable | Required | Default |
| --- | --- | --- |
| `SELDON_API_KEY` | Yes | — |
| `SELDON_BASE_URL` | No | `https://app.seldon.com` |

### If tools fail with "Invalid or revoked API key"

`claude mcp list` showing **✔ Connected is not proof your key is valid.** The server accepts the
connection whenever a key header is present and only checks the key when a tool actually runs. So an
unset or misspelled `SELDON_API_KEY` looks connected and then fails on every call.

If you see that error, check the variable is exported in the environment Claude Code was launched from:

```bash
echo $SELDON_API_KEY     # should print seldon_...
```

Setting it in a shell after Claude Code started will not reach it — restart Claude Code.

Keys are tenant-scoped: an id you do not own returns "not found" rather than a permission error, so it
cannot be used to confirm that someone else's id exists.

## Links

- [Seldon](https://www.seldon.com)
- [Get an API key](https://app.seldon.com)
