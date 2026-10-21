# Game documentation maintenance policy

Adapted from QSD's change classification, layered writing and semantic retrieval rules. QSD container names, test anchors, templates and audit scripts are not FOSquad contracts.

## Minimum correct scope

| Class | Trigger | Update |
| --- | --- | --- |
| `no-doc` | Local implementation changes without changed behavior, contracts, navigation or workflows | No documentation edit; briefly explain the decision |
| `map-only` | A file, module, guide or entrypoint moves while behavior stays the same | Owning index/map and affected references |
| `feature-doc` | A durable gameplay/UI/tooling flow changes within one owner | Owning feature page; map only if its anchors changed |
| `contract-flow` | Script/content/native/config handoff or serialization changes | Affected feature pages, producers/consumers and migration guidance |
| `ownership-boundary` | Responsibility moves between modules or between game and engine | Both owners' maps and contract direction; load the other repository's skill if authorized |
| `convention-or-agent-instructions` | Agent rules, commands, skill routing or verification policy changes | `AGENTS.md`, relevant skill and workflow guidance; keep adapters thin |

Choose the smallest class that remains accurate, not the smallest edit count. A local bug fix with a documented behavior change is not `no-doc`.

## Evidence ladder

1. Fresh source, runtime configuration and generator inputs establish their own content.
2. Existing compiler/validator output establishes only what that specific check covers.
3. Direct application observations establish behavior only for the exercised configuration and revision.
4. CodeGraph relationships guide inspection; negative/exhaustive results require independent confirmation.
5. Prior docs, plans, version labels and historical reports are claims to reconcile, not substitutes for current evidence.

For a cross-file claim, verify the narrowest decisive anchors: callback declaration and subscription, resource entry and consumer, annotation and generated binding, schema/property and serialized reader. For a deletion, search authored consumers and build/resource wiring; absence from the graph alone is insufficient.

## Layered writing

- `AGENTS.md`: instructions and restrictions; one source of truth, not a duplicate manual.
- Maps/indexes: ownership, reading order, entrypoints, links and update triggers.
- Feature pages: a durable flow and contract that is costly to rediscover.
- Plans: proposals, decisions and pending work with explicit status.
- History: dated evidence that remains historical after the implementation changes.
- Generated data: emitted by its owning tool, never hand-maintained prose or pasted graph output.

Prefer short bullets, one claim per bullet, repo-relative paths and explicit producer -> consumer direction. Useful labels are `owner:`, `entrypoint:`, `code:`, `contract:`, `verification:`, `doc:` and `update-when:`. Use `verification:` rather than inventing test anchors for removed suites.

Aim for roughly 80-220 lines for a new map and 40-180 for a new focused page, but keep necessary verified detail. Existing documents need not be renumbered or forced into QSD's ten-section template. Do not compress away caveats or duplicate the engine's reusable API guide.

## Optional outline for a new feature page

1. Scope and owner; what the page does not own.
2. Entry points and minimal source anchors.
3. Main flow, including server/client authority where relevant.
4. Contracts, generated inputs and persistence/network implications.
5. Failure clues and existing verification route, with limits.
6. Related docs and update triggers.

Add a feature page only for repeated discovery cost, a durable multi-step flow or a meaningful cross-owner contract. Prefer updating the owning page for small changes. When merging/removing a page, fix inbound links in the same change.

## Changes to the skills

Keep canonical skill source in the owning repository and installation as a link, not a second editable copy. Update routing descriptions when trigger scope changes. Do not add delegation, publication, destructive Git authority or test creation through a documentation skill. The owner's instructions remain authoritative, including the standing Russian-description commit plus fork push after code edits in `AGENTS.md`. Do not replace that policy with a blanket no-commit mandate; delegated workers must still respect coordinator-owned Git operations.
