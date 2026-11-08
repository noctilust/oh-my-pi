# Nebius dedicated deployments: capabilities and effort ladders

How the `nebius` provider derives reasoning, native-tool support, and thinking effort
tiers for models that no public catalog describes. Written for future agents editing
this code: read this before touching `nebiusReasoning()`, the Nebius `mapModel`,
`classes/minimax.kdl`, or any Nebius model rule.

## The problem in one paragraph

Nebius Token Factory serves two kinds of model ids. **Serverless** ids look like
`MiniMaxAI/MiniMax-M3.1-Flash` and are published by `models.dev` (mirrored as
`catalog.stencil.so`) and by the bundled `packages/catalog/src/models.json`.
**Dedicated** ids look like `dedicated/<project>/<Model>-<variant>-<suffix>` or
`<project>/<Model>-<variant>` (e.g.
`dedicated/sa-example/MiniMax-M3.1-Flash-NVFP4-TP2-abc123` and
`sa-example/DeepSeek-V4.1-Flash-Coding-Optimized-abc123`); they are project-scoped,
carry a random per-deployment suffix, and appear in **no** public catalog. Before this
design, every dedicated deployment resolved with `reasoning: false` and no effort
ladder, even though the live roster advertises `reasoning` and the endpoint reasons
correctly.

## Invariants — do not break these

1. **A dedicated deployment must never depend on a public-catalog row.** Everything
   it needs comes from the live roster (`/v1/models?verbose=true`) or from KDL.
2. **A missing, null, or empty capability field must not revoke inherited
   capabilities.** Such a row has no opinion about reasoning or native tools.
   Where no bundled row exists to inherit from and a reviewed family/residue
   selector matches, that ladder repairs the capability through
   `thinking-upgrade-neutral`; an unreviewed generation gets no invented ladder.
   Historical endpoints sometimes returned `[]` for no opinion, so an empty list
   is neutral for reasoning as well.
3. **Policy selectors must be lineage-based, never deployment-id-based.** A future
   deployment carries a new project name and a new random suffix.
4. **`packages/catalog/src/models.json` is generated.** Never hand-edit it; the
   `compat-parity` test will fail if the engine and the baked catalog disagree.
5. **Reasoning capability and effort ladder are separate decisions.** Capability
   comes from the wire; the ladder is reviewed policy that lives in KDL.

## Where the data comes from

`resolveProviderModels` merges four sources in this order:

```
static (models.json) → cache (models.db) → stencil.so (models.dev) → live roster
```

For `reasoning`, the merge is additive unless the provider declares authority
(`mergeDynamicModel` in `packages/catalog/src/model-manager.ts`):

```ts
const reasoning = authority.reasoning
  ? dynamicModel.reasoning                       // live roster wins outright
  : existingModel.reasoning || dynamicModel.reasoning;
```

Nebius sets `dynamicReasoningAuthoritative: true` in `nebiusModelManagerOptions`
(`packages/catalog/src/provider-models/openai-compat.ts`), so a live row's verdict
wins over anything inherited. That is only safe because absence is handled explicitly:
the mapper inherits an exact bundled reference, and the KDL lineage rules promote a
reviewed ladder when the roster says nothing.

### Why the roster can be authoritative

A fresh raw `GET /v1/models?verbose=true` audit on 2026-11-06 returned 33 rows. Counts
drift as deployments are added and removed; only public model ids are recorded here.

| Check | Result |
| --- | --- |
| Populated list without `"tools"` | Only `openbmb/MiniCPM-V-4_5`: `["json_mode", "structured_outputs"]` |
| Missing `supported_features` | `Qwen/Qwen3-Embedding-8B` (excluded as embedding) and `deepseek-ai/DeepSeek-V4.1-Flash` (inherits bundled reasoning) |
| Empty list | No rows in this snapshot; historical responses did contain `[]`, so neutrality remains required |

The explicit MiniCPM feature list selects in-band tool calling. The absent-field
DeepSeek row retains inherited capabilities, including reasoning. No dedicated model
id is needed to exercise this logic: `mapModel` follows the row's fields, not its id.

## Mechanism 1 — derive capability in the mapper

`nebiusReasoning()` in `packages/catalog/src/provider-models/openai-compat.ts`:

```ts
function nebiusReasoning(entry: OpenAICompatibleModelRecord, inherited: boolean): boolean {
    const features = entry.supported_features;
    if (!Array.isArray(features) || features.length === 0) return inherited;
    return features.some(feature => feature === "reasoning");
}
```

