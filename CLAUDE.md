# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**handoff** — 人間とAIの共同タスクボード (a shared task board for human–AI collaboration).

This is a **greenfield repository**. As of this writing the only tracked files are `README.md`, `LICENSE` (MIT), and `.gitignore`. There is no source code, `package.json`, build tooling, or test setup yet — so there are no build/lint/test commands to run. The `.gitignore` is the standard Node.js template and lists Next.js, Vite, Nuxt, SvelteKit, etc., so a JavaScript/TypeScript web stack is anticipated but not yet committed.

When the first real implementation lands, update this file with the actual commands (build, dev, lint, single-test invocation) and the architecture once it spans more than one file.

## Working methodology (`.claude/skills/engineering/`)

This repo carries an installed set of engineering skills (the "Matt Pocock" collection) under `.claude/` that define how work should flow here. They are not committed to git but are part of the working tree. Key ones, invoked as slash commands:

- **to-prd** / **to-issues** — turn a conversation or plan into a PRD, then break it into independently-grabbable issues using vertical slices.
- **triage** — move issues through a state machine of triage-role labels.
- **tdd** — red-green-refactor, one vertical slice at a time (preferred for new features and bug fixes).
- **diagnose** — disciplined loop for hard bugs/perf regressions: reproduce → minimise → hypothesise → instrument → fix → regression-test.
- **prototype** — throwaway prototype to flesh out a design before committing to it.
- **improve-codebase-architecture** / **zoom-out** / **grill-with-docs** — architecture deepening, higher-level orientation, and sharpening plans against the domain model.

Some of these (`to-issues`, `to-prd`, `triage`) have a **hard dependency** on per-repo config (issue tracker, triage label vocabulary, domain doc layout). If that config is missing, run **setup-matt-pocock-skills** first — otherwise their output targets the wrong tracker/labels. The other skills degrade gracefully without it. See `.claude/docs/adr/0001-*.md` for the rationale.

## Domain language & decisions

- `.claude/CONTEXT.md` holds the domain glossary (canonical terms, e.g. *Issue tracker* / *Issue* / *Triage role*, and which phrasings to avoid). Consult and update it when terminology shifts.
- `.claude/docs/adr/` holds architecture decision records. Check the relevant ADR before changing behavior in the area it covers, and add a new ADR for non-obvious decisions.
