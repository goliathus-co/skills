---
name: agent/architect-prime-brief-pre-dispatch-checklist
confidence: experimental
captured_during: intake-w3-v0.1
captured_at: 2026-04-26
amended_at: 2026-04-27
amended_during: v2-w4-brief-v1-verdict
correlation_id: 28878d40-59e5-452f-9681-ae88d789acab
amendment_correlation_id: c2b03d8b-0abd-448e-bb15-1dcc4e7a881d
applies_to: [studio-architect, brief-authoring, agent-pipeline]
supersedes: []
version: 1.1
---

# Pre-dispatch checklist for ARCHITECT_PRIME (Studio Architect) briefs

## Why this skill exists

Three (now five) Studio Architect failure modes recurred across the first
four Goliathus briefs (`template-astro-content-v0.1`, `heart-engine-v0.1`,
`intake-w3-v0.1`, `goliathus-site-v2-v0.1`), surfaced by SENTINEL pre-build
review each time:

1. **Inherited spec errors** — vendoring patterns from prior briefs without
   updating them with BUILDER's deviation fixes. Tailwind dependency error
   discovered and fixed by BUILDER at template T1 close reappeared verbatim
   in W3 brief T2.2 dependency table.

2. **Canon-existence claims without verification** — brief asserted
   "verified before dispatch" (Heart Engine M6 canon pointers, W3
   carry-forward #3) while referencing canon sections that did not exist.

3. **Dedup_key dead code** — state-changing function specified an
   idempotency check whose dedup_key construction varied per call,
   making the check functionally inert. Same class as Heart Engine M1
   `IMMEDIATE` vs `DEFERRED` constraint contradiction.

4. **(NEW v1.1) PC5 skill naming recurrence** — provider names leak into
   skill slugs despite frontmatter prose-promise. Caught at template
   v1 (Sanity), Heart Engine v1 (Supabase × 2 + Postgres), W4 v1
   (Sanity × 3 again). Prose self-check insufficient; structural grep
   needed.

5. **(NEW v1.1) Existing atomic function source not read before
   specifying caller** — brief assumed `project_advance_phase()` accepts
   multi-step jumps (`foundation → cabin`); SENTINEL verified actual
   function source rejects skips per Master Plan PC2 ("project lifecycle
   is one-directional"). Brief author specified invocation pattern
   without reading function body.

Each failure was caught by SENTINEL pre-build review, costing ~1 review
cycle plus modification work. This skill moves detection upstream — to
the Studio Architect's own pre-dispatch pass — so most or all of these
issues resolve before SENTINEL ever sees the brief.

The skill is **not** a replacement for SENTINEL review. SENTINEL still
reviews. This is a self-discipline pass that catches the recurrent classes,
which lets SENTINEL spend its scrutiny on novel architectural concerns
rather than repeating "you forgot to update the dep pin again."

## When to run this checklist

Run **before** sending a brief to SENTINEL for pre-build review. Run
after the brief content is structurally complete (all tasks T1-Tn drafted,
verification gates V0-Vn defined, deliverables enumerated). Run before
the final commit/save of the brief markdown file.

Do **not** run this during brief authoring — it disrupts flow. Authoring is
a generative phase; this checklist is a gate phase. Separate the modes.

If any check fails, fix or annotate before dispatch. Do not pass forward
unresolved failures.

## Check 1 — inherited dependency pin verification

**Failure mode it prevents:** vendoring obsolete or wrong-version dependency
pins from prior briefs without checking BUILDER's discovered deviations.

### Procedure

For every dependency listed in the brief's Version Pin tables (T1.x or
T2.x typically):

1. Identify the prior brief(s) the pattern was vendored from. Brief usually
   names this explicitly ("vendored from `template-astro-content`", "same
   stack as Heart Engine"). If unclear, grep prior briefs for the same
   library name.

2. For each prior brief, search its closing artifacts (SENTINEL final
   verdict, BUILDER dispatch summary, decisions log) for terms like:
   - `deviation`
   - `BUILDER discovered`
   - `T<n> close — actual`
   - `spec was wrong`
   - `corrected`
   - the dependency name itself

   ```bash
   # Example grep over saved SENTINEL verdicts and BUILDER summaries:
   grep -rin "@astrojs/tailwind\|tailwind" \
     /path/to/prior-brief-artifacts/
   ```

3. If a deviation is found: cross-reference the current brief's spec.
   Does it inherit the wrong version? If yes → update to the corrected
   version. If the new brief is intentionally re-vendoring the original
   to verify the deviation still applies → flag explicitly with
   rationale in the brief.

4. For dependencies not previously used: skip this check. New pins are
   subject to a different validation (publish status of the named version
   on the registry — which BUILDER will discover at T1 install if wrong).

### Failure example (W3 v1)

W3 v1 T2.2 specified `@astrojs/tailwind ^6.0.0`, vendored from `template-astro-content`
brief's T1.2 spec. Template T1 close included a deviation entry:
"`@astrojs/tailwind` v6 does not exist; corrected to `@tailwindcss/vite ^4.0.0`."
W3 brief author missed this deviation when vendoring. SENTINEL caught it as M3
on pre-build review. With this checklist applied at brief authoring time, M3
would not have surfaced.

### Field test results

- W4 v1 (first brief authored after skill committed): Check 1 caught
  Tailwind 4 + pnpm + robots.txt all in carry-forward; zero inherited
  spec errors in v1. Skill working as designed.

## Check 2 — canon reference grep verification

**Failure mode it prevents:** asserting "canon section exists" while
referencing sections that don't.

### Procedure

For every canon reference in the brief — any string matching
`<DOCUMENT>.md §<section>`, `<DOCUMENT> §<section>`,
`<BOOK_NAME> §<section>`, or `Layer N §<section>`:

1. Identify the canonical document file. Documents live in the studio's
   canon repo or project knowledge folder, indexed in
   `GOLIATHUS-SYSTEMIC-MAP.md`.

2. For each reference, run grep against the actual canon document:
   ```bash
   grep -i "<section-name>" /path/to/canon/<DOCUMENT>.md
   ```

3. Three possible outcomes:
   - **Found** → reference is legitimate; keep as-is.
   - **Not found** → reference is fabricated or stale. Resolution paths:
     - Path A: author/expand the missing canon section before dispatch
       (clean long-term, costs 30-60min)
     - Path B: amend brief to remove the reference or replace with a
       verified alternative (operationally faster)
   - **Found but ambiguous** (multiple matches, unclear which is meant) →
     tighten the reference, e.g. `Master Plan PC2 §events-table` rather
     than just `Master Plan §events`.

4. For references to documents not yet authored (e.g. brief mentions
   "future Sealed Book section X"): flag with explicit `[stub]` or
   `[deferred]` annotation in the brief — never silent.

### Field test results

- W4 v1: Check 2 caught all canon refs verified; CLAUDE-DESIGN-REFERENCE
  Part 7 confirmed to contain pre-flight pattern. Zero missing-canon-section
  modifications in v1. Skill working as designed.

### Tooling shortcut — `verify-canon-refs.sh`

A small script kept alongside brief authoring directory:

```bash
#!/usr/bin/env bash
# verify-canon-refs.sh <brief.md>
# Greps every "X §Y" pattern in the brief against canon docs.

BRIEF="$1"
CANON_DIR="${CANON_DIR:-/mnt/project}"

# Extract all "<doc> §<section>" patterns
grep -oE '[A-Z][A-Za-z -]+ §[a-z0-9-]+' "$BRIEF" | sort -u | while read ref; do
  doc=$(echo "$ref" | sed 's/ §.*//')
  section=$(echo "$ref" | sed 's/.*§//')

  # Map document name to filename (manual mapping)
  case "$doc" in
    "Master Plan") file="GOLIATHUS-MASTER-PLAN-v1_1.md" ;;
    "Intake Book") file="GOLIATHUS-INTAKE-BOOK.md" ;;
    "Quality Book") file="GOLIATHUS-QUALITY-BOOK.md" ;;
    "Billing Book") file="GOLIATHUS-BILLING-BOOK.md" ;;
    "Foundations Inventory") file="GOLIATHUS-FOUNDATIONS-INVENTORY.md" ;;
    "Brand DNA") file="GOLIATHUS-BRAND-DNA.md" ;;
    "Scaffold Book") file="GOLIATHUS-SCAFFOLD-BOOK.md" ;;
    "Communication Book") file="GOLIATHUS-COMMUNICATION-BOOK.md" ;;
    "Sealed Book") file="GOLIATHUS-SEALED-BOOK.md" ;;
    "Retainer Book") file="GOLIATHUS-RETAINER-BOOK.md" ;;
    "Legal Book") file="GOLIATHUS-LEGAL-BOOK.md" ;;
    "Content Book") file="GOLIATHUS-CONTENT-BOOK.md" ;;
    "Metrics Book") file="GOLIATHUS-METRICS-BOOK.md" ;;
    "Load Book") file="GOLIATHUS-LOAD-BOOK.md" ;;
    "Agent Book") file="GOLIATHUS-AGENT-BOOK.md" ;;
    *) echo "UNKNOWN DOC: $doc"; continue ;;
  esac

  if grep -qi "$section" "$CANON_DIR/$file" 2>/dev/null; then
    echo "OK    $ref"
  else
    echo "MISS  $ref  (in $file)"
  fi
done
```

Run before brief dispatch. Any `MISS` line = action item.

## Check 3 — atomic function semantic verification (extended in v1.1)

**Failure mode it prevents (v1.0):** specifying an idempotency check whose
dedup_key varies per call, making the check inert (dead code that
appears to work but always misses).

**Failure mode it prevents (v1.1 extension):** assuming behavior of
EXISTING Heart Engine function without reading actual SQL source —
e.g. assuming multi-step phase jump works when function source enforces
linear single-step transitions only.

### Procedure (v1.1)

This check now covers TWO classes of function:

**Class A — NEW state-changing functions introduced by current brief:**
For every NEW state-changing function (SECURITY DEFINER + dedup_key +
early-return-on-match pattern):

1. Locate the dedup_key construction line. Typical form:
   ```sql
   v_dedup_key := '<verb>:' || <input1> || ':' || md5(<input2> || ... || <inputN>);
   ```

2. Trace each input in the construction:
   - Is it a function parameter (caller-controlled)?
   - Is it a generated value inside the function body (`now()`, `random()`,
     `gen_random_uuid()`)?
   - Is it derived from another input deterministically?

3. Test the stability question: **if the same caller intent invokes this
   function twice with identical input, will the dedup_key be identical?**

   - If `now()` or any non-deterministic source contributes → **NO** → dead code.
   - If only caller-supplied parameters and deterministic derivations → **YES** → live.

4. If dead: pick remediation path (A genuine idempotency / B no idempotency /
   C caller-controlled). Document choice in function header comment.

**Class B (NEW v1.1) — EXISTING function calls in brief:**
For every call to an existing Heart Engine atomic function (e.g.
`projectCreate`, `projectAdvancePhase`, `intakeSubmit`, `intakeQualify`,
`intakeDecline`, `intakeConvert`, `recognizeDeferredRevenue`,
`systemClientSeed`):

1. Locate actual function source in `goliathus-admin/src/db/functions/<n>.sql`.
   Read it. Do not assume from name or prior usage.

2. Verify caller's intent matches function's actual behavior:
   - **Parameter shape:** does the brief's call signature match function's
     `CREATE OR REPLACE FUNCTION <name>(p_a, p_b, ...)` exactly?
     Including parameter types and order?
   - **Validation logic:** does the function reject inputs the brief assumes
     valid? (e.g. enum constraints, range checks, state-machine transitions)
   - **Idempotency contract:** does the brief's call shape match the
     function's dedup_key construction? Calling with different
     `p_correlation_id` per invocation when dedup_key includes
     correlation_id = idempotency check misses on retry.
   - **Transition rules:** for state-machine functions
     (`projectAdvancePhase`, `intakeQualify`, `intakeDecline`,
     `intakeConvert`), verify caller's transition request is a valid
     transition from the projection's current state.

3. If mismatch found: amend brief's invocation pattern OR amend brief's
   intent (e.g. break single multi-step call into multiple single-step
   calls).

### Failure example (W4 v1, Class B)

W4 v1 T6.5 called `projectAdvancePhase(toPhase: 'cabin')` from a project
at `current_phase='foundation'`. Brief author assumed multi-step jump
acceptable. SENTINEL read function source — found `CASE` table enforcing
linear single-step only:

```sql
v_valid_next := CASE v_current_phase
  WHEN 'foundation'   THEN 'connections'
  WHEN 'connections'  THEN 'floors'
  ...
END;
```

Function returns SQLSTATE 22023 on skip attempts. Caught as M2.
Remediation: split single multi-step call into four sequential
single-step calls at T2/T4/T5/T6 gate closes (Path A).

### Quick mental test

When reading any function call spec, ask:
- "If I copy-paste this exact call twice in a row, do I get one row or two?" (Class A — idempotency)
- "Have I read this function's source today, or am I assuming behavior from name?" (Class B — semantic verification)

If Class B answer is "assuming" — read source before dispatch.

## Check 4 (NEW v1.1) — PC5 audit grep on skill names

**Failure mode it prevents:** provider names leak into skill slugs
despite frontmatter prose-promise to use capability-first naming.
Recurred at four briefs in a row: template (Sanity × 1), Heart Engine
(Supabase × 2 + Postgres × 1), W3 (none caught — successful), W4 v1
(Sanity × 3).

### Procedure

For every skill captured or referenced in the current brief:

1. Extract all skill slugs from the brief. Pattern matches:
   - `<category>/<slug>.md` in deliverables checklists
   - "Skill captured: `<path>`" in task narrative
   - "Trap captured: `<path>`" in task narrative

   ```bash
   # Extract all skill paths from brief:
   grep -oE '`[a-z-]+/[a-z0-9-]+\.md`' "$BRIEF_FILE" | sort -u
   ```

2. For each slug, run provider-name regex against the **slug only**
   (not body — body legitimately cites current implementation):

   ```bash
   # Provider blocklist — extend as new providers added to studio stack
   PROVIDERS="sanity|strapi|payload|contentful|directus|wordpress|ghost|"
   PROVIDERS+="clerk|auth0|supabase-only|postgres-only|neon|planetscale|"
   PROVIDERS+="resend|sendgrid|mailgun|"
   PROVIDERS+="vercel-only|netlify-only|cloudflare-only|"
   PROVIDERS+="stripe|paddle|lemonsqueezy"

   for skill in $skill_slugs; do
     basename=$(basename "$skill" .md)
     if echo "$basename" | grep -iE "^($PROVIDERS)"; then
       echo "PC5 VIOLATION: $skill"
     fi
   done
   ```

3. **Edge cases — when provider name in slug IS legitimate:**
   - **Canonical-platform skills** (per Amendment-01 PC8): CF Pages,
     Vercel, Supabase named explicitly because they are non-swappable
     studio commitments. Slugs like `cloudflare/cloudflare-pages-deploy.md`
     are acceptable. Body must still describe capability.
   - **Implementation files** (not skills): `src/lib/sanity.ts` is the
     module; capability-named skill describes it. The MODULE keeps the
     provider name (it IS the implementation), the SKILL doesn't.
   - **Provider-specific traps** that genuinely don't generalize: e.g.
     `traps/supabase-cli-include-directive-quirks.md` is repo-local
     because the bug is Supabase CLI specific, no other CLI has it.
     Acceptable IF in `repo-local docs/traps.md`, NOT in cross-cutting
     `goliathus-co/skills/traps/`.

4. If violation found:
   - Rename slug to capability-first: `cms/sanity-x.md` → `cms/headless-cms-x.md`
   - Body may still cite Sanity as current implementation
   - Update all references in brief to new slug name

### Failure example (W4 v1)

W4 v1 T4 deliverables specified:
- `cms/sanity-project-provisioning-checklist.md`
- `cms/sanity-fixture-seeding-pattern.md`
- `traps/sanity-token-scoping.md`

All three contain `sanity` in slug. SENTINEL caught as M1.
Remediation: rename to `cms/headless-cms-project-provisioning-checklist.md`,
`cms/cms-fixture-seeding-pattern.md`, `traps/cms-token-scoping-discipline.md`.

### Frontmatter prose-promise is insufficient

Briefs from intake-w3 onwards include prose like "PC5 self-trap remains
active: capability-first naming on every skill." Despite this written
self-check, the violation recurred in W4 v1. The lesson is that **prose
self-checks at brief frontmatter time do not survive to brief authoring
time**. Structural grep at pre-dispatch closes the gap.

## How this skill compounds

After three or four briefs of consistent application, Studio Architect
should observe:

- SENTINEL pre-build modifications drop in count per brief
- Modifications that remain are novel architectural concerns, not
  recurrent oversights
- Briefs ship faster (fewer revision cycles before APPROVED)

If the pattern holds, this skill graduates from `experimental` to
`tested` confidence. The graduation criterion: three consecutive
briefs where pre-dispatch checks catch ≥80% of what SENTINEL would
have flagged in their absence.

If the pattern does not hold (modifications continue to recur on
checked categories), the skill is broken — investigate why, amend.

### Field test progress

Brief | v1 mods | Recurrent-class mods that pre-dispatch SHOULD have caught
---|---|---
template-astro-content v1 | 7 | (skill not yet authored)
heart-engine v1 | 4 | (skill not yet authored)
intake-w3 v1 | 4 | (skill not yet authored)
intake-w3 v2 | 0 | clean (skill not yet authored, but discipline already learned by ARCHITECT_PRIME)
**v2-w4 v1** | 5 | 2 (M1 PC5 skill naming, M2 function source not read) — both addressed by v1.1 amendment

W4 v1 was first field test of v1.0. Checks 1+2+3 worked (zero misses on
their categories). Checks 4 (PC5) and Class B of Check 3 (existing function
verification) surfaced as new pattern classes — added in v1.1 amendment.

Next two briefs (W5+) will test v1.1. If they hit zero pre-dispatch-class
mods, skill graduates from `experimental` to `tested`.

## What this skill does not cover

- **Novel architectural concerns** — SENTINEL's primary value. This
  skill catches recurrent classes only.
- **Brand voice / vocabulary checks** — handled by separate
  `agent/vocabulary-self-check.md` skill (recommended next, not yet authored).
- **Blast-radius classification** — separate concern; see
  `CLAUDE.md §7` matrix.
- **EVIDENCE template adherence** — that's BUILDER discipline,
  not Studio Architect discipline.

## Run-time estimate (v1.1)

Adding this checklist to the brief-authoring workflow costs roughly
**8-15 minutes per brief**:

- Check 1 (inherited deps): 2-3 minutes if grep tooling is set up
- Check 2 (canon refs): 2-4 minutes (script-aided)
- Check 3 (function source verification, Class A + Class B): 2-5 minutes
  per state-changing function (typically 1-3 functions per brief; existing
  function check adds ~1min per existing function call)
- Check 4 (PC5 skill grep): 1-2 minutes (single grep pass)
- Total: 8-15 min per brief

Cost vs. savings: each SENTINEL pre-build review cycle that surfaces a
recurrent failure costs roughly 30-60 minutes (modification work +
re-review). Catching three such failures pre-dispatch saves
~90-180 minutes. ROI is 8-15x at v1.1 cost basis.

## Closing note

This skill exists because four briefs in, the same classes of failure
kept surfacing. The right place to fix recurrent failure is not at the
gate that catches it (SENTINEL is doing its job correctly), but at
the upstream source (Studio Architect brief authoring). This skill is
the upstream fix.

If a sixth class of recurrent failure surfaces in W5 or W6 brief reviews,
extend this skill with a new check. If a class drops below the recurrence
threshold (zero occurrences across 5+ briefs), consider removing the
corresponding check to keep the skill focused.

---

**Trap captured during skill authoring (cross-cutting):**
The "deviation directory" structure (`~/goliathus-briefs/closed/`)
referenced in Check 1 does not yet exist as a maintained operator-curated
artifact. Until operator action establishes it, Check 1's grep step
operates on whatever ad-hoc location SENTINEL verdicts and BUILDER
summaries are saved to. Recommend operator commits to a canonical
directory structure as a separate small task — not blocking this
skill's adoption, but limiting its full effectiveness.

**v1.1 amendment notes (2026-04-27):**
Two new checks added based on W4 v1 SENTINEL field-test feedback:
- Check 4: PC5 audit grep on skill names — closes prose-promise-insufficient
  recurrence pattern at structural layer
- Check 3 extension (Class B): existing atomic function source verification
  — closes "assumed-behavior-from-name" failure mode caught at W4 v1 M2

Both extensions remain `experimental` confidence until field-tested on W5+
briefs. If two consecutive briefs hit zero recurrent-class mods including
these new categories, skill graduates to `tested`.


### Success example (W3 v2 + future briefs)

Before sending to SENTINEL, Studio Architect runs:
```bash
grep -rin "tailwind\|cloudflare-adapter\|astro" \
  ~/goliathus-briefs/closed/template-astro-content-v0.1-DISPATCH-SUMMARY.md
```
Finds: "deviation #1: @astrojs/tailwind v6 does not exist, used @tailwindcss/vite v4."
Updates W3 brief T2.2 dep table accordingly before dispatch.

### What "found a deviation" looks like in a saved artifact

Studio Architect should keep a directory of prior brief artifacts:

```
~/goliathus-briefs/
├── closed/
│   ├── template-astro-content-v0.1-FINAL-VERDICT.md
│   ├── template-astro-content-v0.1-DISPATCH-SUMMARY.md
│   ├── heart-engine-v0.1-FINAL-VERDICT.md
│   ├── heart-engine-v0.1-DISPATCH-SUMMARY.md
│   ├── heart-engine-v0.1-T7-CLOSE-OBSERVATIONS.md
│   └── intake-w3-v0.1-FINAL-VERDICT.md
└── active/
    └── <current brief in authoring>
```

The DISPATCH-SUMMARY files are operator-curated when each brief ships,
listing every deviation BUILDER discovered + how it was resolved.
This is what `grep` searches over.

If this directory does not yet exist as an organized artifact set,
operator action: collect closing artifacts from previous SENTINEL
verdicts and dispatch summaries into the structure above as they
ship. This skill assumes the directory exists.

