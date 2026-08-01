# Agent Instructions — MySecondBrain

This is Zihan's personal **Obsidian vault** ("an external knowledge base shared with AI system") — not a code repository. There is no build/lint/test. Mixed Chinese/English content is normal; do not translate or "clean up" language.

## Vault map

Three top-level modules, each with a **narrow, non-overlapping scope** and its own `Agent.md` for module-specific detail. Read the relevant module's `Agent.md` before writing into it — this file only covers what's vault-wide.

```
Agent.md              ← this file (vault-wide)
Quicknotes.md         ← ACTIVE task queue for LEARNING topics only (see below) — feeds My Notes/
README.md
My Notes/              learning notes + organized content absorbed from external sources
  Agent.md              ← module-specific instructions
  My notes root.md      ← top-level index
  LLM/                   the only knowledge domain so far (agent frameworks, prompting, base models, security, techs...)
My Creations/           personal ideas → research → PRD/方案 → implementation
  Agent.md              ← module-specific instructions
  My creations root.md  ← top-level index (currently empty, no ideas promoted yet)
  Quick Ideas.md        ← ACTIVE capture-only list of spontaneous ideas (lighter-weight than Quicknotes.md — see below)
My Works/               specific work-project records ONLY (not general learning, not personal ideas)
  Agent.md              ← module-specific instructions
  My works root.md      ← top-level index
  IntelProjects/         ← gitignored (see below). Work/company-sensitive notes, debug transcripts.
```

**Routing rule for new content** (check before writing anywhere):
- Learned/organized fact from an external source (article, paper, tech blog, "survey this topic") → `My Notes/`
- Zihan's own idea/venture being developed (idea → research → spec → build) → `My Creations/`
- A record tied to a specific, named work project (his day job) → `My Works/`

If genuinely unsure which module a piece of content belongs to, ask rather than guessing — the three scopes are intentionally kept from overlapping.

## Two parallel task queues (do not conflate them)

- **`Quicknotes.md`** (vault root) — *learning* topics. Heavyweight protocol: research the topic yourself, write a real note, file it under the right module (almost always `My Notes/`), then check it off.
  1. Read the "Topic List" checkboxes (`- [ ]`).
  2. For each unfinished topic, do the research yourself.
  3. Decide which module + folder the finding belongs in (see routing rule above), and write a note there — reuse the existing folder taxonomy inside that module, don't invent a new top-level category without a good reason.
  4. Check the box (`- [x]`) in `Quicknotes.md` once the note is written.
- **`My Creations/Quick Ideas.md`** — *build* ideas ("灵机一动"). Capture-only: just log the idea as a checklist item, don't research or build anything unprompted. Only promote an idea into its own `My Creations/<Idea Name>/` subfolder when the user explicitly says to move forward with it — see `My Creations/Agent.md` for that pipeline.

If asked to "survey/research topics" → `Quicknotes.md`. If the user drops a spontaneous idea/thing-to-build → `My Creations/Quick Ideas.md`, not `Quicknotes.md`.

## Note conventions (Obsidian-specific — preserve these, vault-wide)

- **Internal links use wiki-link syntax**: `[[Note Name]]`, not markdown `[text](path)`. Use the exact filename including quirks (e.g. `LLM  Root` has a double space) — do not "fix" filenames when linking or renaming, it breaks existing backlinks. **Caveat**: every module now has its own `Agent.md` — these are *not* mutually unique note titles, so don't wiki-link them (`[[Agent]]` would be ambiguous); reference them by relative path in prose instead.
- **Images/attachments** live in auto-generated sibling folders named `<Note Name> assets/` or `<Note Name>_assets/` (Obsidian manages these inconsistently — don't rename or restructure them). Embed with `![[Pasted image ....png]]`.
- `==text==` is used for inline highlight emphasis.
- Note quality varies intentionally: many notes are just a single pasted URL (bookmark-style) or raw pasted AI-chat transcripts — this is fine and expected. When writing a genuinely new researched note, prefer the structured style of `My Notes/LLM/Techs/LLM router/LLMRouter.md` (blockquote summary, TOC with anchor links, tables, code fences) as the target quality bar, not the one-line bookmark style.

## Git behavior

- The `obsidian-git` plugin auto-commits on a schedule with messages like `vault backup: 2026-07-30 10:26:04`. This is automatic backup history, not meaningful development history — don't try to write "conventional" commit messages to match it, and don't commit unless explicitly asked.
- `.gitignore` excludes `.obsidian/` (app config), `copilot/` (Obsidian Copilot plugin cache/chat data), and `*/IntelProjects/` (anywhere in the tree) — content placed under any `IntelProjects/` folder is intentionally untracked/private; treat it as sensitive.

## Installed Obsidian plugins (context only)

`obsidian-git`, `omnisearch`, `copilot` (AI chat plugin, data in gitignored `copilot/`), `table-editor-obsidian`, `editing-toolbar`.
