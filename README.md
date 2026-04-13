# Agent Autopsy

Catalog and analysis of agentic AI security research from [Embrace The Red](https://embracethered.com/blog/) (wunderwuzzi / Johann Reberger). Each write-up is indexed by **exploit type** (primary) and **ecosystem** (secondary). Per-post extractions capture: vulnerability, research method, and exploit process.

## Scope

- **In scope:** Agentic AI security (prompt injection, data exfiltration, RCE, zombie agents, MCP, tool abuse, jailbreaks, invisible injection).
- **Corpus:** (1) Month of AI Bugs 2025 (29 posts, August 2025) from the [wrap-up page](https://embracethered.com/blog/posts/2025/wrapping-up-month-of-ai-bugs/); (2) agentic-AI security posts from the [full blog index](https://embracethered.com/blog/) (2022–2026). Merged and deduplicated: **105 posts** (ETR-001 … ETR-105).

## IDs

Stable post IDs assigned **newest-first**: `ETR-001` = most recent post (2026-02-11), `ETR-105` = oldest post (2022-04-01). IDs are used in extraction records, cross-references, and lesson file names.

## Layout

| Path | Purpose |
|------|---------|
| `embrace-the-red-master-index.md` | All 105 posts by ETR-ID with exploit type, ecosystem, and video links. |
| `embrace-the-red-index-by-exploit-type.md` | All 105 posts grouped by exploit type (primary view). |
| `embrace-the-red-index-by-ecosystem.md` | All 105 posts grouped by target ecosystem (secondary view). |
| `data/master-index.json` | Machine-readable merged index (105 posts, both sources). |
| `data/month-of-ai-bugs-2025.json` | The 29 Month of AI Bugs posts (with video URLs). |
| `data/blog-index-additions.json` | The 76 blog index posts. |
| `data/extractions-etr-001-to-etr-010.json` | Phase 2 extractions: ETR-001–010 |
| `data/extractions-etr-011-to-etr-020.json` | Phase 2 extractions: ETR-011–020 |
| `data/extractions-etr-021-to-etr-031.json` | Phase 2 extractions: ETR-021–031 |
| `data/extractions-etr-032-to-etr-036.json` | Phase 2 extractions: ETR-032–036 |
| `data/extractions-etr-037-to-etr-050.json` | Phase 2 extractions: ETR-037–050 |
| `data/extractions-etr-096-to-etr-100.json` | Phase 2 extractions: ETR-096–100 |
| `data/extractions-etr-101-to-etr-105.json` | Phase 2 extractions: ETR-101–105 |
| `lessons/` | Standalone narrative lessons keyed by ETR-ID. Each lesson has: one-sentence takeaway, architecture + exploit Mermaid diagrams, step table, collapsible detail sections, security notes. |
| `lessons/README.md` | Lesson index, format requirements, and validation notes. |
| `scripts/build_index_views.py` | Regenerates the three markdown index files from `data/master-index.json`. |
| `scripts/validate_mermaid.mjs` | Validates all Mermaid diagrams in lesson files (`npm run validate:mermaid`). |
| `package.json` | Node dependencies (mermaid for validation). |

## Extractions

Phase 2 extractions cover the **top 50 posts by most recent date** (ETR-001–050), plus ETR-096–105. Each extraction record contains:

- `vulnerability`: short label, attack class, affected component, CVE (if assigned)
- `research_method`: discovery, tooling, reproduction, validation
- `exploit_process`: steps, prerequisites, impact, detection notes

## Lessons

30 narrative lessons currently exist, covering recent high-signal posts (ETR-001–015, ETR-032–036) and foundational/introductory posts (ETR-096–105). Each lesson follows a shared format:

- **In one sentence:** one-line takeaway
- **Overview**
- **Core Technologies and Architecture** (Mermaid diagram + text)
- **Core Concepts** (optional third diagram)
- **Exploit Mechanism** (Mermaid diagram, step table, numbered walkthrough, optional `<details>`)
- **Security**, **Summary**, and **References**

Run `npm run validate:mermaid` to verify all Mermaid syntax before committing.

## Status

| Phase | Status |
|-------|--------|
| Phase 1: Discovery and master index | Complete. 105 posts, both index views produced. |
| Phase 2: Per-post extractions | In progress. Top 50 by recency extracted (ETR-001–050). ETR-096–105 also extracted. |
| Phase 3: Full collection and unified output | Pending completion of Phase 2. |

## Attribution

All source material is from [Embrace The Red](https://embracethered.com/blog/) (Johann Reberger / wunderwuzzi). This project is for research and educational reference; attribution is retained in all outputs.
