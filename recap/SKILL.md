---
name: recap
description: Generate an end-of-day recap summarizing work done across all Claude Code sessions. Use when user wants a daily summary, end-of-day notes, daily recap, journal entry, or daily log.
user-invocable: true
disable-model-invocation: true
allowed-tools: Bash(git *), Bash(gh *), Bash(python3 *), Read, Glob, Grep, Write, Agent, Skill, mcp__monday-api-mcp__*, mcp__claude_ai_monday_com__*
---

# Daily Recap

Generate a daily recap note by reading full conversation transcripts and cross-referencing with git, GitHub, and Monday.com activity.

## Argument parsing

Parse $ARGUMENTS for:
- A **date**: "yesterday", a YYYY-MM-DD string, or nothing (defaults to today)

Examples: `/recap`, `/recap yesterday`, `/recap 2026-03-28`

Resolve the date to a YYYY-MM-DD string for use in all subsequent steps.

## Vault routing

The recap is saved as a markdown file inside an Obsidian vault. Read the vault path from the `OBSIDIAN_VAULT` environment variable:

```bash
echo "${OBSIDIAN_VAULT:-}"
```

If `$OBSIDIAN_VAULT` is set, save to `$OBSIDIAN_VAULT/daily-recaps/YYYY-MM-DD.md`. If it is not set, ask the user for a vault path (and point them at the README for how to configure `$OBSIDIAN_VAULT` persistently).

You may also support per-project routing: if the user has exported multiple vault env vars (e.g. `OBSIDIAN_VAULT_WORK`, `OBSIDIAN_VAULT_PERSONAL`) and the current working directory matches a documented mapping, prefer that. Otherwise fall back to `$OBSIDIAN_VAULT`.

Create the output directory if it doesn't exist.

## Step 1: Identify sessions for the target date

Run this script to find session IDs and their transcript file paths:

```bash
RECAP_DATE="<resolved date>" python3 << 'PYEOF'
import json, os
from datetime import datetime, timedelta

raw = os.environ.get('RECAP_DATE', 'today').strip()
if raw == 'today':
    target = datetime.now().strftime('%Y-%m-%d')
elif raw == 'yesterday':
    target = (datetime.now() - timedelta(days=1)).strftime('%Y-%m-%d')
else:
    target = raw

project = os.getcwd()
home = os.path.expanduser('~')
encoded_project = project.replace('/', '-')
project_dir = os.path.join(home, '.claude/projects', encoded_project)

session_ids = set()
with open(os.path.join(home, '.claude/history.jsonl'), 'r') as f:
    for line in f:
        try:
            entry = json.loads(line.strip())
        except:
            continue
        if entry.get('project') != project:
            continue
        ts = entry.get('timestamp', 0) / 1000
        if datetime.fromtimestamp(ts).strftime('%Y-%m-%d') == target:
            session_ids.add(entry.get('sessionId'))

if not session_ids:
    print(f"No sessions found for {target} in this project.")
    exit(0)

print(f"Date: {target}")
print(f"Sessions: {len(session_ids)}")
print(f"Project dir: {project_dir}")
print()
for sid in sorted(session_ids):
    filepath = os.path.join(project_dir, f'{sid}.jsonl')
    exists = os.path.exists(filepath)
    subagent_dir = os.path.join(project_dir, sid, 'subagents')
    subagents = []
    if os.path.isdir(subagent_dir):
        subagents = [f for f in os.listdir(subagent_dir) if f.endswith('.meta.json')]
    print(f"Session: {sid}")
    print(f"  Transcript: {filepath} (exists={exists})")
    print(f"  Subagents: {len(subagents)}")
    for sa in subagents:
        meta_path = os.path.join(subagent_dir, sa)
        try:
            with open(meta_path) as mf:
                meta = json.load(mf)
                print(f"    - {sa.replace('.meta.json','')}: {meta.get('agentType','')} - {meta.get('description','')}")
        except:
            pass
    print()
PYEOF
```

## Step 2: Extract full transcripts using per-session agents

For **each session** found in Step 1, spawn an Agent to read and summarize that session's transcript.

Each agent should:

