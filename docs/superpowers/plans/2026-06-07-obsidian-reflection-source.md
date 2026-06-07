# Obsidian-Sourced Daily & Weekly Reflection — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `/reflect` and `/weekly` read the user's reflections from their Obsidian vault, give Korean feedback appended into the notes, and save project copies — replacing both interactive flows.

**Architecture:** Two skill rewrites (`.claude/skills/reflect/SKILL.md`, `.claude/skills/weekly/SKILL.md`) plus one config addition (`config.yaml`) and a new weekly template in the vault. Skills read `<vault>/Reflections/daily/<date>.md` and `<vault>/Reflections/weekly/<Monday>.md` directly from disk (no Obsidian MCP). Daily scores 1–5; weekly gives goal feedback + one pattern (no weekly score) and carries the two-week goal forward via the project weekly copy. `status` is untouched and keeps working off the project daily copies.

**Tech Stack:** Claude Code skills (markdown instruction files), YAML config, local filesystem (iCloud-synced Obsidian vault).

**Reference spec:** `docs/superpowers/specs/2026-06-07-obsidian-reflection-source-design.md`

---

## File Structure

- **Modify** `config.yaml` — add an `obsidian:` section with `vault`, `reflections_path` (`Reflections/daily`), `weekly_path` (`Reflections/weekly`), `date_format`.
- **Modify** `.claude/skills/reflect/SKILL.md` — full rewrite: Obsidian-source daily read/score/append/save.
- **Modify** `.claude/skills/weekly/SKILL.md` — full rewrite: Obsidian-source weekly read/feedback/append/save, retaining two-week-goal carry-forward.
- **Create** `<vault>/Reflections/weekly/_template.md` — the weekly note template.
- **No changes** to `.claude/skills/status/SKILL.md`, `.claude/skills/add-program/SKILL.md`.

Already done outside this plan: `english-based-contribution-publishing/program.yaml` → `status: planned`; vault `Reflections/daily/` created with the two existing reflections moved in; vault `Reflections/weekly/` folder created (empty).

The vault path root is:
`/Users/mingyumkim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Anna/DeliveryHero/DeliveryHero`

---

## Task 1: Add the `obsidian` section to `config.yaml`

**Files:**
- Modify: `config.yaml`

- [ ] **Step 1: Read the current config**

Run: `cat config.yaml`
Expected: existing `reminders:` block, no `obsidian:` key yet.

- [ ] **Step 2: Append the `obsidian` section**

Append to the end of `config.yaml` (keep existing content intact):

```yaml

obsidian:
  vault: "/Users/mingyumkim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Anna/DeliveryHero/DeliveryHero"
  reflections_path: "/Users/mingyumkim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Anna/DeliveryHero/DeliveryHero/Reflections/daily"
  weekly_path: "/Users/mingyumkim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Anna/DeliveryHero/DeliveryHero/Reflections/weekly"
  date_format: "YYYY-MM-DD"
```

- [ ] **Step 3: Verify the YAML parses and both paths exist**

Run:
```bash
python3 - <<'PY'
import yaml
c = yaml.safe_load(open('config.yaml'))['obsidian']
import os
for k in ('reflections_path','weekly_path'):
    print(k, 'OK' if os.path.isdir(c[k]) else 'MISSING', c[k])
PY
```
Expected: both lines print `OK`.

- [ ] **Step 4: Commit**

```bash
git add config.yaml
git commit -m "feat: add obsidian daily/weekly paths to config"
```

---

## Task 2: Rewrite the `reflect` (daily) skill

**Files:**
- Modify: `.claude/skills/reflect/SKILL.md` (full replacement)

- [ ] **Step 1: Replace the entire file with the new flow**

Write `.claude/skills/reflect/SKILL.md` with exactly this content:

````markdown
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
````

- [ ] **Step 2: Verify key sections are present**

Run:
```bash
grep -nE "reflections_path|## 점수|## 피드백|## Score|## Feedback|status == active" .claude/skills/reflect/SKILL.md
```
Expected: matches for the config key, both Korean headings, both English project-copy headings, and the active-status filter.

- [ ] **Step 3: Confirm no interactive-question remnants remain**

Run:
```bash
grep -nE "What did you do today|Wait for the user's answer|Yesterday you planned" .claude/skills/reflect/SKILL.md || echo "CLEAN"
```
Expected: `CLEAN`.

- [ ] **Step 4: Commit**

```bash
git add .claude/skills/reflect/SKILL.md
git commit -m "feat: rewrite reflect skill to read daily reflections from Obsidian"
```

---

## Task 3: Create the weekly template in the vault

**Files:**
- Create: `<vault>/Reflections/weekly/_template.md`

