# Obsidian-Sourced Daily Reflection — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `/reflect` read the user's free-form daily reflection from their Obsidian vault, score it, append Korean feedback into the note, and save a scored copy into the project — replacing the old interactive Q&A flow.

**Architecture:** A single skill rewrite (`.claude/skills/reflect/SKILL.md`) plus one config addition (`config.yaml`). The skill reads `<vault>/Reflections/<date>.md` directly from disk (no Obsidian MCP), scores against the active program's `scoring_system.criteria`, appends `## 점수`/`## 피드백` to the Obsidian note, and writes a scored copy to `programs/{slug}/reflections/daily/<date>.md` so `/status` and `/weekly` keep working unchanged.

**Tech Stack:** Claude Code skills (markdown instruction files), YAML config, local filesystem (iCloud-synced Obsidian vault).

**Reference spec:** `docs/superpowers/specs/2026-06-07-obsidian-reflection-source-design.md`

---

## File Structure

- **Modify** `config.yaml` — add an `obsidian:` section holding the reflections folder path and date format. This is the single source of truth for where notes live.
- **Modify** `.claude/skills/reflect/SKILL.md` — full rewrite of the flow from interactive Q&A to Obsidian-source read/score/append/save.
- **No changes** to `.claude/skills/weekly/SKILL.md`, `.claude/skills/status/SKILL.md`, `.claude/skills/add-program/SKILL.md`.

Already done outside this plan: `programs/english-based-contribution-publishing/program.yaml` set to `status: planned`; `<vault>/Reflections/` created with the two existing reflections moved in.

---

## Task 1: Add the `obsidian` section to `config.yaml`

**Files:**
- Modify: `config.yaml`

- [ ] **Step 1: Read the current config**

Run: `cat config.yaml`
Expected: the existing `reminders:` block, no `obsidian:` key yet.

- [ ] **Step 2: Append the `obsidian` section**

Append to the end of `config.yaml` (keep existing content intact):

```yaml

obsidian:
  reflections_path: "/Users/mingyumkim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Anna/DeliveryHero/DeliveryHero/Reflections"
  date_format: "YYYY-MM-DD"
```

- [ ] **Step 3: Verify the path exists and the YAML is well-formed**

Run:
```bash
python3 -c "import yaml; c=yaml.safe_load(open('config.yaml')); print(c['obsidian']['reflections_path'])"
```
Expected: prints the reflections path with no traceback.

Run:
```bash
test -d "$(python3 -c "import yaml;print(yaml.safe_load(open('config.yaml'))['obsidian']['reflections_path'])")" && echo "PATH OK"
```
Expected: `PATH OK`.

- [ ] **Step 4: Commit**

```bash
git add config.yaml
git commit -m "feat: add obsidian reflections path to config"
```

---

## Task 2: Rewrite the `reflect` skill to read from Obsidian

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
Wait for confirmation. If the user declines, skip this program. If the user confirms, replace the previously appended `## 점수`/`## 피드백` section in the Obsidian note (do not stack a second one) and overwrite the project copy.

### 2c. Read and score

Read the note's full content. Using the program's `scoring_system` (its `scale` and `criteria`), assign a score. Score from what is written — do not ask follow-up questions. Let `{scale_max}` be the top of the program's scale (e.g. `5` for a `1-5` scale).

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

Keep the `## Score` heading and the `**{score}/{scale_max}**` line in exactly this format — `/status` and `/weekly` parse them. The headings here stay in English for parsing stability even though the text is Korean.

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

- [ ] **Step 2: Verify the frontmatter and key sections are present**

Run:
```bash
grep -nE "reflections_path|## 점수|## 피드백|## Score|## Feedback|status == active" .claude/skills/reflect/SKILL.md
```
Expected: matches for the config key, both Korean headings, both English project-copy headings, and the active-status filter.

- [ ] **Step 3: Confirm no interactive-question remnants remain**

Run:
```bash
grep -nE "What did you do today|Ask:|Wait for the user's answer" .claude/skills/reflect/SKILL.md || echo "CLEAN"
```
Expected: `CLEAN` (the old interactive prompts are gone).

