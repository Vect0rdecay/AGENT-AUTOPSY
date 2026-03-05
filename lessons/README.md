# Agent Autopsy: Standalone Lessons

Narrative lessons for the Embrace The Red corpus. Each lesson summarizes the source post, explains the concepts and frameworks needed to understand the vulnerability and exploit, and is written so a reader can follow without searching for background. Analogies are used where they clarify technical ideas.

Each lesson includes a **Core Technologies and Architecture** section that covers: (1) the relevant **AI/LLM components** (e.g., tokenization, transformers and attention, how the prompt is assembled, where system vs user text lives, inference pipeline), and (2) **integration with web and application stacks** (e.g., HTTP/API flow, how chat UIs and backends build and send prompts, where safety layers sit, how "analyze this webpage" becomes part of the prompt). This gives students enough of the underlying architecture to see why the vulnerabilities and exploits work.

Each lesson ends with a **References** section listing links for sources used to provide additional information beyond the original ETR post (e.g., API docs, protocol specs, related ETR posts). ETR-036 through ETR-032 include References; ETR-105 through ETR-096 will be updated to add References in a follow-up pass.

**Audience:** Students and engineers learning AI security. Fix status is omitted; the focus is on attack concepts, infrastructure, and defensive thinking.

**Lesson format (required for new lessons):** Each lesson must include two **Mermaid diagrams**: (1) an **architecture diagram** in the Core Technologies and Architecture section (data flow, components, or where the attack surface lives), and (2) an **exploit mechanism diagram** in the Exploit Mechanism section (step-by-step attack flow). Use fenced code blocks with the `mermaid` language tag. Prefer `sequenceDiagram` for step-by-step flows or `flowchart` for component/decision layout. Avoid double quotes inside flowchart node labels (e.g. use `[User asks: Explain this file]` not `[User: "Explain this file"]`) so diagrams render reliably on GitHub. Run `npm run validate:mermaid` from the project root to check all diagrams parse correctly.

## Lessons (by ETR-ID)

| ID | Title | Concepts covered |
|----|-------|------------------|
| [ETR-001_scary_agent_skills_unicode.md](ETR-001_scary_agent_skills_unicode.md) | Scary Agent Skills: hidden Unicode instructions | Invisible Unicode in Skills, supply-chain backdoor, Tag codepoints, scanner/aid |
| [ETR-002_openai_data_exfiltration_mitigation_paper.md](ETR-002_openai_data_exfiltration_mitigation_paper.md) | OpenAI URL-based data exfiltration mitigations paper | Allow-list, crawler, dynamic URL block, bypass ideas |
| [ETR-003_agentic_probllms_39c3.md](ETR-003_agentic_probllms_39c3.md) | Agentic ProbLLMs: exploiting computer-use and coding agents (39C3) | RCE via agent tools, ZombAI/C2, AI ClickFix, Month of AI Bugs, multi-vendor agentic systems |
| [ETR-004_normalization_of_deviance_in_ai.md](ETR-004_normalization_of_deviance_in_ai.md) | The normalization of deviance in AI | Vaughan/Challenger, untrusted LLM output, cultural drift, trust boundaries |
| [ETR-007_cross_agent_privilege_escalation.md](ETR-007_cross_agent_privilege_escalation.md) | Cross-agent privilege escalation: when agents free each other | Multi-agent config writes, indirect injection, escalation loop, shared filesystem |
| [ETR-036_chatgpt_chat_history_exfiltration.md](ETR-036_chatgpt_chat_history_exfiltration.md) | ChatGPT: chat history and memories exfiltration | url_safe bypass, system prompt as exfil target, indirect injection via PDF/web |
| [ETR-035_chatgpt_codex_zombai.md](ETR-035_chatgpt_codex_zombai.md) | ChatGPT Codex ZombAI | Common Dependencies allowlist, indirect injection via GitHub issue, C2 on azure.com |
| [ETR-034_anthropic_filesystem_mcp_bypass.md](ETR-034_anthropic_filesystem_mcp_bypass.md) | Anthropic Filesystem MCP directory bypass | Path validation, startsWith bypass, CVE-2025-53109, MCP server security |
| [ETR-033_cursor_mermaid_exfiltration.md](ETR-033_cursor_mermaid_exfiltration.md) | Cursor: data exfiltration via Mermaid | Mermaid image URL fetch, CVE-2025-54132, memories and context exfil, zero-click style |
| [ETR-032_amp_code_rce.md](ETR-032_amp_code_rce.md) | Amp Code: arbitrary command execution | Config file write, MCP/server allowlist abuse, sandbox escape via settings.json |
| [ETR-105_gpt3_phishing.md](ETR-105_gpt3_phishing.md) | GPT-3 and Phishing Attacks | LLMs, generative AI abuse, phishing at scale, dual-use |
| [ETR-104_chatgpt_database.md](ETR-104_chatgpt_database.md) | ChatGPT as a database server | System vs user prompt, role-play, direct prompt injection, simulation vs execution |
| [ETR-103_yolo_natural_language_to_shell.md](ETR-103_yolo_natural_language_to_shell.md) | Yolo: NL to shell commands | NL-to-command pipelines, RCE, API usage, treating model output as untrusted |
| [ETR-102_bing_chat_bank_robbery.md](ETR-102_bing_chat_bank_robbery.md) | Bing Chat "bank robbery" jailbreak | Jailbreaks, indirect phrasing, keyword filter bypass, persona shift |
| [ETR-101_ai_injections_direct_and_indirect.md](ETR-101_ai_injections_direct_and_indirect.md) | AI Injections: direct and indirect | Prompt injection, direct vs indirect (second-order), comparison to SQLi/XSS, cross-context |
| [ETR-100_dont_trust_llm_responses.md](ETR-100_dont_trust_llm_responses.md) | Don't blindly trust LLM responses | Output as untrusted, link unfurling/exfil, custom commands, XSS/SQLi/RCE in context |
| [ETR-099_mlsecops_podcast_threat_modeling.md](ETR-099_mlsecops_podcast_threat_modeling.md) | MLSecOps Podcast: threat modeling ML | Threat modeling ML/LLM apps, red teaming, MLSecOps, data flow and trust boundaries |
| [ETR-098_prompt_injections_intro_video.md](ETR-098_prompt_injections_intro_video.md) | Video: Prompt Injections intro | Natural language bypass of validation, LLM as exploitation engine, indirect + plugins |
| [ETR-097_adversarial_prompting_tutorial_lab.md](ETR-097_adversarial_prompting_tutorial_lab.md) | Adversarial Prompting: tutorial and lab | JSON/HTML injection via model, OrderBot overwrite, Colab lab, output as untrusted |
| [ETR-096_indirect_injection_youtube_transcripts.md](ETR-096_indirect_injection_youtube_transcripts.md) | Indirect injection via YouTube transcripts | Plugin flow, transcript as channel, session hijacking, content owner as attacker |

## Related data

- Structured extractions for ETR-032 to ETR-036 (August 2025): `data/extractions-etr-032-to-etr-036.json`
- Structured extractions for ETR-096 to ETR-100: `data/extractions-etr-096-to-etr-100.json`
- Structured extractions for ETR-101 to ETR-105: `data/extractions-etr-101-to-etr-105.json`

Fix status is not included in any extraction file.
