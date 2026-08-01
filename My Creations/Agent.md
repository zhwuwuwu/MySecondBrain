# Agent Instructions — My Creations

**Scope**: personal ideas taken through a pipeline: **capture → idea → research → PRD/方案 → implementation**. ONLY things Zihan is personally originating/building. If it's a fact/topic learned from an external source with no personal build attached → belongs in `My Notes/` instead (captured via the root `Quicknotes.md`). If it's a record of a specific work project (not a personal venture) → belongs in `My Works/` instead.

## Entry point: `Quick Ideas.md` (capture-only, read its own protocol)

`Quick Ideas.md` in this folder is a lightweight, capture-only list of spontaneous ideas — the equivalent of `Quicknotes.md` but for "things to build" instead of "things to learn". Ideas sit there unpromoted, possibly forever, until Zihan explicitly says to move forward with one. Don't research or expand an item there unprompted — that only happens once it's promoted into its own subfolder below.

## Current state

One idea sits loose at the top level, not yet promoted into the pipeline below: `MetaEquipe 打造任意AI agent 团队.md` (a meta-agent-team-builder concept). It predates this module's current structure and hasn't been formalized into its own `My Creations/<Idea Name>/idea.md` yet — don't move/restructure it unprompted, that's a scope decision for the user. No idea has otherwise gone through the pipeline below.

## Proposed structure

The structure below is a **proposed convention** for whichever idea gets promoted first — adjust if the user directs otherwise, but don't invent a wildly different structure without checking with the user first, since it will become the precedent for everything after it. One subfolder per idea/project:

```
My Creations/
  <Idea Name>/
    idea.md              ← the raw one-liner / spark, dated
    research.md           ← feasibility, prior art, market/tech research backing the idea
    PRD.md / 方案.md       ← the actual spec once the idea is validated enough to scope
    implementation notes  ← build log, decisions, status — as the project moves into build phase
```

Not every idea will reach every stage — many will legitimately stop at `idea.md` or `research.md` forever. Don't force an idea through the full pipeline; the folder should honestly reflect how far it actually got.

## Workflow guidance for agents

1. New spontaneous idea → goes into `Quick Ideas.md` as a one-line checklist item, capture-only, no action taken.
2. When the user explicitly asks to move forward with an idea (from `Quick Ideas.md` or stated fresh), create its subfolder and start with `idea.md` — capture it as stated, don't editorialize or expand scope yet.
3. When asked to research an idea, write findings into that idea's `research.md`, not into `My Notes/` — the research is specifically in service of this idea, not general knowledge.
4. Only produce a `PRD.md`/`方案.md` when explicitly asked to formalize an idea into a spec — don't jump straight from `idea.md` to a PRD unprompted.
5. Implementation records (once building starts) stay in the idea's own subfolder too, unless the user says otherwise.

## Note conventions (same as rest of vault)

- Internal links use wiki-link syntax `[[Note Name]]`, not markdown `[text](path)`.
- Mixed Chinese/English is normal.
- `==text==` for inline highlight emphasis.
