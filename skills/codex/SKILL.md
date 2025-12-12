---
name: codex
description: Use when the user asks to run Codex CLI (codex exec, codex resume) or references OpenAI Codex for code analysis, refactoring, or automated editing
---

# Codex Skill Guide

## Configuration Management

### Config File
User preferences are stored in `.codex-config.json` in the project root directory:

```json
{
  "model": "gpt-5.1-codex-max",
  "reasoning_effort": "medium",
  "collaboration_mode": 1
}
```

### Available Options

| Option | Values | Default |
|--------|--------|---------|
| **model** | See model list below | `gpt-5.1-codex-max` |
| **reasoning_effort** | `high`, `medium`, `low` | `medium` |
| **collaboration_mode** | `1`, `2`, `3` | `1` |

### Supported Models

> **Maintenance Note**: Update this list periodically from [Codex Models Documentation](https://developers.openai.com/codex/models/)

| Model | Description |
|-------|-------------|
| `gpt-5.2` | Latest general agentic model |
| `gpt-5.1-codex-max` | Optimized for long-horizon agentic coding tasks (Recommended) |
| `gpt-5.1-codex-mini` | Smaller, more cost-effective version |
| `gpt-5.1-codex` | Predecessor to Max version |
| `gpt-5-codex` | Older tuned version for coding |
| `gpt-5-codex-mini` | Older budget option |
| `gpt-5` | Base reasoning model |

Last updated: 2025-12-12

### First Run
If `.codex-config.json` does not exist, use `AskUserQuestion` to ask all three options in a single question panel:
1. Model selection
2. Reasoning effort level
3. Collaboration mode

Then save the choices to `.codex-config.json`.

### Subsequent Runs
1. Read `.codex-config.json`
2. Display current configuration to user briefly: "Using: [model], reasoning=[level], mode=[N]"
3. Proceed with the task directly (no need to ask again)

### User-Initiated Switching
Users can change settings at any time by saying:
- "Switch model to o3" / "Use o4-mini"
- "Set reasoning effort to high" / "Use low reasoning"
- "Switch to Mode 2" / "Use deep collaboration mode"
- "Reset Codex config" (delete config and re-ask all options)

When a switch command is detected:
1. Update `.codex-config.json` with the new value
2. Confirm the change to the user
3. Continue with the current or next task

## Running a Task

1. **Load or Initialize Configuration**
   - If `.codex-config.json` exists, read it and display current settings
   - If not, use `AskUserQuestion` to ask model, reasoning effort, and collaboration mode, then save to `.codex-config.json`

2. **Determine Sandbox Mode** based on collaboration mode:
   - Mode 1 & 3: `--sandbox read-only`
   - Mode 2: `--sandbox workspace-write`

3. **Assemble the Command** with configured options:
   - `-m <model>`
   - `--config model_reasoning_effort="<high|medium|low>"`
   - `--sandbox <read-only|workspace-write|danger-full-access>`
   - `--full-auto`
   - `-C, --cd <DIR>`
   - `--skip-git-repo-check`

4. Always use `--skip-git-repo-check`.

5. **Resuming a Session**: Use `codex exec --skip-git-repo-check resume --last` via stdin. When resuming, don't use configuration flags unless explicitly requested by the user. Resume syntax:
   ```bash
   echo "your prompt here" | codex exec --skip-git-repo-check resume --last 2>/dev/null
   ```
   All flags must be inserted between `exec` and `resume`.

6. **IMPORTANT**: By default, append `2>/dev/null` to all `codex exec` commands to suppress thinking tokens (stderr). Only show stderr if the user explicitly requests to see thinking tokens or if debugging is needed.

7. **Execute and Follow Collaboration Mode Workflow** (see below).

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
- Before you use high-impact flags (`--full-auto`, `--sandbox danger-full-access`) ask the user for permission using `AskUserQuestion` unless it was already given.
- When output includes warnings or partial results, summarize them and ask how to adjust using `AskUserQuestion`.
