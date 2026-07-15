# PROJECT LOG

This project was extracted from the `hacklet-league` monorepo without its git history. The pre-extraction
work, roughly 130 commits over three days (2026-07-08 to 07-10), is distilled below as arcs 1 through 7.
The work since, arcs 8 through 12 (2026-07-11 to 07-15), happened in the league monorepo under
`fuzz-runner/` and synced here commit for commit. The appendix carries the raw dated subjects. This is
the why behind the code. When something looks deliberate, it probably is.

## The arcs

**1. The vertical slice (07-08 → early 07-09).** Prove `deploy → discover → applicability → execute →
aggregate → report` end to end against two reference apps with the same surface: a **vulnerable** one
that accrues slop and a **hardened** one that stays clean. Three probes, one per bundle
(security / qa / performance). Established the catalog YAML format, the untrusted-submission ingest
(zip-slip guarded), and the applicability model (a probe that doesn't apply is N/A, not a pass).

**2. The scoring model.** Settled the core measurement decisions that everything else depends on:
deduction-only **slop** (lower is better), **unbounded**, decomposed **per axis** — and deliberately
*not* normalized to 0–100 (that was tried and reverted; normalization destroys the "how bad, in absolute
terms" signal). Penalties are **risk-priced** = frequency × severity. Added **dampers** so one weakness
can't dominate: a variant group fires once, and penalties decay geometrically within a category. Priced
the security headers (CSP 8→12), and — a real bug caught on DVWA — made the grader **never submit a
password-change form**, because resetting the admin password locks the grader out of its own session.
The vulnerable reference anchors at **642**.

**3. Probe breadth.** Grew the catalog toward comprehensive, intent-independent coverage: the injection
family (SQLi via boolean/auth oracle, plus XSS/SSTI/LFI/cmdi/XXE/SSRF), crash-resistance on malformed
input, security headers / CORS / compression, static source secret scan, a Supabase/Firebase
exposed-backend (missing-RLS) probe, Core Web Vitals (throttled, sampled, best-of-N), browser a11y via
axe-core, and QA probes (data-integrity round-trip, content-type correctness, debug-mode). Principle:
each vuln class runs many techniques that **collapse to one finding**, guarded so it never fires on the
hardened app.

**4. Discovery and the browser.** Turned on browser-based grading by default, then fixed the blind spots
it exposed: render **every** route (not just `/`), synthesize injection targets from **formless** SPA
inputs, and **infer field names** for anonymous name-less React controls. This is the machinery that
makes the runner see a client-rendered surface at all.

**5. The deploy + grade pipeline.** Built `deploy_and_grade.py`: hand an LLM (via OpenRouter) a hackathon
repo, get back a Docker deploy plan, run it, grade it, tear it down. Added `devpost_repos.py` (fetch a
hackathon's project repos, auto-paginating the gallery), a **batch orchestrator** with winner metadata,
and `stats.py` for auditable batch statistics. Cleaned up build output (stream behind `-v`), reclaimed
per-repo build cache and sidecar volumes, and tagged build-timeout kills distinctly — "too slow/heavy to
build" is itself a signal.

**6. The parity instrument.** The insight that reframed the work: *an undiscovered surface reads as a
clean surface*, so blindness biases the score. Built a **cross-stack parity dashboard** — the LLM
identifies the stack and the source-implied **expected** surface, the runner records the **observed**
surface, and parity compares them by stack. Added a **test-coverage fingerprint** (% and *kinds* of
tests that ran vs N/A) and **per-phase wall-clock** timings (clone/plan/deploy/grade) as measurement,
not just gates.

**7. Hardening from real batches (07-10).** Running actual hackathons surfaced the messy-reality bugs and
we fixed them in a tight loop: a **gradeability gate** (skip mobile/CLI/notebook apps instead of faking a
server that poisons the dataset) plus a **feature inventory** as parity ground truth; a **three-layer
timeout defense** ending in grade-subprocess isolation (the lesson that only an external SIGKILL stops a
GIL-holding C-spin); **env-var-dead endpoint** baselining so dummy-key 500s don't confound findings; a
batch of precision fixes from a review of the Civic Hacks run (api-only seeding, vendor-surface
stripping, finding dedup, a rate-limit false positive); **stack-ID retention on wedged apps**;
**resume/skip** so a re-run only retries the untested/failed; the deploy model settled on
**`qwen/qwen3.7-plus`**; and most recently a **build-download cache** so heavy-deps apps stop
re-downloading gigabytes on every retry.

**8. Grade the live URL, not just the repo (07-11).** A self deployed repo ships without the team's
secrets, so its backend runs half dead. The live URL the team already deployed to demo is the honest
artifact. Added a raw fuzz path for live app URLs (`--url`, `--urls`), graded the repo and the live URL
as two source tagged lenses on the same submission, split the stats by cohort, and taught the ingest to
reject a dead deployment and spot a broken live site (a client side 404, a placeholder page) instead of
grading an empty shell.

**9. Reproducibility and the LLM's bounded role (07-11 to 07-12).** An objective score has to be
reproducible, and an LLM is stochastic, so the LLM was let in through one door only. It decides where to
look. It never decides the score. temp-0 on both LLM reads, a per commit cache of the deploy plan, and a
per commit cache of the whole discovered surface, so a re-grade replays the exact crawl and never calls
the model again. A coverage auditor runs on every app and prints what discovery missed, off to the side
of the number. The LLM also points injection at source declared params and body fields, with telemetry
that measures how often the pointer was right and never lets it touch the score. Interaction aware
discovery began clicking client side create actions (New Board, Add Card) that a static crawl never sees.

**10. The precision arc (07-12 to 07-14).** The number that decides whether a machine can grade at all.
Built a false positive audit (`precision.py`) that measures residual FP on scored apps, credited the DNF
gate so a broken or placeholder app ranks below every running app instead of scoring a misleading clean
zero, and closed false positive classes one at a time. Phantom fires suppressed on catch all SPAs. CTA
style login and signup credited, not only password forms. The DNF gate hardened with a source dump
detector and a corroboration veto. Then the innocence check, which presumes every endpoint a phantom
until its response differs from the catch all shell. Proactive perception landed in the same window: the
LLM perceives the interactive surface and the deterministic probes judge it, which ends the chase after
every new SPA convention. Suspect fires fell from about 32% to about 1%.

**11. SPA auth and the cookieless cohort (07-13).** The bolt and Supabase and Firebase cohort
authenticates by a JWT in localStorage and an Authorization header, not a cookie, so the cookie based
session and IDOR probes went dark on the apps that dominate the field. Added registration driven by a
browser that fills and submits a signup, captures the bearer or localStorage token, and reaches signups
on separate routes, plus a provided session fallback (`--header`) for a login the runner cannot complete
itself. That woke a localStorage session exposure probe (`sec-session-005`), a user record IDOR
(`sec-idor-003`, the canonical `/user/124` case), and a missing row level security probe on a managed
Supabase backend (`sec-idor-004`), all under one rule: test the team's own config, never the vendor's
platform. Batches also learned to span many events with the limit balanced across the slugs.

**12. Validation at scale, reasoning off, and the report card (07-14 to 07-15).** Probes that read a
client bundle for a leaked server secret and flag a served source map, a check for a present but
toothless CSP, then the shift to reasoning off by default on the perception and audit calls. A paired
A/B on 1043 apps showed the classification and the effect on the score hold with thinking disabled,
while the audit call dropped from about 36s to under 10s and shed its dominant token cost. The working
lever is `reasoning: {enabled: false}` on OpenRouter, found after the obvious `chat_template_kwargs`
lever turned out to do nothing for a whole run. Two false positive fixes from the full run, an LFI
signature that matched minified bundle noise and a rate limit finding that fired without a real auth
backend, plus a phantom SQL injection fix for a host that serves a distinct shell under a subpath. And
the report card, the piece a mandatory credential needs. Each team gets what failed, what a durable app
does instead, and how to fix it, with the hidden pool withheld so the credential stays ungameable.

## Where to look in the code

The `CLAUDE.md` architecture map ties these arcs to modules. In short: the grader core is
`hacklet_runner/` (scoring in `aggregate.py`, discovery in `discovery.py`, probes in `probes.py`), the
pipeline is `scripts/`, the design rationale is `FUZZ_RUNNER_SPEC.md`, and the reference apps that lock
discrimination are `references/`.

---

## Appendix — raw commit log (newest first)

*From the `hacklet-league` monorepo (paths under `fuzz-runner/`). The 2026-07-11 → 07-15 block is the
post-extraction work, synced to this repo commit for commit. Subjects are verbatim, em dashes and all.*
```
2026-07-15  feat(parity): TEST COVERAGE PER APP distribution (avg/median/stdev/min/max/quartiles)
2026-07-15  fix(precision): kill phantom SQLi on per-prefix catch-all hosts
2026-07-15  feat(reportcard): team-facing durability report card w/ public/hidden disclosure tiers
2026-07-15  fix(precision): kill the two pilot-audit FP classes — LFI signature + rate-limit auth-shape
2026-07-15  feat(llm): default reasoning OFF for perceive+audit — the A/B validated no-think
2026-07-14  fix(llm): use reasoning:{enabled:false} — the chat_template_kwargs lever was a no-op
2026-07-14  feat(batch): breadth-first round-robin across slugs so partial runs stay balanced
2026-07-14  feat(llm): --no-llm-reasoning handle disables CoT for perceive+audit passes
2026-07-14  feat(security): sec-csp-001 — grade a present-but-toothless CSP (Tier-2)
2026-07-14  test: response_leaks_secret fires on a realistic AWS key, not the AKIA…EXAMPLE placeholder
2026-07-14  feat(security): SPA-native probes — client-bundle secret mining + source-map exposure
2026-07-14  feat(precision): innocence check — never fire a phantom finding on a catch-all shell
2026-07-13  feat(batch): show batch progress + ETA in the non-tldr full-dump too
2026-07-13  perf(deploy): cache apt downloads + fix yarn/pnpm cache dirs in build injection
2026-07-13  perf(auth): memoize the browser registration per identity in ctx.register
2026-07-13  feat(security): sec-idor-004 managed-backend RLS IDOR (Supabase) — #2
2026-07-13  feat(security): sec-idor-003 user-record IDOR — the canonical /user/124 case (#1)
2026-07-13  feat(auth): bearer-authed IDOR + Option-B provided-session (--header)
2026-07-13  feat(security): sec-session-005 localStorage session-exposure probe + re-gate auth cluster (A3)
2026-07-13  feat(auth): bearer-authed Account + has_auth_entrypoint capability (A2)
2026-07-13  feat(auth): browser register captures Bearer/localStorage token + reaches separate-route signups (A1)
2026-07-13  fix(stats): sequential section labels (a,b,c,c2,d…) matching print order
2026-07-13  test(auth): give test_api_injection's stub _Ctx a register() — phase 3 follow-up
2026-07-13  feat(auth): thread browser_register into the session/idor probes — --browser-auth (phase 3)
2026-07-13  feat(auth): register_account browser fallback — Account from the SPA's session cookie (phase 2)
2026-07-13  feat(auth): browser-driven SPA registration primitive (register_in_browser) — phase 1
2026-07-13  fix(discovery): baseline perceived endpoints so their reachable/hallucinated telemetry works
2026-07-13  feat(ingest): multi-hackathon slug ingestion — --hackathon takes a list, --limit balanced across them
2026-07-13  feat(stats): richer (i2) PERCEPTION POINTER — per-app breakdown + auth + unjudged signals
2026-07-13  feat(telemetry): record perceived endpoint paths + ghost (404) paths for per-app hallucination audit
2026-07-12  feat(grade): no grade-timeout on a direct run (batches stay bounded) + legible discovery
2026-07-12  fix(dnf): strip <style> in _visible_text — inlined critical CSS false-flagged live sites as source dumps
2026-07-12  fix(proactive): always announce perception status (found / nothing / failed)
2026-07-12  feat(proactive): print the perceived surface as it's handed to the probes
2026-07-12  feat(discovery): perception-pointer telemetry — measure how much perceived surface is real (phase 4)
2026-07-12  feat(discovery): wire proactive perception into the grade flow (phase 3)
2026-07-12  feat(discovery): merge_perceived — fold LLM-perceived surface into the Profile (phase 2)
2026-07-12  feat(discovery): perceive_surface — proactive-discovery perception pass (phase 1)
2026-07-12  feat(dnf): harden the DNF gate — deterministic source-dump detector + corroboration veto
2026-07-12  fix(audit): hard-cap the coverage-audit LLM call at 180s (daemon-thread join)
2026-07-12  feat(batch): --tldr — one compact line per app instead of the full grading dump
2026-07-12  feat(ingest): cache Devpost fetches — re-scrapes of a hackathon do ~zero network
2026-07-12  feat(discovery): capture CTA-login forms via auth-route probing (Part 2) + residual FP mop-up
2026-07-12  feat(discovery): credit CTA-style login/signup (button/link), not only password forms
2026-07-12  fix(precision): credit the DNF gate — measure residual FP on SCORED apps only
2026-07-12  feat(stats): exclude non-functional apps from the score distribution (rank DNF-class)
2026-07-12  fix(precision): broken apps -> DNF, and suppress phantom fires on catch-all SPAs
2026-07-12  feat(precision): false-positive audit for a results run — measure the fuzzer's precision
2026-07-12  fix(batch): launch children with the project .venv python, not sys.executable
2026-07-12  feat(batch): parallel URL grading (--concurrency N) + lock-guarded results append
2026-07-12  feat(discovery): interaction captures client-side create actions (New Board / Add Card)
2026-07-12  feat(scope): off-target deny-list — never fetch or probe a third-party link
2026-07-12  feat(stats): report audit(LLM) timing + the model used, in output and stats
2026-07-12  feat(batch): --repo-only / --url-only cohort filters
2026-07-12  feat(discovery): merge LLM param/body enrichment onto crawler-found endpoints (#2 gap)
2026-07-12  feat(stats): LLM-pointer precision telemetry — measure the pointer, never score it (#2)
2026-07-11  feat(anti-evasion): LLM points injection at source-declared params/body_fields (#2)
2026-07-11  feat(qa): dead-control probe — flag clickable controls wired to nothing (qa-deadctrl-001)
2026-07-11  fix(browser): bound reveal-click interaction to the first ~6 routes
2026-07-11  feat(repro): per-commit SURFACE cache — freeze the crawl+interaction (build 1b)
2026-07-11  feat(repro): temp-0 + per-commit deploy-plan cache — computational reproducibility
2026-07-11  feat(audit): multi-route coverage audit — render the head sub-routes, not just /
2026-07-11  feat(audit): run + PRINT the coverage audit on every graded app (like deploy notes)
2026-07-11  fix(stats): tolerate a concurrent-append-corrupted line instead of crashing
2026-07-11  feat: interaction-aware discovery + LLM coverage auditor (synced from standalone)
2026-07-11  feat(url-ingest): detect broken live sites — client-side 404s + placeholder pages
2026-07-11  fix(url-ingest): reject dead deployments cleanly, don't crash or grade a 404 shell
2026-07-11  fix(stats): (b2) = probes N/A on EVERY app (never-applied), not never-fired
2026-07-11  feat(stats): source cohorts + paired repo-vs-url delta + never-fired probes
2026-07-11  feat(batch): grade repo AND live URL as separate source-tagged lenses + resume ladder
2026-07-11  feat(batch): raw-fuzz live app URLs (--url/--urls) + --no-repeat + cohort-split stats
2026-07-10  feat(deploy): cache pip/npm downloads across build attempts + capped cross-app reuse
2026-07-10  fix(parity/probes): recognize API login/upload; show + record the payload that fired a probe
2026-07-10  feat(batch): grade gets its OWN killable budget; resume skips already-done apps
2026-07-10  chore(batch): raise --app-timeout default 900s -> 1000s
2026-07-10  fix(discovery/probes): env-var-dead endpoints don't confound findings or surface
2026-07-10  feat(batch): retain the LLM stack-ID on wedged apps (deploy-parity)
2026-07-10  fix(discovery/precision): civic-batch review — api-only seeding, vendor surface, dedup, rate-limit FP
2026-07-10  feat(measure): record per-phase wall-clock (clone/plan/deploy/grade/total), not just gates
2026-07-10  feat(report): test-coverage fingerprint — % + kinds of tests that ran vs n/a
2026-07-10  feat(deploy): default model -> qwen/qwen3.7-plus (cheap, strong anti-hallucinator)
2026-07-10  feat(stats/parity): app-kind distribution, out-of-scope skips, timeout signals
2026-07-10  feat(deploy): gradeability gate + feature inventory + gpt-5-mini + web-search retries
2026-07-10  fix(batch): hard per-app wall-clock kill — backstop for a CPU-spin the grade-timeout can't stop
2026-07-10  feat(deploy): wrap the LLM notes across a few lines instead of a 200-char cut
2026-07-10  fix(devpost): repo_for uses app-links only — never the embedded vendor URL
2026-07-10  fix(deploy): hard-bound the grading phase (--grade-timeout) + live heartbeat
2026-07-10  feat(batch): run_batch emits the cross-stack parity dashboard after stats
2026-07-10  feat(parity): cross-stack parity dashboard (observed vs expected surface, by stack)
2026-07-10  feat(deploy): LLM identifies stack + source-implied surface; record both + observed surface
2026-07-10  feat(runner): observed-surface fingerprint on the Report (parity denominator)
2026-07-09  feat(discovery): infer field names for anonymous SPA inputs (name-less React controls)
2026-07-09  feat(discovery): synthesize injection targets from formless SPA inputs
2026-07-09  fix(discovery): browser-render every route for forms, not just "/"
2026-07-09  feat(deploy): tag build-timeout kills distinctly from build errors
2026-07-09  fix(deploy): reclaim build cache + sidecar volumes per repo; --build-timeout (default 480s)
2026-07-09  fix(fuzz-runner): auto-paginate Devpost galleries to fill --limit (was capped at one page)
2026-07-09  feat(fuzz-runner): browser grading on by default in the deploy scripts (--no-browser to opt out)
2026-07-09  feat(fuzz-runner): batch orchestrator + winner metadata (Devpost -> grade -> stats)
2026-07-09  feat(fuzz-runner): record results + scripts/stats.py — auditable batch statistics
2026-07-09  feat(fuzz-runner): deploy grade output reconciles raw penalties with the damped score
2026-07-09  feat(fuzz-runner): deploy_and_grade prints ALL findings, not just security
2026-07-09  feat(fuzz-runner): default summary shows the point breakdown, not misleading fire-counts
2026-07-09  fix(fuzz-runner): remove the db sidecar between deploy attempts (name conflict)
2026-07-09  feat(fuzz-runner): clean build output by default, full stream behind -v
2026-07-09  fix(fuzz-runner): stream docker build output + clean up the per-repo image
2026-07-09  feat(fuzz-runner): scripts/devpost_repos.py — fetch hackathon project repos from Devpost
2026-07-09  chore(fuzz-runner): default the deploy-planner LLM to deepseek/deepseek-v4-flash (cheap)
2026-07-09  feat(fuzz-runner): scripts/deploy_and_grade.py — LLM-assisted deploy + grade of a hackathon repo
2026-07-09  feat(fuzz-runner): Supabase/Firebase exposed-backend probe (missing RLS)
2026-07-09  feat(fuzz-runner): static source secret scan (--source / auto for --submission)
2026-07-09  feat(fuzz-runner): -v appends a score breakdown showing the dampers at work
2026-07-09  fix(fuzz-runner): support a path-bearing --target (health-check + crawl-from-entry-page)
2026-07-09  fix(fuzz-runner): risk-price CSP header (8→12); confirm the rest of the security headers
2026-07-09  fix(fuzz-runner): never submit a password-change form (DVWA admin-password reset)
2026-07-09  feat(fuzz-runner): risk-reprice qa/perf penalties (risk = frequency × severity)
2026-07-09  refactor(fuzz-runner): per-axis slop decomposition (unbounded; no 0-100 normalization)
2026-07-09  feat(fuzz-runner): two FridgIT fixes — template-artifact discovery filter + debug-mode probe
2026-07-09  feat(fuzz-runner): two QA probes — data-integrity round-trip + content-type correctness
2026-07-09  feat(fuzz-runner): Core Web Vitals probe (perf-cwv-002) — throttled, sampled, best-of-N
2026-07-09  feat(fuzz-runner): ingest axe-core for browser a11y (WCAG 2.0/2.1 A/AA)
2026-07-08  feat(fuzz-runner): list_probes shows the worst-case (fully-damped) score
2026-07-08  feat(fuzz-runner): scripts/list_probes.py — catalog export (table/CSV/JSON)
2026-07-08  added one last sqli
2026-07-08  feat(fuzz-runner): decompression-bomb (zip-bomb) resistance probe (sec-dos-001)
2026-07-08  feat(fuzz-runner): host-header injection + HTTP response splitting probes
2026-07-08  docs(fuzz-runner): refresh CONTRIBUTING to current reality
2026-07-08  test(fuzz-runner): calibration self-tracks — one golden score, not four
2026-07-08  fix(fuzz-runner): page-weight must not fetch <a href> nav links (logout de-auth)
2026-07-08  feat(fuzz-runner): evidence on every probe (full transparency)
2026-07-08  feat(fuzz-runner): per-outcome evidence (stats for clean/n-a, not just slop)
2026-07-08  feat(fuzz-runner): grade HTTPS targets + end-to-end TLS pipeline coverage
2026-07-08  feat(fuzz-runner): full-cascade contrast in the browser a11y check
2026-07-08  feat(fuzz-runner): HTTP charset-conformance probe (qa-http-002)
2026-07-08  feat(fuzz-runner): SEO/meta presence probe (qa-seo-001)
2026-07-08  feat(fuzz-runner): mixed-content probe (sec-mixed-001)
2026-07-08  feat(fuzz-runner): accessibility hard-fails + broken-links probes (qa-a11y-002, qa-links-001)
2026-07-08  feat(fuzz-runner): soft-404 HTTP-correctness probe (qa-http-001)
2026-07-08  feat(fuzz-runner): static-asset caching probe (perf-cache-001)
2026-07-08  feat(fuzz-runner): general crash-resistance probe (qa-crash-010)
2026-07-08  feat(fuzz-runner): performance rubric -- tiered, published thresholds on measured primitives
2026-07-07  feat(fuzz-runner): SSRF + XXE probes via out-of-band collaborator (sec-ssrf-001, sec-xxe-001)
2026-07-07  feat(fuzz-runner): SSTI + eval code-injection probe (sec-ssti-001)
2026-07-07  test(fuzz-runner): qa-janky reference -- QA/perf calibration anchor
2026-07-07  feat(fuzz-runner): CSRF redesign (works behind --header) + weak session IDs
2026-07-07  feat(fuzz-runner): insecure file upload probe (sec-upload-001)
2026-07-07  feat(fuzz-runner): path traversal / LFI probe + link query-param capture (sec-lfi-001)
2026-07-07  feat(fuzz-runner): OS command injection probe (sec-cmdi-001)
2026-07-07  feat(fuzz-runner): comprehensive reflected + stored XSS (sec-xss-001)
2026-07-07  fix(fuzz-runner): authenticated crawling works end-to-end (DVWA-validated)
2026-07-07  feat(fuzz-runner): boolean/UNION/time-based SQLi detection (comprehensive coverage)
2026-07-07  fix(fuzz-runner): mine API paths inside template literals (SPA base-URL pattern)
2026-07-07  feat(fuzz-runner): common-param + GET-form error-based SQLi reach
2026-07-07  feat(fuzz-runner): SPA JS-bundle API-path mining (see a form-less SPA's backend)
2026-07-07  fix(fuzz-runner): scope credential-leak matcher to JSON bodies (SPA false positive)
2026-07-06  feat(fuzz-runner): API BOLA / horizontal IDOR probe (sec-idor-002)
2026-07-06  feat(fuzz-runner): excessive-data-exposure probe (sec-exposure-005)
2026-07-06  feat(fuzz-runner): resolve OpenAPI $ref/allOf body schemas (modern-API bodies)
2026-07-06  feat(fuzz-runner): error-based API SQLi probe (sec-sqli-004)
2026-07-06  feat(fuzz-runner): OpenAPI/Swagger surface discovery (see JSON-API endpoints)
2026-07-06  feat(fuzz-runner): JSON-API self-as-oracle path (reach SPA/token auth)
2026-07-06  fix(fuzz-runner): self-as-oracle correctness + CSRF handling (from Juice Shop/Gitea validation)
2026-06-27  feat(fuzz-runner): +5 probes — console-errors, accessibility, open-redirect, AWS-creds exposure, X-Powered-By
2026-06-25  feat(fuzz-runner): per-finding point value + 'why it fired' in verbose and --failed
2026-06-25  feat(fuzz-runner): live progress output — --verbose stream + default progress bar
2026-06-25  feat(fuzz-runner): authenticated probing (--header) + gzip & routing-crash probes
2026-06-25  feat(fuzz-runner): login rate-limiting probe (self-as-oracle) — flags missing brute-force protection
2026-06-25  feat(fuzz-runner): CORS-misconfig + cookie-Secure probes
2026-06-25  feat(fuzz-runner): security-header depth — CSP, HSTS, clickjacking, Referrer-Policy
2026-06-25  fix(fuzz-runner): cap dom_xss probe wall-clock so a slow-loris target can't tie it up ~3 min
2026-06-25  feat(fuzz-runner): JSON-crash probes read N/A when no JSON API is served (no fake clean)
2026-06-25  test(fuzz-runner): end-to-end browser-pipeline calibration (run with render=...)
2026-06-25  feat(fuzz-runner): load/CWV real-target coverage (homepage fallback) + collapse _fanout helper
2026-06-25  fix(fuzz-runner): determinism + gating — median-of-N perf oracles, browser capability via actual render
2026-06-25  fix(fuzz-runner): scoring accuracy — csrf redirect-rejection, race field-fill, load counts dropped connections
2026-06-25  fix(fuzz-runner): auth correctness — self-as-oracle suite works on real apps, not just the references
2026-06-25  fix(fuzz-runner): crash-resilience — hostile-target edge cases degrade one probe, never DNF the grade
2026-06-25  feat(fuzz-runner): load-resilience probe (perf) — concurrent burst, fire if >10% 5xx under load (unsynchronized shared state; perf-load-001, pen 10); references gain /report (vulnerable unsync dict crashes under concurrency, hardened locked snapshot); _concurrent_get primitive; tests (vulnerable 322)
2026-06-25  feat(fuzz-runner): race-condition probe (self-as-oracle) — fire N concurrent resource creates, flag duplicate IDs (non-atomic id allocation; qa-race-001, pen 25); references switch to ThreadingHTTPServer (vulnerable racy increment, hardened locked); concurrency primitive _concurrent_creates; tests (vulnerable 312)
2026-06-25  feat(fuzz-runner): crash-resistance battery (QA) — JSON malformed-input gauntlet (bad JSON / wrong type / missing field → 500; qa-crash-004/005/006) targeting the #1 non-security slop (error handling); references gain /api/items (vulnerable crashes, hardened validates to 400); executor gains a raw `body` field; tests (vulnerable 287)
2026-06-24  feat(fuzz-runner): browser probes — DOM-XSS oracle (render + <img onerror> execution detection; sec-domxss-001, browser-gated, pen 30) catches reflected-that-executes + DOM-sink XSS a source check misses; references gain a /dom sink (vulnerable innerHTML, hardened textContent); discovery src extraction generalized to all tags (img/iframe/source/...); browser capability gate; tests (no-browser 284, --browser 314)
2026-06-24  feat(fuzz-runner): browser harness — Playwright render for client-rendered (SPA) form/route discovery; browser-agnostic launch (bundled chromium → system chromium/chrome/msedge; Ubuntu 26.04 bundled needs Playwright ≥1.61, unreleased, so system Chrome used here); --browser flag; discovery merges rendered DOM; SPA reference + browser-gated tests; playwright optional dep-group
2026-06-24  feat(fuzz-runner): CSRF + SameSite probes (self-as-oracle) — sec-csrf-001 (cross-site POST accepted with no token + no SameSite → pen 25, auto-clears when SameSite/token defends) + sec-session-002 (missing SameSite, 15); generalize session probe to a flag-parameterized predicate; auth.is_csrf_field; tests (vulnerable 284)
2026-06-24  feat(fuzz-runner): two-account IDOR (self-as-oracle flagship) — A creates a note, B reads it → broken access control (sec-idor-001, pen 40); references gain stateful sessions + owned /notes (vulnerable any-logged-in, hardened owner-only); auth.create_form; pipeline keeps the shared client stateless (no cross-form cookie leak); tests (vulnerable 250)
2026-06-24  feat(fuzz-runner): self-as-oracle — auth.py registers the runner own account (find password form, fill, POST, capture session); sec-session-001 fires when the session cookie lacks HttpOnly; references gain /register + sessions (vulnerable no flags, hardened HttpOnly+SameSite); any_form_has_password capability; auth + calibration tests (vulnerable 210)
2026-06-24  fix(fuzz-runner): zip-bomb guard — stream-extract each member with a running cap on actual decompressed bytes (stop trusting forged file_size headers); per-member dir handling; zip-bomb test
2026-06-24  fix(fuzz-runner): review batch — _DOTENV catches TOKEN/bare-KEY/export-prefixed keys; git-head matches detached SHA; discovery attr regexes reject data-* (no-boundary mis-parse); probe emits N/A when all fan-out fetches fail (no silent vanish); _container_ip indexes the target network; RemoteDeployer requires <500 (5xx no longer "healthy"); tests
2026-06-24  feat(fuzz-runner): unauthenticated-exposure family — dotfile/.git matchers (200 + content signature, precision) + sec-exposure-001/002/003 (/.env, /.git config+HEAD variant-grouped); vulnerable reference serves them; matcher + calibration tests (vulnerable 190)
2026-06-24  feat(fuzz-runner): secrets-exposure family — response_leaks_secret matcher (high-confidence corpus, precision over recall) + sec-secrets-001 (route-fanned, variant-grouped, pen 35); discovery captures <script src> for JS scanning; references leak fake AWS key in /config.js; matcher + calibration tests (vulnerable 137)
2026-06-24  feat(fuzz-runner): CLI output modes — human summary by default, --failed for a slop-only table, --json for the full report; grouped --help with examples; renderer tests
2026-06-24  feat(fuzz-runner): forms fan-out — target: forms selector fills + submits each discovered form (GET→params, POST→body); sec-xss-001 fans across forms; reflective /search form on references; tests
2026-06-24  feat(fuzz-runner): per-target fan-out — probes expand a routes selector and fan across discovered routes (one outcome per probe×target); sec-headers-001 per-route; Outcome.target + by_id collapse; trim reference /heavy for fast suite; vulnerable 98→102
2026-06-24  feat(fuzz-runner): discovery depth — bounded same-origin crawl builds a structured surface map (routes + forms with method/fields); Profile gains routes/forms (form_endpoints derived); discovery tests
2026-06-24  feat(fuzz-runner): RemoteDeployer + --target — dogfood any live URL, no Docker (no-op teardown); remote tests; README dogfooding section
2026-06-24  fix(fuzz-runner): DockerDeployer reaches the container by IP on a custom network (-p does not route on --internal); add read-only isolation test
2026-06-24  test(fuzz-runner): verify DockerDeployer hardening (read-only + egress-blocked --internal network preserve 98/0/0; egress-block proof + control); README reflects DockerDeployer is implemented + hardened invocation
2026-06-24  feat(fuzz-runner): production DockerDeployer (build/run/health-gate/teardown, quotas + cap-drop, hardening toggles); references bind 0.0.0.0; guaranteed teardown on deploy failure; Docker-gated 98/0/0 calibration test
2026-06-24  docs(fuzz-runner): CONTRIBUTING.md — probe authoring guide (add/change/remove recipes, matcher/predicate registry, three-way calibration gate)
2026-06-24  feat(fuzz-runner): author core batch — SQLi variant group (3 syntaxes, parameterized oracle → fires once) + crash-resistance category (3 probes, /profile endpoint, response_server_error matcher, POST-body support → diminishing returns); both dampers proven end-to-end (10 fire, raw 184 → damped 98). 8 tests pass
2026-06-24  feat(fuzz-runner): aggregation dampers — variant-group-fires-once + diminishing-returns-within-category (aggregate.py); per-bundle ordering via penalty calibration, not a runtime multiplier; 9 tests pass
2026-06-24  feat(fuzz-runner): expand public catalog to 5 probes (SQLi oracle, XSS reflection, security-header, error-hygiene, TTFB) across 3 bundles; minimal reference app completes the vuln/hardened/minimal triad (N/A path); hidden-pool repo boundary (gitignore backstop + FUZZ_RUNNER_SPEC note). 4 tests pass
```
