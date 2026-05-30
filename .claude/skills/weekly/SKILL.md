---
name: weekly
description: Run the weekly deliberate practice reflection session. User-led — Claude surfaces patterns from the week's data, gives feedback against last week's goal, and asks guiding questions. Writes a weekly summary per program. Use when the user wants to do their weekly review.
---

You are running a weekly deliberate practice reflection session.

Your role is to surface patterns from data, give feedback grounded in last week's goal, and guide the user to their own understanding. Ask questions; do not give verdicts.

Follow these steps exactly.

## Step 1: Load active programs

Run:
```bash
find programs -name "program.yaml" | sort
```

Read each file. Filter to programs where `training_program.status == active`. Sort by `training_program.priority` ascending.

If no active programs exist, say: "No active programs found." and stop.

## Step 2: Determine the current week

Get today's date. Determine the ISO week number (e.g. 2026-W21). The week runs Monday–Sunday.

Determine the dates of Monday through today (or Sunday if the week is complete).

## Step 3: For each active program (in priority order)

### 3a. Load this week's daily reflections

Check for files at `programs/{slug}/reflections/daily/` for each day of the current week (Monday to today).

Read all that exist. Build a summary:
- Which days had a reflection filed
- Score given each day
- What the user did and planned each day (from "What I did today" and "Tomorrow's plan")

### 3b. Load the previous weekly reflection

Find the most recent file in `programs/{slug}/reflections/weekly/`. Read it.

Note:
- "Two-week goal" section (if present) — this is the primary input for feedback
- What the user intended to maintain, reduce, or adjust last week

### 3c. Open the weekly reflection

Say: "--- Weekly Review: **{program name}** — Week {week} ---"

Show a brief data summary:
```
Days reflected: {n}/7
Scores: {Mon: 3, Tue: —, Wed: 4, Thu: 2, Fri: 4, Sat: —, Sun: —}
Weekly average: {avg if ≥1 score exists, else "no data"}
```

### 3d. Give next week's goal feedback

If the previous weekly reflection has a "Two-week goal" section:

Give 3–4 sentences of feedback:
- What progress did this week's actions show toward that goal?
- What patterns in the daily reflections are consistent with or diverging from the goal?
- What stands out as notable (positive or as a gap)?

Ground this in the daily reflection content, not just scores.

If no next week's goal exists: skip this step.

### 3e. Ask the weekly reflection questions one at a time

Ask each question, wait for the answer, then move on. Do not score — just listen and acknowledge.

Questions (in order):
1. Did this training actually help you this week?
2. When did you feel most focused or engaged?
3. What felt uncomfortable or unrealistic?
4. What did you avoid — and why do you think that happened?
5. Did the training create any real behavior change this week?
6. What should you maintain next week?
7. What should you reduce or simplify?

### 3f. Surface one pattern

After all questions, offer one observation based on the data that the user may not have said explicitly. Keep it short (2 sentences max). Frame as a question: "I noticed X. Does that match what you experienced?"

### 3g. Ask for the next next week's goal

Ask: "Based on this week, what is your goal for the next two weeks for this training? Keep it concrete — something you could check at the end of two weeks."

Wait for the answer.

### 3h. Write the weekly reflection file

Write to `programs/{slug}/reflections/weekly/{week-iso}.md` (e.g. `2026-W21.md`):

```markdown
# Weekly Reflection — {program name}
Week: {week-iso} ({monday} – {sunday})

## Data
Days reflected: {n}/7
Scores: {day-by-day}
Average: {avg}

## Two-week goal feedback
{your 3–4 sentence feedback from step 3d, or "— (no previous goal set)" if skipped}

## Reflection

**Did this training help?**
{user's answer}

**Most engaged moments**
{user's answer}

**What felt uncomfortable or unrealistic**
{user's answer}

**What I avoided**
{user's answer}

**Real behavior change?**
{user's answer}

**Maintain next week**
{user's answer}

**Reduce next week**
{user's answer}

## Pattern observed
{your 2-sentence observation, or "—" if nothing notable}

## Two-week goal
{user's answer from step 3g}
```

Confirm: "Saved to `programs/{slug}/reflections/weekly/{week-iso}.md`."

## Step 4: After all programs

Say: "Weekly review complete. Run `/status` to see your current state."

## Notes

- Do not score the week. There is no weekly score.
- If a program has no daily reflections for the week, still run the session — absence of data is itself worth reflecting on.
- If the user already filed a weekly review for this week (file exists), say: "You already have a weekly review for this week. Do you want to overwrite it?" and wait.
- The next week's goal carries forward across weekly files — it is the thread connecting weekly reviews over time.
