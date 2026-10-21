# Game documentation checks and audit procedure

A small documentation update and a full documentation audit are different tasks. Report which one was performed. Installing this skill does not complete either repository's documentation audit.

## Bounded update checklist

- Read applicable instructions, the owning map and only the relevant feature pages.
- Check changed claims against their current source/configuration anchors; check links relative to the containing document and inline source paths relative to the stated repository.
- Distinguish a real source path from an example placeholder, API route, generated output, intentional absence or historical filename. Do not create fake files to satisfy a link.
- Check inbound links when renaming or deleting a maintained page. Include root Markdown and tool READMEs, not only `Docs/`.
- Update `Docs/README.md` for a new maintained game guide and `Docs/Systems.md` for changed gameplay ownership.
- Verify game/engine boundaries, generator ownership, language rules and current verification instructions.
- Inspect the scoped diff and whitespace. These existing read-only commands are useful from the game root:

```sh
git status --short
git diff -- AGENTS.md README.md Nullability.md Docs Tools/NullableEstimate/README.md skills
git diff --check
```

Git diff does not include untracked file contents: read new skill/docs files explicitly. A whole-worktree whitespace failure may predate this task; identify its owner without reverting it or hiding the result.

No standalone game documentation auditor is supplied by this skill. Use targeted `rg`, listings and direct reads. Do not claim engine documentation checks validate the game corpus. Do not create a temporary test suite or copy QSD's infrastructure to fill this gap.

## When a full audit is requested

### 1. Fix the scope and baseline

Record the game root/HEAD, dirty state, actual engine target/HEAD, `EngineRevision`, and whether the engine audit is also authorized. Preserve existing changes. Do not treat the index or Git HEAD alone as the working-tree contents.

Inventory separately:

- active root guides and agent adapters;
- maintained `Docs/` maps and feature pages;
- workflow/build/launch/config instructions;
- tool guides and canonical skill references;
- plans, history and archived workflows;
- generated and ignored local outputs;
- the external engine corpus, owned by its own audit.

Use Git's tracked/untracked inventory and bounded directory listings. Do not traverse `Engine/`, output trees, vendor trees or symlinks indiscriminately. A full audit must account for all in-scope documents, not extrapolate from a few samples.

### 2. Reconcile claims with current owners

For each maintained area, check navigation and links, source paths/symbols, authoritative side, cross-module flow, resource inputs, generated ownership, supported commands and verification limits. Include root/tool/agent documents in the same consistency pass.

Prioritize these known drift risks:

- stale unit-suite paths, deleted test targets and old mandatory test/CI gates;
- submodule commands versus the external engine link, and pinned versus local engine revisions;
- generated GUI edits without the owning `.fogui`, or missing resource-pack input directories;
- copied engine APIs/annotations/settings that no longer match the connected engine;
- plans written as implemented behavior, historical pass counts written as present coverage;
- release/server/persistence claims without evidence for the stated configuration.

Classify each finding by concrete impact, cite document path/line plus decisive evidence, and identify the owning repository and page. A missing link and an incorrect runtime contract are not equally severe. A historical report should remain historical; an active wrong command must be corrected, not merely labelled old.

### 3. Verify proportionately

Documentation-only checks do not require a native build or resource bake. If code/content/configuration changes are explicitly part of the work, use the relevant existing VS Code/CMake/formatter route from `AGENTS.md` and `.vscode/tasks.json`; judge its output against the requirement. Do not create or modify tests without an explicit request.

For commands that install dependencies, generate outputs, launch services, publish, access external systems or mutate saved data, inspect the implementation and obtain the necessary authorization before execution. Mark unexecuted claims as such. Archived CI is not an available validation result.

### 4. Report honestly

State checked scope and counts where actually measured, unresolved findings, corrected pages, actual commands/results and material exclusions. Distinguish structural/link verification from behavior, rendering and deployment verification. Retain concrete blockers; do not report a full pass while active documentation still contradicts its source.

Use a Markdown report only when requested or needed for the agreed audit deliverable; keep it separate from canonical current guides. Report completion separately for game and engine. Follow the standing `AGENTS.md` commit policy: after code edits, create a Russian-description commit and push normally to the configured fork, preserving unrelated dirty changes and separate game/engine commits. In delegated work, the coordinator may own all Git mutations; a worker must not duplicate them. Documentation skills do not grant additional publication authority. Advance `EngineRevision` only to an actually published engine commit after appropriate verification, never to describe an uncommitted working tree.
