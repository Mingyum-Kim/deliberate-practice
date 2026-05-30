---
name: status
description: Show a dashboard of all deliberate practice training programs — name, priority, status, today's score, and streak. Use when the user wants to see their active programs at a glance.
---

Display a status dashboard for all training programs. Follow these steps exactly.

## Step 1: Read all program.yaml files

Use bash to find all program files:
```bash
find programs -name "program.yaml" | sort
```

Read each file.

## Step 2: For each program, gather dashboard data

For each program, determine:

- **Name**: from `training_program.name`
- **Priority**: from `training_program.priority`
- **Status**: from `training_program.status`
- **Today's score**: Look for a daily reflection file at `programs/{slug}/reflections/daily/{today}.md`. If it exists, extract the score line (look for `Score:` or `**Score**`). If not found, show `—`.
- **Streak**: Count how many consecutive calendar days (ending today or yesterday) have a reflection file in `programs/{slug}/reflections/daily/`. A streak of 0 means no reflection filed today or yesterday.

Today's date format: YYYY-MM-DD (e.g. 2026-05-24).

## Step 3: Sort by priority (ascending — 1 is highest)

## Step 4: Display the table

Format as a markdown table:

```
| Priority | Program | Status | Today | Streak |
|----------|---------|--------|-------|--------|
| 1 | Ticket-Based Product Immersion | active | 4/5 | 🔥 7 days |
| 2 | English Writing | paused | — | 0 days |
```

Use 🔥 prefix on streak when streak ≥ 3.

## Step 5: Add a one-line summary

After the table, show:
- Number of active programs
- Whether today's reflection has been filed for each active program (yes/no)
- If no reflection filed today: "Run `/reflect` to log today's session."

## Notes

- If `programs/` directory is empty or does not exist, say: "No training programs found. Run `/add-program` to create your first one."
- Do not show deleted programs.
- Show paused and completed programs in the table with their status — just don't prompt for reflection on them.
