# Design: Obsidian-Sourced Daily & Weekly Reflection

Date: 2026-06-07
Status: Approved (pending spec review)

## Summary

Change both the daily and weekly reflection flows so the user writes their
reflection in an Obsidian note, and the `/reflect` and `/weekly` skills **read**
that note instead of asking interactive questions.

- **Daily:** user writes a free-form note in `<vault>/Reflections/daily/<date>.md`.
  `/reflect` reads it, scores it (1–5) against the active program's criteria,
  appends Korean feedback into the note, and saves a scored copy to the project.
- **Weekly:** user writes a templated note in `<vault>/Reflections/weekly/<Monday>.md`.
  `/weekly` reads it, computes a data summary from the week's daily project
  copies, gives Korean goal-progress feedback + one pattern (no weekly score),
  appends that into the note, and saves a project copy that carries the
  "two-week goal" forward to the next week.

Both keep a project-folder copy so `status` and `weekly` carry-forward keep
working. The terminal stays quiet beyond a short confirmation.

Only **one program is active at a time**. As part of this change, the
English-Based Contribution Publishing program is set to `status: planned` so
only the Ticket-Based Product Immersion program is reflected on.

## Decisions (from brainstorming)

| Topic | Decision |
|-------|----------|
| Active program | Only `ticket-based-product-immersion`. English program → `status: planned`. |
| Vault access | **Direct file read** off disk. No Obsidian MCP, no Local REST API plugin. |
| Vault location | `/Users/mingyumkim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Anna/DeliveryHero/DeliveryHero` (the nested folder containing `.obsidian/`). |
| Daily note location | `<vault>/Reflections/daily/<date>.md`. |
| Daily note format | **Free-form.** User writes with their own headings (currently Korean). |
| Weekly note location | `<vault>/Reflections/weekly/<Monday-date>.md`. Daily and weekly are separate subfolders under `Reflections/`, mirroring the project layout. |
| Weekly note format | **Templated.** A weekly-specific template with English headings + a `## Next two-week goal` section. Template stored at `<vault>/Reflections/weekly/_template.md`. |
| Feedback language | **Korean.** (Headings in project copies stay English for parsing.) |
| Feedback destination | Appended into the same Obsidian note, **and** a copy saved to the project folder. No terminal output beyond a short confirmation. |
| Weekly scoring | **No weekly score** (unchanged rule). Weekly gives goal feedback + one pattern only. |
| Date format | `YYYY-MM-DD`; weekly note named by that week's Monday date; weekly project copy named by ISO week (e.g. `2026-W24`). |

## Architecture

Single skill change. `/reflect` remains the entry point.

```
DAILY
User writes <vault>/Reflections/daily/<date>.md  (free-form, Korean)
                │
                ▼
/reflect ──reads── config.yaml (obsidian.reflections_path)
                ├── reads  <vault>/Reflections/daily/<date>.md
                ├── scores against program.yaml scoring_system.criteria
                ├── appends "## 점수" + "## 피드백" to the note
                └── writes scored copy to
                     programs/{slug}/reflections/daily/<date>.md

WEEKLY
User writes <vault>/Reflections/weekly/<Monday>.md  (from _template.md)
                │
                ▼
/weekly  ──reads── config.yaml (obsidian.weekly_path)
                ├── reads  <vault>/Reflections/weekly/<Monday>.md
                ├── summarizes week from project daily copies
                ├── reads prior project weekly copy → two-week goal
                ├── appends "## 데이터 요약/주간 피드백/발견한 패턴" to the note
                └── writes copy to
                     programs/{slug}/reflections/weekly/<week-iso>.md
                │
                ▼
status ──reads── project daily copies ; /weekly carry-forward ──reads── project weekly copies
```

## Components

### 1. `config.yaml` — add an `obsidian` section

```yaml
obsidian:
  vault: "/Users/mingyumkim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Anna/DeliveryHero/DeliveryHero"
  reflections_path: "/Users/mingyumkim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Anna/DeliveryHero/DeliveryHero/Reflections/daily"
  weekly_path: "/Users/mingyumkim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Anna/DeliveryHero/DeliveryHero/Reflections/weekly"
  date_format: "YYYY-MM-DD"
```

