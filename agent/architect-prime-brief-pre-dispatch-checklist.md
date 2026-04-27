---
name: agent/architect-prime-brief-pre-dispatch-checklist
confidence: experimental
captured_during: intake-w3-v0.1
captured_at: 2026-04-26
correlation_id: 28878d40-59e5-452f-9681-ae88d789acab
applies_to: [studio-architect, brief-authoring, agent-pipeline]
supersedes: []
---

# Pre-dispatch checklist for ARCHITECT_PRIME (Studio Architect) briefs

## Why this skill exists

Three Studio Architect failure modes recurred across the first three Goliathus
briefs (`template-astro-content-v0.1`, `heart-engine-v0.1`,
`intake-w3-v0.1`), surfaced by SENTINEL pre-build review each time:

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

### Failure example (Heart Engine v1, W3 v1)

Heart Engine M6 added five canon pointers to atomic functions:

> Foundations Inventory Layer 3 §project-phase-state-machine

The Foundations Inventory file did not have a `§project-phase-state-machine`
subsection. SENTINEL flagged at pre-build review. Same class of failure
recurred in W3 v1 with `Foundations Inventory Layer 4 §intake-funnel`
and `§correlation-id-propagation`.

### Success example (W3 v2 + future briefs)

Studio Architect, before final brief save, runs:
```bash
# For each canon reference in the brief:
grep -in "intake-funnel" /mnt/project/GOLIATHUS-FOUNDATIONS-INVENTORY.md
# Returns: (no output)
# → Reference does not exist. Decide Path A or Path B.

grep -in "16-question-scoring" /mnt/project/GOLIATHUS-INTAKE-BOOK.md
# Returns: matched line numbers
# → Reference legitimate. Keep.
```

### Tooling shortcut

A small script `verify-canon-refs.sh` can be kept alongside the brief
authoring directory:

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

## Check 3 — dedup_key stability trace

**Failure mode it prevents:** specifying an idempotency check whose
dedup_key varies per call, making the check inert (dead code that
appears to work but always misses).

### Procedure

For every state-changing function in the brief that claims idempotency
(SECURITY DEFINER + dedup_key + early-return-on-match pattern):

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

4. If dead: pick remediation path:
   - **Path A** (genuine idempotency): drop non-deterministic inputs from
     the dedup_key. Rely only on caller-supplied identifying parameters.
   - **Path B** (no idempotency): remove the dedup_key + idempotency check
     entirely. Comment explicitly: "verb is intentionally not deduplicated;
     ops triages duplicates."
   - **Path C** (caller-controlled): function accepts `p_dedup_key` from
     caller. Caller constructs deterministic dedup_key (e.g.
     `clientHash:projectName`). Pushes responsibility to caller.

5. Document the choice in the function's header comment so future readers
   understand whether idempotency is feature, deferred, or caller-controlled.

### Failure examples

**Heart Engine M1 (caught by BUILDER at smoke test, not pre-build):**
constraint defined as `DEFERRED` while brief described `IMMEDIATE` semantics.
The mismatch was about transaction-time vs statement-time enforcement, but
the underlying issue was the same: a discipline named in the spec didn't
behave as the spec claimed.

**W3 v1 M1 (caught by SENTINEL at pre-build):**
```sql
v_submitted_at TIMESTAMPTZ := now();
...
v_dedup_key := 'intake.submitted:' || p_source || ':' ||
  md5(p_email || v_submitted_at::text || coalesce(p_brief_payload::text, ''));
```
`v_submitted_at` derived from `now()` — non-deterministic. Two calls one
second apart produce different `v_submitted_at` → different md5 →
different dedup_key → idempotency check always misses.

### Success example (W3 v2)

```sql
v_dedup_key := 'intake.submitted:' || p_source || ':' ||
  md5(p_email || coalesce(p_brief_payload::text, ''));
```
Inputs: `p_source` (caller-supplied), `p_email` (caller-supplied),
`p_brief_payload` (caller-supplied). All deterministic. Same caller
intent → same dedup_key → check fires correctly.

### Quick mental test

When reading any function spec, ask: "If I copy-paste this exact call
twice in a row, do I get one row in the projection or two?"

- "Two" = idempotency dead. Investigate dedup_key construction.
- "One" = idempotency live. Verify that's the intended semantic.
- "Doesn't matter" = legitimate non-idempotent design (e.g. logging
  every audit attempt). Document explicitly so reader doesn't mistake it.

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

## What this skill does not cover

- **Novel architectural concerns** — SENTINEL's primary value. This
  skill catches recurrent classes only.
- **Brand voice / vocabulary checks** — handled by separate
  `agent/vocabulary-self-check.md` skill.
- **PC5 capability-naming on skills** — handled by separate
  `agent/skill-naming-pc5-check.md` skill (not yet authored; flag for
  future).
- **Blast-radius classification** — separate concern; see
  `CLAUDE.md §7` matrix.
- **EVIDENCE template adherence** — that's BUILDER discipline,
  not Studio Architect discipline.

## Run-time estimate

Adding this checklist to the brief-authoring workflow costs roughly
**5-10 minutes per brief**:

- Check 1 (inherited deps): 2-3 minutes if grep tooling is set up
- Check 2 (canon refs): 2-4 minutes (script-aided)
- Check 3 (dedup_key trace): 1-3 minutes per state-changing function
  (typically 1-3 functions per brief)

Cost vs. savings: each SENTINEL pre-build review cycle that surfaces a
recurrent failure costs roughly 30-60 minutes (modification work +
re-review). Catching three such failures pre-dispatch saves
~90-180 minutes. ROI is 10-20x.

## Closing note

This skill exists because three briefs in, the same classes of failure
kept surfacing. The right place to fix recurrent failure is not at the
gate that catches it (SENTINEL is doing its job correctly), but at
the upstream source (Studio Architect brief authoring). This skill is
the upstream fix.

If a fourth class of recurrent failure surfaces in W4 or W5 brief reviews,
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
