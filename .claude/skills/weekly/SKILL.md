---
name: weekly
description: Run the weekly deliberate practice reflection. Reads this week's weekly note from the Obsidian vault, summarizes the week from daily scores, gives Korean goal feedback and one pattern (no weekly score), appends it into the note, and saves a project copy that carries the two-week goal forward. Use when the user wants to do their weekly review.
---

You are running a weekly deliberate practice reflection session. The user writes their weekly review in Obsidian; you read it, summarize the week, and give feedback. Surface patterns and ground feedback in last week's goal — do not give verdicts, and do not score the week.

Input comes from the Obsidian weekly note, NOT from interactive questions. Do not ask the user the weekly questions. Do not create the note — the user authors it from `_template.md`.

Follow these steps exactly.

## Step 1: Load config and active programs

Read `config.yaml`. Get `obsidian.weekly_path` and `obsidian.date_format` (default `YYYY-MM-DD`).

Run:
```bash
find programs -name "program.yaml" | sort
```

Read each file. Filter to programs where `training_program.status == active`. Sort by `training_program.priority` ascending.

If no active programs exist, say: "No active programs found." and stop.

## Step 2: Determine the current week

Get today's date. Determine the ISO week (e.g. `2026-W24`) and that week's Monday date (`{monday}`) and Sunday date (`{sunday}`). The week runs Monday–Sunday.

## Step 3: For each active program (in priority order)

### 3a. Locate this week's Obsidian note

The note path is `{weekly_path}/{monday}.md`.

If the file does NOT exist, say (in Korean):
"이번 주 주간 회고 노트가 없습니다: `{path}`. Obsidian에서 `_template.md`로 먼저 작성해 주세요."
Then skip this program (continue to the next, or stop if this was the only one).

### 3b. Check for an existing review (idempotency)

If the Obsidian note already contains a `## 주간 피드백` section, OR the project copy `programs/{slug}/reflections/weekly/{week-iso}.md` already exists, ask (in Korean):
"이미 이번 주 주간 회고를 작성했습니다. 다시 작성하고 덮어쓸까요?"
Wait for confirmation. If declined, skip. If confirmed, replace the previously appended block in the note (do not stack) and overwrite the project copy.

### 3c. Load this week's daily data

Read `programs/{slug}/reflections/daily/{date}.md` for each day Monday→today. Build:
- Days reflected: {n}/7
- Day-by-day scores (use `—` for missing days), parsed from the `**{score}/{scale_max}**` line
- Average of the scores that exist (or "no data")

### 3d. Load the previous two-week goal

Find the most recent file in `programs/{slug}/reflections/weekly/` (excluding the current week). Read its `## Two-week goal` section. If none exists, there is no prior goal.

### 3e. Read the weekly note and produce feedback (Korean)

Read the full content of the Obsidian weekly note.

- **Goal feedback:** if a prior two-week goal exists, write 3–4 sentences on progress toward it, grounded in this week's daily content and the user's note — what is consistent or diverging, what stands out. If no prior goal, skip.
- **Pattern:** one short (≤2 sentence) observation from the data the user did not state explicitly.

Do NOT assign a weekly score.

### 3f. Append feedback to the Obsidian note

Append to `{weekly_path}/{monday}.md`, leaving the user's text intact above:

```markdown

---
## 데이터 요약
반영 일수: {n}/7 · 점수: {day-by-day} · 평균: {avg}

## 주간 피드백
{3–4 sentence goal feedback in Korean, or "— (이전 목표 없음)"}

## 발견한 패턴
{one observation in Korean, or "—"}
```

### 3g. Save the project copy

Write `programs/{slug}/reflections/weekly/{week-iso}.md`:

```markdown
# Weekly Reflection — {program name}
Week: {week-iso} ({monday} – {sunday})

## Data
Days reflected: {n}/7
Scores: {day-by-day}
Average: {avg}

## Two-week goal feedback
{3–4 sentence feedback in Korean, or "— (no previous goal set)"}

## Reflection
{full text copied from the Obsidian weekly note — the user's answers only, not the appended feedback}

## Pattern observed
{observation in Korean, or "—"}

## Two-week goal
{copied from the note's "Next two-week goal" section}
```

Keep the `## Two-week goal` heading exactly — the next weekly run reads it for carry-forward. Headings stay English; text is Korean.

### 3h. Confirm

Say (in Korean): "주간 회고 완료: {program name}. Obsidian 노트와 프로젝트에 저장했습니다."

## Step 4: After all programs

Say: "Weekly review complete. Run `/status` to see your current state."

## Notes

- Input is the Obsidian weekly note. Do not ask interactive questions.
- Do not score the week. There is no weekly score.
- The next two-week goal carries forward via the project `## Two-week goal` section — it is the thread connecting weekly reviews.
- The skill reads the note; it never creates it.
- Keep terminal output minimal.
