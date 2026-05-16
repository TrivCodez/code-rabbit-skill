---
name: code-rabbit
description: "Use this skill whenever the user wants to review a pull request, diff, or code change. Triggers include pasting a PR link, a GitHub diff, raw code changes, or asking for a code review, PR review, or review this. Performs a structured review with AI Agent fix prompts for each finding and outputs to chat by default. Only posts to GitHub, Slack, or email if explicitly asked."
license: MIT
---

# code-rabbit — PR Review Skill

Performs a structured PR review. Each finding is output as a **separate inline comment** — like GitHub's "View reviewed changes" reviewer comments — not dumped into one big block. Output goes to **chat by default**.

**Never mention "CodeRabbit" in any output.**  
**Never post to GitHub unless the user explicitly says so.**  
**Never combine all findings into one comment wall.**

---

## Input Modes

- A **GitHub PR URL** → fetch `<url>.diff` via web_fetch if network available
- A **raw diff or code paste** → review inline
- A **PR description + code** → combine for full context
- A **screenshot of a PR** → extract visible code/comments and review

---

## Output Structure — Separate Comments

Produce the review as a sequence of **individual comment cards**, one per file/finding. This mirrors how GitHub displays inline reviewer comments, not a single issue comment.

Start with a small header, then emit each comment as its own standalone block separated by a visible divider.

### Header (once, at the top)

```
🔍 PR Review — <PR title or branch name>
Closes #<issue> · <file count> file(s) · +<lines> lines

Actionable comments: <N>
```

---

### Per-Finding Comment Block

Each finding is a self-contained comment. Emit one block per file/issue — **never merge multiple findings into one block**.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📄 path/to/file.ext · Lines X–Y
⚠️ Potential issue | 🟠 Major | ⚡ Quick win

**Short title of the issue.**

One to two sentences: what is wrong, why it matters.

▼ Suggested fix
- old code line
+ new code line

▼ 📝 Committable suggestion
(full corrected block, ready to apply)

▼ 🤖 Prompt for AI Agents
Verify each finding against current code. Fix only still-valid issues,
skip the rest with a brief reason, keep changes minimal, and validate.

In `@path/to/file.ext` around lines X–Y, [precise instruction: what to
change, what to replace it with, side effects to handle (i18n keys,
shared constants, snapshot tests), and what to verify after the fix].
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Severity Tags

| Level | Severity line |
|---|---|
| Critical | `🔴 Critical` |
| Major | `⚠️ Potential issue | 🟠 Major | ⚡ Quick win` |
| Minor | `⚠️ Potential issue | 🟡 Minor | ⚡ Quick win` |
| Suggestion | `💡 Suggestion` |

Suggestions do **not** need a suggested fix or committable suggestion — just a short description.

---

## AI Agent Prompt Rules

Every Critical/Major/Minor block must include a `🤖 Prompt for AI Agents` section. Rules:

1. Always open with: `Verify each finding against current code. Fix only still-valid issues, skip the rest with a brief reason, keep changes minimal, and validate.`
2. Reference file with `@path/to/file.ext` syntax
3. Include exact line range
4. Be fully self-contained — no assumed context
5. Call out side effects: i18n keys, shared constants, tests, snapshots
6. End with what to verify after the fix

---

## Praise Block (after all findings)

Emit a separate praise block at the end. Keep it specific, not generic.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Praise

• [Specific thing done well and why it matters]
• [Another specific positive if warranted]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Summary Block (always last)

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

## Output Destination Rules

| User says | Action |
|---|---|
| Nothing (default) | Output to chat as separate comment blocks |
| "send to Slack" | Reformat with Slack mrkdwn, use Slack MCP if connected |
| "post to GitHub" / "comment on PR" | Format as GitHub review with inline comments + suggestion blocks |
| "send via email" | Plain email body |
| "save to file" | Write to /mnt/user-data/outputs/pr-review.md |

---

## Slack Format (when requested)

- `*bold*` not `**bold**`
- `:warning:` `:large_orange_circle:` `:large_yellow_circle:` `:white_check_mark:` `:bulb:` for icons
- Triple backticks for code
- Each comment block still separate, divided with `---`

---

## GitHub Format (only when explicitly requested)

When posting to GitHub, each finding becomes a **separate inline review comment** on the specific file + line — not one top-level comment. Use:
- suggestion blocks for committable suggestions (one-click apply)
- One top-level review comment for the summary only

---

## Notes

- If the diff is large, prioritise: auth → state management → API calls → error handling → accessibility → copy.
- If only a description is given with no code, say so and ask for the diff or file.
- If a GitHub URL is given and web_fetch is available, fetch `<url>.diff` automatically.
- Never invent line numbers — write "verify line number" in the agent prompt if unsure.
- Never say "CodeRabbit" anywhere in the output. The review is yours, not attributed to any tool.
