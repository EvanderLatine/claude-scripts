# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository is a collection of Claude Code skills, prompt templates, and reference documentation for content production workflows. There is no build system, package manager, or test suite — the project consists entirely of markdown files, shell scripts, and zip archives.

## Repository Structure

- `.claude/skills/` — Installed Claude Code skills (each skill is a directory with `SKILL.md` as entrypoint)
- `skills/` — Source zip archives of skills (originals before extraction)
- `docs/TEAM_LEAD_PROMPT.md` — Multi-agent Team Lead prompt for YouTube script production (~80KB, detailed orchestration protocol)
- `docs/NICHE_STYLE_GUIDE_TEMPLATE.md` — Шаблон адаптации под конкретный канал и нишу (заполнить и сохранить как `docs/ref/NICHE_STYLE_GUIDE.md` в проекте скрипта)
- `docs/specific/` — Reference documents (video attention strategies, skill installation guide)

## Skills

Six project-level skills are configured in `.claude/skills/`:

| Skill | Invocation | Key Tools |
|---|---|---|
| `article-extractor` | `/article-extractor <url>` | Bash, Write |
| `content-research-writer` | `/content-research-writer` | (default) |
| `editor` | `/editor` | (default) |
| `video-analyzer` | `/video-analyzer <video_path>` | Bash (ffmpeg) |
| `video-report` | `/video-report` | (default) |
| `youtube-transcript` | `/youtube-transcript <url>` | Bash, Read, Write |

`video-analyzer` bundles a helper script at `scripts/extract_frames.sh` — reference it via `${CLAUDE_SKILL_DIR}/scripts/extract_frames.sh`.

## Permissions

Git write operations are denied in `.claude/settings.local.json` — all git commands that modify state (commit, push, checkout, merge, etc.) are blocked. Read-only git commands (status, log, diff) are allowed. File read/write/edit is scoped to this project directory only.

## Language

User communicates in Russian. Documentation and prompts are a mix of Russian and English.