The vault mirrors the project layout: `Reflections/daily/` and
`Reflections/weekly/` parallel `programs/{slug}/reflections/daily|weekly`.

`/reflect` reads `reflections_path` to locate `<reflections_path>/<today>.md`.
`/weekly` reads `weekly_path` to locate `<weekly_path>/<this-Monday>.md`.
Keeping paths in config (not hardcoded in the skills) makes it the single place
to change if the vault moves.

### 2. `programs/english-based-contribution-publishing/program.yaml`

`status: active` → `status: planned`. (Already applied during brainstorming.)

### 3. `.claude/skills/reflect/SKILL.md` — rewritten flow

1. Read `config.yaml`; get `obsidian.reflections_path`.
2. Load active programs (`status == active`, sorted by priority). With current
   data this is only Ticket Immersion.
3. For the active program, locate today's note: `<reflections_path>/<today>.md`.
   - **Missing** → print, in Korean: "오늘 회고 노트가 없습니다: `<path>`. Obsidian에
     먼저 작성해 주세요." and stop. The skill does **not** create the note — the
     user authors it.
   - **Already scored** → if the Obsidian note already contains a `## 점수`
     section OR the project copy already exists, ask before overwriting.
4. Read the free-form note content. Assign a score 1–5 using the program's
   `scoring_system.criteria`. Write 2–3 sentences of feedback grounded in the
   program's `goal`, `motivation`, and `problem_to_solve`. **Score reasoning and
   feedback in Korean.** Be honest; do not inflate.
5. **Append** to the Obsidian note, below the user's reflection:

   ```markdown

   ---
   ## 점수
   **{score}/5** — {reasoning in Korean}

   ## 피드백
   {2–3 sentences in Korean}
   ```

6. **Save scored copy** to
   `programs/ticket-based-product-immersion/reflections/daily/<date>.md`,
   containing the user's reflection text **and** the score/feedback. The score
   line MUST use the format `**{score}/5** — {reasoning}` under a `## Score`
   heading so `status` and `weekly` can parse it. Project-copy headings stay in
   English (`## Score`, `## Feedback`) for parsing stability; the feedback text
   itself is Korean.

   ```markdown
   # Daily Reflection — Ticket-Based Product Immersion Training
   Date: {date}

   ## Reflection
   {full text copied from the Obsidian note}

   ## Score
   **{score}/5** — {reasoning in Korean}

   ## Feedback
   {2–3 sentences in Korean}
   ```

7. Confirm quietly, in Korean: "회고 완료. 점수 {score}/5. Obsidian 노트와
   프로젝트에 저장했습니다." No further terminal output.

### 4. `<vault>/Reflections/weekly/` folder + `_template.md`

Create the `Reflections/weekly/` folder and a weekly template at
`<vault>/Reflections/weekly/_template.md`. The `_` prefix sorts it to the top and
keeps `/weekly` from treating it as a real week's note. The user copies it to
`<vault>/Reflections/weekly/<Monday-date>.md` each week and fills it in.

```markdown
## Did this training help this week?

## Most engaged moments

## What felt uncomfortable or unrealistic

## What I avoided (and why)

## Real behavior change?

## Maintain next week

## Reduce or simplify next week

## Next two-week goal

```

### 5. `.claude/skills/weekly/SKILL.md` — rewritten flow

1. Read `config.yaml`; get `obsidian.weekly_path`. Load active programs
   (`status == active`, by priority) — Ticket Immersion only.
