# ETR-007: Cross-Agent Privilege Escalation: When Agents Free Each Other

**Source:** [Cross-Agent Privilege Escalation: When Agents Free Each Other](https://embracethered.com/blog/posts/2025/cross-agent-privilege-escalation-agents-that-free-each-other/) (Embrace The Red, September 2025)

**In one sentence:** One compromised coding agent (via indirect prompt injection) writes another agent's config or instruction files so that when the developer uses the second agent, it loads poisoned config and runs the attacker's code, with optional escalation loop if the second agent can reconfigure the first.

---

## Overview

The post describes a cross-agent privilege escalation pattern: one compromised coding agent can overwrite another agent's configuration and instruction files, effectively "freeing" the second agent from its sandbox or constraints. "Freeing" here means one agent gives another additional capabilities (e.g., arbitrary code execution, allowlisted shell commands, or malicious MCP servers) by writing to that agent's config files. The scenario is common because many developers run multiple coding agents (e.g., GitHub Copilot and Claude Code) on the same codebase. Those agents often write to shared or adjacent config locations (e.g., `.vscode/`, `.mcp.json`, `AGENTS.md`, `CLAUDE.md`) without requiring user approval. An attacker who achieves indirect prompt injection against one agent can direct it to modify another agent's config; when the developer later uses the second agent, it loads the poisoned config and runs arbitrary code. The chain can form a loop if the second agent can in turn write to the first agent's config. The post ties this to prior work on agents overwriting their own settings (e.g., GitHub Copilot, AWS Kiro) and argues that cross-agent config writes are a design flaw that accumulates security debt and is not well mitigated today.

---

## Core Technologies and Architecture

```mermaid
sequenceDiagram
  participant Victim
  participant AgentA as Agent A
  participant AgentB as Agent B
  participant Attacker
  Victim->>AgentA: Use agent on repo (injection in file/comment)
  AgentA->>AgentA: Follows injected instructions
  AgentA->>AgentB: Writes to B's config (.mcp.json, CLAUDE.md)
  Victim->>AgentB: Invoke second agent
  AgentB->>AgentB: Loads poisoned config
  AgentB->>Attacker: Runs attacker code or connects to malicious MCP
```

### Multi-Agent Setups on a Single Codebase

Developers often use more than one coding agent on the same project (e.g., GitHub Copilot and Claude Code) for comparison, review, or complementary tasks. These agents share the same filesystem and often the same workspace metadata. There is no process or permission boundary between "Copilot's view" and "Claude's view" of the repo; both can read and write the same files and folders. From a security perspective, any file one agent can create or modify can be read and acted on by the other agent, and by the user's tools (e.g., the IDE) when the other agent is invoked.

### Where Agents Store Configuration and Instructions

Different agents use different file paths for settings and instructions, but many of these live under the project or in the user's home directory and are plain text or JSON:

<details>
<summary>Config paths by agent</summary>

- **GitHub Copilot:** `.vscode/settings.json`, `.vscode/mcp.json`, `.vscode/copilot-instructions.md` (and related `.vscode/` files). Copilot also respects project-level instruction files.
- **Claude Code:** `.mcp.json` (MCP server config, including local path), `.claude/settings.local.json`, and instruction files such as `CLAUDE.md`.
- **Other agents:** The post mentions `.gemini/settings.json` and similar patterns. Generic instruction files like `AGENTS.md` are often read by multiple agents.

</details>

These files control what commands an agent can run, which MCP servers it talks to, and what high-level instructions it follows. Writing to them is equivalent to changing the agent's capabilities and behavior. Many agents create or update these files as part of normal operation (e.g., adding a task or MCP server), and historically many did so without explicit user confirmation. Even when vendors add "do not overwrite config without permission," agents can still create or modify other files and folders, so cross-agent writes to another agent's config remain possible.

### How Agents Achieve Arbitrary Code Execution

A single agent can escape its sandbox or achieve arbitrary code execution through several mechanisms that depend on config and project files: allowlisting shell commands in settings, adding or modifying MCP server configuration (so the agent connects to a malicious server that runs code), adding tasks in `.vscode/tasks.json`, or changing instruction files so the agent follows attacker-supplied steps. When the trigger is indirect prompt injection (e.g., from a comment, doc, or web content the agent reads), the result is remote code execution from the perspective of the victim. The important point for cross-agent escalation is that the same primitives (writing to config and instruction files) are available not only for an agent to modify its own environment but for one agent to modify another's.

---

## Core Concepts

### Cross-Agent Configuration Write

A cross-agent configuration write is when agent A creates or modifies files that agent B uses for its configuration or instructions. Because both agents operate on the same codebase and filesystem, agent A can write to paths like `.mcp.json`, `CLAUDE.md`, or `.vscode/settings.json` that agent B will load the next time the user runs agent B. No separate API or "agent-to-agent" channel is required; the channel is the shared filesystem. If agent A is compromised (e.g., via indirect prompt injection), the attacker can use agent A to poison agent B's config and thus escalate from one agent to the other.

### "Freeing" Another Agent

"Freeing" in this post means one agent giving another agent capabilities it did not have before, typically by writing to that agent's config or instruction files. For example, agent A adds a malicious MCP server to agent B's `.mcp.json` or inserts instructions into `CLAUDE.md` that tell agent B to run specific commands. When the developer next uses agent B, agent B loads the modified config and executes the attacker's code or follows the attacker's instructions. So the first agent "frees" the second from its default sandbox or policy by reconfiguring it.

### Escalation Loop

If agent B can also write to agent A's config (e.g., Copilot's `.vscode/` or instruction files), then after agent A has "freed" agent B, agent B can "return the favor" and reconfigure agent A. That creates a loop: one indirect prompt injection leads to agent A being compromised, agent A reconfigures agent B, and agent B reconfigures agent A (and possibly others). Privilege and control escalate across agents without further injection. The post emphasizes that agents collaborating (or being coerced) to mutually reinforce malicious behavior is a plausible and already observable pattern.

