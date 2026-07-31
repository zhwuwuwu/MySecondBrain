# Agent Instructions — My Notes

**Scope**: learning notes and organized content absorbed from external sources ONLY. If it's a personal idea/venture you're developing → belongs in `My Creations/` instead. If it's a record of a specific work project → belongs in `My Works/` instead.

## What lives here

```
My notes root.md      ← index, links out via [[wiki-links]]
LLM/
  LLM  Root.md          ← index for the LLM knowledge domain (note: literal double space in filename)
  Agent Application/     agent application/use-case notes
  Agent Framework/       agent framework comparisons/notes (opencode, Claude Code, Claws, MetaEquipe)
  Agent Self Evo/        agent self-evolution / RL / prompt auto-optimization notes
  Base Models/           architecture/training evolution surveys (e.g. GPT-2 → Kimi K3), model-specific deep dives
  Biz/                   commercial/market research (OpenClaw use cases, WAIC, etc.)
  LLM security/          security/attack-case notes
  Techs/
    Agent Platform/      raw agent system-prompt dumps (e.g. cline)
    LLM router/          most mature research area — see LLMRouter.md as the gold-standard note format
```

This folder listing is a snapshot — the user actively reorganizes subfolders here (folder names/taxonomy may have shifted since this was written). If it looks stale, trust the actual directory listing over this doc, and consider refreshing this section while you're at it.

`LLM/` is currently the only knowledge domain. If a genuinely new domain shows up (not LLM-related), give it its own top-level folder here (sibling to `LLM/`) rather than cramming it into an unrelated existing subfolder.

## Primary workflow: `Quicknotes.md` (at vault root, not here)

The vault-root `Quicknotes.md` is the live task queue that feeds this module: it lists topics to survey, and the resulting notes get filed under here (usually `LLM/...`, or a new sibling domain folder). See the root `Agent.md` for the full checklist protocol. When a research task's output is a "learned/organized fact from an external source", it lands in `My Notes/` — that's the defining criterion for this module.

## Note conventions (Obsidian-specific — preserve these)

- **Internal links use wiki-link syntax**: `[[Note Name]]`, not markdown `[text](path)`. Use the exact filename including quirks (e.g. `LLM  Root` has a double space) — do not "fix" filenames when linking or renaming, it breaks existing backlinks.
- **Images/attachments** live in auto-generated sibling folders named `<Note Name> assets/` or `<Note Name>_assets/` (Obsidian manages these inconsistently — don't rename or restructure them). Embed with `![[Pasted image ....png]]`.
- `==text==` is used for inline highlight emphasis.
- Note quality varies intentionally: many notes are just a single pasted URL (bookmark-style) or raw pasted AI-chat transcripts — this is fine and expected for quick capture. When writing a genuinely researched note, prefer the structured style of `LLM/Techs/LLM router/LLMRouter.md` or `LLM/Base Models/GPT2 to Kimi K3 技术路径.md` (blockquote summary, TOC with anchors, tables, sourced citations) as the quality bar.
- Mixed Chinese/English is normal — don't translate or "clean up" language.
