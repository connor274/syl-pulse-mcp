# Syl Pulse MCP

<!-- mcp-name: io.sylai/pulse -->

Syl Pulse is causal analytics for business questions. Give your agent a dataset and a
question like "Does ad spend drive sales?" and get back the full Pulse report: the
adjusted causal effect with its confidence interval, a by-channel breakdown, caveats and
assumptions, and charts (causal graph, effect, trend). When the question asks about sample
size or test duration, it also returns a spend-cut test design.

All numbers are deterministic statistics. They are never computed by an LLM, and the
figures in the report prose are checked against the computed numbers. Optional LLM help
(dataset cleaning, causal-graph pruning, estimator choice, report prose) is controlled per
step with `config`; see [Engine config](#engine-config).

This repository holds the public listing metadata and connection instructions for the
hosted Pulse MCP server. Pulse itself is a hosted service.

- Endpoint: `https://pulse.sylai.io/mcp` (stateless streamable HTTP)
- Auth: `X-API-Key` header on every request. It is the same key as Syl Guardian; get a
  trial key at <https://sylai.io/guardian>.
- Product page: <https://sylai.io/pulse>
- Docs: <https://sylai.io/guardian/docs> (MCP connector section)

## Connect

Claude Code:

```bash
claude mcp add --transport http syl-pulse https://pulse.sylai.io/mcp \
  --header "X-API-Key: <your key>"
```

Claude Desktop (via `mcp-remote`):

```json
{
  "mcpServers": {
    "syl-pulse": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://pulse.sylai.io/mcp",
               "--header", "X-API-Key:${SYL_API_KEY}"],
      "env": { "SYL_API_KEY": "<your key>" }
    }
  }
}
```

Then ask your agent: *"Use pulse_analyze on this data: does ad_spend drive new_customers,
adjusting for seasonality? How long would a spend-cut test need to run?"*

Every request needs the key, including tool listing. Each successful analysis counts
against your Pulse quota or trial.

## Tool: `pulse_analyze`

Estimates a causal effect from a dataset and answers a business question.

| Argument | Description |
|---|---|
| `question` | Required. The business question, e.g. "Does ad spend drive sales?". |
| `csv` or `data` | Dataset as CSV text or a list of row objects (exactly one). |
| `cause`, `effect` | Optional. Treatment and outcome columns; inferred from the question if omitted. |
| `confounders` | Optional. Columns to adjust for, in addition to those discovered. |
| `outcomes` | Optional. Several outcome columns, for by-channel effects. |
| `time_col` | Optional. Time column, used for the test-design cadence. |
| `include_guardian_verdict` | Also return a `pass` / `caution` / `hold` verdict, computed locally (uses no Guardian quota). |
| `include_charts` | Attach chart images (default true). |
| `include_test_design` | Force the spend-cut test design on or off; by default it appears only when the question asks about sample size, power or test duration. |
| `estimator` | Optional. One of `ols_adjusted`, `aipw`, `ipw`, `psm`, `dml_forest`, or `rule` (the deterministic default). Omit to follow `config.estimator`. |
| `config` | Optional object choosing `rule` or `llm` per step: `cleaning`, `discovery`, `estimator`, `summary`. See [Engine config](#engine-config). |

The result's first content block is the report text, followed by chart images. The
structured result has the effect size with its confidence interval, per-channel effects,
confidence, caveats, and the optional verdict and test design. It also reports the
`estimator_used`, the resolved `config`, and `engine_steps`. An invalid `config` (unknown
key or a value other than `rule` or `llm`) comes back as a tool error, and misspelled
argument names are rejected rather than ignored.

### Engine config

`config` lets you choose, per step, whether the engine uses fixed rules or an LLM. Each
value is `"rule"` or `"llm"`. Omitted keys use the defaults:

| Step | Default | What it controls |
|---|---|---|
| `cleaning` | `llm` | Dataset cleaning: derived columns and null imputation. The LLM only picks a strategy per column; the values are computed deterministically, and your cause and effect columns are never imputed. |
| `discovery` | `llm` | Pruning and orienting the causal graph, naming cause and effect from your question when you omit them, and reading your question to decide on optional extras such as a subgroup breakdown, a robustness check or a test design. |
| `estimator` | `rule` | Choosing the estimator. With `llm`, the model picks from the supported set based on each method's documentation, and may also choose a matching method. With `rule`, the deterministic default is used. |
| `summary` | `llm` | `llm` writes the report prose; `rule` uses a fixed template. |

The model key is provided by the service; you never supply one. `rule` always works. If an
`llm` step cannot run, it falls back to `rule`, and `engine_steps` reports
`{requested, used, fallback_reason}` for each step. Setting all four to `rule` makes no LLM
calls.

Example: `{"config": {"estimator": "llm", "summary": "rule"}}`.

Which settings can change a number: `cleaning: llm` (imputation strategy, derived-column
drops), `discovery: llm` (the adjustment set) and `estimator: llm` (the method).
`summary: llm` changes prose only. For results that don't depend on an LLM at all, set
`cleaning`, `discovery` and `estimator` to `rule`.

For a quick pass/caution/hold check on a single claim, see the sibling server
[Syl Guardian MCP](https://github.com/connor274/syl-guardian-mcp).

## Support

Questions or issues: <https://sylai.io> or connor@sylai.io.

## License

The files in this repository (metadata and documentation) are MIT licensed; see
[LICENSE](LICENSE). The Syl Pulse service is proprietary.