Wired into the Nebius `mapModel`, which also derives `explicitFeatures` from non-empty
arrays only. If that populated list omits `"tools"`, the mapper sets `supportsTools:
false`; if it includes `"tools"`, it does not force a positive value. Missing, null,
and empty lists leave both reasoning and tool metadata neutral. The logic is
deployment-agnostic: it follows roster fields, not model ids.

A populated list that omits `reasoning` is an explicit negative, not a missing opinion.
The mapper then clears the inherited `thinking` dial and sets
`compat.trustExplicitThinkingOnly`, preventing a reviewed lineage ladder from
re-promoting the model during `buildModel`.

For native tools, only explicit `false` is a verdict. `undefined` keeps automatic
selection intact; `false` makes `tools.format: auto` select the in-band dialect chosen
by `resolveDialect` in `packages/coding-agent/src/sdk.ts` (GLM when the model has no
known family). This changes request dispatch, not a claim about endpoint compatibility.

Do **not** generalize this into a shared helper for other providers without auditing
their `supported_features` semantics. Only CoralBricks, Synthetic, and Nebius declare
`dynamicReasoningAuthoritative` today, and most providers do not publish the field at
all.

## Mechanism 2 — reviewed effort ladder in KDL

`packages/catalog/src/compat/rules/classes/minimax.kdl`:

```kdl
on "nebius" {
    family "m2" {
        thinking-mode "effort"
        thinking-efforts "low" "medium" "high"
        thinking-requires-effort #true
        // The roster omits `supported_features` on some deployments; the reviewed
        // lineage ladder is the capability fallback, never a silent downgrade.
        thinking-upgrade-neutral #true
        reasoning-disable-mode "lowest-effort"
    }
    family "m3" {
        thinking-mode "effort"
        thinking-efforts "low" "medium" "high" "xhigh" "max"
        thinking-requires-effort #false
        thinking-upgrade-neutral #true
        reasoning-disable-mode "none-effort"
        models "MiniMax-M3" "minimax-m3" "MiniMaxAI/MiniMax-M3" "*minimax-m3-*" {
            thinking-efforts "minimal" "low" "medium" "high" "xhigh"
            reasoning-disable-mode "lowest-effort"
        }
    }
}
```

`thinking-upgrade-neutral` is the capability fallback for a dedicated row with no
bundled reference: the cascade promotes the row to reasoning-capable when an
identity-scoped rule owns a reviewed ladder. The same opt-in is present on every
reviewed Nebius ladder: MiniMax (`family "m3"`), DeepSeek (`revision ">=4"`), Qwen 3.8,
GLM, Kimi K3, Nemotron, Hermes-4-405B, `muse-spark-open`, and `beam`. The provenance and model
policy still live in KDL; the mapper only reads the wire.

### Why the rule stops at the reviewed families

Taxonomy facts (`classifyModel("nebius", …)`):

| id | class | family |
| --- | --- | --- |
| `MiniMax-M2.7` | minimax | `m2` |
| `MiniMax-M3` | minimax | `m3` |
| `MiniMax-M3.1-Flash` | minimax | `m3` |

The taxonomy only declares `*minimax-m1*`, `*minimax-m2*`, `*minimax-m3*` globs, and
the rule is scoped to the reviewed family selectors. A generation the taxonomy does
not classify into `m2` or `m3` matches **no Nebius rule**: it gets no ladder and no
`thinking-upgrade-neutral` repair until its vocabulary is measured and a selector is
added. That is deliberate — an unreviewed generation must not advertise a ladder the
endpoint has not been shown to accept. The `family "m2"` and `family "m3"` blocks are
siblings, not a class-wide default with overrides.

### Worked example: the generic fallback ladder is not a vocabulary

A row the roster reports as reasoning-capable still resolves with the API-generic
`fallbackEfforts` ladder while no Nebius rule matches it — for `openai-completions`
with `supportsReasoningEffort` that is `minimal…xhigh`. That ladder is the engine's
API default, not a measured Nebius vocabulary, and the endpoint validates
`reasoning_effort` per deployment.

`NousResearch/Hermes-4-405B` (classifies as `unknown`; the taxonomy declares no Hermes
class) is the worked example. Measured 2026-11-08:

