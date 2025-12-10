---
name: codex
description: Use when the user asks to run Codex CLI (codex exec, codex resume) or references OpenAI Codex for code analysis, refactoring, or automated editing
---

# Codex Skill Guide

## Running a Task
1. Ask the user (via `AskUserQuestion`) which reasoning effort to use (`high`, `medium`, or `low`). The model is fixed to `gpt-5.1-codex-max`.
2. Select the sandbox mode required for the task; default to `--sandbox read-only` unless edits or network access are necessary.
3. Assemble the command with the appropriate options:
   - `-m gpt-5.1-codex-max`
   - `--config model_reasoning_effort="<high|medium|low>"`
   - `--sandbox <read-only|workspace-write|danger-full-access>`
   - `--full-auto`
   - `-C, --cd <DIR>`
   - `--skip-git-repo-check`
4. Always use --skip-git-repo-check.
5. When continuing a previous session, use `codex exec --skip-git-repo-check resume --last` via stdin. When resuming don't use any configuration flags unless explicitly requested by the user e.g. if he specifies the model or the reasoning effort when requesting to resume a session. Resume syntax: `echo "your prompt here" | codex exec --skip-git-repo-check resume --last 2>/dev/null`. All flags have to be inserted between exec and resume.
6. **IMPORTANT**: By default, append `2>/dev/null` to all `codex exec` commands to suppress thinking tokens (stderr). Only show stderr if the user explicitly requests to see thinking tokens or if debugging is needed.
7. Run the command, capture stdout/stderr (filtered as appropriate), and summarize the outcome for the user.

## Collaboration Modes

This skill supports three collaboration modes between Claude Code and Codex:

### Mode 1: Analysis + Independent Evaluation (Recommended for most tasks)
Use `--sandbox read-only`. Codex analyzes, Claude Code independently evaluates, then user decides.

**Workflow:**
1. Run codex with `--sandbox read-only` to get analysis and suggestions
2. **Claude Code independently evaluates** Codex's suggestions:
   - Verify correctness of each suggestion
   - Identify potential risks or issues Codex may have missed
   - Add supplementary suggestions if needed
3. Present to user with clear structure:
   - **Consensus**: Points both agree on
   - **Claude Code's additions/corrections**: Supplementary insights or corrections
   - **Concerns**: Any risks or issues identified
4. Use `AskUserQuestion` to ask user which suggestions to implement
5. If user agrees, **Claude Code implements the changes** using its own tools (Edit, Write, Bash, etc.)
6. After implementation, summarize what was changed

### Mode 2: Codex Direct Execution
Use `--sandbox workspace-write`. Codex directly modifies files.

**Workflow:**
1. Run codex with `--sandbox workspace-write --full-auto`
2. Codex directly makes changes to files
3. Summarize what Codex modified for the user

### Mode 3: Deep Collaboration (For complex/critical tasks)
Use `--sandbox read-only` with multiple rounds. Claude Code and Codex engage in back-and-forth discussion until reaching consensus.

**Workflow:**
1. Run codex with `--sandbox read-only` to get initial analysis
2. Claude Code evaluates and identifies points of disagreement or uncertainty
3. If disagreements exist, **resume the Codex session** with Claude Code's counterpoints:
   ```bash
   echo "I have concerns about your suggestions: [specific concerns]. Please address these points." | codex exec --skip-git-repo-check resume --last 2>/dev/null
   ```
4. Codex responds, may revise or defend its position
5. Repeat steps 2-4 until:
   - Consensus is reached, OR
   - Clear disagreement points are identified (max 3 rounds recommended to control token usage)
6. Present final conclusion to user:
   - **Consensus items**: Both agree
   - **Resolved disagreements**: Initially differed, now agreed
   - **Unresolved disagreements**: Still differ, user decides
7. User makes final decision
8. Claude Code implements approved changes

**Token Usage Note:** Each resume round accumulates context. Limit to 2-3 rounds for cost efficiency.

### Quick Reference
| Use case | Mode | Sandbox | Who modifies files | Token cost |
| --- | --- | --- | --- | --- |
| Code review / analysis | Mode 1 | `read-only` | Claude Code (after user approval) | Low |
| Refactoring suggestions | Mode 1 | `read-only` | Claude Code (after user approval) | Low |
| Automated code changes | Mode 2 | `workspace-write` | Codex directly | Low |
| Complex multi-file edits | Mode 2 | `workspace-write` | Codex directly | Low |
| Critical decisions / Architecture | Mode 3 | `read-only` | Claude Code (after consensus) | Medium-High |
| Controversial changes | Mode 3 | `read-only` | Claude Code (after consensus) | Medium-High |

## Following Up
- After every `codex` command, use `AskUserQuestion` to confirm next steps or decide whether to resume the session.
- When resuming, pipe the new prompt via stdin: `echo "new prompt" | codex exec --skip-git-repo-check resume --last 2>/dev/null`.
- Inform the user: "You can resume this Codex session at any time by saying 'codex resume'."

## Error Handling
- Stop and report failures whenever `codex --version` or a `codex exec` command exits non-zero; request direction before retrying.
- Before you use high-impact flags (`--full-auto`, `--sandbox danger-full-access`, `--skip-git-repo-check`) ask the user for permission using AskUserQuestion unless it was already given.
- When output includes warnings or partial results, summarize them and ask how to adjust using `AskUserQuestion`.
