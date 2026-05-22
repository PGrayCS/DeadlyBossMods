---
name: vibe-code
description: Structured vibe coding workflow. Plan first, review hard, then let the vibes roll. Use when starting any new coding task to enforce plan mode, test case generation, git snapshots, and optional subagent review before writing any code.
---

# Vibe Code Skill

A structured workflow for vibe coding: plan first, review hard, then let the vibes roll.

## Instructions

When this skill is invoked with a task description, follow these steps in order:

### Step 1: Enter Plan Mode

Switch to plan mode immediately. Do NOT write any code yet.

### Step 2: Build the Plan

Create a detailed implementation plan that includes:

- **What** is changing and **why**
- **Files** that will be created or modified, with a one-line description of each change
- **Test cases** listed in plain English (not code) — describe what each test validates, e.g.:
  - "it returns the sum of two positive integers"
  - "it returns an error when no arguments are provided"
  - "it handles negative numbers correctly"
- **Risks or unknowns** — anything that could go wrong or needs a decision
- **Database changes** (if any) — flag these prominently and remind the user to back up first

Keep each section short and scannable. If the plan is longer than ~20 bullet points total, it is too big — split it and say so explicitly.

### Step 3: Wait for Plan Approval

Present the plan and STOP. Do not proceed until the user explicitly approves it. Encourage them to:
- Ask about anything unclear ("What does X mean?")
- Push back on scope
- Request the plan be broken into smaller chunks if it feels overwhelming

### Step 4: Complexity Check

If the change touches more than 3 files, modifies core logic, or involves security-sensitive code, automatically spin up three subagents in parallel before writing any code:

1. **Critical plan reviewer** — looks for logical gaps, missing edge cases, wrong assumptions
2. **Security reviewer** — checks for injection, auth issues, data exposure, OWASP top 10
3. **Testing auditor** — verifies the test cases cover the important paths and aren't just happy-path

Surface their findings to the user as a short summary. Pause again if anything significant is flagged.

### Step 5: Pre-flight Git Snapshot

Before writing any code, run:

```
git add -A && git commit -m "snapshot: before <short description of task>"
```

This saves the current state so the user can always roll back. Confirm the commit succeeded.

### Step 6: Auto Mode — Implement

Now write the code. Follow the approved plan exactly. Do not add unrequested features, refactors, or abstractions.

Implement the test cases described in the plan as actual tests.

### Step 7: Post-implementation Git Commit

After all code and tests are written and passing, commit:

```
git add -A && git commit -m "<concise description of what was built>"
```

### Step 8: Report

Summarize:
- What was built
- Test results (pass/fail count)
- Any deviations from the plan and why
- Next suggested step if the task was part of something larger

---

## Key Rules

- **Plan mode is sacred.** Never write code before the plan is approved.
- **Read the plan out loud in your head.** If a section doesn't make sense, say so.
- **Small plans only.** If it's too big to hold in your head, break it down.
- **Tests are not optional.** Every plan must include test cases.
- **Git before and after.** Always snapshot before, always commit after.
- **DB changes are scary.** Always flag them and remind the user to have a backup.
