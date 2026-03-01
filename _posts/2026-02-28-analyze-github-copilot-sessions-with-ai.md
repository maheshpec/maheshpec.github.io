---
layout: post
title: "Let AI Audit Your Copilot Sessions: How to Actually Get Better at GitHub Copilot"
date: 2026-02-28 10:00:00 +0000
categories: productivity
tags: github-copilot ai developer-tools vscode
---
You use GitHub Copilot every day. You accept suggestions, you reject suggestions, you occasionally swear at suggestions. But do you ever stop and ask: am I actually getting the most out of this tool, or am I just using it on autopilot?

Most developers treat Copilot as a black box. A suggestion appears, you tab or you escape, and you move on. The problem is that the quality of Copilot's suggestions is almost entirely determined by *your* habits — what context you give it, how you structure your workspace, how you write the code that surrounds the cursor. Without auditing those habits, you have no way to improve them.

There is a straightforward fix: ask Copilot Chat to analyze your session logs. VS Code has been writing them to disk every time you work. A pattern that shows up across a week of sessions is a real habit — one bad session is just noise.

## How Copilot Works in Local Mode

Before you can audit your sessions, you need to understand what Copilot is actually doing — as of early 2026. The details below apply to the VS Code extension in standard local use.

**The context payload**

Copilot does not just read your current file. For every completion request, the extension assembles a payload using fill-in-the-middle (FIM) format: a prefix block (everything above your cursor), a suffix block (everything below), and a `<MID>` marker where the completion should be inserted. This gives the model bidirectional context — it knows not just where you have been but where you are going.

The payload also includes snippets from other open editor buffers. These "neighboring documents" are scored by Jaccard similarity against the current cursor context, tokenized, and the top-ranked ones are prepended to the prompt as `// Path: ...` labeled blocks. The entire payload is bounded by a token budget — typically 2048 to 8192 tokens depending on your Copilot tier. When the budget runs out, lower-scored neighbor snippets are dropped first.

The direct implication: suggestion quality is a direct function of what files you have open and how much typed context surrounds your cursor. If you are editing a blank function in an isolated file, Copilot is working with almost nothing.

**The LSP layer**

The extension registers as a `vscode.languages.registerInlineCompletionItemProvider`. This gives it access to the VS Code language server: resolved types, hover information, workspace symbol resolution, and diagnostics (errors and warnings in your current files).

A healthy language server means richer context. If your TypeScript project has type errors, or your Python environment is misconfigured so the LSP cannot resolve imports, that diagnostic state is visible to Copilot and degrades the quality of what it can infer about your code.

**How suggestions fire and what gets logged**

After a short debounce idle (default around 100ms), the payload is assembled and sent to `api.githubcopilot.com` over HTTPS. The response streams back as Server-Sent Events. Accepted and rejected suggestions are telemetry events — but the same data is locally observable through VS Code's Output panel, which is exactly what we will use.

## The Optimization Workflow

1. **Enable debug logging.** Add this to your VS Code `settings.json`:

```json
{
  "github.copilot.advanced": {
    "debug.overrideLogLevels": {
      "*": "DEBUG"
    }
  }
}
```

Open the Output panel (`View > Output`) and select "GitHub Copilot Log" from the dropdown. You will start seeing verbose entries for each completion request: context size, latency, and request details.

2. **Work normally for a few days.** Just code. VS Code is already writing a `GitHub Copilot.log` file for every session in the background. You do not need to do anything special. If you want to jot down notes on sessions where suggestions were particularly good or bad, that adds useful signal — but it is optional.

3. **Find your session logs.** Open `Help > Open Logs Folder`. VS Code creates a timestamped subdirectory for each window session:

```
~/.config/Code/logs/
  20260225T090000/exthost1/GitHub Copilot.log
  20260226T093000/exthost1/GitHub Copilot.log
  20260227T091500/exthost1/GitHub Copilot.log
```

Pick the logs from your last several working days. The more sessions you include, the more confidently patterns separate from one-off noise.

4. **Open Copilot Chat and attach the logs.** Open the Copilot Chat panel (`Ctrl+Shift+I` on Windows/Linux, `Cmd+Shift+I` on macOS). Type `#file:` once for each log file to attach them, then append this prompt:

