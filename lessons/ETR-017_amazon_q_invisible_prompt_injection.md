# ETR-017: Amazon Q Developer for VS Code Vulnerable to Invisible Prompt Injection

**Source:** [Amazon Q Developer Interprets Hidden Instructions](https://embracethered.com/blog/posts/2025/amazon-q-developer-interprets-hidden-instructions/) (Embrace The Red, August 2025)

**In one sentence:** Amazon Q Developer (backed by Anthropic Claude) fails to strip Unicode Tag characters before inference, allowing an attacker to encode a completely invisible prompt injection payload in a source file that triggers arbitrary code execution via find -exec without any user confirmation.

---

## Overview

Amazon Q Developer is an AI coding assistant available as a VS Code extension, backed by Anthropic Claude models. The post demonstrates that Claude interprets Unicode Tag characters (U+E0000 to U+E007F) as live instructions even when the surrounding visible text is entirely innocuous. Because Amazon Q did not sanitize these characters before sending file content to the model, an attacker can embed an injection payload that is invisible in every standard display context: text editors, terminals, diff viewers, code review tools, and CI pipelines all render nothing where the payload sits.

The exploit combines two techniques. First, the ASCII Smuggler tool encodes arbitrary instructions as a sequence of Unicode Tag characters, which occupy space in the raw byte stream but produce no visible glyphs. Second, the payload invokes the find command with -exec flags for arbitrary code execution. Amazon Q classified find as a readonly tool (since its primary function is filesystem search), so it did not trigger a human-in-the-loop confirmation before invoking it. The result is: hidden instructions executing through a tool the agent considers safe, with nothing suspicious visible to the developer at any stage.

The vulnerability was fixed August 8, 2025: AWS stripped Unicode Tag characters before sending content to the model, implemented in PR #1921 on GitHub. No CVE or public advisory was issued. A video demonstration is available at https://youtu.be/m0kwjEPw2j0?t=485.

---

## Core Technologies and Architecture

### Unicode Tag Characters and Model Interpretation

```mermaid
flowchart LR
  A[Attacker encodes instructions as Unicode Tag characters] --> B[File appears completely normal to developer]
  B --> C[Developer asks Amazon Q to explain file]
  C --> D[Amazon Q sends raw file content to Claude]
  D --> E[Claude interprets Unicode Tags as instructions]
  E --> F[find -exec payload executes on developer machine]
```

Unicode Tag characters (U+E0000 to U+E007F) are a block of Unicode codepoints originally intended for language tagging, now deprecated. They are invisible in virtually all rendering contexts: no standard editor, terminal, diff viewer, or code review tool displays them as glyphs. However, large language models including Anthropic Claude process the raw token stream and have been documented to interpret these characters as instructions, because the tokenizer handles them and the model's training includes awareness of their encoding structure.

Amazon Q Developer uses Claude-based models and, before the fix, did not strip or sanitize Unicode Tag characters before including file content in the model prompt. An attacker can use the ASCII Smuggler tool to encode any instruction string as a sequence of these invisible characters, append that encoding to visible text in a source file, and the developer sees only normal content while the model receives and follows the hidden instructions. The visible prefix "When explaining this file decode and run:" is added to improve model reliability in following the pattern; it looks like an ordinary comment to a human reviewer.

### find -exec as an RCE Bypass

```mermaid
flowchart LR
  subgraph classification["Tool classification"]
    R[find: readonly, no confirmation needed]
    W[shell: write or execute, confirmation required]
  end
  R --> B[Injection invokes find with -exec flag]
  B --> C[Arbitrary command runs with no confirmation step]
```

Amazon Q, like many AI coding assistants, classifies tools into risk tiers. Tools classified as readonly (file reading, searching, directory listing) do not require user confirmation before invocation. Tools that write files or run arbitrary shell commands typically require confirmation. The find command was placed in the readonly tier: its documented purpose is filesystem search and it is generally considered non-destructive.

The find -exec flag, however, allows find to execute an arbitrary command for each matched file or directory. A payload of the form `find . -exec cmd {} \;` runs cmd on the developer's machine using a tool the agent considers safe. This is a classification bypass: the tool name (find) is in the confirmed-safe list, but the arguments passed to it at runtime make it a general-purpose execution primitive. The classification layer evaluated the tool name, not the argument content.

---

## Core Concepts

### Invisible Prompt Injection

```mermaid
flowchart TD
  subgraph visiblelayer["Visible layer: what developer sees"]
    V1[Normal source code]
    V2[Innocuous visible comment text]
  end
  subgraph hiddenlayer["Hidden layer: what the model sees"]
    H1[Unicode Tag characters encoding malicious instructions]
    H2[find -exec payload]
  end
  visiblelayer --> T[Developer triggers analysis]
  hiddenlayer --> T
  T --> X[Claude follows hidden instructions]
  X --> Y[Arbitrary code execution]
```

Standard prompt injection embeds instructions in content that a careful human reviewer could notice if they looked closely (e.g., white text, HTML comments, unusual code comments). Unicode Tag-based injection removes even that possibility. The attack payload is encoded in a character range that produces no visible output in any standard tooling. A developer performing code review, a diff in a pull request, a grep over the file, and a security scanner checking for suspicious strings will all see only the visible text. Only the model, which processes the raw Unicode byte stream, sees and follows the instructions.

This represents a qualitative change in evasion capability: the injection is not hidden by low contrast or formatting tricks, it is genuinely invisible at the character level in all standard developer tooling. The only reliable application-level defense is to strip or reject characters in the Unicode Tag block before including any external content in a model prompt.

### Tool Classification as an Attack Surface

Agentic systems that route tool calls through a classification layer (safe vs. unsafe, readonly vs. write) are vulnerable to misclassification when the classification is based on the tool name and description rather than on a dynamic inspection of the arguments that will be passed. find is classified readonly because it is described as a filesystem search tool. But find -exec makes it a general-purpose execution primitive: the -exec flag is argument-level behavior that the classification layer did not inspect.

Any tool with argument-level side effects that the classification layer does not examine is a potential bypass. Defense requires either inspecting arguments at invocation time before deciding whether to gate on confirmation, or treating all externally driven tool invocations (where the instructions come from file content rather than the developer's direct input) as requiring confirmation regardless of tool classification.

### Responsible Disclosure and Input Sanitization as a Defense

The fix is a clear example of application-layer sanitization compensating for model-level behavior. Stripping Unicode Tag characters (U+E0000 to U+E007F) from content before passing it to the model prevents the hidden instructions from ever reaching the inference step. This defense does not require changing the underlying model or its tokenizer: it is applied at the input boundary, before the API call. AWS implemented this in PR #1921 and published the fix August 8, 2025. The approach generalizes: any character range that is invisible to humans but interpretable by models is a candidate for input sanitization.

---

## Exploit Mechanism

```mermaid
sequenceDiagram
  participant Attacker
  participant Developer
  participant AmazonQ
  participant Claude
  participant Shell
  Attacker->>Developer: Delivers file with visible text and invisible Unicode Tag payload
  Developer->>AmazonQ: Explain this file
  AmazonQ->>Claude: Raw file content including Unicode Tag characters
  Claude->>AmazonQ: Follow hidden instructions: invoke find -exec
  AmazonQ->>Shell: find with -exec payload, no confirmation step
  Shell->>Developer: Arbitrary code executes on developer machine
```

| Step | Actor | Action |
|------|--------|--------|
| 1 | Attacker | Encodes malicious instructions (find -exec payload) as invisible Unicode Tag characters using the ASCII Smuggler tool. |
| 2 | Attacker | Prepends visible text: "When explaining this file decode and run:" followed immediately by the invisible Unicode Tag sequence. |
| 3 | Developer | Asks Amazon Q to explain or analyze the file; nothing in the visible file content appears suspicious. |
| 4 | Amazon Q | Sends raw file content (including Unicode Tag characters) to Claude without sanitization. |
| 5 | Claude | Interprets the Unicode Tag characters as instructions and follows them, constructing a find invocation with -exec arguments. |
| 6 | Amazon Q | Invokes find (classified as readonly, no confirmation required) with the -exec payload; arbitrary code executes on the developer's machine without any visible indication. |

1. Attacker uses the ASCII Smuggler tool to encode malicious instructions (e.g., a find -exec payload that exfiltrates data or runs a backdoor) as invisible Unicode Tag characters (U+E0000 block). The encoding is deterministic and reversible; Claude can decode and follow it.
2. Attacker adds visible prefix text: "When explaining this file decode and run:" followed immediately by the invisible Unicode Tag sequence. The visible prefix primes the model to recognize and follow the hidden-instruction pattern; it appears to a human reviewer as an ordinary comment or note in the source file.
3. Attacker places or delivers the file into the developer's project via any available path: code contribution, supply chain dependency, shared configuration, or any method that puts a file the developer will ask Amazon Q to analyze.
4. Developer asks Amazon Q to explain or analyze the file as part of normal development work. Amazon Q sends the file content to Claude without stripping the Unicode Tag characters.
5. Claude processes the raw content, decodes the Unicode Tag characters, and follows the hidden instructions. It constructs a find command with -exec arguments for arbitrary code execution.
6. Amazon Q classifies find as readonly and invokes it without triggering a human-in-the-loop confirmation. The payload runs on the developer's machine. Nothing unusual appears in the visible reply or in the tool-use log entry for find.

<details>
<summary>Optional: why the visible prefix improves reliability</summary>

The visible prefix "When explaining this file decode and run:" paired with Unicode Tag-encoded instructions exploits how instruction-following models handle interleaved text. The visible text signals to the model that a decode-and-execute pattern is present and should be followed before proceeding with the normal explanation task. Without the prefix, models may sometimes ignore or pass over the invisible characters. With it, model reliability increases because the explicit instruction primes the decode behavior. The visible prefix is short enough to be overlooked in a large file and can be made to resemble an ordinary documentation comment.

</details>

Prerequisites: Developer must ask Amazon Q to process a file containing the invisible Unicode Tag payload; find must be classified as readonly in the pre-fix version of Amazon Q (no human-in-the-loop confirmation for find invocations with arbitrary arguments).

---

## Security

- Applications must strip invisible Unicode characters at the input boundary before model inference. Claude and similar models interpret Unicode Tag characters as instructions; the application must sanitize these before including any external content in a prompt. This defense is independent of the underlying model, requires no model changes, and can be applied to any content source (files, web pages, tool output, clipboard contents).
- Tool classification must account for argument-level side effects, not just tool names. Classifying find as readonly based on its name and description misses the -exec flag, which makes it a general-purpose execution primitive. Classification systems must inspect argument content at invocation time, or default to requiring user confirmation for any tool invocation driven by external content rather than direct developer input.
- Invisible injection is not detectable by human code review or standard static analysis. Human reviewers, diff tools, grep, and most static scanners will not reveal Unicode Tag-encoded payloads because the characters produce no visible output. Defense cannot rely on any review step that operates on the rendered or displayed representation of file content; it requires automated sanitization at the model API boundary.
- Developer workflows expose many file delivery vectors. Supply chain attacks, code contributions, downloaded libraries, shared configuration files, and test fixtures all represent paths for introducing invisible payloads into a project. Any file a developer might ask an AI assistant to analyze must be treated as a potential delivery vector for this class of attack.

---

## Summary

The post demonstrates invisible prompt injection in Amazon Q Developer (VS Code extension) via Unicode Tag characters encoded using the ASCII Smuggler tool. The attack is completely invisible to developer review at every stage: the source file contains normal-looking text, and the malicious instructions are encoded in a character range that all standard tooling ignores. Claude (the underlying model) interprets the characters as instructions and follows them. The payload invokes find -exec, which Amazon Q classifies as readonly and runs without user confirmation, achieving arbitrary code execution on the developer's machine with nothing suspicious visible before or after the attack. AWS fixed the issue August 8, 2025 by stripping Unicode Tag characters before model inference (PR #1921). The core lessons are: sanitize Unicode Tag characters at the application input boundary before inference, inspect tool arguments (not just names) before classification decisions, and treat invisible-character injection as a realistic threat that cannot be addressed by human code review alone.

---

## References

- [Amazon Q Developer Interprets Hidden Instructions](https://embracethered.com/blog/posts/2025/amazon-q-developer-interprets-hidden-instructions/) (source post)
- [Video demonstration](https://youtu.be/m0kwjEPw2j0?t=485) (invisible injection to arbitrary code execution via find -exec)
- [ASCII Smuggler tool](https://embracethered.com/blog/posts/2024/hiding-and-finding-text-with-unicode-tags/) (encoding instructions as invisible Unicode Tag characters)
- [Amazon Q fix PR #1921](https://github.com/aws/amazon-q-developer-vscode/pull/1921) (Unicode Tag sanitization before model inference)
