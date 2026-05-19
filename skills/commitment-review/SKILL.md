---
name: commitment-review
description: "Run a daily commitment review workflow. Use this skill whenever the user says 'start my commitment review', 'run my commitment review', 'check my tasks from today', 'what did I commit to today', 'let's review the commitments I made today', 'let's review tasks from yesterday', 'let's check what commitments I made on [date]', or any similar phrase indicating they want to review recent messages for tasks and commitments. This skill orchestrates a multi-source search across configured sources (Slack, Outlook, and/or Zoom), identifies tasks and commitments, and manages creation of confirmed tasks in the user's Asana project. Always use this skill for the full end-to-end workflow rather than attempting it ad hoc."
---

# Commitment Review Skill

This skill runs a daily commitment review: searching recent messages and meetings
for tasks and commitments, reviewing them together with the user, and creating
confirmed new tasks in their Asana project.

---

## Setup

Before using this skill for the first time, fill in the Configuration block below.
The skill will not work correctly until the required fields are populated.

---

## Configuration

Update this section as needed without modifying the rest of the skill.

```
# --- REQUIRED ---

USER_NAME: [Your name as it appears in Slack and meeting transcripts]

ASANA_PROJECT: [Name of your Asana project or board where tasks should be created]
  # This should be a project you own or have write access to.
  # Example: "My Kanban", "Personal Backlog", "Sprint Board"

# --- SOURCES ---
# Set each source to "enabled" or "disabled".
# You must have the corresponding integration connected in Claude.ai for it to work.

SOURCES:
  Slack: enabled
  Outlook: enabled
  Zoom: enabled

# --- SLACK (only needed if Slack is enabled) ---

SLACK_CHANNELS:
  # List the Slack channels you want searched. Add one per line with a leading dash.
  # Example:
  #   - general
  #   - my-team
  #   - project-alpha
  - [Add channels here]

SLACK_DMS: all (1:1 and group DMs)

# --- LOOKBACK WINDOW ---
# The time window is determined by how you invoke the review.
# If no mode is indicated, the skill defaults to "Today" and confirms before searching.

LOOKBACK_WINDOW:
  Mode: Today
    Trigger phrases: "commitments I made today", "tasks from today", "review today", etc.
    Window: Current date, 6:00 AM to 8:00 PM (user's local time).
    If the current time is before 8:00 PM, use the current time as the end of
    the window instead of 8:00 PM.

  Mode: Yesterday
    Trigger phrases: "commitments I made yesterday", "tasks from yesterday", "review yesterday", etc.
    Window: Previous calendar date, 6:00 AM to 8:00 PM (user's local time).

  Mode: Specific Date
    Trigger phrases: "commitments I made on [date]", "tasks from [date]", "review [date]", etc.
    Window: The named date, 6:00 AM to 8:00 PM (user's local time).
    Parse the date from the user's message. If the date is ambiguous or
    unparseable, ask for clarification before searching.

# --- VIPS ---
# Messages and mentions from VIPs are always included for review, even if they
# contain no clear action language. Mark them with a "(VIP)" label in the review list.

VIPS:
  - Mike Derezin (CEO)
  - Denis Khazan (CTO)
  - Scott Balwinski (COO)
  - Matt Harsh-Strong (CFO)
  - Celia Stokes (Chief Product Officer)
  - Jonathan Lister (Chief Growth Officer)
  - Thomas Yamamoto (General Counsel)
  - Matt Merry (Chief of Staff)
  - May Reid (Chief People Officer)
  - Nicole Russell (Chief Academic Officer)
  - Breeyn Mack (Senior Vice President of Education)
  - Jonah Stuart (Chief Policy & Government Relations Officer)
  - Kevin Roden (Vice President, Public Sales)
  - [Add additional VIPs here — use name and title]

# --- FILTERS ---

FILTERS:
  - Ignore all messages or notifications from Happyfox
  - Ignore tasks originating from any Asana project other than ASANA_PROJECT
  - Do not include tasks the user has assigned to others; only tasks assigned to
    the user or commitments the user has made themselves
```

