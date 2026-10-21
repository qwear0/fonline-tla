---
name: fosquad-doc-maintainer
description: Maintain concise, source-grounded documentation for the FOSquad game repository. Use for repository orientation, documentation audits and impact checks, and changes to gameplay ownership, AngelScript, authored content, GUI generation, native extensions, configuration, build workflows, the external engine boundary, skills or agent instructions. Keep game maps and feature docs synchronized without copying engine documentation or historical test requirements.
metadata:
  version: "1.0.0"
  adapted-from: "qsd-doc-maintainer"
---

# FOSquad documentation maintainer

Make the next maintainer faster at finding the owner, understanding a contract and choosing the documentation that must change. This skill is for the **game**, not the reusable engine.

## 1. Establish scope

1. Read the active checkout's `AGENTS.md` and any nearer instructions before editing.
2. Determine the game root from the working directory and Git, not from this skill's installed location. A global installation may point to another checkout.
3. Inspect Git status without staging or reverting anything. Preserve earlier user changes.
4. Start with `Docs/README.md`; use `Docs/Systems.md` for gameplay and `Docs/FOSquadWorkspace.md` for the engine connection and local commands.
5. Read [references/repository-map.md](references/repository-map.md) for evidence anchors and boundaries. Read only the relevant feature doc next.
6. Use [references/maintenance-policy.md](references/maintenance-policy.md) to classify impact, retrieve evidence and keep writing compact.
7. Before completion, follow [references/audit-checklist.md](references/audit-checklist.md). Use its full-audit route only when a documentation audit is requested.

## 2. Retrieve evidence selectively

- For unfamiliar multi-file implementation flows, start with a bounded CodeGraph query, normally `maxFiles: 1..3`, with the exact game `projectPath`. Check freshness if uncertain.
- Fresh verbatim source is evidence for those lines. Relationships are candidates, not proof of complete ownership or impact. Do not reread identical source without a reason.
- For Markdown, configuration, exact strings, AngelScript/content references or absent index coverage, use targeted `rg`, file listings and direct reads.
- A graph for native code does not establish coverage of `.fos`, `.fogui`, `.fodlg`, prototypes or `TLA.fomain` resource wiring.
- Verify decision-critical claims about deletion, callbacks, generated inputs, serialization, saves and network boundaries against actual source/configuration.
- If CodeGraph times out or is stale, use repository evidence immediately; do not install or rebuild it for this task.
- After a non-trivial code change that used the graph, make one bounded missed-impact pass. Skip it for documentation-only work or an unavailable index.

## 3. Update the minimum correct documents

Classify the change as `no-doc`, `map-only`, `feature-doc`, `contract-flow`, `ownership-boundary` or `convention-or-agent-instructions` using the maintenance policy.

- Keep `AGENTS.md` operational, `Docs/README.md` navigational, `Docs/Systems.md` an ownership map, and feature pages focused on durable behavior.
- Register a new maintained game guide in `Docs/README.md`; update `Docs/Systems.md` when gameplay ownership or entrypoints change.
- Update relevant README/build guidance when supported commands or workflows change. A checked-in example is not proof that it currently works.
- Game rules and authored assets stay in this repository. `Engine/` is a link to an independent repository, not a submodule or permission to edit engine docs.
- When engine documentation is in scope, also load `fosquad-engine-doc-maintainer` from that checkout's `skills/fosquad-engine-doc-maintainer/SKILL.md`. Keep changes and evidence separate per repository.
- Treat `EngineRevision` as a published baseline, not the current dirty engine state. Never invent a commit SHA or advance it just to match uncommitted changes.
- Describe generated files through their owning input and generator. Do not repair generated output as a substitute for correcting its owner.

## 4. Fact and verification discipline

- Write game-maintainer Markdown in English (preserve the established Russian `Docs/ScriptStyle.md`); communicate with the user in Russian unless requested otherwise. Script comments remain Russian; serialized identifiers are not translated.
- Separate current contracts, proposed work, old migration notes, historical execution evidence and unverified assumptions. Do not disguise live broken instructions as history merely to silence a check.
- The owner's testing policy in `AGENTS.md` overrides imported guidance. Do not create or change automated tests or test infrastructure, including temporary/delegated tests, without an explicit request. Missing new tests are not a defect.
- Do not port QSD's container scanner/auditor, impose its directory layout or require a nonexistent local audit command. Use existing checks and direct evidence; do not create a replacement verification framework.
- Do not claim inactive archived workflows run on GitHub. Local source state and remote settings/publication are different facts.
- Check documentation-only edits without automatically rebuilding or baking the game. Runtime claims require appropriate runtime evidence; Markdown checks alone cannot establish them.
- Preserve real failures and verification gaps. Do not guess replacement paths, weaken validators or overwrite historical evidence to manufacture success.
- Follow `AGENTS.md` for commits: code edits require a Russian-description commit and a normal push to the configured fork, not upstream. Keep game and engine commits separate and preserve unrelated dirty work. A delegated worker follows its narrower Git authority; the coordinator can own publication. This skill neither forbids standing authorized publication nor grants new Git authority.

## 5. Local distribution

The canonical source is `skills/fosquad-doc-maintainer/` in the game repository. Its references are relative to this `SKILL.md`; implementation paths in those references are relative to the **target game checkout**.

Local installation uses a symlink from `~/.agents/skills/fosquad-doc-maintainer` to the canonical directory. Do not overwrite an existing independent installation without inspecting it. This is a live local installation, not a Git/npm publication or an immutable release. Refresh Pi with `/reload` or start a new session after installation; `/skill:fosquad-doc-maintainer` explicitly loads it. Other agents can read the canonical file directly.

No scripts, dependencies, test suites or remote publication are bundled. The references together replace the QSD-specific scanner/auditor assumptions.

## 6. Completion report

State:

- changed maps, feature docs, agent instructions and skill files;
- any changed ownership or public/serialized contract, or explicitly that none changed;
- evidence used, including graph limitations when relevant;
- exact checks actually run, their results and significant gaps;
- whether this was a bounded update or a full audit, and what remains open;
- whether local installation was verified and whether commits/publication occurred.
