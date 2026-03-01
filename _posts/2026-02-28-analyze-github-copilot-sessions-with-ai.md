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

## How Copilot Decides What to Suggest

Copilot does not just read the file you are editing. For each suggestion, it pulls context from two places:

**What's around your cursor.** Everything above and below the insertion point goes into the prompt. A function with a meaningful name, typed parameters, and a short comment describing its goal gives Copilot far more to work with than an empty stub.

**Your other open files.** Copilot scans your open buffers and pulls in snippets from the ones most related to what you are currently writing. If your types, interfaces, or related modules are open in other tabs, they are more likely to show up in the suggestion context. If you have 20 unrelated files open, the useful ones get crowded out.

**Your language server's health.** Type errors, unresolved imports, and misconfigured environments are visible to Copilot and degrade what it can infer. A clean error panel means cleaner suggestions.

The practical upshot: suggestion quality is largely a function of what you have open and how much context surrounds the cursor.

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

Copilot works by sending the text around my cursor plus snippets from other open
files to a completion API. Suggestion quality depends on: how much useful context
surrounds the cursor, which related files are open in the editor, and whether the
language server is reporting errors.

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
- **Write signatures before bodies.** Copilot reads both above and below the cursor. A function with a name, parameter types, a return type, and a one-line comment describing its goal is a far richer signal than an empty stub. Write the skeleton first.
- **Resolve errors before a session.** TypeScript errors, unresolved imports, or a broken Python environment degrade what Copilot can infer about your code. A clean error panel before you start means cleaner suggestions throughout.
- **Use meaningful names in surrounding scope.** Copilot picks related open files based on vocabulary overlap with what you are writing. Descriptive, domain-specific names make it more likely the right context files get pulled in.

---

Copilot suggestions are only as good as the context you give them, and most developers never measure whether they are giving good context. The audit loop closes entirely inside VS Code: the same tool generating your suggestions can tell you why they were not good enough.