- [ ] **Step 1: Write the weekly template**

Run:
```bash
WP="/Users/mingyumkim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Anna/DeliveryHero/DeliveryHero/Reflections/weekly"
mkdir -p "$WP"
cat > "$WP/_template.md" <<'EOF'
## Did this training help this week?

## Most engaged moments

## What felt uncomfortable or unrealistic

## What I avoided (and why)

## Real behavior change?

## Maintain next week

## Reduce or simplify next week

## Next two-week goal

EOF
echo "wrote $WP/_template.md"
```
Expected: `wrote .../Reflections/weekly/_template.md`.

- [ ] **Step 2: Verify the template has all sections**

Run:
```bash
WP="/Users/mingyumkim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Anna/DeliveryHero/DeliveryHero/Reflections/weekly"
grep -cE "^## " "$WP/_template.md"
```
Expected: `8` (eight headings).

(No git commit — the template lives in the iCloud vault, which is outside the repo.)

---

## Task 4: Rewrite the `weekly` skill

**Files:**
- Modify: `.claude/skills/weekly/SKILL.md` (full replacement)

- [ ] **Step 1: Replace the entire file with the new flow**

Write `.claude/skills/weekly/SKILL.md` with exactly this content:

````markdown
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
````

- [ ] **Step 2: Verify key sections are present**

Run:
```bash
grep -nE "weekly_path|## 주간 피드백|## 데이터 요약|## 발견한 패턴|## Two-week goal|status == active" .claude/skills/weekly/SKILL.md
```
Expected: matches for the config key, the three Korean headings, the carry-forward `## Two-week goal` heading, and the active-status filter.

- [ ] **Step 3: Confirm no interactive-question remnants remain**

Run:
```bash
grep -nE "Ask each question, wait for the answer|Ask the weekly reflection questions|Wait for the answer" .claude/skills/weekly/SKILL.md || echo "CLEAN"
```
Expected: `CLEAN`.

- [ ] **Step 4: Commit**

```bash
git add .claude/skills/weekly/SKILL.md
git commit -m "feat: rewrite weekly skill to read weekly review from Obsidian"
```

---

## Task 5: End-to-end behavioral verification (daily)

**Files:**
- Temp: `<vault>/Reflections/daily/<today>.md`

- [ ] **Step 1: Create a sample daily note for today**

```bash
RP="/Users/mingyumkim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Anna/DeliveryHero/DeliveryHero/Reflections/daily"
TODAY=$(date +%F)
cat > "$RP/$TODAY.md" <<'EOF'
## What I did
- 오늘 티켓의 전체 플로우를 그려보고 내 작업이 어디에 속하는지 정리했다.
- 이 작업이 유저에게 어떤 문제를 해결하는지, 실패 시나리오는 무엇인지 적어봤다.

## What I learned from
- SNS/SQS 흐름을 다시 확인하면서 트리거 경로를 명확히 이해했다.

## What points needed to be improved
- 시니어에게 정리한 질문을 공유하지는 못했다. 내일은 공유까지 해보자.
EOF
echo "created $RP/$TODAY.md"
```
Expected: `created .../Reflections/daily/<today>.md`.

- [ ] **Step 2: Run `/reflect`**

Expected: finds today's note, scores 1–5, prints the short Korean confirmation, no interactive questions.

- [ ] **Step 3: Verify feedback appended to the Obsidian note (not duplicated)**

```bash
RP="/Users/mingyumkim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Anna/DeliveryHero/DeliveryHero/Reflections/daily"
TODAY=$(date +%F)
grep -cE "## 점수" "$RP/$TODAY.md"; grep -cE "## 피드백" "$RP/$TODAY.md"
```
Expected: `1` and `1`.

- [ ] **Step 4: Verify the project scored copy is parseable**

```bash
TODAY=$(date +%F)
F="programs/ticket-based-product-immersion/reflections/daily/$TODAY.md"
grep -nE "^## Reflection|^## Score|\*\*[0-9]/5\*\*|^## Feedback" "$F"
```
Expected: matches for `## Reflection`, `## Score`, a `**N/5**` line, `## Feedback`.

- [ ] **Step 5: Verify `/status` reads the score**

Run `/status`. Expected: Ticket row shows today's score; English program shows `planned`.

- [ ] **Step 6: Verify idempotency**

Run `/reflect` again. Expected: detects the existing score and asks before overwriting; no second `## 점수` block appended.

- [ ] **Step 7: Clean up (skip if you wrote a real reflection to keep)**

