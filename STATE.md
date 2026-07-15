# STATE: where this project stands

*Snapshot: 2026-07-15. Point in time, and it will go stale. `CLAUDE.md` holds the durable stuff.*

## Status

The grader is functionally complete and validated at scale. The catalog is 72 probes across three
bundles (security, qa, performance). The vulnerable reference app anchors at 642 slop, the hardened one
at 0, and the test suite locks that gap. Around the grader sits an LLM-assisted deploy and grade
pipeline (clone, plan a Docker deploy, run, grade, tear down), a live URL path that fuzzes an already
deployed app with no Docker, a batch orchestrator over whole hackathons, a cross stack parity
instrument, and a team facing report card. The offline suite is green at 347 passing with 18 skipped
for want of Docker or a browser (365 pass in a checkout that has both).

## What is validated

The last run graded 1043 real hackathon apps over about 30 collegiate events, URL only: 644 scored, 71
ranked DNF (broken, placeholder, or not an app), and 328 unreachable. Three results hold at that scale.

- **Precision.** 43 of 4348 fires on scored apps look suspect, about 1%, and every one is the same
  catch-all / soft 404 class. Early runs sat near 32%. The gap closed one false positive class at a time.
- **The score does not track who won.** Winners came in at median slop 61, non-winners at 64. That is
  the point. The grader measures durability, which stands on its own as a prize, not a re-ranking of the
  judges.
- **Coverage is tight.** The median app had 74% of the battery apply to it, the floor was 49%, and the
  spread was small (stdev 7.9). Scores are comparable app to app because no app was scored on almost
  nothing.

Reasoning is off by default on the perception and audit LLM calls. A paired A/B on the same apps showed
the score, the precision, and the page state classification all hold with thinking disabled (paired
median delta 0), while the audit call fell from about 36s to under 10s and shed its dominant token cost.
The working lever is `reasoning: {enabled: false}` on OpenRouter. The obvious one,
`chat_template_kwargs.enable_thinking`, is silently ignored by the provider and did nothing for a whole
run before we caught it.

## Recently landed

- **The report card** (`scripts/report_card.py`). Per finding it states what a durable app should have
  done, what we saw, what the failure indicates, and how to fix it. Public findings render in full.
  Hidden pool findings (the set that keeps the credential ungameable) show up as an opaque count in the
  team card and are itemized only in the organizer view. Both count toward the score the same way. Only
  the disclosure differs.
- **Three false positive fixes from the last full run.** The LFI signature no longer matches noise
  inside a minified bundle. Rate limiting fires only once the login endpoint returns a real
  authentication rejection, so a client side auth SPA with no server login stops tripping it. And a SQL
  injection finding is suppressed when the endpoint answers a nonexistent sibling under its own path the
  same way it answers a benign request, which kills the phantom injectable on a catch-all host that
  serves a distinct shell under a subpath.

## Do next

Run one hackathon live, inside its judging window. Every run to date grades a scraped event whose URLs
have partly rotted. A live pilot is the one thing scraping cannot test: cold starts on free tiers, live
auth walls, the difference between a team's own deployed app and a share link on someone else's
platform, and consent that comes with submitting for the prize. It is both the last validation and the
first real use.

Two known residuals to watch, both better characterized by a real pilot than by more scraping.

1. Rate limiting still fires on third party platform logins, an app submitted as an agentverse or base44
   page where the login belongs to the platform, not the team. This is the "is this the team's own app"
   question, and it mostly disappears when submissions are teams' own URLs.
2. Web Vitals is the noisiest axis on live URLs. A cold free tier instance can swing an app's score by 20
   points or more between grading days. Warm the app before measuring, or take a median of several
   samples.

## Why the deep security probes read N/A on this cohort

Data integrity round-trip, the race probe, and three of the four IDOR probes never fired on the 1043-app
run. That is correct, not a bug. Each one has to register two accounts and exercise a real server side
create and read on the app's own backend. The scraped URL cohort is mostly static SPAs with client side
or vendor hosted auth, platform pages that cannot be self registered, and backends crippled by missing
keys, so the precondition never holds and the probes read N/A instead of inventing a finding. They light
up where the surface exists: the league's self contained builds (24 minutes each), and a live pilot with
real registerable apps. `sec-idor-001` shares the same gate and did apply, which is the tell that this is
the cohort, not the code.

## Settled: do not relitigate without a reason

- Deploy planner model is `qwen/qwen3.7-plus`. Cheap, and strong at abstaining rather than hallucinating.
- Reasoning off is the default for perception and audit. `--llm-reasoning` turns it back on.
- Results are append only JSONL, deduped at read time. Resume a batch by re-running the same `--results`
  file. It retries only the untested or failed apps.
- Grading gets its own killable budget (`--grade-timeout`), separate from build. A shared total starved
  grading after slow deploys.
- Scoring stays unbounded and deduction only. No 0 to 100 normalization.
- Non-web apps are skipped, not faked.
- Multi hackathon batches run breadth first across slugs, so a partial or interrupted run is still a
  balanced sample. `--no-interleave` restores slug by slug.
- The LLM perceives surface and audits coverage. It never sets the score. Determinism comes from temp-0
  plus a per commit cache of the discovered surface.

## What did NOT come across in the extraction

- **Git history.** Distilled into `PROJECT_LOG.md` instead.
- **The hidden probe pool.** It lives in a separate private catalog repo, by design. The report card and
  the `pool` field already handle the public and hidden split.
- **League integration.** The league's `backend/rounds/scoring.py` will consume the Slop Score at Stage
  5. Today there is no code link, only anticipatory comments there, so nothing is broken.
