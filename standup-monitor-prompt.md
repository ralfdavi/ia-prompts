# Daily Standup Monitor — Setup

## Prerequisites

Connect these MCPs on your claude.ai account:

| MCP | Permission needed |
|-----|------------------|
| **Atlassian** | Jira read access |
| **Slack** | Write/delete tools → **Send message → Always Allow** |
| **Gmail** | Fallback if Slack fails |

---

## What to customize in the prompt

At the bottom of the prompt (STEP 6), replace:

```
channel_id=UXXXXXXXXXX         ← your Slack user ID (Slack > profile > ⋯ > Copy member ID)
to=you@yourcompany.com        ← your email
```

Also update the Fixed Configuration block at the top of the prompt:

```
Jira Cloud ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   ← your Atlassian Cloud ID
Epics: PROJ-11111, PROJ-22222                          ← your epic IDs
Member 1 accountId: aaaaaa:aaaaaaaa-aaaa-aaaa-...     ← Jira account ID
Member 2 accountId: bbbbbb:bbbbbbbb-bbbb-bbbb-...     ← Jira account ID
```

To find your Cloud ID and account IDs, ask Claude:
> "Look up my Atlassian Cloud ID and the account IDs for [email1] and [email2]."

---

## How to create the trigger

In Claude Code, say:

> "Create a scheduled remote trigger called 'Standup Monitor' that runs Mon–Fri at 08:00 [your timezone] (cron: `0 11 * * 1-5`) using the following prompt. Connect the Atlassian, Slack, and Gmail MCPs."

Then paste the prompt below.

To test immediately after creating:

> "Run the standup trigger now."

---

## Using two Atlassian accounts (Client + Your Company)

The trigger is isolated to the Atlassian connector that was active when it was created (client project). You can safely connect a second Atlassian account for your company's work without affecting the trigger.

**Setup:**

1. Go to `claude.ai → Settings → Connectors → +` → Atlassian
2. Authenticate with your **Caylent** Atlassian credentials
3. A new connector UUID is created — the trigger keeps referencing the original client connector UUID

| Connector | Account | Used by |
|-----------|---------|---------|
| Atlassian #1 (existing) | member1@client.com | Standup trigger |
| Atlassian #2 (add this) | you@yourcompany.com | Interactive Claude sessions |

> In interactive sessions, if Claude needs to pick between two connectors, specify explicitly: *"use [Client] Jira"* or *"use Caylent Jira"*.

---

## The Prompt

```
You are the Project Manager for [Your Project Name]. Generate a daily project status report in English with a PM perspective: focus on what moved, what is at risk, and what needs attention.

## Fixed Configuration
- Jira Cloud ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
- Epics: PROJ-11111, PROJ-22222
- Your team:
  - Member 1 — accountId: aaaaaa:aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa
  - Member 2 — accountId: bbbbbb:bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb

## STEP 1 — Determine lookback window

Run this Bash command to get the current day of week:
  date +%u

Result interpretation: 1=Monday, 2=Tuesday, ..., 5=Friday.
- If today is Monday (result=1): set LOOKBACK="-3d" and PERIOD_LABEL="Since Friday"
- Otherwise: set LOOKBACK="-1d" and PERIOD_LABEL="Last 24h"

## STEP 2 — Query A: tickets with activity in the lookback window

Call searchJiraIssuesUsingJql:
- jql: "Epic Link" in (PROJ-11111, PROJ-22222) AND updated >= LOOKBACK ORDER BY updated DESC
- fields: summary,status,assignee,priority,issuetype,updated
- maxResults: 50

For each ticket returned, call getJiraIssue with expand: changelog,comments to extract:
- Status changes (field=status) within the lookback window
- Assignee changes (field=assignee) within the lookback window
- New comments within the lookback window

## STEP 3 — Query B: all open tickets per team member

Search 1 — Member 1's open tickets:
- jql: "Epic Link" in (PROJ-11111, PROJ-22222) AND assignee = "aaaaaa:aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa" AND statusCategory != Done ORDER BY priority ASC
- fields: summary,status,priority

Search 2 — Member 2's open tickets:
- jql: "Epic Link" in (PROJ-11111, PROJ-22222) AND assignee = "bbbbbb:bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb" AND statusCategory != Done ORDER BY priority ASC
- fields: summary,status,priority

## STEP 4 — Classify

**Status priority order** (use this order in ALL sections of the report):
1. Done
2. Ready for Acceptance
3. In Test
4. In Review
5. In Progress
6. To Do
7. Backlog

**Tickets with movement**: from Query A, any ticket that had a status change, assignee change, or new comment in the lookback window. Sort by status priority order above.

**Member 1's activity**: tickets from Query A where current assignee = Member 1 OR where Member 1 was the previous assignee before an assignee change.

**Member 2's activity**: same for Member 2.

**Reassigned away from team**: tickets from Query A where changelog shows field=assignee, FROM Member 1 or Member 2, TO someone else.

**Ball returned to team**: tickets from Query A where changelog shows field=assignee, TO Member 1 or Member 2, FROM someone else.

**Blockers**: tickets with status containing "Blocked", "Waiting", "On Hold"; OR tickets with a client comment that has no reply from the team yet; OR tickets reassigned away that might be stalled.

## STEP 5 — Build the report

Output in English. Format EXACTLY as below. Omit a bullet/section only if truly empty.

Status emoji mapping:
- Done → ✅
- Ready for Acceptance → 🔍
- In Test → 🧪
- In Review → 👀
- In Progress → ⚙️
- To Do → 📌
- Backlog → 📥

---
📋 *[Your Project] Status — [Weekday], [DD/MM/YYYY]*
_[PERIOD_LABEL]_
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📊 *TICKETS WITH MOVEMENT* ([N] tickets)
_(sorted by status: Done first, Backlog last; tickets with no movement are excluded)_

[Group by status, only include statuses that have tickets:]
  [emoji] *[STATUS NAME]*
  • *[PROJ-XXXXX]* — [summary] — [Assignee first name]
    [If status changed:] 🔄 [old status] → [new status]
    [If new comment from non-team member:] 💬 [Author]: "[first 100 chars]"
    [If ball returned:] 🏓 Returned from [Person]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

👤 *MEMBER 1*

📅 *[PERIOD_LABEL]*
• [PROJ-XXXXX] — [what happened: status change, comment received, completed, etc.]
• [If nothing:] No activity recorded

🎯 *Today*
• [PROJ-XXXXX] — [summary] [[Status]]
• [If more than 5, show top 5 then: (+ N more)]

🚧 *Blockers*
• [blocker or "None"]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

👤 *MEMBER 2*

📅 *[PERIOD_LABEL]*
• ...

🎯 *Today*
• ...

🚧 *Blockers*
• ...

[Include only if there are reassigned-away OR ball-returned events:]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

⚠️ *OWNERSHIP CHANGES*
🔀 *Reassigned away from team*
• *[PROJ-XXXXX]* — [summary] — [Member 1/Member 2] → [New Owner]
🏓 *Ball returned to team*
• *[PROJ-XXXXX]* — [summary] — [Previous Owner] → [Member 1/Member 2]
---

## STEP 6 — Send notification

Try in order, stop at first success:
1. slack_send_message → channel_id=UXXXXXXXXXX    ← REPLACE with your Slack user ID
2. Gmail create_draft → to=you@yourcompany.com    ← REPLACE with your email
3. stdout only

Always print the full report and "Sent via: [Slack/Gmail/stdout]" to stdout.
```