| `reasoning_effort` | `minimal` | `low` | `medium` | `high` | `xhigh` | `max` | `none` |
| --- | --- | --- | --- | --- | --- | --- | --- |
| response | 400 | 200 | 200 | 200 | 400 | 400 | 200 |

The 400s are vLLM `literal_error`s naming the accepted set (`none`, `low`, `medium`,
`high`). Before its residue rule, the fallback ladder offered `minimal` and `xhigh`
and `lowest-effort` disabled thinking with `minimal` — every one of them a rejected
request. `models "*hermes-4-405b*"` now carries the measured `low`/`medium`/`high`
ladder with `none-effort`; copy that pattern for the next such row: probe every tier,
then pin the accepted subset.

### Why the M3 pin exists

M3 and M3.1-Flash share `family "m3"` — family alone cannot separate them. Without
the pin, the `family "m3"` ladder re-resolved `MiniMax-M3` from the aggregator ladder
it already bakes (`minimal…xhigh`, see the `on "aimlapi" …` block in the same file) to
the M3.1 ladder (`low…max`). The `compat-parity` test caught it:

```
nebius/MiniMaxAI/MiniMax-M3.compat.reasoningDisableMode: baked="lowest-effort" engine="none-effort"
nebius/MiniMaxAI/MiniMax-M3.thinking.efforts: baked=["minimal",…,"xhigh"] engine=["low",…,"max"]
```

The pin restores M3 while leaving M3.1 on the tuned ladder. Its glob `*minimax-m3-*`
deliberately cannot match `minimax-m3.1-flash` (`m3.` never satisfies `m3-`); the
regression test asserts this.

### Why these exact axes

Each value was measured against the live endpoint, not assumed:

| Axis | Value | Evidence |
| --- | --- | --- |
| `thinking-efforts` | `low medium high xhigh max` | all tiers return HTTP 200 on both M3.1-Flash rows |
| `thinking-requires-effort` | `#false` | `reasoning_effort: "none"` returns 200 with no reasoning — thinking **can** be turned off on Nebius, unlike MiniMax's own API (which 400s with error 2013) |
| `reasoning-disable-mode` | `none-effort` | sends literal `reasoning_effort: "none"` (`packages/ai/src/providers/openai-shared.ts`), the observed off switch |
| `thinking-effort-map` | **omitted** | value names already match the wire; Nebius has no inherited adaptive map, so an identity map is dead weight (`mergeEffortMap` returns `undefined` without one) |
| `supports-sampling-params` | **omitted** | Nebius accepts `temperature`, `top_p`, `top_k` on these rows (HTTP 200). The minimax-provider rule sets `#false`; extending it to Nebius would strip working controls |

## Selector semantics you must know

From `packages/catalog/src/compat/rules/README.md` and
`packages/catalog/src/compat/cascade.ts`:

- `models "X"` (no `*`) is an **exact, case-sensitive** match against the
  provider-relative model id. For Nebius that is the whole id, including
  `dedicated/<project>/`.
- `models "X*"` (contains `*`) is an **anchored, ASCII-case-insensitive** wildcard.
  `*minimax-m3-*` matches any id containing `minimax-m3-`.
- `token="x"` matches a non-alphanumeric-bounded token.
- Rules are ranked by `(exactness, selector dimensions, priority)`; two rules of equal
  rank contesting the same axis throw `AmbiguousOverlapError` **at resolve time**, not
  compile time. That is why "regenerate and run the tests" is mandatory.

## Adding support for a new model or generation

1. If the model is a **new generation outside the reviewed families**: it matches no
   Nebius rule, so it deliberately gets no ladder or `thinking-upgrade-neutral`
   repair. Measure the endpoint, then add a reviewed selector — a taxonomy family
   glob when the generation becomes a family, otherwise a documented `models`
   residue rule. Do not widen the rule to the class to pre-cover it.
2. If a **reviewed family** has a different effort vocabulary (like M2's
   `low/medium/high`): give it its own `family` block. Families are siblings, not a
   class-wide default with narrower overrides.
3. If the model belongs to a **different lineage** (`deepseek`, `qwen`, `glm`, `kimi`
   already have Nebius rules in `providers/nebius.kdl`): add it there, including
   `thinking-upgrade-neutral #true` so an absent `supported_features` does not
   downgrade it.
4. Regenerate and verify:

```sh
bun run gen:compat
bun run gen:models --provider nebius
bun test packages/catalog/test/nebius-provider.test.ts
bun test packages/catalog/test/compat-parity.test.ts
```

