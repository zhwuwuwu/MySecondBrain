# Agent Instructions — My Creations

**Scope**: personal ideas taken through a pipeline: **idea → research → PRD/方案 → implementation**. ONLY things Zihan is personally originating/building. If it's a fact/topic learned from an external source with no personal build attached → belongs in `My Notes/` instead. If it's a record of a specific work project (not a personal venture) → belongs in `My Works/` instead.

## Current state

Empty — no ideas logged yet. The structure below is a **proposed convention** for the first agent/session that adds something here; adjust if the user directs otherwise, but don't invent a wildly different structure without checking with the user first, since it will become the precedent for everything after it.

## Proposed structure

One subfolder per idea/project:

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

1. When the user drops a new idea, create its subfolder and start with `idea.md` — capture it as stated, don't editorialize or expand scope yet.
2. When asked to research an idea, write findings into that idea's `research.md`, not into `My Notes/` — the research is specifically in service of this idea, not general knowledge.
3. Only produce a `PRD.md`/`方案.md` when explicitly asked to formalize an idea into a spec — don't jump straight from `idea.md` to a PRD unprompted.
4. Implementation records (once building starts) stay in the idea's own subfolder too, unless the user says otherwise.

## Note conventions (same as rest of vault)

- Internal links use wiki-link syntax `[[Note Name]]`, not markdown `[text](path)`.
- Mixed Chinese/English is normal.
- `==text==` for inline highlight emphasis.
