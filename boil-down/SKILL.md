---
name: boil-down
description: Boil down current session to notes and update daily log. Use when user says "boil down", "boil down session", or wants to summarize the conversation into session notes.
---

# Boil Down Session Skill

## INSTALLATION

1. Place this file at:
   - **Mac/Linux:** `~/.claude/skills/boil-down/SKILL.md`
   - **Windows:** `C:\Users\<you>\.claude\skills\boil-down\SKILL.md`
2. Set `NOTES_DIR` below to your preferred notes folder.
3. (Optional) Customize the Project Detection Rules table to match your own projects.

---

## CONFIGURATION — Set This First

Before using this skill, open this file and set your notes directory. This is the **only** path you need to configure. All output files will be created inside it.

```
NOTES_DIR = <your absolute path here>
```

**Examples:**
- Windows: `C:\Users\alice\notes`
- Mac/Linux: `/Users/alice/notes`

**Folder structure created automatically under NOTES_DIR:**
```
NOTES_DIR/
  sessions/
    YYYY-MM/
      YYYY-MM-DD/
        session-N.md
        handoff-session-N.md
  daily/
    thoughts-YYYY-MM-DD.md
  backlog.md
```

Create missing folders on first use.

---

## IMPORTANT REMINDERS

1. **OS path separator** — use backslash on Windows, forward slash on Mac/Linux
2. **Session summary + file links together** — keep them in one block, never split
3. **Merge repeated edits to same file** — if the same file was modified multiple times, combine into one record; separate changes with `·`; use a time range `HH:MM–HH:MM`

---

## Trigger Conditions

Activate when user says any of:
- "boil down"
- "boil down session"
- "summarize this session"
- "create session notes"

---

## Execution Steps

### Step 1 — Detect Session Number

Read:
```
NOTES_DIR/daily/thoughts-YYYY-MM-DD.md
```

Scan for lines matching `**Session N**:` format. New session number = highest N + 1.
If the daily file does not exist yet → this is Session 1.

---

### Step 2 — Create Session Note

**Path:**
```
NOTES_DIR/sessions/YYYY-MM/YYYY-MM-DD/session-N.md
```

**Format:**

```markdown
project: [TAG]

# Session N - YYYY-MM-DD

## Summary

1. Task description 1 (key metric or result)
2. Task description 2 (outcome)
3. Task description 3 (decision made)

---

### 1. Task Name

**Problem**: What was asked or what issue was found

**Root Cause** (if applicable): Why it happened

**Solution**:
- Step 1: specific action taken
- Step 2: specific action taken

**Conclusion**: Key finding or takeaway

---

## Files Created / Modified

- `NOTES_DIR/relative/path/file.md` - what it is | HH:MM | prompt: user instruction
- `NOTES_DIR/relative/path/file2.py` - what it is | HH:MM | prompt: same as above
```

**Rules:**
- `project:` must be **line 1**, format exactly `project: Tag` (no quotes)
- Summary: 1–8 numbered items, one sentence each, key numbers in parentheses
- Action verbs: fixed, created, resolved, updated, generated
- Each detailed section uses at least 3 of the 5 elements (Problem / Root Cause / Solution / Data / Conclusion)
- **Max 100 lines total**
- File list always goes in the session note, never in the daily log

**File list merge rule:**
Same file edited multiple times → one line, changes separated by `·`, time as range:
```
- `path/file.md` - created draft · added section 2 · fixed formatting | 14:00–14:20 | prompt: write the analysis
```

---

### Step 3 — Update Daily Log

**Path:**
```
NOTES_DIR/daily/thoughts-YYYY-MM-DD.md
```

If it doesn't exist, create it:
```markdown
# Daily Log - YYYY-MM-DD

## Sessions

```

Append this block to the `## Sessions` section:

```markdown
**Session N**: one-line summary of what was done (~20 words)
📋 Session Note: `NOTES_DIR/sessions/YYYY-MM/YYYY-MM-DD/session-N.md`
🎯 Main Output: `path/to/main/deliverable.ext` ← omit if no file deliverable this session
🤝 Hand-off: `NOTES_DIR/sessions/YYYY-MM/YYYY-MM-DD/handoff-session-N.md`
- bullet 1: specific operation with filenames or numbers
- bullet 2: specific operation with filenames or numbers
- bullet 3–6: (4–6 bullets total)
```

**CRITICAL — every session block has exactly these elements:**
1. Title line: `**Session N**: ~20-word summary`
2. 📋 link to session note
3. 🎯 main output (skip if none)
4. 🤝 link to hand-off file
5. 4–6 bullet points with concrete details

The daily log contains only this brief block. Detailed content and the file list live in the session note.

---

### Step 4 — Create Hand-off File

**Path:**
```
NOTES_DIR/sessions/YYYY-MM/YYYY-MM-DD/handoff-session-N.md
```

**Audience:** The next AI agent picking up this work. Optimized for fast onboarding.

**Format (≤30 lines):**

```markdown
# Hand-off: Session N — YYYY-MM-DD

## Task Description

1–2 sentences: what was done and why.

## Main Output

- `path/to/output.ext` — what it is and its current state

## Key Files

- `path/to/file1.md` — purpose
- `path/to/file2.py` — purpose
(up to 5 files)

## Context & Notes

- Key constraint or decision
- Known issue or edge case

## Unfinished / Next Steps

- [ ] Next task 1
- [ ] Next task 2
(write "All tasks completed this session." if nothing pending)
```

---

### Step 5 — Sync backlog.md

**Path:**
```
NOTES_DIR/backlog.md
```

If `backlog.md` doesn't exist, create it with a section for the current project and a `## Archive` section at the bottom.

**Logic:**
1. Read `## Unfinished / Next Steps` from the hand-off file
2. Extract all `- [ ]` items
3. If that section says "All tasks completed this session." → skip, no changes to backlog
4. Use the `project:` tag to find the matching section in backlog.md; create a new section if needed
5. For each `- [ ]` item: skip if a semantically identical item already exists; otherwise append it
6. If this session clearly completed a backlog item → move it to `## Archive` with ` ✓` suffix (plain text, no checkbox)
7. Update the `> Last updated: YYYY-MM-DD` line at the top
8. If a new section was created, add it to the table of contents

**Format rules:**
- New items: `- [ ] Task description`
- Do not delete `[~]` in-progress items
- Archive: `- Task description ✓`

---

## Project Detection Rules

First match wins. Use the `project:` tag that best fits the session content.

| Tag | Match when the session involves… |
|:---|:---|
| `Research` | data analysis, statistics, papers, clinical study, experiment |
| `WebApp` | web app, React, Next.js, API, backend, frontend |
| `MobileApp` | iOS, Android, React Native, Expo, mobile |
| `Writing` | documents, essays, reports, editing, drafting |
| `DevOps` | deployment, CI/CD, Docker, servers, infrastructure |
| `Skills` | skill files, workflow tools, AI configuration |
| `System` | notes system, daily tracker, workflow setup |
| `Other` | fallback — use when nothing above fits clearly |

> **Customize this table** to match your own projects. Add rows for your specific domains (e.g., `Climbing`, `Thesis`, `ClientName`). The detection logic — first match wins, fallback to `Other` — stays the same.

**Do not invent new tags on the fly.** Use `Other` and note it in the session file if a new category seems warranted, then update this table manually.

---

## Quality Checklist

Before telling the user "done", verify:
- [ ] Session number correct (detected from daily log)
- [ ] `project:` is line 1 of session file, format: `project: Tag`
- [ ] Summary has 1–8 numbered items
- [ ] Hand-off file is ≤30 lines
- [ ] Daily log has the 5-element session block
- [ ] File list is in session note, NOT in daily log
- [ ] backlog.md updated (items added / completed items archived / date updated)
- [ ] All paths use correct separator for user's OS
