# Domain Docs

How the engineering skills should consume this workspace's domain documentation when exploring the codebase.

## Before exploring, read these

- **`GLOSSARY-MAP.md`** at the workspace root: it points at one `GLOSSARY.md` per context (one per repository in `repositories/`). Read each one relevant to the topic.
- **`docs/adr/`** at the workspace root: workspace-wide decisions. Also check `repositories/<repo>/docs/adr/` for context-scoped decisions.

If any of these files don't exist, **proceed silently**. Don't flag their absence; don't suggest creating them upfront. The `/domain-modeling` skill (reached via `/grill-with-docs` and `/improve-codebase-architecture`) creates them lazily when terms or decisions actually get resolved.

## File structure

Multi-context workspace:

```
/
├── GLOSSARY-MAP.md
├── docs/adr/                          ← workspace-wide decisions
└── repositories/
    ├── CLIProxyAPI/
    │   ├── GLOSSARY.md
    │   └── docs/adr/                  ← context-specific decisions
    ├── CPA-Manager-Plus/
    │   ├── GLOSSARY.md
    │   └── docs/adr/
    └── DevScripts/
        ├── GLOSSARY.md
        └── docs/adr/
```

Note: per-repository glossaries and ADRs live inside the git submodules. Creating or editing them requires a commit (and push) in that submodule before the workspace's pinned reference is updated.

## Use the glossary's vocabulary

When your output names a domain concept (in an issue title, a refactor proposal, a hypothesis, a test name), use the term as defined in the relevant `GLOSSARY.md`. Don't drift to synonyms the glossary explicitly avoids.

If the concept you need isn't in the glossary yet, that's a signal: either you're inventing language the project doesn't use (reconsider) or there's a real gap (note it for `/domain-modeling`).

## Flag ADR conflicts

If your output contradicts an existing ADR, surface it explicitly rather than silently overriding:

> _Contradicts ADR-0007 (event-sourced orders), but worth reopening because…_
