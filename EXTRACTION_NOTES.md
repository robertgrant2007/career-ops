# career-ops — Architecture Extraction Notes

Five reusable patterns lifted from career-ops, each with the key file, a critical snippet, and one concrete port to either **(a)** a 4-agent meeting-intelligence pipeline (Fathom transcripts → ClickUp tasks + Drive notes) or **(b)** a multi-skill ad-copy compliance QC chain (meta + health-claims + agency-qc-accuracy).

---

## 1. Mode Dispatcher — one slash command, many modes

**File:** `.agents/skills/career-ops/SKILL.md` (symlinked into `.claude/skills/`, `.qwen/skills/`)

One `user-invocable` skill is a thin **router**: it maps an arg to a mode, then lazy-loads only the files that mode needs. No business logic lives in the router itself.

```text
| Input                          | Mode            |
| (empty / no args)              | discovery (menu)|
| JD text or URL (no sub-command)| auto-pipeline   |
| oferta / scan / batch / ...    | {that mode}     |
# then: Read modes/_shared.md + modes/{mode}.md  (shared modes)
#   or: Read modes/{mode}.md                      (standalone modes)
```

**Apply to (b):** Build one `/adqc` skill whose router dispatches `meta | health | accuracy | all`. Default (no arg) = run all three in sequence; a single keyword runs just that check. Each mode loads a shared `_rules.md` plus its own `meta.md` / `health-claims.md` / `accuracy.md` — so the three compliance skills share one entrypoint instead of three disconnected commands.

---

## 2. Sub-agent Batch Orchestration — conductor + headless workers

**File:** `batch/batch-runner.sh` (orchestration), `batch/batch-prompt.md` (self-contained worker prompt)

A conductor loops over a work queue and spawns **stateless `claude -p` workers**, each with a clean context and the full instructions injected as a system prompt. The conductor only schedules, tracks state, and merges.

```bash
local -a claude_args=(-p --dangerously-skip-permissions)
[[ -n "$MODEL" ]] && claude_args+=(--model "$MODEL")
claude_args+=(--append-system-prompt-file "$resolved_prompt" "$prompt")
claude "${claude_args[@]}" > "$log_file" 2>&1 || exit_code=$?
```

Worker prompt is explicitly **self-contained** ("Tienes TODO lo necesario aquí. No dependes de ningún otro skill"), placeholders (`{{URL}}`, `{{REPORT_NUM}}`) resolved by the conductor via `sed`, and each worker emits a result JSON to stdout that the conductor parses for score/status.

**Apply to (a):** One conductor reads a list of Fathom transcript IDs as the queue; spawns one headless worker per meeting. Run 4 workers in parallel (`--parallel 4`) per meeting — extract-actions, summarize, tag-attendees, draft-followup — each emitting result JSON. Conductor merges their outputs, then calls ClickUp + Drive MCP tools. Resumable: re-run skips meetings already `completed`.

---

## 3. Skill Structure — flat router + shared/mode references

**File:** `.agents/skills/career-ops/SKILL.md` + `modes/_shared.md` + `modes/{mode}.md`

Maps cleanly to the standard `SKILL.md` + references pattern, but with a twist: the SKILL file is tiny (routing only); the "references" are mode files in repo root, split into a **shared base** loaded for most modes and **per-mode** files. CLAUDE.md/AGENTS.md add a strict **System vs User layer** rule so updates never clobber personalization.

```text
SKILL.md            → router (frontmatter: name, description, argument-hint)
modes/_shared.md    → shared rules/scoring  (System layer, auto-updatable)
modes/{mode}.md     → one file per mode      (System layer)
modes/_profile.md   → user overrides         (User layer, NEVER auto-updated)
```

**Apply to (b):** `adqc/SKILL.md` (router) + `references/_shared.md` (shared severity scale, output format) + one reference per regime (`meta.md`, `health-claims.md`, `accuracy.md`) + `client-overrides.md` as the user layer for brand-specific claim allowlists — so engine rules and per-client exceptions never overwrite each other.

---

## 4. Human-in-the-Loop Gates — stop vs. proceed

**File:** `AGENTS.md` (Ethical Use), `modes/_shared.md` (score thresholds)

The system auto-generates everything up to the irreversible/outward-facing act, then **hard-stops for human review**. Score thresholds gate recommendations; a separate quantitative `--min-score` gate auto-skips low-value downstream work.

```text
- NEVER submit an application without the user reviewing it first.
  Fill forms, draft answers, generate PDFs — but STOP before Submit/Send/Apply.
- Below 3.5 → recommend AGAINST applying.
# batch-runner.sh: if score < MIN_SCORE → mark "skipped", don't PDF/track
```

**Apply to (a):** Workers may draft ClickUp tasks and Drive notes freely, but the conductor stops before *creating* anything external — present the proposed task list + note for one approval, then write. Add a confidence gate: action items below a confidence threshold land in a "review" section rather than being created as live tasks automatically.

---

## 5. Persistent State — tracker + reports + outputs, append-not-edit

**File:** `data/applications.md` (tracker), `reports/` (full records), `batch/tracker-additions/` (merge staging), `batch/batch-state.tsv` (run state)

State is split by purpose and write discipline is enforced: workers **never edit the shared tracker directly** — they drop one TSV line per item into a staging dir, and a merge script reconciles (dedup, column-swap, integrity check). A separate `batch-state.tsv` gives **resumability** (skip `completed`, retry `failed`).

```text
batch/tracker-additions/{id}.tsv   # workers append one line each (no contention)
node merge-tracker.mjs             # reconcile → data/applications.md (dedup)
batch-state.tsv: id status report_num score error retries  # resumable
RULE: NEVER add rows to applications.md directly; only UPDATE existing rows.
```

**Apply to (a):** A `meetings.md` tracker (one row per meeting: date, attendees, #tasks, note link, status), full per-meeting records in `notes/{date}-{slug}.md`, and a `meeting-state.tsv` for resumability. Each worker writes its own staging file; a merge step reconciles into the tracker and dedupes meetings already pushed to ClickUp — so a re-run never double-creates tasks.

---

*Reconnaissance only — no scripts run, no dependencies installed, no user data touched.*