```text
Analyze the attached Copilot session logs (multiple sessions) and identify patterns
in my usage that I can improve.

Copilot works by sending the text above and below my cursor (fill-in-the-middle) plus
snippets from other open files (ranked by token similarity) to a completion API.
Suggestion quality depends on: context density around the cursor, which related
files are open in the editor, and whether the language server is reporting errors.

Optional notes I kept across these sessions:
- [Sessions where suggestions were accurate and useful]
- [Sessions where I kept rejecting and rewriting]
- [What kinds of tasks I was working on]

Please tell me:
1. What patterns appear consistently across sessions where context quality was weakest?
2. What workspace habits would have improved suggestions across these sessions?
3. Are there recurring situations where suggestions were likely rejected?
4. Give me 3 specific, actionable changes to my workflow based on what you see across all sessions.
```

5. **Act on the output.** Findings that appear across multiple sessions are the ones worth acting on first. Common recurring patterns: editing isolated functions with no type imports visible, switching files frequently before context stabilizes, writing imperative comments ("do X") rather than descriptive context ("this function handles Y so that Z"). Each maps directly to a habit you can change.

## Managing Context Length

Debug logs grow fast. A single productive coding day with DEBUG-level logging enabled will produce a `GitHub Copilot.log` in the 2–10 MB range — roughly 500K to 2.5M tokens. Stack several sessions together and you will blow past the context window of any model currently available in Copilot Chat before the analysis even starts.

**Switch to a premium model before attaching files.** In the Copilot Chat panel, click the model selector (bottom-left of the input box) and choose the highest-context option you have access to. As of early 2026, Claude Sonnet offers the largest context window (200K tokens), followed by GPT-4o at 128K. The default models used for quick chat are smaller and will silently truncate large inputs — dropping the earliest sessions and giving you a skewed picture of your patterns.

**Trim the logs before attaching.** You do not need every line — just the events that describe what happened at each completion. This command extracts the signal from all recent session logs into one file:

```bash
grep -iE "accept|reject|latency|contextSize|token|completion" \
  ~/.config/Code/logs/*/exthost1/"GitHub Copilot.log" \
  > ~/copilot-audit.txt
```

The result is typically 5–20× smaller than the raw logs and contains everything needed for pattern analysis. Attach `~/copilot-audit.txt` with `#file:` instead of the individual session files.

If the trimmed file is still over a few MB, split it by date range and run two separate analyses — one on older sessions, one on recent. The model produces more reliable output on 10K focused lines than on 200K lines it had to aggressively compress to fit.

## Workspace Hygiene — The Quick Wins

Based on the internals above, these changes help immediately without any tooling setup:

- **Keep related files open.** Copilot picks the most relevant open buffers using similarity scoring. If you have your interfaces, types, and the file you are editing open at the same time, the neighbor snippets are far more useful than if you have 20 unrelated tabs cluttering the workspace.
- **Write signatures before bodies.** FIM uses both prefix and suffix. A function with a name, parameter types, a return type, and a one-line comment describing its goal is a far richer context signal than an empty stub. Write the skeleton first.
- **Resolve LSP errors before a session.** TypeScript errors, unresolved imports, or a broken Python environment thin out the language server context. A clean error panel before you start means Copilot has cleaner type information to work with.
- **Use meaningful names in surrounding scope.** The similarity-based retrieval that selects which neighbor snippets to include responds to token overlap. If your surrounding code uses descriptive, domain-specific names, the right context files rank higher.

## Going Deeper — Intercepting the Raw Payloads

If you want to see exactly what Copilot sends to the API rather than inferring it from logs, a local HTTPS proxy gives you the raw request and response JSON. With [mitmproxy](https://mitmproxy.org/) running locally:

```bash
# Start mitmproxy on port 8080
mitmproxy --listen-port 8080 --ssl-insecure

# Configure VS Code to route through it (in settings.json)
# "http.proxy": "http://localhost:8080"
```

The request body will show the `prompt` field (the assembled FIM prefix), `suffix`, and `extra.neighbors` — the actual neighbor snippets that were included. Seeing which files made it into the payload and which were excluded is the most direct way to understand what context Copilot had for any given suggestion.

This is optional and mainly useful if the debug logs are not giving you enough detail. The traffic stays on your machine — you are not routing it anywhere new by intercepting it locally.

---

Copilot suggestions are only as good as the context you give them, and most developers never measure whether they are giving good context. The audit loop closes entirely inside VS Code: the same tool generating your suggestions can tell you why they were not good enough.
