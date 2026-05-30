---
name: add-program
description: Add a new deliberate practice training program. Asks the user 7 lightweight questions one at a time, then generates a program.yaml with a confirmation step before writing.
---

You are setting up a new deliberate practice training program.

Your goal is to collect only the minimum necessary information to create a usable training module.

Do not ask redundant questions. Do not make the setup feel heavy. Ask one question at a time.

Important:
- Do not separately ask "motivation" and "specific problem" unless the user's answer is unclear.
- Combine them into one "purpose" question.
- Keep daily actions small and concrete.
- After the user gives a long answer, compress it into a short structured training module.
- At the end, ask the user to confirm or edit the summary.

---

## Step 1. Name

Ask: "What is the name of this training program?"

---

## Step 2. Priority

Ask: "What priority is this program? (1 = highest priority among active programs)"

---

## Step 3. Duration

Ask: "How long should this training stay active? For example: 2 weeks, 4 weeks, 8 weeks, or until a specific date."

---

## Step 4. Purpose

Ask: "What is the purpose of this training?

Please include:
- why this matters to you right now,
- what problem you want to solve,
- what behavior you want to change."

Do not ask separate motivation/problem questions unless the answer is too vague.

---

## Step 5. Daily Actions

Ask: "What are the 1–3 daily actions you want to practice? Please keep them small enough to do on a normal workday."

If the user gives a long answer, extract 1–3 core daily actions.

---

## Step 6. Daily Score

Ask: "How should we measure daily progress?

You can either:
1. define your own scoring system, or
2. let me suggest a simple checklist based on your daily actions."

If the user chooses option 2, create a 3–5 point checklist based on their daily actions.

---

## Step 7. Reflection Schedule

Ask: "When should daily and weekly reflections happen?

Please provide:
- daily reflection time,
- weekly reflection day/time."

---

## After collecting answers

Output the training module in this structure and ask: "Does this look right? Edit anything you want, or say 'confirm' to save it."

```yaml
training_program:
  name:
  slug:
  priority:
  status: active
  start_date:
  end_date:
  duration:
  purpose:
  daily_actions:
    -
  daily_score:
    scale:
    criteria:
      -
  daily_reflection:
    questions:
      -
  weekly_reflection:
    focus:
      - actual experience
      - friction
      - effectiveness
      - system adjustment
  reminders:
    daily:
      time:
      message:
    weekly:
      day_time:
      message:
  review_rules:
    adjust_if:
    pause_if:
    complete_if:
```

Wait for the user to confirm or request edits.

---

## After confirmation

1. Infer a slug from the name (lowercase, hyphenated). Example: "Ticket-Based Product Immersion Training" → `ticket-based-product-immersion`.

2. Run:
```bash
mkdir -p programs/{slug}/reflections/daily
mkdir -p programs/{slug}/reflections/weekly
```

3. Write the confirmed YAML to `programs/{slug}/program.yaml`.

4. Confirm:
"Saved to `programs/{slug}/program.yaml`.
- Edit any field directly in the file.
- Run `/status` to see it in the dashboard.
- Run `/reflect` to start your first daily reflection."
