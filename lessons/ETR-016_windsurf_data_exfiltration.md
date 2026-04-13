# ETR-016: Hijacking Windsurf: How Prompt Injection Leaks Developer Secrets

**Source:** [Windsurf Data Exfiltration Vulnerabilities](https://embracethered.com/blog/posts/2025/windsurf-data-exfiltration-vulnerabilities/) (Embrace The Red, August 2025)

**In one sentence:** Prompt injection embedded in a source code file hijacks Windsurf Cascade to automatically invoke the read_url_content tool (or render a markdown image) pointing to an attacker server with .env file contents in the query string, exfiltrating developer secrets with no user approval step.

---

## Overview

Windsurf is an AI-powered code editor whose Cascade agent can read files, invoke tools, and browse URLs. The post demonstrates two independent data exfiltration channels that activate when a developer asks Cascade to analyze a file containing a prompt injection payload. No special developer action is required: a normal "explain this file" request is sufficient to trigger exfiltration.

The first channel is the read_url_content tool. Cascade can be instructed via injection to call this tool with an attacker-controlled URL, appending .env file contents as query parameters. The critical finding is that the tool fires automatically without a user approval step. A review of the Windsurf Cascade system prompt confirmed that read_url_content was available to the agent without a human-in-the-loop gate. The second channel is markdown image rendering: the injected payload instructs Cascade to output a markdown image tag pointing to the attacker server with sensitive data in the query string, and the chat renderer fetches that URL to display the image.

The vulnerability was disclosed May 30, 2025, with specific payloads redacted pending a fix. Recommended mitigations are: require human-in-the-loop before read_url_content connects to untrusted servers, maintain a domain allowlist, and block markdown image rendering from untrusted domains. A full video demonstration is available at https://www.youtube.com/watch?v=lTkiCe3uhEY.

---

## Core Technologies and Architecture

### Windsurf Cascade and the read_url_content Tool

```mermaid
flowchart LR
  A[Injection in source file] --> B[Developer asks Cascade to analyze]
  B --> C[Cascade reads file and gets hijacked]
  C --> D[read_url_content auto-invoked]
  D --> E[Attacker server receives .env contents in URL params]
```

Windsurf Cascade is the agentic layer of the Windsurf IDE. It has access to a toolset including read_url_content, which fetches a URL and returns the content to the model. In normal use, a developer might ask Cascade to browse documentation or check an API endpoint. The key finding is that read_url_content was invoked automatically based on model output, with no user approval prompt appearing before the outbound request was made.

The researcher reviewed the Cascade system prompt and found the tool listed as available without a human-in-the-loop gate. This means any prompt injection that can instruct the model to call read_url_content with an attacker URL and sensitive data in the query string constitutes a complete exfiltration chain. The developer's .env file, any config file Cascade has read access to, or any secrets in the current project can be included in the URL parameters by the injected instructions.

### Markdown Image Rendering as a Second Channel

```mermaid
flowchart LR
  B1[Injection instructs: render markdown image] --> C1[Model outputs markdown image tag]
  C1 --> D1[Chat renderer fetches URL]
  D1 --> E1[Attacker server receives data in query string]
```

The second channel operates at the rendering layer rather than the tool layer. Windsurf's chat UI renders markdown, including image syntax. If the model outputs a markdown image whose URL is attacker-controlled with sensitive data encoded in the query string, the renderer fetches that URL to display the image. No named tool invocation is needed: the fetch happens as a rendering side effect, making it harder to spot in any activity or tool-use log.

This pattern is structurally similar to CVE-2025-54132 in Cursor (Mermaid image exfiltration): the renderer treats model output as trusted and fetches URLs contained within it. Having two independent channels means that patching one (e.g., adding a confirmation gate to read_url_content) does not close the vulnerability if the rendering channel remains open.

<details>
<summary>Optional: data accessible to the injected payload</summary>

At the time of the injection, Cascade may have read access to: the current file, other files in the project (via file-reading tools), the .env file if present in the project root, and any additional context loaded for the session. The injection payload can instruct Cascade to read a specific file and append its contents to the exfiltration URL before making the request. This means the attacker can target specific secrets (API keys, database credentials, private tokens) rather than relying on ambient context.

</details>

---

## Core Concepts

### Automatic Tool Invocation Without User Approval

```mermaid
flowchart TD
  subgraph expected["Expected behavior"]
    S1[Cascade proposes outbound request] --> S2[User reviews and approves]
    S2 --> S3[read_url_content fires]
  end
  subgraph actual["Actual behavior under injection"]
    A1[Injection instructs Cascade to call read_url_content] --> A2[Tool fires with no approval dialog]
    A2 --> A3[Developer secrets leave the machine silently]
  end
```

A fundamental design question for agentic AI tools is which actions require explicit human approval before execution. Outbound network requests to domains outside the developer's project context are a natural candidate for gating, because they can carry data to arbitrary servers. When read_url_content fires automatically based on model output, any prompt injection that can construct an attacker URL with sensitive data in the query string becomes a complete, silent exfiltration chain. The fix is to require human-in-the-loop confirmation before outbound requests to untrusted domains, or to enforce a strict domain allowlist at the tool layer.

### Dual Exfiltration Channels and Defense in Depth

The post identifies two independent channels: tool invocation (read_url_content) and rendering (markdown images). Two independent channels mean that fixing one does not close the attack surface. Defense requires addressing both: gating tool-layer network requests AND sandboxing the rendering layer so it does not fetch arbitrary URLs from model output. An attacker only needs one channel to succeed; a defender must close both.

This is a general pattern in AI agent security: features that independently perform outbound HTTP requests (tool calls, image rendering, link previews, Mermaid diagrams) each represent a separate exfiltration surface. Enumerating all such surfaces and applying consistent controls to each is part of designing a defensible agent.

### Indirect Prompt Injection from Source Files

Prompt injection in a source code file (or any content Cascade reads) is the entry point for both channels. The payload does not require a special file type or privileged location: any file a developer might ask Cascade to explain or analyze can carry the injection. The developer's ordinary workflow ("explain this file," "analyze this project") is the trigger. No additional interaction is needed after the developer initiates the analysis, making this effectively a zero-click exfiltration from the developer's perspective.

---

## Exploit Mechanism

```mermaid
sequenceDiagram
  participant Attacker
  participant Developer
  participant Cascade
  participant TargetFile
  participant AttackerServer
  Attacker->>TargetFile: Embed prompt injection payload at file start
  Developer->>Cascade: Analyze or explain this file
  Cascade->>TargetFile: Read file content
  TargetFile->>Cascade: Returns injection payload and file content
  Cascade->>AttackerServer: read_url_content with .env contents as URL params
  AttackerServer->>Attacker: Receives exfiltrated secrets in request log
```

| Step | Actor | Action |
|------|--------|--------|
| 1 | Attacker | Embeds a prompt injection payload at the start of a source code file in the developer's project. |
| 2 | Developer | Asks Windsurf Cascade to analyze or explain the file during normal development work. |
| 3 | Cascade | Reads the file, encounters the injection, and processes it as instructions. |
| 4 | Cascade | Automatically invokes read_url_content with an attacker-controlled URL and .env file contents appended as query parameters; no user approval dialog appears. |
| 5 | Cascade | Alternatively (second channel), outputs a markdown image tag whose URL contains the sensitive data; the chat renderer fetches the URL when rendering the reply. |
| 6 | Attacker | Server receives the outbound request and extracts the leaked secrets from the URL parameters. |

1. Attacker embeds a prompt injection payload at the start of a source code file placed (or already present) in the developer's project. The payload instructs Cascade: when analyzing this file, read the .env file and invoke read_url_content with the attacker URL, appending the .env contents as query parameters.
2. Developer opens the project and asks Windsurf Cascade to analyze or explain the file as part of ordinary development work. Nothing about the request is unusual from the developer's perspective.
3. Cascade reads the file, processes the injection payload as instructions, and invokes read_url_content with the attacker-controlled URL. No user approval dialog appears. API keys, database passwords, and other credentials from the .env file are sent as URL parameters to the attacker's server.
4. In the second channel variant, the injection instructs Cascade to output a markdown image tag whose URL contains the sensitive data encoded in the query string. When the Cascade chat window renders the reply, the image fetch sends the data to the attacker's server as a side effect of rendering.
5. Attacker's server receives the outbound request and logs the exfiltrated secrets from the query string. The developer sees a normal-looking reply and has no indication that secrets were transmitted.

<details>
<summary>Optional: channel comparison</summary>

Channel 1 (read_url_content) requires the tool to be available and auto-invokable; the exfiltration appears as an explicit HTTP GET from a named tool, which could in principle appear in a tool-use log. Channel 2 (markdown image) requires only that the chat renderer displays images from arbitrary URLs; the request comes from the renderer, not a named tool, and is less likely to appear in any agent activity log. Channel 2 is harder to detect and audit after the fact.

</details>

Prerequisites: Developer must ask Cascade to analyze a file containing the injection payload; read_url_content must be invokable without user approval (pre-fix state); the injection has access to files Cascade can read, such as the .env file in the project root.

---

## Security

- Outbound network tools must require user approval before connecting to untrusted domains. read_url_content and any tool that makes an HTTP request should gate on explicit user confirmation for domains not on an allowlist. Automatic invocation of network tools creates a complete exfiltration chain whenever prompt injection is possible, regardless of how the injection is delivered.
- Rendering layers must not fetch arbitrary URLs from model output. Markdown image rendering, Mermaid diagrams with image nodes, HTML img tags, and link previews all represent rendering-layer exfiltration surfaces if the URL comes from model output. Renderers must sandbox outbound fetches or block requests to untrusted domains.
- Domain allowlists reduce blast radius for both channels. Even if automatic invocation or rendering is permitted, restricting outbound requests to a verified allowlist of domains prevents data from reaching attacker-controlled servers. Allowlists are a second line of defense when strict per-action gating is not feasible.
- Indirect injection from source files is a credible entry point. Any content the agent reads as part of normal development (source files, READMEs, config files, dependency manifests) can carry injection payloads. Every file the agent processes must be treated as potentially adversarial when designing the tool invocation and rendering pipeline.

---

## Summary

The post demonstrates two independent data exfiltration channels in Windsurf Cascade triggered by prompt injection embedded in a source code file. The first channel uses read_url_content, which fires automatically and sends .env contents to an attacker server as URL parameters with no user approval. The second uses markdown image rendering: the model outputs an image tag with sensitive data in the URL and the chat renderer fetches it when displaying the reply. Both channels require only that a developer ask Cascade to analyze a file containing the injection. The vulnerability was disclosed May 30, 2025, with payloads redacted pending fix. The core lessons are: gate outbound network tools behind human-in-the-loop approval, maintain domain allowlists, and sandbox rendering layers so they cannot fetch arbitrary URLs from model output.

---

## References

- [Windsurf Data Exfiltration Vulnerabilities](https://embracethered.com/blog/posts/2025/windsurf-data-exfiltration-vulnerabilities/) (source post)
- [Video demonstration](https://www.youtube.com/watch?v=lTkiCe3uhEY) (full exploit chain: injection to .env exfiltration)
- [Cursor Mermaid exfiltration CVE-2025-54132](https://embracethered.com/blog/posts/2025/cursor-data-exfiltration-with-mermaid/) (related: rendering-layer exfiltration via Mermaid image nodes)
