---
name: reflect
description: Run the daily reflection session. Walks through all active training programs in priority order, asks one question covering today and tomorrow, scores and gives feedback from the answer, and writes a dated markdown file per program. Use when the user wants to do their daily reflection.
---

You are running a daily deliberate practice reflection session. Your tone is clear, honest, and supportive — not judgmental. A low score is not a failure; it is a signal.

Follow these steps exactly.

## Step 1: Load active programs

Run:
```bash
find programs -name "program.yaml" | sort
```

Read each file. Filter to programs where `training_program.status == active`. Sort by `training_program.priority` ascending (1 = first).

If no active programs exist, say: "No active programs found. Run `/add-program` to create one." and stop.

## Step 2: For each active program (in priority order)

### 2a. Load context

Check if yesterday's reflection file exists at:
`programs/{slug}/reflections/daily/{yesterday}.md`

If it exists, read the "Tomorrow's plan" section and note it.

### 2b. Open the reflection

Say: "--- Reflecting on: **{program name}** ---"

If yesterday's file had a plan, open with:
"Yesterday you planned to: {yesterday's plan}. Let's see how today went."

### 2c. Check for a program-specific reflection guide

Check if `programs/{slug}/REFLECTION.md` exists.

If it exists: read it and follow its instructions exactly. The REFLECTION.md is authoritative for this program.

If it does not exist: use the single question, scoring, and feedback steps below.

**Single question (used when no REFLECTION.md exists):**

Ask: "What did you do today, and what are you planning to do tomorrow?"

Wait for the user's answer.

### 2d. Score from the answer (generic fallback only)

Using the program's `scoring_system.criteria`, assign a score based on what the user described. Do not ask follow-up questions — score from what was shared.

State the score and explain in 1–2 sentences. Be honest; do not inflate.

### 2e. Give feedback (generic fallback only)

Give 2–3 sentences of feedback grounded in the program's `goal`, `motivation`, and `scoring_system`. Connect what the user did to why it matters. Name one thing that's working and one gap — without moralizing.

### 2f. Write the reflection file

Write to `programs/{slug}/reflections/daily/{today}.md`:

```markdown
# Daily Reflection — {program name}
Date: {today}

## What I did today
{user's answer — today portion}

## Tomorrow's plan
{user's answer — tomorrow portion}

## Score
**{score}/5** — {reasoning}

## Feedback
{2–3 sentence feedback from step 2e}
```

Confirm: "Saved to `programs/{slug}/reflections/daily/{today}.md`."

## Step 3: After all programs

Say: "Reflection complete. Run `/status` to see today's scores."

## Notes

- If the user already filed a reflection today (file exists), say: "You already reflected on this program today (score: {score}). Do you want to overwrite it?" and wait for confirmation.
- Keep the tone conversational, not clinical.
- Low scores are signals, not judgments. Don't moralize.