- [ ] **Step 4: Commit**

```bash
git add .claude/skills/reflect/SKILL.md
git commit -m "feat: rewrite reflect skill to read reflections from Obsidian"
```

---

## Task 3: End-to-end behavioral verification

This verifies the rewritten skill against a real note. It uses a throwaway dated note so it does not pollute real history.

**Files:**
- Temp: `<vault>/Reflections/<today>.md` (created for the test, then removed)

- [ ] **Step 1: Create a sample reflection note for today**

Run (creates today's note in the vault Reflections folder):
```bash
VR="/Users/mingyumkim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Anna/DeliveryHero/DeliveryHero/Reflections"
TODAY=$(date +%F)
cat > "$VR/$TODAY.md" <<'EOF'
## What I did
- 오늘 티켓의 전체 플로우를 그려보고 내 작업이 어디에 속하는지 정리했다.
- 이 작업이 유저에게 어떤 문제를 해결하는지, 실패 시나리오는 무엇인지 적어봤다.

## What I learned from
- SNS/SQS 흐름을 다시 확인하면서 트리거 경로를 명확히 이해했다.

## What points needed to be improved
- 시니어에게 정리한 질문을 공유하지는 못했다. 내일은 공유까지 해보자.
EOF
echo "created $VR/$TODAY.md"
```
Expected: `created .../Reflections/<today>.md`.

- [ ] **Step 2: Run the reflection flow**

In the Claude session, run `/reflect`.
Expected: it finds today's note, scores it (1–5), and prints the short Korean confirmation. No interactive questions are asked.

- [ ] **Step 3: Verify feedback was appended to the Obsidian note**

Run:
```bash
VR="/Users/mingyumkim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Anna/DeliveryHero/DeliveryHero/Reflections"
TODAY=$(date +%F)
grep -nE "## 점수|## 피드백" "$VR/$TODAY.md"
```
Expected: one `## 점수` and one `## 피드백` heading (not duplicated).

- [ ] **Step 4: Verify the project scored copy exists and is parseable**

Run:
```bash
TODAY=$(date +%F)
F="programs/ticket-based-product-immersion/reflections/daily/$TODAY.md"
grep -nE "^## Score|\*\*[0-9]/5\*\*|^## Reflection|^## Feedback" "$F"
```
Expected: matches for `## Reflection`, `## Score`, a `**N/5**` line, and `## Feedback`.

- [ ] **Step 5: Verify `/status` reads the score**

In the Claude session, run `/status`.
Expected: the Ticket-Based Product Immersion row shows today's score (e.g. `4/5`), and the English program shows `planned` (not prompted for reflection).

- [ ] **Step 6: Verify idempotency**

Run `/reflect` again.
Expected: it detects the existing score and asks before overwriting (Korean confirmation prompt), rather than appending a second `## 점수` block.

- [ ] **Step 7: Clean up the test artifacts**

Run:
```bash
VR="/Users/mingyumkim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Anna/DeliveryHero/DeliveryHero/Reflections"
TODAY=$(date +%F)
rm -f "$VR/$TODAY.md" "programs/ticket-based-product-immersion/reflections/daily/$TODAY.md"
echo "cleaned"
```
Expected: `cleaned`. (Skip this step if you ran the test on a real reflection you want to keep.)

---

## Self-Review notes

- **Spec coverage:** config path (Task 1) ✓; skill rewrite with read/score/append/save (Task 2) ✓; Korean feedback ✓; English program excluded via `planned` (verified Task 3 Step 5) ✓; idempotency (Task 2 §2b, Task 3 Step 6) ✓; status/weekly compatibility via `## Score` + `**N/5**` (Task 2 §2e, Task 3 Step 4) ✓; missing-note handling (Task 2 §2a) ✓.
- **No automated unit tests:** skills are markdown instructions; verification is behavioral (Task 3). This is the honest fit, not a gap.
- **Format consistency:** the `**{score}/{scale_max}**` line and `## Score` heading match the format `/status` and `/weekly` already parse from the existing project reflection files.
