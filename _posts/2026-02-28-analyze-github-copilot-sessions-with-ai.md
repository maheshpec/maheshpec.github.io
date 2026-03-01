---
layout: post
title: "Let AI Audit Your Copilot Sessions: How to Actually Get Better at GitHub Copilot"
date: 2026-02-28 10:00:00 +0000
categories: productivity
tags: github-copilot ai developer-tools vscode
---
You use GitHub Copilot every day. You accept suggestions, you reject suggestions, you occasionally swear at suggestions. You kick off agentic edits that rewrite half a file and then spend ten minutes undoing them. But do you ever stop and ask: am I actually getting the most out of this tool, or am I just using it on autopilot?

Most developers treat Copilot as a black box. A suggestion appears, you tab or you escape, and you move on. You type a task into Copilot Edits and accept whatever comes back. The problem is that the quality of everything Copilot produces — inline completions, multi-line generations, agentic edits across multiple files — is almost entirely determined by *your* habits: what context you give it, how you scope your tasks, how you structure your workspace. Without auditing those habits, you have no way to improve them.

There is a straightforward fix: ask Copilot Chat to analyze your session logs. VS Code has been writing them to disk every time you work. A pattern that shows up across a week of sessions is a real habit — one bad session is just noise.

## How Copilot Builds Context

Copilot works in two distinct modes and each one determines quality differently.

**Inline completions** trigger as you type. Copilot reads everything above and below the cursor in the current file, then pulls in snippets from your other open buffers — favouring the ones most related to what you are writing. A function with a meaningful name, typed parameters, and a short comment gives it far more to work with than an empty stub. If you have 20 unrelated files open, the useful ones get crowded out by the noise.

**Agentic and multi-line generations** (Copilot Edits, agent mode, long chat responses) work differently. Here the model has a much larger context window and is driven by what you explicitly provide: your task description, the files you add to the working set, and the conversation history. It can also read additional files on its own to complete the task. The limiting factor is not cursor position — it is task clarity. A vague "refactor this" produces different output than "extract the validation logic from UserService into a separate Validator class, keeping the existing public method signatures."

**Language server health affects both modes.** Type errors, unresolved imports, and misconfigured environments degrade what Copilot can infer regardless of which mode you are in. A clean error panel is a baseline requirement, not an optional nicety.

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
in my usage that I can improve. I use both inline completions and agentic/multi-line
generations (Copilot Edits, agent mode).

For inline completions: quality depends on how much useful context surrounds the cursor
and which related files are open. For agentic generations: quality depends on task
description clarity, how well I scope the working set, and whether I break large tasks
into focused steps.

Optional notes I kept across these sessions:
- [Sessions where inline suggestions were accurate and useful]
- [Sessions where I kept rejecting or heavily editing inline completions]
- [Agentic tasks that needed significant rework after generation]
- [What kinds of tasks I was working on]

Please tell me:
1. For inline completions — what patterns appear consistently when context quality was weakest?
2. For agentic generations — were there patterns where task descriptions led to off-target output or excess rework?
3. Were there recurring situations where I should have broken a large agentic task into smaller steps?
4. Give me 3 specific, actionable changes to my workflow — covering both completion and agentic usage.
```

5. **Act on the output.** Findings that appear across multiple sessions are the ones worth acting on first. For completions, common patterns include editing isolated functions with no related types visible, or switching files before context stabilises. For agentic work, the patterns tend to be vague task descriptions ("clean this up"), working sets that are too broad, or large tasks that should have been split into two or three focused requests. Each finding maps directly to a habit you can change.

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

These changes help immediately without any tooling setup.

**For inline completions:**

- **Keep related files open.** Copilot pulls snippets from your open buffers. If your types, interfaces, and the file you are editing are open at the same time, the context is far more useful than if 20 unrelated tabs are cluttering the workspace.
- **Write signatures before bodies.** Copilot reads both above and below the cursor. A function with a name, typed parameters, a return type, and a one-line comment is a far richer signal than an empty stub. Write the skeleton first.
- **Use meaningful names in surrounding scope.** Copilot picks related files based on vocabulary overlap with what you are writing. Descriptive, domain-specific names make it more likely the right files get pulled in.

**For agentic and multi-line generations:**

- **Describe the goal and the constraints, not just the steps.** "Extract the validation logic from UserService into a Validator class, keeping the existing method signatures" outperforms "refactor UserService". The model needs to know what must not change as much as what should.
- **Scope your working set tightly.** In Copilot Edits, add only the files directly relevant to the task. A narrow working set means fewer irrelevant edits and faster iteration.
- **Break large tasks into focused requests.** A task that touches five subsystems in one pass is likely to produce something that needs significant rework. Two or three focused requests with a review step in between reliably produce better output than one ambitious prompt.

**For both:**

- **Resolve errors before you start.** TypeScript errors, unresolved imports, or a broken Python environment degrade what Copilot can infer regardless of which mode you are in. A clean error panel is a baseline, not an optional nicety.

---

Copilot output — whether it is a one-line completion or a multi-file agentic edit — is only as good as the context and task framing you give it. Most developers never measure either. The audit loop closes entirely inside VS Code: the same tool generating your completions and edits can tell you why they were not good enough.
