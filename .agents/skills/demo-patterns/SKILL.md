---
name: demo-patterns
description: >
  Apply the codified better-demo patterns (DP-* rules) when building, enriching,
  reviewing, or deploying any ph-* workshop repo. Loads the pattern catalog from
  ../docs/DEMO-PATTERNS.md and provides per-phase checklists for workshop-orient,
  workshop-do, workshop-act, quality-enrichment, and verify-content work. Use when
  authoring modules, writing qa-automation, structuring automation/gitops, gating
  publish, or auditing an existing demo for quality.
---

# Demo Patterns

The distilled patterns from mining the live ph-* corpus (2026-11-10): what separates
the demos that work and teach from the ones that rot. The catalog is the source of
truth — this skill only tells you WHICH rules bite in YOUR phase and how to check
them.

**Read first:** `../docs/DEMO-PATTERNS.md` (DP-01 … DP-27, severity mapping at the end).

## Phase checklists

### Orient (planning a new workshop or re-orienting an existing one)
- DP-16: plan the schedule-tier table and the teardown exercise if the lab provisions
  billable resources; plan long-running work as declared-and-decoupled.
- DP-22: read `publishing-house/worklog.yaml` FIRST on an existing repo — known drift
  is planning input, append new drift instead of silently fixing.
- DP-24: set spec.yaml statuses in template vocabulary (`not_started`, not `planned`).
- DP-25: if synthesizing `spec/modules/*.md` from RAC, note they are design
  snapshots — schedule the pre-publish re-sync (DP-25) now.

### Do (writing content + automation)
- DP-13 module skeleton; DP-10 expected output after every command; DP-12 callouts as
  the debugging-scar ledger; DP-14 mechanism-before-commands; DP-15 transitions.
- DP-01/DP-02: write the `=== Verify` (literal expected value + one negative test)
  WHILE writing the exercise, not after.
- DP-17: `ui-config.yml` tabs match prose, exist at lab start; DP-18: runnability is
  a decision (`role="execute"` only for in-terminal commands).
- DP-20: new attribute ⇒ antora.yml inventory with canonical placeholder; never
  hardcode versions/cluster values; `subs="specialcharacters,attributes"` on
  substituting blocks.
- DP-21: any Antora extension = local `content/lib/*.js` + WHY-comment in `site.yml`.
- DP-19: multi-version content branches fragments, never whole exercises.
- DP-06/07/08: cluster layer via app-of-apps chart, hook Jobs own transitions,
  patched fields excluded from Git.

### Act (deploy + QA + publish)
- DP-03/04/05: e2e = student replay with `M<N>-Ex<N> |` names, healthcheck separate,
  idempotency discipline, honest skip ledger. DP-05c identity boilerplate in QA plays.
- DP-05a: BEFORE publish, mechanically verify every solve/validate placeholder has
  same-branch playbooks (the autoshift failure).
- DP-09: secrets through the three named mechanisms; run the §0 grep.
- DP-24: final `development.*` status ritual; e2e/healthCheck only `complete` after a
  live-cluster run.
- DP-25/26: spec-vs-reality check + unused-machinery sweep, then publish.

### Quality (quality-enrichment / verify-content runs)
- Audit against DP-01, DP-10, DP-12, DP-13, DP-26 (content) and DP-25 (frame) —
  these are the mechanically checkable ones; severity mapping in the catalog.
- Gold-standard metrics to beat (ph-deploy-configure-rhoai): 40 Verify sections /
  7 modules, e2e replaying M1-M6, 0 orphan images.

## Mechanical checks you can run (grep-level)

```bash
# DP-01: exercises without a Verify (page-level smell)
grep -c '^== Exercise' page.adoc; grep -c '^=== Verify' page.adoc
# DP-05a: placeholders without backing playbooks
grep -rl 'solve-button-placeholder\|validate-button-placeholder' content/ \
  | while read p; do m=$(basename "$p" .adoc | grep -o 'module-[0-9]*'); \
    ls "runtime-automation/$m/" 2>/dev/null || echo "MISSING playbooks: $m ($p)"; done
# DP-09/§0: secrets
grep -rn "AKIA\|aws_secret\|hosted_zone" --include='*' . | grep -v '.git/'
# DP-25: spec module outlines referencing non-existent paths
grep -rhoE 'manifests/[a-z0-9/_.-]+' publishing-house/spec/ | sort -u \
  | while read p; do [ -e "$p" ] || echo "STALE spec path: $p"; done
```

## Related Skills

- `workshop-orient`, `workshop-do`, `workshop-act` — the phases that apply these rules
- `quality-enrichment` — audits against the Warning-severity subset
- `verify-content` — the mechanical gate; DP-* checks complement B.*/E.*/P.* checks
- See `../docs/DEMO-PATTERNS.md` for the full catalog with evidence paths
