# Measuring tool input schema fidelity

[SEP-2106](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/seps/2106-json-schema-2020-12.md) lets an MCP tool's `inputSchema` and `outputSchema` use the full **JSON Schema 2020-12** vocabulary. Before it, a tool input was limited to `type` / `properties` / `required`; now `oneOf`, `$ref` / `$defs`, `if` / `then`, `const` discriminators and 2020-12 tuples (`prefixItems`) are all legal. This topic shows that capability in action and probes whether a sample of models and providers actually respect those keywords when they generate a tool call.

*Last updated 2026-07-28.*

## What models + providers do with input schemas

The grid below includes results from handing each tool's verbatim `inputSchema` to a selection of models and providers over two buckets: open-weight models via Hugging Face's Inference Providers router (each pinned to one verified serving provider) and closed models via each vendor's own SDK.

### How to read these results

- A cell counts as 'honored' if the returned arguments both validate against the verbatim schema and respect the demonstrated keyword (such as populating exactly one `oneOf` branch, handling a 2-element tuple, using the right `const` discriminator). Percentages are **honored-of-served**: honored trials out of the trials actually served (transport `error` rows, like rate-limit noise, are excluded from the denominator).
- A single honored-of-served number pools two different things: whether the provider's API even *accepted* the schema (a deterministic host gate) and, given acceptance, whether the model filled it out correctly (model skill).  So don't rank models by the pooled number: an Anthropic/OpenAI-chat 0.83 represents a deterministic API reject of one keyword, while a lower score elsewhere may be a genuine model failure.
- Each open-model verdict is specific to a (model × serving host) at a point in time and not 100% exhaustive of all possible hosts and configurations. These results are not meant to be a comprehensive decision guide on which providers to use. They are meant to be illustrative of how schema support varies, and what constraints you may need to consider as an MCP server, framework, or SDK author.
- Read the rates as upper bounds under favorable phrasing. The default prompts state each task in a single fixed sentence whose values already conform to the schema, and one of them (`lookup-record`'s "Use whichever single identifier the tool expects") explicitly nudges the expected structure.  See "Does coaching the prompt inflate the numbers?" below the results tables for a control comparison with more natural phrasings.

### How models handled each keyword

This results table below reports one serving arm per model.  For closed vendors, this was the vendor's own API.  For open models, this represents a single pinned host.

- Native inference APIs were used for closed model examples, and/or models that were not available on Hugging Face Inference Providers (Claude, GPT, Gemini). The OpenAI rows in this table are the `/v1/responses` arm.  (OpenAI recommends this endpoint for function tools.)
- A sample of open model host providers through Hugging Face Inference were used for all other models. (The results in the table below are from Deepinfra for all open models, except for Inkling and Kimi 3, which were each available from only one host — Together.)

These results also reflect setting the API parameters to allow for these schema keywords, such as by setting strict mode to 'false' where applicable.

| Model                   | enums | top-level oneOf | const union | $ref | if / then | prefixItems |
| ----------------------- | ----- | --------------- | ----------- | ---- | --------- | ----------- |
| `claude-fable-5`        | ✓     | ⊘               | ✓           | ✓    | ✓         | ✓           |
| `claude-sonnet-5`       | ✓     | ⊘               | ✓           | ✓    | ✓         | ✓           |
| `claude-haiku-4.5`      | ✓     | ⊘               | ✗           | ✓    | ✓         | ✓           |
| `claude-opus-4.8`       | ✓     | ⊘               | ✓           | ✓    | ✓         | ✓           |
| `gpt-5.5`               | ✓     | ✓               | ✓           | ✓    | ✓         | ✓           |
| `gpt-5.4`               | ✓     | ✓               | ✓           | ✓    | ✓         | ✓           |
| `gpt-5.4-mini`          | ✓     | ✓               | ✓           | ✓    | ✓         | ✓           |
| `gpt-5.6-sol`           | ✓     | ✓               | ✓           | ✓    | ✓         | ✓           |
| `gpt-5.6-terra`         | ✓     | ✓               | ✓           | ✓    | ✓         | ✓           |
| `gpt-5.6-luna`          | ✓     | ✓               | ✓           | ✓    | ✓         | ✓           |
| `gemini-3.5-flash`      | ✓     | ✓               | ✓           | ✓    | ✓         | ✓           |
| `gemini-3.1-pro`        | ✗     | ✓               | ✓           | ✓    | ✓         | ✓           |
| `gemini-3.1-flash-lite` | ✗     | ✓               | ✓           | ✓    | ✓         | ✓           |
| `glm-5.2`               | ✓     | ✓               | ✓           | ✓    | ✓         | ✓           |
| `deepseek-v4-flash`     | ✓     | ✓               | ✓           | ✓    | ✓         | ✓           |
| `kimi-k2.6`             | ✓     | ✓               | ✓           | ✓    | ✓         | ✓           |
| `kimi-k2.7-code`        | ✓     | ✓               | 95%         | 85%  | ✓         | 80%         |
| `kimi-k3`               | ✓     | ✓               | ✓           | ✓    | ✓         | ✓           |
| `qwen3.6-35b`           | 90%   | ✗               | ✓           | ✓    | ✓         | ✓           |
| `qwen3.5-397b`          | 95%   | ✗               | ✓           | ✓    | ✓         | ✓           |
| `minimax-m3`            | ✓     | ✓               | ✓           | ✓    | ✓         | ✗           |
| `gemma-4-31b`           | 95%   | 65%             | ✓           | ✓    | ✓         | ✓           |
| `gpt-oss-120b`          | ✗     | ✓               | ✓           | 65%  | ✓         | ✓           |
| `inkling`               | ✓     | ✓               | ✓           | ✓    | ✓         | ✓           |

Key: **✓** honored on every served trial · **n%** honored on only that share of trials (mixed results, honored-of-served) · **✗** produces invalid args (fails 2020-12 validation) on every served trial · **⊘** rejected (the provider refused the schema at request time).

Two of the ✗ cells are attributable to the pinned serving host rather than the model: `minimax-m3`'s `prefixItems` fails only on deepinfra (fireworks-ai, novita, and together all serve it at 100%), and `gpt-oss-120b`'s enums failure is host-conditional (novita and scaleway reproduce it; fireworks-ai and together don't). The measured unit for open models is `(model × serving host)` — see the cross-host appendix in the [dataset card](https://github.com/olaservo/research-hub-public/blob/main/mcp/tool-schemas/data/dataset-card.md).

### Comparing OpenAI Chat Completions vs Responses endpoints

I also tested the same OpenAI models on the older `/v1/chat/completions` endpoint. On the chat-completions endpoint, the top-level `oneOf` was refused on every tier and both generations of model. The GPT-5.6 rows here were measured with `reasoning_effort="none"` because chat-completions refuses function tools at any other effort.

| Model | enums | top-level oneOf | const union | $ref | if / then | prefixItems |
| --- | --- | --- | --- | --- | --- | --- |
| `gpt-5.5` | ✓ | ⊘ | ✓ | ✓ | ✓ | ✓ |
| `gpt-5.4` | ✓ | ⊘ | ✓ | ✓ | ✓ | ✓ |
| `gpt-5.4-mini` | ✓ | ⊘ | ✓ | ✓ | ✓ | ✓ |
| `gpt-5.6-sol` | ✓ | ⊘ | ✓ | ✓ | ✓ | ✓ |
| `gpt-5.6-terra` | ✓ | ⊘ | ✓ | ✓ | ✓ | ✓ |
| `gpt-5.6-luna` | ✓ | ⊘ | ✓ | ✓ | ✓ | ✓ |

### Digging deeper into Gemini and enum failures

The example "enums" tool bundles six keywords (`enum` / `oneOf` / `anyOf` / `const` / `title` / `enumNames`). Both Gemini 3.1 models used the three plain-`enum` fields perfectly, but a follow-up probe (the title-vs-const decomposition in the [dataset card](https://github.com/olaservo/research-hub-public/blob/main/mcp/tool-schemas/data/dataset-card.md)'s appendix) found that `oneOf`/`anyOf` branch `const`s don't survive Google's function-declaration path: asked to echo the allowed values, all three Gemini generations quote the display titles and never the consts, and renaming the consts shows even 3.5 Flash is reconstructing the wire values (from `default` + option order) rather than reading them from the schema. 3.5 Flash reconstructs well enough to score ✓; the 3.1 models echo the title instead.

### The same model on different providers

As a side experiment, I re-ran the identical schemas on other tool-capable Hugging Face providers, then added Moonshot's own API in order to cover multiple Kimi model versions side by side on the same providers.

The `kimi-k3` row is the sharpest case in this table: the same model and the same six schemas on two delivery paths, and one keyword flips. Moonshot's API refuses the 2020-12 tuple (`prefixItems` + `items: false`) at request time, before the model ever sees it; Together serves the identical schema and K3 honors it on every trial. The ⊘ is the endpoint's, not the model's.

<table><thead><tr><th>Model</th><th>Keyword</th><th>deepinfra</th><th>fireworks-ai</th><th>novita</th><th>scaleway</th><th>together</th><th>zai-org</th><th>moonshot</th></tr></thead><tbody><tr><td rowspan="6"><code>deepseek-v4-flash</code></td><td>enums</td><td>100%</td><td>100%</td><td>100%</td><td>–</td><td>–</td><td>–</td><td>–</td></tr><tr><td>top-level oneOf</td><td>100%</td><td>100%</td><td>95%</td><td>–</td><td>–</td><td>–</td><td>–</td></tr><tr><td>const union</td><td>100%</td><td>0%</td><td>100%</td><td>–</td><td>–</td><td>–</td><td>–</td></tr><tr><td>$ref</td><td>100%</td><td>0%</td><td>100%</td><td>–</td><td>–</td><td>–</td><td>–</td></tr><tr><td>if / then</td><td>100%</td><td>100%</td><td>100%</td><td>–</td><td>–</td><td>–</td><td>–</td></tr><tr><td>prefixItems</td><td>100%</td><td>100%</td><td>100%</td><td>–</td><td>–</td><td>–</td><td>–</td></tr><tr><td rowspan="6"><code>kimi-k2.6</code></td><td>enums</td><td>100%</td><td>100%</td><td>100%</td><td>–</td><td>100%</td><td>–</td><td>100%</td></tr><tr><td>top-level oneOf</td><td>100%</td><td>⊘ 0%</td><td>100%</td><td>–</td><td>100%</td><td>–</td><td>100%</td></tr><tr><td>const union</td><td>100%</td><td>100%</td><td>100%</td><td>–</td><td>100%</td><td>–</td><td>100%</td></tr><tr><td>$ref</td><td>100%</td><td>100%</td><td>100%</td><td>–</td><td>100%</td><td>–</td><td>100%</td></tr><tr><td>if / then</td><td>100%</td><td>100%</td><td>100%</td><td>–</td><td>100%</td><td>–</td><td>100%</td></tr><tr><td>prefixItems</td><td>100%</td><td>100%</td><td>100%</td><td>–</td><td>100%</td><td>–</td><td>⊘ 0%</td></tr><tr><td rowspan="6"><code>kimi-k3</code></td><td>enums</td><td>–</td><td>–</td><td>–</td><td>–</td><td>100%</td><td>–</td><td>100%</td></tr><tr><td>top-level oneOf</td><td>–</td><td>–</td><td>–</td><td>–</td><td>100%</td><td>–</td><td>100%</td></tr><tr><td>const union</td><td>–</td><td>–</td><td>–</td><td>–</td><td>100%</td><td>–</td><td>100%</td></tr><tr><td>$ref</td><td>–</td><td>–</td><td>–</td><td>–</td><td>100%</td><td>–</td><td>100%</td></tr><tr><td>if / then</td><td>–</td><td>–</td><td>–</td><td>–</td><td>100%</td><td>–</td><td>100%</td></tr><tr><td>prefixItems</td><td>–</td><td>–</td><td>–</td><td>–</td><td>100%</td><td>–</td><td>⊘ 0%</td></tr><tr><td rowspan="6"><code>glm-5.2</code></td><td>enums</td><td>100%</td><td>100%</td><td>n/s</td><td>100%</td><td>80%</td><td>100%</td><td>–</td></tr><tr><td>top-level oneOf</td><td>100%</td><td>100%</td><td>n/s</td><td>100%</td><td>0%</td><td>100%</td><td>–</td></tr><tr><td>const union</td><td>100%</td><td>100%</td><td>n/s</td><td>100%</td><td>100%</td><td>100%</td><td>–</td></tr><tr><td>$ref</td><td>100%</td><td>⊘ 0%</td><td>n/s</td><td>0%</td><td>100%</td><td>0%</td><td>–</td></tr><tr><td>if / then</td><td>100%</td><td>100%</td><td>n/s</td><td>100%</td><td>100%</td><td>100%</td><td>–</td></tr><tr><td>prefixItems</td><td>100%</td><td>100%</td><td>n/s</td><td>100%</td><td>100%</td><td>100%</td><td>–</td></tr><tr><td rowspan="6"><code>qwen3.6-35b</code></td><td>enums</td><td>90%</td><td>–</td><td>–</td><td>75%</td><td>–</td><td>–</td><td>–</td></tr><tr><td>top-level oneOf</td><td>0%</td><td>–</td><td>–</td><td>0%</td><td>–</td><td>–</td><td>–</td></tr><tr><td>const union</td><td>100%</td><td>–</td><td>–</td><td>100%</td><td>–</td><td>–</td><td>–</td></tr><tr><td>$ref</td><td>100%</td><td>–</td><td>–</td><td>100%</td><td>–</td><td>–</td><td>–</td></tr><tr><td>if / then</td><td>95%</td><td>–</td><td>–</td><td>100%</td><td>–</td><td>–</td><td>–</td></tr><tr><td>prefixItems</td><td>100%</td><td>–</td><td>–</td><td>100%</td><td>–</td><td>–</td><td>–</td></tr><tr><td rowspan="6"><code>qwen3.5-397b</code></td><td>enums</td><td>100%</td><td>–</td><td>⊘ 0%</td><td>95%</td><td>–</td><td>–</td><td>–</td></tr><tr><td>top-level oneOf</td><td>0%</td><td>–</td><td>⊘ 0%</td><td>0%</td><td>–</td><td>–</td><td>–</td></tr><tr><td>const union</td><td>100%</td><td>–</td><td>⊘ 0%</td><td>100%</td><td>–</td><td>–</td><td>–</td></tr><tr><td>$ref</td><td>100%</td><td>–</td><td>⊘ 0%</td><td>100%</td><td>–</td><td>–</td><td>–</td></tr><tr><td>if / then</td><td>100%</td><td>–</td><td>⊘ 0%</td><td>100%</td><td>–</td><td>–</td><td>–</td></tr><tr><td>prefixItems</td><td>100%</td><td>–</td><td>⊘ 0%</td><td>100%</td><td>–</td><td>–</td><td>–</td></tr><tr><td rowspan="6"><code>minimax-m3</code></td><td>enums</td><td>100%</td><td>100%</td><td>100%</td><td>–</td><td>55%</td><td>–</td><td>–</td></tr><tr><td>top-level oneOf</td><td>100%</td><td>100%</td><td>100%</td><td>–</td><td>0%</td><td>–</td><td>–</td></tr><tr><td>const union</td><td>100%</td><td>100%</td><td>0%</td><td>–</td><td>100%</td><td>–</td><td>–</td></tr><tr><td>$ref</td><td>100%</td><td>100%</td><td>100%</td><td>–</td><td>100%</td><td>–</td><td>–</td></tr><tr><td>if / then</td><td>100%</td><td>100%</td><td>100%</td><td>–</td><td>100%</td><td>–</td><td>–</td></tr><tr><td>prefixItems</td><td>0%</td><td>100%</td><td>100%</td><td>–</td><td>100%</td><td>–</td><td>–</td></tr><tr><td rowspan="6"><code>gpt-oss-120b</code></td><td>enums</td><td>0%</td><td>100%</td><td>0%</td><td>0%</td><td>100%</td><td>–</td><td>–</td></tr><tr><td>top-level oneOf</td><td>100%</td><td>⊘ 0%</td><td>100%</td><td>100%</td><td>100%</td><td>–</td><td>–</td></tr><tr><td>const union</td><td>100%</td><td>100%</td><td>100%</td><td>100%</td><td>100%</td><td>–</td><td>–</td></tr><tr><td>$ref</td><td>100%</td><td>100%</td><td>40%</td><td>60%</td><td>95%</td><td>–</td><td>–</td></tr><tr><td>if / then</td><td>100%</td><td>100%</td><td>100%</td><td>100%</td><td>25%</td><td>–</td><td>–</td></tr><tr><td>prefixItems</td><td>100%</td><td>100%</td><td>100%</td><td>100%</td><td>100%</td><td>–</td><td>–</td></tr></tbody></table>

Honored-of-served %, 20 trials/cell. **–** means that the provider doesn't offer this model; **n/s** offered but did not reliably serve a forced tool call, so the pairing was excluded; **⊘** means that the provider's API refused the schema at request time.

### Are the highest-fidelity providers forcing conformance?

I also tested whether Deepinfra's perfect score was a result of a constrained decoder forcing conformance rather than the model following the schema, by telling the model to break the schema and seeing if it would. Deepinfra lets models emit invalid output, so my conclusion is that the results do not reflect forced conformance by the model serving host. I ran the same check on Together, which was the only provider available for `inkling` at the time. 

### Does coaching the prompt inflate the numbers?

The prompts used in the results above hand the model already-conforming values, and one (`lookup-record`) also nudges the expected structure, so there was a chance that the results are measuring prompt-following rather than schema-reading. An 'uncoached' control re-phrasing each task naturally was run across 22 of the 24 models in the table (`claude-opus-4.8` and `inkling` have no uncoached rows).

**115 of 128** comparable cells were identical between 'coached' vs 'uncoached', and **15 of the 22 models** didn't change results (four more changed by only one or two trials in one cell). In these cases, the perfect scores appear to be a result of the model reading the schema, not the prompt coaching the model.

The 13 cells that did change are below. Coaching appears to matter mainly where a weaker model meets a hard keyword (for example: `gemma-4-31b` couldn't handle a top-level `oneOf` without the hint, and `gpt-oss-120b` leaned on the coaching for `$ref`):

| Model | Keyword | Coached | Uncoached | Δ |
| --- | --- | --- | --- | --- |
| `gemma-4-31b` | top-level oneOf | 65% | 0% | \-65 |
| `gpt-oss-120b` | $ref | 65% | 30% | \-35 |
| `qwen3.6-35b` | enums | 90% | 80% | \-10 |
| `glm-5.2` | top-level oneOf | 100% | 90% | \-10 |
| `gpt-oss-120b` | const union | 100% | 95% | \-5 |
| `gemma-4-31b` | enums | 95% | 90% | \-5 |
| `kimi-k2.7-code` | enums | 100% | 95% | \-5 |
| `kimi-k2.7-code` | top-level oneOf | 100% | 95% | \-5 |
| `kimi-k2.7-code` | const union | 95% | 100% | +5 |
| `gemini-3.1-pro` | enums | 0% | 5% | +5 |
| `qwen3.5-397b` | enums | 95% | 100% | +5 |
| `kimi-k2.7-code` | $ref | 85% | 95% | +10 |
| `kimi-k2.7-code` | prefixItems | 80% | 100% | +20 |

Percent of 'honored' out of 'served', coached vs uncoached (20 trials each). Negative Δ indicates that the model leaned on the coaching. The 115 unchanged cells and the 15 models with no changed cells aren't shown. A few cells (mostly `kimi-k2.7-code`) did slightly *better* uncoached, but each positive Δ is only 1–4 trials out of 20 (the 95% CI on a rate measured with 20 trials spans roughly ±15 points) and the sign flips both ways across that model's own cells. This appears consistent with sampling noise rather than a real "coaching hurts" effect.

### Full Results

The full results are a Hugging Face-ready dataset (`runs.jsonl` + `summary.csv` + `transport_diff.csv` + `cycle.csv`, full provenance per record including the back-end that served each call).

[Dataset card (GitHub)](https://github.com/olaservo/research-hub-public/blob/main/mcp/tool-schemas/data/dataset-card.md) · [summary.csv](https://github.com/olaservo/research-hub-public/blob/main/mcp/tool-schemas/data/summary.csv) (first-class `honored_of_served` with a served-based Wilson CI, plus the `accepted_pct` / `honored_given_accepted` gate-vs-skill split and an `underpowered` flag; the legacy `honored_rate` divides by all trials — see the dataset card for the conventions)

Source, the Markdown write-ups, and the servers live on [GitHub](https://github.com/olaservo/research-hub-public/tree/main/mcp/tool-schemas). An independent survey of tool-call support across model providers is at [evalstate/tool-research](https://huggingface.co/spaces/evalstate/tool-research) by Shaun Smith.

## Try it

### Interactive playground — coming soon

An interactive playground is in the works: pick one of the six demo tools, fill its form, and validate against its schema, with a panel showing how each model + provider handled that `inputSchema`. It isn't published yet. Follow [Ola on LinkedIn](https://www.linkedin.com/in/olahungerford/) for updates, more research, and errata.

<!-- Playground published-links section — restore when the playground ships.
### Interactive playground

Pick one of the six demo tools, fill its form, and validate against its schema. A model-results panel then shows, from the static results dataset, how that model + provider handled the `inputSchema`.

[Open playground](https://olaservo-sandyland.static.hf.space/mcp/tool-schemas/playground.html)
-->

By [Ola Hungerford](https://www.olahungerford.com/)
