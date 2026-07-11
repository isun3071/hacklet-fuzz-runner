# STATE — where this project stands

*Snapshot: 2026-07-10. This is point-in-time and will go stale; `CLAUDE.md` holds the durable stuff.*

## Status

The **Stage-5 grader is functionally complete** and discrimination is validated: the vulnerable
reference anchors at **642 slop**, the hardened one at **0**, and the suite locks that. On top of the
pure grader we built an **LLM-assisted deploy+grade pipeline** (clone → plan a Docker deploy → run →
grade → tear down) and a **cross-stack parity instrument**, and hardened both across two real
hackathon batches (Civic Hacks 2026, n≈14; Boston 2025, n=33 — see below).

**Headline validation so far:** winners scored *lower* slop than non-winners (avg 85 vs 96, median
76 vs 98) — suggestive at small n, which is why we keep running bigger batches.

## In flight

- **`parityboston25` batch (n=33)** was running in the *original* `hacklet-league` checkout when this
  repo was extracted. Its `*.jsonl` output lives there (gitignored here). When it finishes: pull the
  stats + parity and analyze.

## Recently landed (last session)

- **Build-download cache** (`deploy_and_grade._inject_build_cache` + capped teardown). Fixes the pattern
  where a heavy-deps app (e.g. `TimeTalk`: torch/scipy/pyarrow) blew the 480s build on downloads alone
  and *every retry re-downloaded from cold*. Now pip/npm downloads persist across the 3 attempts (and a
  timeout-kill) via BuildKit cache mounts, and common wheels download once per batch. Offline suite green
  (250). The end-to-end "survives a timeout-kill" behavior is best confirmed on the next live heavy build.

## Do next (priority order)

1. **Analyze `parityboston25`** when it lands. Three questions: (a) do the heavy-deps apps that were
   wedging on builds now deploy? (b) does winners-<-non-winners hold at n=33? (c) does the SPA
   login/upload blind spot recur, or was it a two-app fluke?
2. **SPA login/upload discovery blind spot** — the next *recall* investigation. On some `spa-path` apps a
   client-rendered login/upload control is never reached (hash-routing, formless control, or an
   un-rendered route). Needs a deliberate start; don't begin without confirming it's the priority.

## Settled — don't relitigate without a reason

- Deploy planner model = **`qwen/qwen3.7-plus`** (cheap, strong on AA-Omniscience-style abstention).
- Results are **append-only JSONL**; dedup at read time.
- Grading gets its **own** killable budget (`--grade-timeout`), separate from build — a shared total
  starved grading after slow deploys.
- Scoring stays **unbounded deduction-only**; no 0–100 normalization.
- Non-web apps are **skipped, not faked**.
- Resume a batch by re-running the **same `--results` file** — it skips already-graded/skipped apps and
  retries only the untested/failed ones.

## What did NOT come across in the extraction

- **Git history** — distilled into `PROJECT_LOG.md` instead (130 commits, 2026-07-08…10).
- **The hidden probe pool** — lives in a separate *private* fuzz-catalog repo, by design.
- **League integration** — the league's `backend/rounds/scoring.py` will consume the Slop Score at
  Stage 5; today there is **no code link** (only anticipatory comments there), so nothing is broken.