---

## Exploit Mechanism

```mermaid
flowchart LR
  A[Injection in file/comment] --> B[Agent A reads, follows]
  B --> C[A writes B's config]
  C --> D[Victim invokes Agent B]
  D --> E[B loads poisoned config]
  E --> F[RCE / malicious MCP]
  F -.->|optional| C
```

| Step | Actor | Action |
|------|--------|--------|
| 1 | Attacker | Achieves control of one agent via indirect prompt injection (e.g., instructions in a comment, README, or file the agent reads). |
| 2 | Compromised agent (A) | Follows injected instructions and writes to another agent's config (e.g., .mcp.json, CLAUDE.md, .vscode/settings.json) to add malicious MCP server or allowlisted commands. |
| 3 | Victim | Invokes the second agent (B). Agent B loads the modified config and instruction files. |
| 4 | Agent B | Connects to malicious MCP, runs allowlisted commands, or follows injected instructions; arbitrary code execution occurs. |
| 5 | Optional | Agent B reconfigures agent A's config; compromise propagates and can form a persistent escalation loop. |

1. **Attacker gains control of one agent via indirect prompt injection.** The victim uses a coding agent (e.g., GitHub Copilot) on a codebase or document that contains hidden instructions (e.g., in a comment, README, or file the agent reads). The agent follows those instructions instead of (or in addition to) the user's intent.

2. **Injected instructions direct the compromised agent to write to another agent's config.** For example, the attacker instructs Copilot to create or modify Claude's MCP config (e.g., `.mcp.json` or the path Claude uses) to add a malicious MCP server, or to update `CLAUDE.md` (or similar) with instructions that cause Claude to run arbitrary commands or allowlist dangerous operations.

3. **Developer uses the second agent.** When the developer later invokes Claude Code (or another agent), it loads the modified config and instruction files. It may connect to the malicious MCP server, run allowlisted commands, or follow the injected instructions. Arbitrary code execution occurs in the context of the second agent.

4. **Optional: second agent reconfigures the first.** If the second agent has write access to the first agent's config (e.g., Copilot's `.vscode/` or instruction files), the attacker can instruct the second agent to modify those files. When the developer uses the first agent again, it too runs with poisoned config. The compromise propagates and can form a persistent escalation loop.

<details>
<summary>Optional: prior work on self-overwrite</summary>

The post ties cross-agent escalation to prior work on agents overwriting their own settings (e.g., GitHub Copilot, AWS Kiro). Cross-agent config writes are the same primitive (writing to config that controls capabilities) applied from one agent to another's config; vendors have added mitigations for self-overwrite but cross-agent writes remain a generic, under-mitigated design issue.

</details>

Prerequisites: multiple agents used on the same codebase or shared filesystem; at least one agent that can be influenced by indirect prompt injection and that can write to files the other agent(s) read for config or instructions; no strong isolation or permission checks preventing one agent from writing another agent's dotfiles or project config.

---

## Security

- **Least privilege for agents.** Agents should not have blanket permission to create or overwrite arbitrary files. Restricting writes (e.g., no writes to dotfiles or config directories without explicit user approval) reduces the chance that one agent can reconfigure another. Proposing changes for user approval instead of applying them automatically is a practical mitigation.

- **Indirect prompt injection enables the initial compromise.** The first agent is typically compromised via untrusted data (e.g., a file, comment, or webpage the agent reads). Defenses include limiting which content the agent sees, treating agent output as untrusted, and not trusting one agent's file writes as input to another agent's behavior without review.

- **Multiple agents on shared data increase blast radius.** If several agents operate on the same repo or config space, a single compromise can spread. Users and vendors should be aware that agents can interfere with each other. Isolating each agent's configuration (e.g., separate directories or permission boundaries) makes cross-agent config writes harder.

- **Secure defaults for config writes.** Vendors should not allow agents to overwrite or create config and instruction files without user consent. At minimum, dotfiles and config directories should be protected by default. The post notes that some vendors have added mitigations for self-overwrite but that cross-agent writes remain a generic, under-mitigated design issue.

---

## Summary

The post introduces cross-agent privilege escalation: one compromised coding agent can "free" another by writing to that agent's configuration and instruction files (e.g., `.mcp.json`, `CLAUDE.md`, `.vscode/settings.json`). Because many developers run multiple agents on the same codebase and agents share the filesystem, one agent can add malicious MCP servers, allowlisted commands, or injected instructions for another. An attacker who achieves indirect prompt injection against the first agent can use it to poison the second agent's config; when the developer uses the second agent, it runs arbitrary code. The chain can form a loop if the second agent can write back to the first. The pattern is a design flaw in multi-agent setups and is not limited to a single vendor or IDE. Mitigations include least-privilege execution, avoiding automatic config overwrites, isolating agent config, and treating indirect prompt injection as the primary enabler for the initial compromise.

---

## References

- [Cross-Agent Privilege Escalation: When Agents Free Each Other](https://embracethered.com/blog/posts/2025/cross-agent-privilege-escalation-agents-that-free-each-other/) (source post)
- [GitHub Copilot: Remote Code Execution via Prompt Injection (CVE-2025-53773)](https://embracethered.com/blog/posts/2025/github-copilot-remote-code-execution-via-prompt-injection/)
- [AWS Kiro: Arbitrary Command Execution with Indirect Prompt Injection](https://embracethered.com/blog/posts/2025/aws-kiro-aribtrary-command-execution-with-indirect-prompt-injection/)
- [Month of AI Bugs](https://monthofaibugs.com/)