```bash
RP="/Users/mingyumkim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Anna/DeliveryHero/DeliveryHero/Reflections/daily"
TODAY=$(date +%F)
rm -f "$RP/$TODAY.md" "programs/ticket-based-product-immersion/reflections/daily/$TODAY.md"
echo cleaned
```
Expected: `cleaned`.

---

## Task 6: End-to-end behavioral verification (weekly)

**Files:**
- Temp: a prior-week project weekly copy (to test carry-forward), this week's vault weekly note.

- [ ] **Step 1: Seed a previous two-week goal (to test carry-forward)**

```bash
D="programs/ticket-based-product-immersion/reflections/weekly"
mkdir -p "$D"
cat > "$D/2026-W23.md" <<'EOF'
# Weekly Reflection — Ticket-Based Product Immersion Training
Week: 2026-W23 (2026-06-01 – 2026-06-07)

## Two-week goal
모든 티켓에서 유저 관점 질문을 최소 한 개씩 시니어와 공유하기.
EOF
echo "seeded W23"
```
Expected: `seeded W23`.

- [ ] **Step 2: Create this week's weekly note in the vault from the template**

```bash
WP="/Users/mingyumkim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Anna/DeliveryHero/DeliveryHero/Reflections/weekly"
MON=$(date -v -monday +%F 2>/dev/null || python3 -c "import datetime;t=datetime.date.today();print((t-datetime.timedelta(days=t.weekday())).isoformat())")
cp "$WP/_template.md" "$WP/$MON.md"
echo "created $WP/$MON.md"
```
Expected: `created .../Reflections/weekly/<this-monday>.md`. Fill a few sections (including "Next two-week goal") before running the skill, or leave as-is to test empty handling.

- [ ] **Step 3: Run `/weekly`**

Expected: finds this week's note, shows a Korean confirmation, no interactive questions.

- [ ] **Step 4: Verify feedback appended to the weekly note**

```bash
WP="/Users/mingyumkim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Anna/DeliveryHero/DeliveryHero/Reflections/weekly"
MON=$(date -v -monday +%F 2>/dev/null || python3 -c "import datetime;t=datetime.date.today();print((t-datetime.timedelta(days=t.weekday())).isoformat())")
grep -cE "## 주간 피드백|## 데이터 요약|## 발견한 패턴" "$WP/$MON.md"
```
Expected: `3`.

- [ ] **Step 5: Verify the project weekly copy carries the new goal forward**

```bash
ISO=$(date +%G-W%V)
F="programs/ticket-based-product-immersion/reflections/weekly/$ISO.md"
grep -nE "^## Two-week goal|^## Two-week goal feedback|^## Reflection" "$F"
```
Expected: matches for `## Two-week goal feedback`, `## Reflection`, and `## Two-week goal`. The feedback section should reference the W23 goal seeded in Step 1.

- [ ] **Step 6: Clean up test artifacts (skip what you want to keep)**

```bash
WP="/Users/mingyumkim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Anna/DeliveryHero/DeliveryHero/Reflections/weekly"
MON=$(date -v -monday +%F 2>/dev/null || python3 -c "import datetime;t=datetime.date.today();print((t-datetime.timedelta(days=t.weekday())).isoformat())")
ISO=$(date +%G-W%V)
rm -f "$WP/$MON.md" "programs/ticket-based-product-immersion/reflections/weekly/$ISO.md" "programs/ticket-based-product-immersion/reflections/weekly/2026-W23.md"
echo cleaned
```
Expected: `cleaned`.

---

## Self-Review notes

- **Spec coverage:** config paths incl. `Reflections/daily` + `Reflections/weekly` (Task 1) ✓; daily skill rewrite (Task 2) ✓; weekly template (Task 3) ✓; weekly skill rewrite with carry-forward (Task 4) ✓; Korean feedback ✓; English program excluded via `planned` (Task 5 Step 5) ✓; daily + weekly idempotency (Tasks 2/4 §b, Task 5 Step 6) ✓; status compatibility via `## Score` + `**N/5**` (Task 2 §2e, Task 5 Step 4) ✓; weekly carry-forward via `## Two-week goal` (Task 4 §3d/§3g, Task 6 Step 5) ✓; missing-note handling (Tasks 2/4 §a) ✓.
- **Type/format consistency:** the daily `**{score}/{scale_max}**` line and `## Score` heading match what `/status` and `/weekly` parse. The weekly `## Two-week goal` heading matches between writer (Task 4 §3g) and reader (Task 4 §3d).
- **No automated unit tests:** skills are markdown instructions; verification is behavioral (Tasks 5–6) — the honest fit.
- **ISO week note:** `date +%G-W%V` gives ISO year + week (e.g. `2026-W24`), matching the spec's weekly project-copy naming.
