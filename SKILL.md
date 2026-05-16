---
name: code-rabbit
description: "Use this skill whenever the user wants to review a pull request, diff, or code change. Triggers include pasting a PR link, a GitHub diff, raw code changes, or asking for a code review, PR review, or review this. Performs a structured review with AI Agent fix prompts for each finding and outputs to chat by default. Only posts to GitHub, Slack, or email if explicitly asked."
license: MIT
---

# code-rabbit — PR Review Skill

Performs a structured PR review. Output goes to **chat by default** as separate comment cards.

**Never mention "CodeRabbit" anywhere in output.**
**Never post to GitHub unless the user explicitly says so.**
**Never dump all findings into one single comment.**

---

## Step 1 — Parse the Input

Accept any of:
- GitHub PR URL → use GitHub MCP to fetch the PR diff and file list
- Raw diff or code paste → review inline
- PR description + code → combine for context
- Screenshot → extract visible code and review

From the input, identify: files changed, PR purpose, related issue number, tech stack.

---

## Step 2 — Run the Review

Analyse the diff and produce findings across these categories in priority order:
auth → state management → API calls → error handling → accessibility → copy/UX

For each finding assign a severity:
- 🔴 Critical — data loss, security hole, crash, broken auth
- 🟠 Major — broken retry, bad error handling, wrong API usage, UX bug
- 🟡 Minor — misleading copy, naming issue, style inconsistency
- 💡 Suggestion — optional improvement, no fix required

---

## Step 3 — Output Format (chat default)

### Header (once)
```
🔍 PR Review — <branch or PR title>
Closes #<N> · <N> file(s) · +<N> lines
Actionable comments: <N>
```

### Each finding = its own separate block

NEVER merge two findings into one block. Each gets its own `━━━` card:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📄 path/to/file.ext · Lines X–Y
⚠️ Potential issue | 🟠 Major | ⚡ Quick win

**Short title of the issue.**

What is wrong and why it matters (1–2 sentences).

▼ Suggested fix
```diff
- old line
+ new line
```

▼ 📝 Committable suggestion
```language
// full corrected block ready to apply
```

▼ 🤖 Prompt for AI Agents
```
Verify each finding against current code. Fix only still-valid issues,
skip the rest with a brief reason, keep changes minimal, and validate.

In `@path/to/file.ext` around lines X–Y, [exact instruction: what to
change, what to replace with, side effects (i18n keys, shared constants,
snapshot tests), and what to verify after].
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Suggestions (💡) only need a short description — no fix or committable suggestion required.

### Praise block (after all findings)
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Praise
• Specific thing done well and why it matters.
• Another specific positive if warranted.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Summary block (always last)
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📋 REVIEW SUMMARY

| Category       | Count |
|----------------|-------|
| 🔴 Critical     |   X   |
| 🟠 Major        |   X   |
| 🟡 Minor        |   X   |
| 💡 Suggestions  |   X   |

Verdict: REQUEST CHANGES / APPROVE / NEEDS DISCUSSION
Reason: One sentence.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Step 4 — GitHub Posting (only when user explicitly asks)

When the user says "post to GitHub", "submit review", or "comment on the PR":

Use the GitHub MCP to submit a **proper pull request review** — NOT a single issue comment.

### Exact GitHub MCP flow:

1. **Fetch PR details** via GitHub MCP to get: `owner`, `repo`, `pull_number`, and the latest commit SHA (`head.sha`)

2. **For each Critical/Major/Minor finding**, call the GitHub MCP `create_pull_request_review` or equivalent tool with:
   - `commit_id`: the latest commit SHA
   - `event`: `REQUEST_CHANGES` (or `APPROVE` / `COMMENT` based on verdict)
   - `comments`: array of inline comment objects, one per finding:
     ```json
     {
       "path": "src/path/to/file.ext",
       "line": <end line number of the finding>,
       "body": "<severity line>\n\n**Title**\n\nExplanation.\n\n```suggestion\n// corrected code\n```\n\n🤖 Prompt for AI Agents\n..."
     }
     ```

3. **Top-level review body**: post only the Summary block as the main review body — NOT the individual findings (those go as inline comments).

4. **Do NOT** post a separate issue comment or PR comment with all findings merged — that is the wrong format.

### Inline comment body format (GitHub markdown):
```
⚠️ Potential issue | 🟠 Major | ⚡ Quick win

**Title of the issue.**

Explanation of the problem.

```suggestion
// corrected code block
```

<details>
<summary>🤖 Prompt for AI Agents</summary>

Verify each finding against current code. Fix only still-valid issues,
skip the rest with a brief reason, keep changes minimal, and validate.

In `@path/to/file.ext` around lines X–Y, [instruction].
</details>
```

---

## Output Destination Rules

| User says | Action |
|---|---|
| Nothing (default) | Chat — separate ━━━ cards |
| "send to Slack" | Slack mrkdwn format, use Slack MCP |
| "post to GitHub" / "submit review" / "comment on PR" | GitHub MCP — inline comments per finding + summary as review body |
| "save to file" | /mnt/user-data/outputs/pr-review.md |

---

## AI Agent Prompt Rules

1. Open with: `Verify each finding against current code. Fix only still-valid issues, skip the rest with a brief reason, keep changes minimal, and validate.`
2. Use `@path/to/file.ext` syntax
3. Give exact line range
4. Be fully self-contained
5. Mention side effects: i18n keys, constants, snapshots, related components
6. End with what to verify

---

## Notes

- Never invent line numbers — write "verify line number" in agent prompt if unsure.
- Never say "CodeRabbit" anywhere in output.
- If GitHub MCP tools aren't available when trying to post, tell the user clearly.
- If only a description is given with no code, ask for the diff.
