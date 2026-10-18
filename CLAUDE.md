# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Claude Code plugin marketplace of Agent Skills for TM Forum's Open Digital Architecture (ODA), backed by a generated, machine-readable knowledge base. It is mostly Markdown/JSON content plus a Python pipeline (`tools/`); there is no test suite or linter. Design rationale lives in `spec/spec.md`; build history and bug/decision log in `spec/tasks.md` (each feature area has its own `spec-*.md` / `tasks-*.md` pair).

## Pipeline: references → knowledge → skills → dist

```
references/  raw DOCX/PDF/Excel, unmodified (member-gated TM Forum content; committed on purpose)
knowledge/   GENERATED agent-facing corpus (use-cases/, components/, apis/, etom/, index/)
skills/      canonical, hand-authored SKILL.md files (read knowledge/, never references/)
dist/        GENERATED plugin bundles: dist/consumer and dist/creator (marketplace points here)
```

- **Do not hand-edit generated files** under `knowledge/` (component.yaml, `*.meta.json`, index JSON, `TMFSxxx.md`/`TMFCxxx.md` bodies) or anything under `dist/`; re-run the producing tool or the next refresh overwrites it. `knowledge/AGENTS.md` and each subfolder's `AGENTS.md` list the few deliberately hand-maintained files — read them before touching `knowledge/`.
- Cross-references between use cases, components and APIs live only in `knowledge/index/*.json`. Look ids up there rather than guessing paths.
- Absence is meaningful: a missing `.md`/`.yaml` is surveyed information (planned / not published). Never fabricate content to fill it.
- Component `component.yaml` version and its PDF spec version are independent; report both, don't reconcile.

## Commands

```bash
pip install -r tools/requirements.txt          # PyYAML, python-docx, pdfplumber, openpyxl

# Automated refresh track (public sources)
python tools/fetch_component.py --all-referenced && python tools/fetch_api.py

# Always finish a refresh with:
python tools/build_index.py                    # regenerates use-cases/components/apis.json (idempotent; runs validate_envelope.py)
python tools/refresh_report.py                 # appends dated entry to CHANGELOG.md

python tools/validate_envelope.py --strict     # check every artefact has id/type/name/version/status

# After ANY change to skills/ or knowledge/:
python tools/build_plugin.py                   # regenerate dist/consumer and dist/creator
```

The assisted track (IG1228 + `TMFSxxx` use cases) needs a logged-in TM Forum session and manual download; follow `spec/refresh-runbook.md`. Every script in `tools/` opens with a `Reads:/Writes:/Track:` docstring header — check it before running.

`generate-component-svg-diagram` is the one skill with executable code (Node ≥ 18, D3 + jsdom): `npm install` in its `scripts/` folder; helm needed only for rendering a Helm chart.

## Skills authoring and packaging

- `skills/*/SKILL.md` are written with plain relative paths (`knowledge/use-cases/...`) so they work in a direct clone. `build_plugin.py` rewrites them to `${CLAUDE_PLUGIN_ROOT}/knowledge/...` in `dist/` copies only (plugins install to a cache dir, not the user's cwd). Never put `${CLAUDE_PLUGIN_ROOT}` in `skills/`.
- Audience split is defined in `tools/build_plugin.py`: consumer vs creator skill sets, a `SHARED_SKILLS` bucket shipped in both (currently `generate-component-svg-diagram`), and `INTERNAL_ONLY_SKILLS` (`process-usecase-media`, `process-component-media`) which write to `knowledge/` and are excluded from both plugins. Each plugin bundles a full verbatim copy of `knowledge/`.
- A new skill must be added to the appropriate set in `build_plugin.py` (and its `SKILL_EXAMPLES` entry) and to the README skill tables; `skills/` stays flat (no stage subfolders — see `spec/spec-skills-consumer.md` §9).
- Plugin/marketplace manifests: `.claude-plugin/marketplace.json` (repo root) and `dist/*/.claude-plugin/plugin.json`; bump plugin versions on skill changes.
- Skills are read-only against `knowledge/` and make no network calls at run time (except the noted build/deploy skill `implement-oda-component`).
