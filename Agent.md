# Agent Instructions — MySecondBrain

This is Zihan's personal **Obsidian vault** ("an external knowledge base shared with AI system") — not a code repository. There is no build/lint/test. Mixed Chinese/English content is normal; do not translate or "clean up" language.

## Vault map

```
Agent.md              ← this file
Quicknotes.md         ← ACTIVE agent task queue, read this first (see below)
README.md
My Works/
  My works root.md     ← top-level index, links out via [[wiki-links]]
  IntelProjects/        ← gitignored (see below). Work/company-sensitive notes, debug transcripts.
  LLM/
    LLM  Root.md        ← index for the LLM knowledge domain (note: literal double space in filename)
    Agent Framework/     agent framework comparisons/notes (opencode, Claude Code, Claws, MetaEquipe)
    Agent Prompt/        numbered notes (1..5) on prompting/benchmarks/RL
    Biz/                 commercial/market research (OpenClaw use cases, WAIC, etc.)
    LLM security/        security/attack-case notes
    Techs/
      Agent Platform/    raw agent system-prompt dumps (e.g. cline)
      LLM router/        most mature research area — see LLMRouter.md as the gold-standard note format
    models/               currently empty
    实用模板/             curated reference links/articles (NOT literal note templates despite the name)
```

## Existing agent protocol — `Quicknotes.md` (do not skip this)

`Quicknotes.md` is a live checklist the user maintains for agents:
1. Read the "Topic List" checkboxes (`- [ ]`).
2. For each unfinished topic, do the research yourself.
3. Decide which existing folder (see map above) the finding belongs in, and write a note there — reuse the folder taxonomy, don't invent a new top-level category without a good reason.
4. Check the box (`- [x]`) in `Quicknotes.md` once the note is written.

If asked to "survey/research topics", this file is the source of truth for what's pending.

## Note conventions (Obsidian-specific — preserve these)

- **Internal links use wiki-link syntax**: `[[Note Name]]`, not markdown `[text](path)`. Use the exact filename including quirks (e.g. `LLM  Root` has a double space) — do not "fix" filenames when linking or renaming, it breaks existing backlinks.
- **Images/attachments** live in auto-generated sibling folders named `<Note Name> assets/` or `<Note Name>_assets/` (Obsidian manages these inconsistently — don't rename or restructure them). Embed with `![[Pasted image ....png]]`.
- `==text==` is used for inline highlight emphasis.
- Note quality varies intentionally: many notes are just a single pasted URL (bookmark-style) or raw pasted AI-chat transcripts — this is fine and expected. When writing a genuinely new researched note, prefer the structured style of `My Works/LLM/Techs/LLM router/LLMRouter.md` (blockquote summary, TOC with anchor links, tables, code fences) as the target quality bar, not the one-line bookmark style.

## Git behavior

- The `obsidian-git` plugin auto-commits on a schedule with messages like `vault backup: 2026-07-30 10:26:04`. This is automatic backup history, not meaningful development history — don't try to write "conventional" commit messages to match it, and don't commit unless explicitly asked.
- `.gitignore` excludes `.obsidian/` (app config), `copilot/` (Obsidian Copilot plugin cache/chat data), and `*/IntelProjects/` (anywhere in the tree) — content placed under any `IntelProjects/` folder is intentionally untracked/private; treat it as sensitive.

## Installed Obsidian plugins (context only)

`obsidian-git`, `omnisearch`, `copilot` (AI chat plugin, data in gitignored `copilot/`), `table-editor-obsidian`, `editing-toolbar`.
