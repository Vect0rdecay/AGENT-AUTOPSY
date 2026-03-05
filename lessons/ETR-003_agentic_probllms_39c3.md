# ETR-003: Agentic ProbLLMs: Exploiting AI Computer-Use And Coding Agents (39C3 Video + Slides)

**Source:** [Agentic ProbLLMs: Exploiting AI Computer-Use And Coding Agents (39C3 Video + Slides)](https://embracethered.com/blog/posts/2025/39c3-agentic-probllms-exploiting-computer-use-and-coding-agents/)

**In one sentence:** Attacker places instructions in content the agent consumes (issues, READMEs, docs); the victim asks the agent to work on it; the model follows those instructions as a confused deputy and invokes tools (config write, curl and bash), leading to RCE, exfil, or ZombAI/C2.

---

## Overview

The post announces a 39C3 (Chaos Communication Congress 2025) talk that presents security research on agentic AI systems and the Month of AI Bugs (August 2025). The talk demonstrates end-to-end prompt injection exploits against computer-use and coding agents across multiple vendors (e.g., Anthropic Claude Code, GitHub Copilot, Google Jules, Devin AI, ChatGPT Operator, Amazon Q, AWS Kiro). Exploits affect confidentiality, integrity, and automation trust: remote code execution (RCE), exfiltration of access tokens and sensitive data, and recruitment of agents into traditional command-and-control (C2) infrastructure (ZombAIs). The talk also covers adaptation of nation-state tactics such as ClickFix to computer-use systems (AI ClickFix) and how they can lead to full system compromise. Recurring issues include over-reliance on LLM behavior for trust decisions, inadequate tool sandboxing, and weak user-in-the-loop controls. The session summarizes over two dozen disclosed findings and presents mitigations and forward-looking recommendations for probabilistic, autonomous AI systems.

---

## Core Technologies and Architecture

```mermaid
sequenceDiagram
  participant Victim
  participant Agent
  participant Model
  participant Tools
  participant Attacker
  Victim->>Agent: Task - fix this issue
  Agent->>Agent: Load context - issue, code, docs, injection present
  Agent->>Model: Prompt and tool descriptions
  Model->>Model: Follows injected instructions - confused deputy
  Model->>Tools: Tool call - write config or curl and bash
  Tools->>Attacker: RCE, exfil, or C2 - ZombAI
```

### Computer-Use and Coding Agents

Computer-use agents are AI systems that can operate a machine interface (e.g., keyboard, mouse, screen) or invoke tools (browsers, terminals, editors) to accomplish tasks. Coding agents extend this to software development: reading and editing code, running commands, opening PRs, and using external services. These agents are built on probabilistic LLMs (ProbLLMs): the model’s outputs are not deterministic, so security cannot depend on the model “choosing” to refuse malicious instructions. The architecture typically includes: (1) an LLM that receives prompts and tool descriptions, (2) a tool layer (file read/write, shell, browser, API calls), (3) a loop that passes tool results back to the model for the next action. Trust boundaries are often enforced only by model behavior (e.g., “don’t run untrusted code”) rather than hard technical controls, which creates a large attack surface when untrusted content is in context.

### Tool Invocation and the Confused Deputy

Agents call tools on behalf of the user. The model decides which tool to call and with what arguments; the host application executes the call. So the application is a deputy: it has authority (file system, network, shell), and it is “confused” when the model’s decision is driven by attacker-controlled content in the prompt. Prompt injection can instruct the model to call a benign-looking tool with malicious arguments (e.g., write a file, run a command, fetch a URL). The AI Kill Chain in the talk is: prompt injection, confused deputy behavior (model chooses malicious action), and automatic tool invocation (application executes it without sufficient user confirmation). No traditional vulnerability in the application code is required; the chain relies on the model following instructions that were embedded in data the agent consumed (e.g., a GitHub issue, a webpage, a dependency’s README).

### Multi-Vendor and Multi-Product Scope

<details>
<summary>Optional: AgentHopper and persistence at scale</summary>

Long-term prompt injection persistence in coding agents (e.g., instructions in a repo that every developer's agent will read) and the ability to combine multiple issues (e.g., AgentHopper as a conceptual "AI virus") show how single findings can chain into broader compromise. Vendor response varies: some issues are patched quickly; others remain unaddressed or only quietly fixed.

</details>

The research spans many products and ecosystems. Computer-use and coding agents from different vendors share similar patterns: they accept rich context (code, issues, docs, web content), they have tools for file and shell access and sometimes network access, and they often allow network allowlists (e.g., “Common Dependencies”) that include high-level domains where an attacker can host C2 or malware. So the exploit patterns (indirect injection, RCE via tool abuse, ZombAI via allowlisted C2) apply across vendors; the talk distills the most severe findings and recurring systemic flaws rather than a single CVE.

---

## Core Concepts

### ZombAI and C2 Recruitment

ZombAI (a term used in the talk) refers to recruiting an AI agent into a botnet: the agent is tricked via prompt injection into downloading and executing an implant that connects to the attacker’s C2 server. The agent’s environment (e.g., cloud sandbox with code and env vars) becomes a controlled node. Prerequisites are: (1) the agent can be instructed to perform network and execution actions (e.g., curl and execute a binary), and (2) the agent’s network allowlist includes a domain the attacker controls (e.g., a cloud VM hostname). The impact is equivalent to compromising the host or sandbox the agent runs on: full remote control, credential theft, and use of the agent for further attacks.

### Remote Code Execution via Agent Tools

RCE in this setting means the attacker achieves arbitrary code execution in the agent’s environment (or on the host the agent can influence) without exploiting a classic memory or logic bug in the application. Instead, prompt injection causes the model to invoke tools that write or execute code: for example, editing a config file that defines allowed commands or MCP servers, or running shell commands that download and execute a payload. Overly broad tool permissions (e.g., writing to user or system config, or running arbitrary allowlisted commands) and weak or absent user-in-the-loop for dangerous operations make RCE feasible across multiple platforms.

### AI ClickFix and Nation-State TTP Adaptation

ClickFix-style attacks traditionally target humans (e.g., social engineering to click a malicious link or approve a malicious action). The talk describes AI ClickFix: the same idea applied to computer-use systems. The AI is tricked into “clicking” or approving actions that compromise the system (e.g., loading malicious content, granting access, executing code). So existing nation-state TTPs that assume a human in the loop can be adapted to systems where an AI agent is the actor; the agent becomes the target of the social-engineering or deception step, and automation amplifies the impact.

### Month of AI Bugs and Responsible Disclosure

The Month of AI Bugs (August 2025) was a coordinated disclosure effort. The talk references over two dozen security issues reported across major agentic coding assistants. Findings range from quick vendor patches and CVEs to long delays or quiet fixes. The talk highlights that systemic issues (trust in LLM decisions, sandboxing, allowlists) recur across products and require design-level mitigations, not only point fixes.

---

## Exploit Mechanism

```mermaid
flowchart LR
  A[Injection in issue or README or doc] --> B[Victim asks agent to work on it]
  B --> C[Agent fetches content into prompt]
  C --> D[Model follows - tool calls]
  D --> E[Config write or curl and bash]
  E --> F[RCE or ZombAI or C2]
```

| Step | Actor | Action |
|------|--------|--------|
| 1 | Attacker | Places instructions in content the agent will consume (GitHub issues, READMEs, dependency docs, web pages, or file contents), written as natural language or repro steps. |
| 2 | Victim | Asks the agent to work on that content (e.g., investigate this issue, summarize this doc, fix this bug). The agent fetches the content into the prompt and has tools (file edit, shell, network). |
| 3 | Model | Follows the injected instructions (confused deputy); selects tool calls that achieve the attacker's goal (e.g., write malicious MCP or allowlist to config, or curl and execute a binary). |
| 4 | Host application | Executes the tool calls. New config is loaded and attacker code runs, or an implant runs and connects to C2 (ZombAI). |
| 5 | Attacker | RCE or full compromise of the agent environment; persistence and scale possible via repo-level injection or chained issues. |

1. **Injection channel.** Attacker places instructions in content the agent will consume: GitHub issues, READMEs, dependency docs, web pages, or file contents. The instructions are written in natural language or as “repro steps” so the model interprets them as legitimate task directives.

2. **Context and tool access.** Victim (or an automated workflow) asks the agent to work on that content (e.g., “investigate this issue,” “summarize this doc,” “fix this bug”). The agent fetches the content; it is concatenated into the prompt. The agent has tools (file edit, shell, network) and possibly an allowlisted set of domains or commands.

3. **Confused deputy.** The model, following the injected instructions, selects tool calls that achieve the attacker’s goal: for example, writing a malicious MCP server or command allowlist entry to a config file, or running `curl` to download a binary from an allowlisted domain and executing it.

4. **Execution.** The host application executes the tool calls. If the result is config change, the new config is loaded and attacker code runs (e.g., MCP server, or allowlisted command). If the result is download-and-execute, an implant runs and connects to C2 (ZombAI). RCE or full compromise of the agent environment follows.

5. **Persistence and scale.** Long-term prompt injection persistence in coding agents (e.g., instructions in a repo that every developer’s agent will read) and the ability to combine multiple issues (e.g., AgentHopper as a conceptual “AI virus”) show how single findings can chain into broader compromise. Vendor response varies: some issues are patched quickly; others remain unaddressed or only quietly fixed.

<details>
<summary>Optional: AI ClickFix and nation-state TTP adaptation</summary>

ClickFix-style attacks traditionally target humans (social engineering to click or approve). The talk describes AI ClickFix: the same idea applied to computer-use systems. The AI is tricked into clicking or approving actions that compromise the system. Nation-state TTPs that assume a human in the loop can be adapted to systems where an AI agent is the actor; the agent becomes the target and automation amplifies impact.

</details>

Prerequisites: The agent has powerful tools (file write, shell, network) and permissive policies (e.g., writable config that defines capabilities, or network allowlists that include attacker-controlled domains). The agent must consume attacker-controlled or attacker-influenced content as part of normal use.

---

## Security

- **Do not rely on LLM behavior for security.** Probabilistic models can be steered by prompt injection. Treat model output as untrusted; enforce hard boundaries (allowlists, path restrictions, user approval for dangerous operations) instead of assuming the model will refuse malicious instructions.

- **Sandbox tools and config.** Limit what the agent can write (e.g., no writes to capability-defining config) and what it can execute (minimal command allowlists, no execution of content fetched from arbitrary allowlisted URLs). Assume any content the agent reads can instruct it to abuse its tools.

- **Tighten network allowlists.** Preset allowlists that include broad domains (e.g., cloud provider base domains) allow attackers to host C2 and malware on allowlisted hosts. Prefer custom allowlists with exact hostnames; treat preset “Common Dependencies”–style lists as high risk when the agent has sensitive code or credentials.

- **User-in-the-loop for high-risk actions.** Require explicit user confirmation for file writes outside project scope, new network endpoints, or execution of downloaded binaries. Automation should not execute such actions on the sole basis of model output when untrusted data was in context.

- **Threat model computer-use and coding agents like high-privilege users.** They have access to code, env vars, and often network. Compromise via prompt injection plus tool abuse is equivalent to host or tenant compromise. Apply least privilege, monitoring, and supply-chain awareness (e.g., which repos and issues the agent reads).

---

## Summary

The post points to a 39C3 talk that summarizes security research on agentic computer-use and coding agents and the Month of AI Bugs. The talk demonstrates prompt injection-driven exploits across multiple vendors: RCE via abuse of agent tools (e.g., config writes, command execution) and ZombAI-style C2 recruitment when network allowlists include attacker-controlled domains. Recurring themes are over-reliance on the model for trust, inadequate sandboxing of tools and config, and weak user-in-the-loop. The talk also covers adaptation of nation-state TTPs (e.g., ClickFix) to AI systems (AI ClickFix) and presents mitigations and strategic recommendations. The lesson for AI security is that agentic systems must enforce strict technical boundaries and assume that any consumed content can weaponize the agent’s tools; design and allowlists must be attacker-aware across the ecosystem.

---

## References

- [Agentic ProbLLMs: Exploiting AI Computer-Use And Coding Agents (39C3 Video + Slides)](https://embracethered.com/blog/posts/2025/39c3-agentic-probllms-exploiting-computer-use-and-coding-agents/) (source post)
- [39C3 Talk – media.ccc.de](https://media.ccc.de/v/39c3-agentic-probllms-exploiting-ai-computer-use-and-coding-agents) (video and abstract)
- [Slides (PDF)](https://embracethered.com/blog/downloads/Agentic-ProbLLMs-39c3-December-2025.pdf)
- [Month of AI Bugs](https://monthofaibugs.com/)