---

## Startup Check

At the start of every run:

1. Ask the user: "What's your name, and which Asana project should I create tasks in?"
2. Use their answer to set USER_NAME and ASANA_PROJECT for this session.
   - If USER_NAME is already filled in the Configuration block above, skip asking for the name.
   - If ASANA_PROJECT is already filled in, skip asking for the project.
3. If either value is still missing after asking, do not proceed until it is provided.

---

## Workflow Overview

1. [Phase 1: Gather](#phase-1-gather) — Search enabled sources in parallel
2. [Phase 2: Identify](#phase-2-identify) — Extract tasks and commitments; present numbered list for review
3. [Phase 3: Confirm](#phase-3-confirm) — User selects which items are real tasks
4. [Phase 4: Duplicate Check](#phase-4-duplicate-check) — Search the Asana project for potential duplicates
5. [Phase 5: Create](#phase-5-create) — Create confirmed non-duplicates immediately; review duplicates together

---

## Phase 1: Gather

Search only the sources listed as "enabled" in SOURCES above. Treat each as a
parallel workstream and consolidate results before proceeding. Skip any source
marked "disabled" without comment.

### Slack
*(Skip if Slack is disabled)*
- Search all channels listed in SLACK_CHANNELS above
- Search all DMs (1:1 and group) where the user is a participant. DM search
  requires two separate queries to capture both sides of conversations:
  one for messages sent TO the user (`to:<@user ID>`), and one for messages
  sent FROM the user (`from:<@user ID>`). Both queries are needed to catch
  assignments directed at the user and commitments the user has made.
- Lookback: the window defined by the active LOOKBACK_WINDOW mode (see Configuration)
- Look for: direct assignments ("[Name], can you..."), action items directed at
  the user, and soft commitments the user has made ("I'll look into...", "I can
  handle that", "Let me check on...")
- Skip any messages from Happyfox
- @-mention notification emails (e.g., from SharePoint or Office 365 alerting
  that someone mentioned the user in a document) are low-signal and should
  generally not be treated as tasks unless the mention contains clear action
  language. However, if the sender is a VIP (see Configuration), always include
  the item for review rather than filtering it out.

### Outlook / Microsoft 365
*(Skip if Outlook is disabled)*
- Search the user's inbox and sent items for the active LOOKBACK_WINDOW
- Look for: emails where the user is asked to do something, or emails where the
  user has committed to an action in a reply
- Skip any emails from Happyfox or Happyfox-generated notifications
- @-mention notification emails (e.g., from SharePoint or Office 365 alerting
  that someone mentioned the user in a document or presentation) are low-signal
  and should generally not be treated as tasks unless the notification body
  contains clear action language. Exception: if the sender or person who
  mentioned the user is a VIP (see Configuration), always include the item for
  review rather than filtering it out.

### Zoom Meeting Transcripts
*(Skip if Zoom is disabled)*
- Search for all Zoom meetings the user hosted or participated in within the
  active LOOKBACK_WINDOW
- Not all meetings will have transcripts available, even if they occurred within
  the lookback window. Transcript availability depends on whether AI Companion
  was enabled for that meeting. This is expected behavior; note any meetings
  where no transcript was available, and continue.
- Review transcripts for: action items assigned to the user by name, or
  statements the user made indicating a commitment ("I'll follow up", "I'll send
  that over", "Let me pull that together")
- If no meetings at all are found in the window, note that and continue

---

## Phase 2: Identify

After gathering, consolidate all findings and apply filters:

- Remove anything originating from Happyfox
- Remove tasks the user assigned to others
- Deduplicate across sources (if the same commitment appears in both a Slack
  message and a follow-up email, treat it as one item)
- For any message or mention from a VIP (see Configuration): always include it
  for review unless you are highly confident it contains no actionable content
  whatsoever. When in doubt, include it. Mark VIP-sourced items with a "(VIP)"
  label in the review list.

Present a numbered list in this format:

```
COMMITMENT REVIEW
Sources searched: [list of enabled sources] ([active window label, e.g. "Today — May 18, 2026, 6:00 AM – 2:30 PM"])
Found: [N] potential tasks and commitments

---

1. Task Name (generated, interpretive)
   Description: Brief explanation of the commitment or assignment
   Source: [Slack / Outlook / Zoom] — [channel name, sender, or meeting title]
   Due Date: [date if mentioned, or blank]
   Type: [Assignment / Soft Commitment]
   VIP: [Yes — Name, Title] or omit if not a VIP

2. ...
```

After presenting the list, say:

> "Please review the list above. Tell me which numbers represent real tasks for
> your Asana project, and I'll proceed from there. You can also say 'all of them'
> or 'none of them'."

---

## Phase 3: Confirm

Wait for the user's response. They will provide a list of item numbers (or "all" /
"none").

Record the confirmed list. Any items not confirmed are discarded.

---

## Phase 4: Duplicate Check

Search the user's Asana project (ASANA_PROJECT) across ALL sections present.

For each confirmed task:
- Search for tasks with similar topics, keywords, or subject matter
- Flag as a potential duplicate if the topic or subject is similar, even if the
  wording differs
- Flag completed tasks as potential duplicates too; note their completed status
  when flagging

Separate confirmed tasks into two groups:

**Group A: No duplicates found** — proceed to create immediately (Phase 5a)

**Group B: Possible duplicates** — hold for review (Phase 5b)

---

## Phase 5: Create and Resolve

### Phase 5a: Create Non-Duplicates

For each task in Group A, create a new task in the user's Asana project
immediately, in the Backlog section (or the first available section if no
Backlog section exists), using this structure:

- **Task Name**: Generated name based on your interpretation of the action item
  (clear, action-oriented, e.g., "Follow up with Sarah on Q3 budget approval")
- **Description**: Include the original source context — what was said or written,
  where it came from, and by whom. Include the due date if one was identified.
- **Section**: Backlog (or first available section)

Confirm each creation inline as you go.

### Phase 5b: Review Possible Duplicates

After non-duplicates are created, present the duplicate candidates for review:

```
POSSIBLE DUPLICATES TO REVIEW

1. New Task: [generated task name]
   Existing Asana Task: [existing task name] — Section: [section] — Status: [open/complete]
   Reason flagged: [brief explanation of similarity]

   Options:
   a) Create as new task anyway
   b) Update the existing task with new context
   c) Skip — not a real task

2. ...
```

Wait for the user's decision on each item. Then act accordingly:
- Option a: Create new task in Backlog (same structure as Phase 5a)
- Option b: Update the existing Asana task's description to include the new
  context; note the update source and date
- Option c: Discard

---

## Completion Summary

After all tasks are resolved, provide a brief summary:

```
REVIEW COMPLETE

Created [N] new tasks in [ASANA_PROJECT] (Backlog)
Updated [N] existing tasks
Skipped [N] items

[List task names created or updated]
```

---

## Error Handling

- If a source returns no results, note it and continue with other sources
- If Zoom transcript search returns no meetings in the window, note it and continue
- If Asana is unreachable during duplicate check, flag all confirmed tasks as
  "duplicate check skipped" and ask the user whether to create them anyway
- If a source returns an auth error, surface it clearly and ask the user to
  reconnect the integration before continuing

---

## Notes

- Do not create tasks in any Asana project other than ASANA_PROJECT
- Task names should be action-oriented and interpreted, not copy-pasted from
  source messages
- Soft commitments are included by default; the user will decline them during
  review if they are not actionable
- This skill supports three lookback modes: Today, Yesterday, and Specific Date.
  Detect the mode from the user's phrasing before searching. Default to Today if
  ambiguous, but confirm before proceeding.
- The skill is designed to be run once per day, typically at the start of the
  workday, but can also be run retroactively for a past date.