1. **Read the session JSONL file** using the Read tool (it may be large, read in chunks if needed)
2. **Read any subagent transcripts** if they exist (from Step 1 output). Subagent JSONL files are at `<project_dir>/<sessionId>/subagents/agent-<id>.jsonl`.
3. **Extract a comprehensive summary** covering:
   - What the user asked for (key prompts and goals)
   - What the assistant did (actions taken, findings, decisions made)
   - Files created or modified (full paths)
   - Commands run and their notable output (errors, test results, API responses)
   - Skills invoked (look for `<command-name>` tags) and what they produced
   - Any items identified as "to do", "next steps", or "follow up"
   - Key discussions and decisions, especially those that inform unfinished work

The agent is reading Claude Code's own transcript format - it will understand the message structure naturally. Focus on capturing *substance* (what was done, what was decided, what's pending) and skip tool coordination noise (glob/grep/read exploration that didn't lead to action).

**Important**: Do NOT truncate messages. Capture the full content so nothing is missed.

## Step 3: Gather git activity for the target date

Adjust the `--since` and `--until` flags based on the resolved date. For today, use `--since="midnight"`. For other dates, use `--since="YYYY-MM-DD" --until="YYYY-MM-DDT23:59:59"`.

```bash
git log --all --since="<date>" --until="<next day>" --format="%h %s (%an, %ar) [%D]" --reverse
```

```bash
git status --short
```

```bash
git branch --sort=-committerdate --format="%(refname:short) %(committerdate:relative)" | head -10
```

## Step 4: Gather PR activity for the target date

Check for PRs created, reviewed, and commented on by the user. Gather enough detail to determine the *state* of each PR and what interactions happened.

```bash
# PRs authored by me
gh pr list --author="@me" --state=all --search="created:>=<date>" --json number,title,state,createdAt,url,reviews,comments
```

```bash
# PRs I reviewed
gh api search/issues --method GET -f q="type:pr repo:$(gh repo view --json nameWithOwner -q .nameWithOwner) reviewed-by:@me updated:>=<date>" -f per_page=20 --jq '.items[] | "#\(.number) \(.title) \(.pull_request.html_url)"'
```

```bash
# PRs I commented on
gh api search/issues --method GET -f q="type:pr repo:$(gh repo view --json nameWithOwner -q .nameWithOwner) commenter:@me updated:>=<date>" -f per_page=20 --jq '.items[] | "#\(.number) \(.title) \(.pull_request.html_url)"'
```

For each PR found, get interaction details:

```bash
# Get comments and reviews on a specific PR to understand what happened
gh pr view <number> --json reviews,comments,state,mergedAt,author
```

Categorize each PR by your relationship to it:
- **Your PRs**: what state are they in? Did someone leave comments or approve? Are they waiting on reviewers?
- **Others' PRs**: did you review, comment, or push commits? What was your action?

## Step 5 (optional): Gather Monday.com activity for the target date

**Skip this step if the Monday MCP is not configured.** Check whether any `mcp__monday-api-mcp__*` tools are available in the session. If they are not, move on to Step 6 without the Monday source.

If Monday MCP is available, fetch activity relevant to you:

1. **Get your Monday user context** to identify your user ID
2. **Search for items assigned to you** that had updates on the target date
3. **Get updates/comments** on those items to find messages from others that need your attention
4. **Search for items newly assigned to you** on the target date

Scope tightly - only capture:
- **Comments/updates from others on items assigned to you** - someone left you a message
- **New items assigned to you** - something landed on your plate

Skip status change automations and activity on items you're not assigned to. Include real Monday item URLs in the output.

See the repo README for how to enable the Monday MCP. Without it, the recap still works - it just won't contain a Monday section.

## Step 6: Cross-reference and synthesize

You now have up to five sources (Monday is optional):
- **Session summaries** (Step 2): Full picture of what was asked, discussed, decided, attempted, and produced
- **Git history** (Step 3): What was actually committed and shipped
- **PR activity** (Step 4): What was created, reviewed, or commented on
- **Monday activity** (Step 5, optional): What needs your attention from the team
- **Working tree** (Step 3): Uncommitted changes

Cross-reference to determine:
- What conversation topics resulted in actual commits?
- What was discussed but reverted or abandoned?
- What work is in progress (uncommitted changes, open branches)?
- What action items or follow-ups were identified but not started?
- What PRs need your attention (comments to address, reviews requested)?
- What Monday items need your response?

## Step 7: Write the daily note

Save to: `<resolved vault path>/YYYY-MM-DD.md`

Derive the repo URL from `gh repo view --json url -q .url` for constructing links.

### Frontmatter

Every recap starts with YAML frontmatter containing tags and counts. Compute the counts after assembling all sections:

```yaml
---
tags:
  - recap
  - {year}
date: {YYYY-MM-DD}
prs_merged: {count of your PRs that were merged}
prs_reviewed: {count of others' PRs you reviewed/commented on}
items_completed: {count of items in Completed section}
items_needs_attention: {count of items in Needs Attention section}
cases_resolved: {count of Monday cases you resolved today, omit this line if Monday MCP is not configured}
---
```

### Format

```markdown
# Recap - {date}

> [!tip] {date} at a glance
> **{N}** items completed | **{N}** PRs merged | **{N}** PRs reviewed | **{N}** items need attention

> [!warning] Needs Attention
> - {Monday comments from others on your items, with [item link](monday-url) — only if Monday MCP is configured}
> - {PR review feedback on your PRs that you need to address}
> - {New Monday items assigned to you — only if Monday MCP is configured}

> [!info] In Progress
> - [ ] {Uncommitted work or open branches, what they're for}
>   - Context: {relevant discussion outcome or approach decided, if applicable}

> [!todo] Not Started
> - [ ] {Action items identified during conversations but not yet begun}
>   - Context: {relevant discussion outcome or approach decided, if applicable}

> [!success]+ Completed
> - {What was done, with linked commit hash and branch. Focus on "what and why"}

> [!example]- PRs
> **Your PRs:**
> - [#123 Title](url) - {state: merged/open/draft}. {Comments received, approvals, changes requested}
>
> **Others' PRs:**
> - [#456 Title](url) - {what you did: reviewed, commented, pushed commits}

> [!quote]- Notes
> - {Decisions that closed off a path and why}
> - {Context or learnings worth remembering}
```

**Callout behavior:**
- `[!warning]` Needs Attention - expanded, visually loud (orange)
- `[!info]` In Progress - expanded (blue)
- `[!todo]` Not Started - expanded (blue)
- `[!success]+` Completed - expanded by default (green) - the main content
- `[!example]-` PRs - collapsed by default (purple) - reference material
- `[!quote]-` Notes - collapsed by default (grey) - reference material

### Guidelines

- **No duplication across sections.** Each topic appears in exactly one section. If work is in Completed, don't repeat it in Not Started or In Progress.
- **Inline context under checklist items** rather than having a separate Discussed section. If a discussion informed an unfinished task, add a `Context:` line under that task's checkbox.
- Be concise. Each bullet should be 1-2 sentences max. Completed items can be longer for archive/search value.
- Focus on substance: what was built, why, what decisions were made.
- Skip meta-conversations (tool configuration, "are you stuck?" exchanges) unless they led to meaningful outcomes.
- **Skip callout sections with no items.** Do not render empty callouts.
- Group related items together rather than listing every micro-task.
- The summary callout counts should match the frontmatter counts.
- **Use hyperlinks wherever possible.** The recap is an Obsidian note, so markdown links are clickable:
  - PRs: `[#123 Title](https://github.com/owner/repo/pull/123)`
  - Commits: `[abc1234](https://github.com/owner/repo/commit/abc1234)`
  - Files: use `cursor://file/` URIs so they open in the IDE, e.g. `[use-file-processor.tsx](cursor://file/absolute/path/to/use-file-processor.tsx)` - use the absolute path (swap the URI scheme to match your editor if you don't use Cursor)
  - Monday items: use real Monday.com URLs (only if Monday MCP is configured)

## Step 8: Brag evaluation

After writing the daily note, invoke the `/brag` skill with the recap date as the argument. The brag skill will evaluate the recap content already in your conversation context (it will not re-read the file) and append any brag-worthy items to the yearly brag document.

Example: if the recap date is `2026-04-07`, invoke `/brag 2026-04-07`.
