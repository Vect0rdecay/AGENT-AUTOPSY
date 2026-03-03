# Agent Autopsy

Catalog and analysis of agentic AI security research from [Embrace The Red](https://embracethered.com/blog/) (wunderwuzzi). Each write-up is indexed by **exploit type** (primary) and **ecosystem** (secondary). Per-post extraction captures: vulnerability, research method, and exploit process.

## Scope

- **In scope:** Agentic AI security (prompt injection, data exfiltration, RCE, zombie agents, MCP, tool abuse).
- **Corpus:** (1) Month of AI Bugs 2025 (29 posts, August 2025) from the [wrap-up page](https://embracethered.com/blog/posts/2025/wrapping-up-month-of-ai-bugs/); (2) agentic-AI security posts from the [full blog index](https://embracethered.com/blog/) (2022–2026). Merged and deduplicated: **105 posts** (ETR-001 … ETR-105).

## Layout

| Path | Purpose |
|------|---------|
| `embrace-the-red-master-index.md` | Master index: 29 from Month of AI Bugs (table); full 105 in `data/master-index.json`. |
| `embrace-the-red-index-by-exploit-type.md` | Index by exploit type (primary view; all 105 posts). |
| `embrace-the-red-index-by-ecosystem.md` | Index by ecosystem (secondary view; all 105 posts). |
| `data/master-index.json` | Machine-readable merged index (105 posts, both sources). |
| `data/month-of-ai-bugs-2025.json` | The 29 from Month of AI Bugs (with video URLs). |
| `data/blog-index-additions.json` | The 76 from the blog index (video URLs unknown). |
| `lessons/` | Standalone narrative lessons: 01–10 (ETR-030–ETR-039, blog index); 11–15 (ETR-001–ETR-005, August 2025). Each lesson has Core Technologies and Architecture; 11–15 include a References section. |

## IDs

Stable post IDs: `ETR-001` … `ETR-029` = Month of AI Bugs (2025-08-01 through 2025-08-29). `ETR-030` … `ETR-105` = Blog index additions (chronological by post date). Use in extraction records and cross-references.

## Attribution

All source material is from Embrace The Red (Johann Reberger / wunderwuzzi). This project is for research and reference; attribution is retained in all outputs.
