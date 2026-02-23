# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AI Daily Digest — a zero-dependency, single-file TypeScript script that fetches RSS feeds from 90 top Hacker News blogs, uses AI (Gemini/OpenAI) to score, filter, and summarize articles, then generates a bilingual (Chinese/English) Markdown digest with Mermaid charts and ASCII visualizations.

## Running the Script

```bash
# Primary invocation (uses Bun runtime)
npx -y bun scripts/digest.ts

# With options
npx -y bun scripts/digest.ts --hours 24 --top-n 20 --lang en --output ./output/digest.md
```

No package.json, no build step, no test suite. The project is a standalone script.

## Environment Variables

- `GEMINI_API_KEY` — Primary AI provider (Gemini 2.0 Flash, free tier)
- `OPENAI_API_KEY` — Fallback provider (optional)
- `OPENAI_API_BASE` — Fallback API base URL (optional)
- `OPENAI_MODEL` — Fallback model name (optional)

User config persists at `~/.hn-daily-digest/config.json`.

## Architecture

Everything lives in `scripts/digest.ts` (~1,186 lines), organized as a five-stage pipeline:

1. **RSS Fetching** (lines ~293-363) — Concurrent fetch of 90 feeds (10 concurrent, 15s timeout per feed)
2. **Time Filtering** — Filter articles to specified hour window (default 48h)
3. **AI Scoring** (lines ~505-623) — Batch scoring (relevance/quality/timeliness 1-10) with category classification
4. **AI Summarization** (lines ~625-728) — Structured summaries + Chinese title translation
5. **Trend Analysis + Report** (lines ~730-995) — Macro trends, Mermaid charts, ASCII charts, tag clouds, final Markdown output

Key sections in the file:
- Lines 1-111: Constants, API URLs, RSS feed list
- Lines 113-173: TypeScript interfaces
- Lines 175-291: Manual RSS/Atom XML parser (no external libs)
- Lines 365-503: AI provider clients (Gemini primary, OpenAI-compatible fallback with automatic failover)
- Lines 997-1186: CLI argument parsing and interactive flow

## Design Decisions

- **Zero dependencies**: No npm packages — uses native `fetch`, manual XML parsing, built-in Bun APIs
- **Graceful degradation**: Gemini → OpenAI-compatible automatic failover; failed feeds don't halt the process
- **Batch processing**: Articles scored in batches of 10, max 2 concurrent AI calls
- **Single file by design**: Intentionally monolithic for portability — don't split into modules

## Six Article Categories

AI/ML, Security, Engineering, Tools/Open Source, Opinion, Other

## Skill Integration

This project is also a Claude Code skill (`/digest`). The skill definition is in `SKILL.md` and provides an interactive guided flow for parameter collection.
