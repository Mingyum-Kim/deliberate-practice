# Design: Obsidian-Sourced Daily Reflection

Date: 2026-06-07
Status: Approved (pending spec review)

## Summary

Change the daily reflection flow so the user writes their reflection in an
Obsidian note (free-form), and the `/reflect` skill **reads** that note instead
of asking interactive questions. The skill scores the note (1–5) against the
active program's scoring criteria, appends Korean feedback back into the same
Obsidian note, and saves a scored copy into the project so `status` and `weekly`
keep working.

Only **one program is active at a time**. As part of this change, the
English-Based Contribution Publishing program is set to `status: planned` so
only the Ticket-Based Product Immersion program is reflected on.

## Decisions (from brainstorming)

| Topic | Decision |
|-------|----------|
| Active program | Only `ticket-based-product-immersion`. English program → `status: planned`. |
| Vault access | **Direct file read** off disk. No Obsidian MCP, no Local REST API plugin. |
| Vault location | `/Users/mingyumkim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Anna/DeliveryHero/DeliveryHero` (the nested folder containing `.obsidian/`). |
| Note location | `<vault>/Reflections/<date>.md` (a `Reflections/` subfolder). |
| Note format | **Free-form.** User writes with their own headings (currently Korean). |
| Feedback language | **Korean.** |
| Feedback destination | Appended into the same Obsidian note, **and** a scored copy saved to the project folder. No terminal output beyond a short confirmation. |
| Date format | `YYYY-MM-DD`. |

## Architecture

Single skill change. `/reflect` remains the entry point.

```
User writes <vault>/Reflections/<date>.md  (Obsidian, free-form, Korean)
                │
                ▼
/reflect  ──reads── config.yaml (obsidian.reflections_path)
                │
                ├── reads  <vault>/Reflections/<date>.md
                ├── scores against program.yaml scoring_system.criteria
                ├── appends "## 점수" + "## 피드백" to the Obsidian note
                └── writes scored copy to
                     programs/ticket-based-product-immersion/reflections/daily/<date>.md
                │
                ▼
status / weekly  ──read── project scored copies  (unchanged)
```

## Components

### 1. `config.yaml` — add an `obsidian` section

```yaml
obsidian:
  reflections_path: "/Users/mingyumkim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Anna/DeliveryHero/DeliveryHero/Reflections"
  date_format: "YYYY-MM-DD"
```

The skill reads `reflections_path` to locate `<reflections_path>/<today>.md`.
Keeping the path in config (not hardcoded in the skill) makes it the single
place to change if the vault moves.

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

## Data flow / compatibility

- `status` reads `programs/{slug}/reflections/daily/{today}.md` and extracts the
  score via `Score:` / `**Score**`. The project copy's `## Score` +
  `**{score}/5**` line satisfies this.
- `weekly` reads the same daily files for the week, using both the score and the
  reflection content. The project copy includes the full reflection text, so
  weekly has what it needs. `weekly` also filters `status == active`, so the
  `planned` English program is correctly excluded.

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

## Out of scope

- Obsidian MCP server / Local REST API plugin (not needed for local file read).
- Changes to `weekly`, `status`, `add-program`.
- Auto-generating the daily Obsidian note (user authors it).
- Multi-program-per-day note structure.
```
