---
layout: post
title: "Let AI Audit Your AI Coding Sessions: How to Actually Get Better at Copilot, Claude, and ChatGPT"
date: 2026-02-28 10:00:00 +0000
categories: productivity
tags: github-copilot claude chatgpt ai developer-tools vscode cursor
---
You use AI coding tools every day — GitHub Copilot, Claude Code, ChatGPT, Cursor, or some combination of all of them. You accept suggestions, you reject suggestions, you occasionally swear at suggestions. You kick off agentic edits that rewrite half a file and then spend ten minutes undoing them. But do you ever stop and ask: am I actually getting the most out of these tools, or am I just using them on autopilot?

Most developers treat AI coding assistants as a black box. A suggestion appears, you tab or you escape, and you move on. You type a task and accept whatever comes back. The problem is that the quality of everything these tools produce — inline completions, multi-line generations, agentic edits across multiple files — is almost entirely determined by *your* habits: what context you give them, how you scope your tasks, how you write your prompts. Without auditing those habits, you have no way to improve them.

There is a straightforward fix: feed your session logs back into an AI model and ask it to find the patterns. Every major AI coding tool has been writing logs to disk every time you work. A pattern that shows up across a week of sessions is a real habit — one bad session is just noise.

## How AI Coding Tools Build Context

All of these tools work in two distinct modes and each one determines quality differently.

**Inline completions** trigger as you type. The tool reads everything above and below the cursor in the current file, then pulls in snippets from your other open buffers — favouring the ones most related to what you are writing. A function with a meaningful name, typed parameters, and a short comment gives it far more to work with than an empty stub. If you have 20 unrelated files open, the useful ones get crowded out by the noise.

**Agentic and multi-line generations** (Copilot Edits, Claude Code, ChatGPT with a long task, Cursor Composer) work differently. Here the model has a much larger context window and is driven by what you explicitly provide: your task description, the files you include, and the conversation history. The tool can also read additional files on its own to complete the task. The limiting factor is not cursor position — it is task clarity. A vague "refactor this" produces different output than "extract the validation logic from UserService into a separate Validator class, keeping the existing public method signatures."

**Language server health affects inline completions.** Type errors, unresolved imports, and misconfigured environments degrade what the tool can infer about your code. A clean error panel before you start means cleaner inline output throughout the session.

## Where the Logs Live

Each tool stores session data differently. Find the right path for what you use.

**GitHub Copilot (VS Code)**

Copilot does not log verbosely by default. First enable debug logging by adding this to your VS Code `settings.json`:

```json
{
  "github.copilot.advanced": {
    "debug.overrideLogLevels": {
      "*": "DEBUG"
    }
  }
}
```

VS Code then writes one log file per window session, in a timestamped subdirectory:

```
# Linux
~/.config/Code/logs/<timestamp>/exthost1/GitHub Copilot.log

# macOS
~/Library/Application Support/Code/logs/<timestamp>/exthost1/GitHub Copilot.log
```

Open `Help > Open Logs Folder` in VS Code to jump straight to the directory.

**Claude Code**

No setup needed. Claude Code writes a full JSONL transcript of every session automatically, keyed by project:

```
~/.claude/projects/<project-hash>/<session-id>.jsonl
```

The project hash is derived from your working directory path. Each `.jsonl` file contains the complete message history including tool calls, token counts, and timestamps — much richer than extension debug logs.

**ChatGPT**

For the web interface: Settings → Data controls → Export data. You receive a zip file containing `conversations.json` with your full chat history across all conversations.

For the desktop app: conversation data is stored locally. On macOS it lives under `~/Library/Application Support/ChatGPT/`.

**Cursor**

Cursor is built on VS Code and uses the same timestamped session structure under `~/.cursor/logs/`. Cursor Composer (the agentic mode) logs appear in the relevant extension host subfolder alongside inline completion events.

## The Optimization Workflow

1. **Enable debug logging if you use Copilot.** For all other tools the logs above are written automatically — skip this step. For Copilot, apply the `settings.json` change from the section above and verify you can see entries in `View > Output > GitHub Copilot Log`.

2. **Work normally for a few days.** Just code. All of the tools above are already writing session data in the background. If you want to jot down notes on sessions where output was particularly good or bad, that adds useful signal — but it is optional.

3. **Collect your session logs.** Using the paths from the section above, find the logs covering your last several working days. The more sessions you include, the more confidently patterns separate from one-off noise.

4. **Open an AI assistant and attach the logs.** Use any large-context model — Copilot Chat, Claude.ai, or ChatGPT. Type `#file:` (Copilot Chat) or use the file attachment button to load the log files, then append this prompt:

```text
Analyze the attached AI coding session logs (multiple sessions, possibly from multiple
tools) and identify patterns in my usage that I can improve.

I use both inline completions and agentic/multi-line generations. For inline completions,
quality depends on how much useful context surrounds the cursor and which related files
are open. For agentic sessions, quality depends on task description clarity, how well I
scope what the tool can see, and whether I break large tasks into focused steps.

Tool(s) these logs are from: [Copilot / Claude Code / ChatGPT / Cursor]

Optional notes I kept across these sessions:
- [Sessions where inline suggestions were accurate and useful]
- [Sessions where I kept rejecting or heavily editing completions]
- [Agentic tasks that needed significant rework after generation]
- [What kinds of tasks I was working on]

Please tell me:
1. For inline completions — what patterns appear consistently when output quality was weakest?
2. For agentic sessions — were there patterns where my prompts or task descriptions led to off-target output or excess rework?
3. Were there recurring situations where I should have broken a large task into smaller steps?
4. Give me 3 specific, actionable changes to my workflow — covering both completion and agentic usage.
```

5. **Act on the output.** Findings that appear across multiple sessions are the ones worth acting on first. For completions, common patterns include editing isolated functions with no related types visible, or switching files before context stabilises. For agentic work, the patterns tend to be vague task descriptions ("clean this up"), working sets that are too broad, or large tasks that should have been split into two or three focused requests. Each finding maps directly to a habit you can change.

## Managing Context Length

Debug logs grow fast. A single productive coding day with DEBUG-level logging enabled will produce a `GitHub Copilot.log` in the 2–10 MB range — roughly 500K to 2.5M tokens. Stack several sessions together and you will blow past the context window of any model currently available in Copilot Chat before the analysis even starts.

**Switch to a premium model before attaching files.** In the Copilot Chat panel, click the model selector (bottom-left of the input box) and choose the highest-context option you have access to. As of early 2026, Claude Sonnet offers the largest context window (200K tokens), followed by GPT-4o at 128K. The default models used for quick chat are smaller and will silently truncate large inputs — dropping the earliest sessions and giving you a skewed picture of your patterns.

**Trim the logs before attaching.** You do not need every line. For Copilot logs, extract just the completion signal:

```bash
grep -iE "accept|reject|latency|contextSize|token|completion" \
  ~/.config/Code/logs/*/exthost1/"GitHub Copilot.log" \
  > ~/copilot-audit.txt
```

For Claude Code JSONL files, the data is already structured — you can pull just the message-level entries and strip the raw file content from tool call outputs:

```bash
cat ~/.claude/projects/<hash>/*.jsonl \
  | jq -c 'del(.content[]? | select(.type=="tool_result") | .content)' \
  > ~/claude-audit.jsonl
```

For ChatGPT exports, the `conversations.json` file is already compact — attach it directly once you locate the relevant conversations.

The result is typically 5–20× smaller than the raw logs and contains everything needed for pattern analysis.

If the trimmed file is still over a few MB, split it by date range and run two separate analyses — one on older sessions, one on recent. The model produces more reliable output on 10K focused lines than on 200K lines it had to aggressively compress to fit.

## Workspace Hygiene — The Quick Wins

These changes help immediately without any tooling setup.

**For inline completions:**

- **Keep related files open.** Inline completion tools pull snippets from your open buffers. If your types, interfaces, and the file you are editing are open at the same time, the context is far more useful than if 20 unrelated tabs are cluttering the workspace.
- **Write signatures before bodies.** These tools read both above and below the cursor. A function with a name, typed parameters, a return type, and a one-line comment is a far richer signal than an empty stub. Write the skeleton first.
- **Use meaningful names in surrounding scope.** Tools pick related files based on vocabulary overlap with what you are writing. Descriptive, domain-specific names make it more likely the right files get pulled in.

**For agentic and multi-line generations:**

- **Describe the goal and the constraints, not just the steps.** "Extract the validation logic from UserService into a Validator class, keeping the existing method signatures" outperforms "refactor UserService". The model needs to know what must not change as much as what should.
- **Scope your working set tightly.** In Copilot Edits, add only the files directly relevant to the task. A narrow working set means fewer irrelevant edits and faster iteration.
- **Break large tasks into focused requests.** A task that touches five subsystems in one pass is likely to produce something that needs significant rework. Two or three focused requests with a review step in between reliably produce better output than one ambitious prompt.

**For both:**

- **Resolve errors before you start.** TypeScript errors, unresolved imports, or a broken Python environment degrade what any inline completion tool can infer about your code. A clean error panel is a baseline, not an optional nicety.

---

AI coding output — whether it is a one-line completion or a multi-file agentic edit — is only as good as the context and task framing you give it. Most developers never measure either. The logs are already there. Feed them back into the same class of tool that generated them and you will get specific, session-grounded feedback that no generic tips article can match.
