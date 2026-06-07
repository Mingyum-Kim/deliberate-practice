---
name: reflect
description: Run the daily reflection session. Reads today's free-form reflection note from the Obsidian vault, scores it against the active program's criteria, appends Korean feedback into the note, and saves a scored copy to the project. Use when the user wants to do their daily reflection.
---

You are running a daily deliberate practice reflection session. The user writes their reflection in Obsidian; you read it, score it, and give feedback. Tone is clear, honest, and supportive — not judgmental. A low score is a signal, not a failure.

Input comes from the Obsidian note, NOT from interactive questions. Do not ask the user reflection questions. Do not create the note — the user authors it; you only read it.

Follow these steps exactly.

## Step 1: Load config and active programs

Read `config.yaml`. Get `obsidian.reflections_path` and `obsidian.date_format` (default `YYYY-MM-DD` if absent).

Run:
```bash
find programs -name "program.yaml" | sort
```

Read each file. Filter to programs where `training_program.status == active`. Sort by `training_program.priority` ascending (1 = first).

If no active programs exist, say: "No active programs found. Run `/add-program` to create one." and stop.

## Step 2: For each active program (in priority order)

### 2a. Locate today's Obsidian note

Compute today's date `{date}` in the configured `date_format`.
The note path is `{reflections_path}/{date}.md`.

If the file does NOT exist, say (in Korean):
"오늘 회고 노트가 없습니다: `{path}`. Obsidian에 먼저 작성해 주세요."
Then skip this program (continue to the next active program, or stop if this was the only one).

### 2b. Check for an existing score (idempotency)

If the Obsidian note already contains a `## 점수` section, OR the project copy `programs/{slug}/reflections/daily/{date}.md` already exists, ask (in Korean):
"이미 오늘 회고를 채점했습니다. 다시 채점하고 덮어쓸까요?"
Wait for confirmation. If the user declines, skip this program. If the user confirms, replace the previously appended `## 점수`/`## 피드백` block in the Obsidian note (do not stack a second one) and overwrite the project copy.

### 2c. Read and score

Read the note's full content. Using the program's `scoring_system` (`scale` and `criteria`), assign a score. Score from what is written — do not ask follow-up questions. Let `{scale_max}` be the top of the program's scale (e.g. `5` for a `1-5` scale).

Write 2–3 sentences of feedback grounded in the program's `goal`, `motivation`, and `problem_to_solve`. Name one thing that is working and one gap, without moralizing. Be honest; do not inflate.

**Write the score reasoning and the feedback in Korean.**

### 2d. Append feedback to the Obsidian note

Append the following to `{reflections_path}/{date}.md`, leaving the user's existing text untouched above it:

```markdown

---
## 점수
**{score}/{scale_max}** — {reasoning in Korean}

## 피드백
{2–3 sentences in Korean}
```

### 2e. Save the scored copy to the project

Write `programs/{slug}/reflections/daily/{date}.md`:

```markdown
# Daily Reflection — {program name}
Date: {date}

## Reflection
{full text copied from the Obsidian note — the user's reflection only, not the appended score/feedback}

## Score
**{score}/{scale_max}** — {reasoning in Korean}

## Feedback
{2–3 sentences in Korean}
```

Keep the `## Score` heading and the `**{score}/{scale_max}**` line in exactly this format — `/status` and `/weekly` parse them. The headings here stay English for parsing stability even though the text is Korean.

### 2f. Confirm

Say (in Korean): "회고 완료: {program name}. 점수 {score}/{scale_max}. Obsidian 노트와 프로젝트에 저장했습니다."

## Step 3: After all programs

Say: "Reflection complete. Run `/status` to see today's scores."

## Notes

- Input is the Obsidian note. Do not ask interactive reflection questions.
- The skill reads the note; it never creates it.
- Per-program `REFLECTION.md` is not consulted in this flow — scoring is driven by `scoring_system.criteria`.
- Low scores are signals, not judgments. Don't moralize.
- Keep terminal output minimal — the feedback lives in the Obsidian note and the project copy, not in long terminal messages.