2. Determine the current ISO week and that week's Monday date.
3. Locate this week's note: `<weekly_path>/<this-Monday>.md`.
   - **Missing** → print, in Korean: "이번 주 주간 회고 노트가 없습니다:
     `<path>`. Obsidian에서 `_template.md`로 먼저 작성해 주세요." and stop. The
     skill does **not** create the note.
   - **Already reviewed** → if the note already contains a `## 주간 피드백`
     section OR the project weekly copy for this ISO week exists, ask before
     overwriting (replace the prior appended block, don't stack).
4. Load this week's daily project copies from
   `programs/{slug}/reflections/daily/` (Monday→today): days reflected,
   day-by-day scores, average.
5. Load the previous weekly project copy (most recent file in
   `programs/{slug}/reflections/weekly/`) and read its `## Two-week goal`.
6. Read the weekly Obsidian note's content. Produce, in **Korean**:
   (a) 3–4 sentences of goal-progress feedback vs the prior two-week goal,
   grounded in the daily content (skip if no prior goal); (b) one surfaced
   pattern as a short observation. **Do not score the week.**
7. **Append** to the Obsidian note (leaving the user's text intact above):

   ```markdown

   ---
   ## 데이터 요약
   반영 일수: {n}/7 · 점수: {day-by-day} · 평균: {avg}

   ## 주간 피드백
   {3–4 sentence goal-progress feedback in Korean, or "— (이전 목표 없음)"}

   ## 발견한 패턴
   {one 2-sentence observation in Korean, or "—"}
   ```

8. **Save project copy** to `programs/{slug}/reflections/weekly/<week-iso>.md`,
   preserving the existing weekly file format so the next run's carry-forward
   works. The `## Two-week goal` section is filled from the note's
   `## Next two-week goal`. Headings stay English; text is Korean.

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
   {full text copied from the Obsidian weekly note — user's answers only}

   ## Pattern observed
   {2-sentence observation in Korean, or "—"}

   ## Two-week goal
   {from the note's "Next two-week goal" section}
   ```

9. Quiet Korean confirmation: "주간 회고 완료: {program name}. Obsidian 노트와
   프로젝트에 저장했습니다."

## Data flow / compatibility

- `status` reads `programs/{slug}/reflections/daily/{today}.md` and extracts the
  score via `Score:` / `**Score**`. The project copy's `## Score` +
  `**{score}/5**` line satisfies this.
- `weekly` reads the same daily files for the week, using both the score and the
  reflection content. The daily project copy includes the full reflection text,
  so weekly has what it needs. `weekly` also filters `status == active`, so the
  `planned` English program is correctly excluded.
- The weekly **two-week goal thread** is preserved: each `/weekly` run reads the
  previous weekly project copy's `## Two-week goal` and writes the new one from
  the user's note, exactly as the old interactive flow did — so carry-forward is
  unchanged, only the input source moved to Obsidian.

## Error handling / edge cases

- **No note today:** stop with a Korean message telling the user to write it.
- **Idempotency / re-run:** detect a prior `## 점수` in the Obsidian note or an
  existing project copy; ask before overwriting both.
- **Vault unreachable** (e.g. iCloud path missing): report the path and stop;
  do not silently create files elsewhere.
- **Date/timezone:** use local date in `YYYY-MM-DD`. Project uses CET reminders;
  the date is taken from the system clock at run time.

## Known limitation

This assumes **one active program ⇒ one daily note**. If a second program is
later reactivated, a single free-form note cannot separate per-program content;
the note structure would need revisiting at that point. Acceptable given the
explicit one-program-at-a-time decision.

## Behavior changes from the current skill

- The interactive Q&A path (asking "What did you do today…" and the per-program
  question lists) is removed — input now comes from the Obsidian note.
- The "Yesterday you planned to… " opener is removed (no live conversation, and
  free-form notes have no fixed "tomorrow's plan" anchor).
- Per-program `REFLECTION.md` is **no longer consulted**. Scoring is driven by
  `scoring_system.criteria` directly. (No active program currently has a
  `REFLECTION.md`, so this changes nothing today.)
- **Weekly:** the interactive guided session (data summary, goal feedback, then
  7 questions one-at-a-time, then asking for the next two-week goal) is removed.
  The user answers in the Obsidian weekly note instead; `/weekly` reads it,
  appends feedback, and saves the project copy. The two-week-goal carry-forward
  thread is retained.

## Out of scope

- Obsidian MCP server / Local REST API plugin (not needed for local file read).
- Changes to `status`, `add-program`.
- Auto-generating the daily or weekly Obsidian note (user authors them; a weekly
  `_template.md` is provided to copy from).
- Multi-program-per-day note structure.
```
