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
  "model": "gpt-5.2-codex",
  "reasoning_effort": "medium",
  "collaboration_mode": 1
}
```

### Available Options

| Option | Values | Default |
|--------|--------|---------|
| **model** | See model list below | `gpt-5.2-codex` |
| **reasoning_effort** | `high`, `medium`, `low` | `medium` |
| **collaboration_mode** | `1`, `2`, `3`, `4` | `1` |

### Supported Models

> **Maintenance Note**: Update this list periodically from [Codex Models Documentation](https://developers.openai.com/codex/models/)

| Model | Description |
|-------|-------------|
| `gpt-5.2-codex` | Latest codex model optimized for coding tasks (Recommended) |
| `gpt-5.2` | Latest general agentic model |
| `gpt-5.1-codex-max` | Optimized for long-horizon agentic coding tasks |
| `gpt-5.1-codex-mini` | Smaller, more cost-effective version |
| `gpt-5.1-codex` | Predecessor to Max version |
| `gpt-5-codex` | Older tuned version for coding |
| `gpt-5-codex-mini` | Older budget option |
| `gpt-5` | Base reasoning model |

Last updated: 2025-12-19

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
   - Mode 1, 3 & 4: `--sandbox read-only`
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
Use `--sandbox read-only` with multiple rounds. Claude Code designs the solution, Codex reviews and critiques, then iterate until alignment.

**Role Assignment:**
- **Claude Code (Proposer)**: Requirements analysis, solution design, plan formulation
- **Codex (Reviewer)**: Critical review, identify risks, provide alternative perspectives

**Workflow:**
1. **Claude Code analyzes requirements and designs solution**:
   - Understand the user's requirements thoroughly
   - Research relevant code in the codebase
   - Formulate a detailed implementation plan
   - Document key decisions and trade-offs

2. **Submit design to Codex for review**:
   ```bash
   echo "Please review my proposed solution for [task]:

   ## Requirements Analysis
   [Claude's analysis]

   ## Proposed Solution
   [Detailed design]

   ## Implementation Plan
   [Step-by-step plan]

   Please critically evaluate this proposal:
   - Are there any flaws or risks in this approach?
   - What alternatives would you suggest?
   - What have I potentially overlooked?

   Be objective and direct. Do not simply agree - provide genuine critical feedback." | codex exec --skip-git-repo-check -m <model> --config model_reasoning_effort="<level>" --sandbox read-only --full-auto 2>/dev/null
   ```

3. **Claude Code evaluates Codex's feedback objectively**:
   - Consider each point on its merits, not to defend the original proposal
   - Acknowledge valid criticisms
   - Identify points of genuine disagreement (not just misunderstanding)
   - Distinguish between: factual errors, preference differences, and genuine trade-offs

4. **If significant disagreements exist, continue dialogue**:
   ```bash
   echo "Thank you for the review. Here is my response:

   ## Points I Accept
   [Acknowledged valid criticisms and how to address them]

   ## Points I Disagree With
   [Specific disagreements with reasoning - not defensive, but substantive]

   ## Questions for Clarification
   [Any unclear points]

   Please respond to my disagreements with specific reasoning. It's fine to maintain your position if you believe it's correct." | codex exec --skip-git-repo-check resume --last 2>/dev/null
   ```

5. **Iterate until alignment** (max 2-3 rounds for cost efficiency):
   - Goal is mutual understanding, not forced agreement
   - Accept "agree to disagree" on genuine trade-offs
   - Focus on reaching clarity, not winning the argument

6. **Claude Code summarizes the final outcome**:
   - **Consensus**: Points both agree on (with reasoning)
   - **Accepted Changes**: Original design modified based on valid feedback
   - **Unresolved Differences**: Genuine disagreements where both positions have merit
     - Clearly state each side's reasoning
     - Identify what factors would favor each approach

7. **Present to user for final decision**:
   - User reviews the summary
   - User decides on unresolved differences
   - User approves or requests modifications

8. **Claude Code implements the approved solution**

**Discussion Principles:**
- Be objective and fact-based, not defensive
- Acknowledge when the other side has a valid point
- Do not agree just to be agreeable - genuine disagreement is valuable
- Seek truth and best solution, not consensus for its own sake
- Clearly distinguish facts from opinions/preferences
- Favor simplicity - complete functionality and correct logic suffice, avoid over-engineering

**Token Usage Note:** Each resume round accumulates context. Limit to 2-3 rounds for cost efficiency.

### Mode 4: Deep Collaboration - Reversed Roles (Alternative perspective)
Use `--sandbox read-only` with multiple rounds. Codex designs the solution, Claude Code reviews and critiques, then iterate until alignment.

**Role Assignment:**
- **Codex (Proposer)**: Requirements analysis, solution design, plan formulation
- **Claude Code (Reviewer)**: Critical review, identify risks, provide alternative perspectives

**When to Use Mode 4 vs Mode 3:**
- Mode 3: When you want Claude Code's design sensibilities to lead
- Mode 4: When you want a fresh perspective from Codex, or to leverage Codex's specific strengths

**Workflow:**
1. **Request Codex to analyze requirements and design solution**:
   ```bash
   echo "Please analyze the following task and provide a complete solution design:

   ## Task
   [User's requirements]

   ## Context
   [Relevant codebase information Claude has gathered]

   Please provide:
   1. Requirements Analysis - your understanding of what needs to be done
   2. Proposed Solution - detailed design approach
   3. Implementation Plan - step-by-step plan
   4. Trade-offs - key decisions and their implications

   Be thorough and specific." | codex exec --skip-git-repo-check -m <model> --config model_reasoning_effort="<level>" --sandbox read-only --full-auto 2>/dev/null
   ```

2. **Claude Code critically reviews Codex's proposal**:
   - Evaluate the design objectively, not to find fault but to improve
   - Identify genuine risks or oversights
   - Consider alternative approaches
   - Distinguish between: factual errors, preference differences, and genuine trade-offs

3. **If Claude Code has feedback, continue dialogue**:
   ```bash
   echo "Thank you for the proposal. Here is my review:

   ## Strengths
   [What works well in the proposal]

   ## Concerns
   [Specific issues with reasoning - not nitpicking, but substantive]

   ## Alternative Considerations
   [Other approaches worth considering]

   ## Questions
   [Clarifications needed]

   Please respond to my concerns. Defend your position where you believe it's correct, and acknowledge where adjustments might be needed." | codex exec --skip-git-repo-check resume --last 2>/dev/null
   ```

4. **Claude Code evaluates Codex's response objectively**:
   - Consider whether Codex's defense is valid
   - Acknowledge when own concerns were unfounded
   - Identify remaining genuine disagreements

5. **Iterate until alignment** (max 2-3 rounds for cost efficiency):
   - Goal is mutual understanding, not forced agreement
   - Accept "agree to disagree" on genuine trade-offs
   - Focus on reaching clarity, not winning the argument

6. **Claude Code summarizes the final outcome**:
   - **Consensus**: Points both agree on (with reasoning)
   - **Accepted Modifications**: Changes to Codex's original design based on review
   - **Unresolved Differences**: Genuine disagreements where both positions have merit
     - Clearly state each side's reasoning
     - Identify what factors would favor each approach

7. **Present to user for final decision**:
   - User reviews the summary
   - User decides on unresolved differences
   - User approves or requests modifications

8. **Claude Code implements the approved solution**

**Discussion Principles (same as Mode 3):**
- Be objective and fact-based, not defensive
- Acknowledge when the other side has a valid point
- Do not agree just to be agreeable - genuine disagreement is valuable
- Seek truth and best solution, not consensus for its own sake
- Clearly distinguish facts from opinions/preferences

**Token Usage Note:** Each resume round accumulates context. Limit to 2-3 rounds for cost efficiency.

### Quick Reference
| Use case | Mode | Sandbox | Proposer | Reviewer | Token cost |
| --- | --- | --- | --- | --- | --- |
| Code review / analysis | Mode 1 | `read-only` | Codex | Claude | Low |
| Refactoring suggestions | Mode 1 | `read-only` | Codex | Claude | Low |
| Automated code changes | Mode 2 | `workspace-write` | Codex | - | Low |
| Complex multi-file edits | Mode 2 | `workspace-write` | Codex | - | Low |
| Architecture design (Claude leads) | Mode 3 | `read-only` | Claude | Codex | Medium-High |
| Critical decisions (Claude leads) | Mode 3 | `read-only` | Claude | Codex | Medium-High |
| Architecture design (Codex leads) | Mode 4 | `read-only` | Codex | Claude | Medium-High |
| Fresh perspective needed | Mode 4 | `read-only` | Codex | Claude | Medium-High |

## Following Up
- After every `codex` command, use `AskUserQuestion` to confirm next steps or decide whether to resume the session.
- When resuming, pipe the new prompt via stdin: `echo "new prompt" | codex exec --skip-git-repo-check resume --last 2>/dev/null`.
- Inform the user: "You can resume this Codex session at any time by saying 'codex resume'."

## Error Handling
- Stop and report failures whenever `codex --version` or a `codex exec` command exits non-zero; request direction before retrying.
- Before you use high-impact flags (`--full-auto`, `--sandbox danger-full-access`) ask the user for permission using `AskUserQuestion` unless it was already given.
- When output includes warnings or partial results, summarize them and ask how to adjust using `AskUserQuestion`.