5. Never add a rule keyed on a deployment id (`dedicated/<project>/…` or
   `<project>/<Model>-…`). Project names and the trailing suffix are per-deployment
   and will change; use class/family/revision or a lineage glob.

## Verification recipes

Resolve a hypothetical future deployment without any network access:

```ts
import { buildModel } from "@oh-my-pi/pi-catalog/build";
import { getSupportedEfforts } from "@oh-my-pi/pi-catalog/model-thinking";

const model = buildModel({
    id: "dedicated/sa-newteam/MiniMax-M3.1-Flash-NVFP4-TP2-FUTURE99",
    name: "future",
    api: "openai-completions",
    provider: "nebius",
    baseUrl: "https://api.tokenfactory.nebius.com/v1",
    reasoning: false,          // roster omitted supported_features
    input: ["text", "image"],
    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
    contextWindow: 1048576,
    maxTokens: null,
});
console.log(getSupportedEfforts(model), model.reasoning, model.compat.reasoningDisableMode);
// → ["low","medium","high","xhigh","max"] true none-effort
```

A generation that matches no reviewed family resolves without a ladder:
`getSupportedEfforts` is `[]` and `reasoning` stays `false` unless the roster or a
bundled row says otherwise. That is the intended evidence-only boundary, not a
missing rule.

Check what the live product actually reports:

```sh
bun packages/coding-agent/src/cli.ts models refresh --json \
  | python3 -c 'import json,sys; [print(m["selector"], m["reasoning"], m["thinking"]) for m in json.load(sys.stdin)["models"] if m["provider"]=="nebius"]'
```

Counts and deployment ids drift; current examples include:

```
nebius/dedicated/sa-example/DeepSeek-V4.1-Flash-Coding-Optimized-abc123  True  ['low','high','max']
nebius/dedicated/sa-example/GLM-5.3-Flash-Fixed-abc123                   True  ['low','high','max']
nebius/MiniMaxAI/MiniMax-M3                                              True  ['minimal','low','medium','high','xhigh']
nebius/MiniMaxAI/MiniMax-M3.1-Flash                                      True  ['low','medium','high','xhigh','max']
```

## Known failure modes

| Symptom | Cause |
| --- | --- |
| Dedicated row shows `reasoning: false` | mapper lost the `supported_features` read, or `dynamicReasoningAuthoritative` was removed |
| Dedicated row omits `supported_features` and still shows `reasoning: false` | its lineage rule lacks `thinking-upgrade-neutral #true`, it classifies as `unknown`, or it is an unreviewed generation with no matching family/residue selector |
| A row whose `supported_features` omits `reasoning` still shows a ladder | the mapper did not clear `thinking` and set `trustExplicitThinkingOnly`, or a KDL rule overrode that flag |
| An empty `supported_features` list revokes reasoning or native tools | the mapper treated `[]` as a verdict; only populated lists may do so |
| A populated list without `"tools"` still dispatches native tools | the mapper no longer sets `supportsTools: false`, or a configured model override reverses it |
| `AmbiguousOverlapError` at resolve time | two equally-ranked Nebius rules contest one axis; add `priority=` or narrow a selector |
| `compat-parity` diff on `nebius/MiniMaxAI/MiniMax-M3` | the `family "m3"` ladder leaked onto M3; the `models` pin is missing or its glob is wrong |
| A new model shows a ladder but no `max` | it matched only the generic `fallbackEfforts` ladder — the class rule did not apply |
| An offered tier answers 400 with a vLLM `literal_error` naming the accepted set | the row resolved to the generic `fallbackEfforts` ladder; probe the tiers and add a measured residue rule (`*hermes-4-405b*` is the worked example) |
| A new generation shows no ladder | intended until a reviewed family/residue selector exists; measure the endpoint, then add the selector |
| An embedding SKU appears as a chat model | the Nebius `filterModel` no longer applies the `runtime/behavior.kdl` exclusion |

Unexpected in-band calls are the intended result of a populated list without `"tools"`;
they do not by themselves prove the endpoint lacks native support. Verify the actual
endpoint and correct its roster. If the roster is wrong, `modelOverrides` can restore
native dispatch for that one model (see `docs/models.md`):

```yaml
providers:
  nebius:
    modelOverrides:
      "openbmb/MiniCPM-V-4_5":
        supportsTools: true
```

Use this override only after verifying native support; it takes precedence over
discovery and can re-enable native calls on an endpoint that cannot process them.
