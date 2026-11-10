# Demo Patterns — what makes a BETTER workshop demo

**Version**: 1.0 (2026-11-10)

Codified from pattern-mining the live ph-* corpus: `ph-deploy-configure-rhoai`,
`ph-openshift-autoshift`, `ph-openshift-monorepo-example`, `ph-operations-rhoai`,
plus the scaffold trio (`ph-llm-gpu-test2`, `ph-nate-whatever`,
`ph-whatever-awesome`) as the counter-example. Full evidence:
`demo-platform-investigate/findings/` (clones under `~/git/` or /sandbox; analysis in
`demo-platform-investigate/better-demo-patterns.md`).

Rules marked **[RULE]** are mandatory for new workshops; **[HABIT]** are strong
defaults. `DP-*` IDs are grepable and referenced by the pipeline skills.

---

## A. Content ↔ QA coupling (the defining trait of the best repo)

### DP-01 — Every exercise ends in a Verify with a literal expected value [RULE]
Each `== Exercise N` contains a `=== Verify` section whose commands are copy-pasteable
and whose expected output is stated literally ("STATUS shows `Running`, READY `1/1`",
"returns `{openshift_cluster_admin_username}`"). A Verify the learner cannot diff
against reality is not a Verify.
Evidence: ph-deploy-configure-rhoai 40× (`module-01-install-connected.adoc` L66-73);
ph-operations-rhoai per-exercise with expected literals (`module-02-…adoc` L152-170).

### DP-02 — Include negative tests [RULE]
Prove the control is enforced, not decorative: same request without auth → HTTP 401
(deploy-configure `module-04` L156-164); RBAC negatives via impersonation —
`oc auth can-i create notebooks.kubeflow.org -n <ns> --as=<user> --as-group=<group>`
must answer `yes` for the granted persona and `no` for the other, no persona
passwords needed (ph-operations-rhoai `qa-automation/e2e.yml` M2).

### DP-03 — e2e replays the student's commands with `M<N>-Ex<N> |` task names [RULE]
`qa-automation/e2e.yml` is the automated solve: it repeats the exact manifest targets
and waits the content teaches. Task names embed the exercise they validate
(`"M5-Ex2 | Polling for RHCL CSV"`, `"M1 | VERIFY — DSC phase is Ready"`) so a red
test names the broken exercise. Header carries the contributor contract: Purpose /
Idempotency / Scope / When to run / Runtime / Retry semantics / "Adding modules for
your lab" steps.
Evidence: ph-deploy-configure-rhoai `qa-automation/e2e.yml` L1-60, L104-666.

### DP-04 — healthcheck and e2e are different jobs [RULE]
`healthcheck.yml`: provisioning-only (<30 s, exit codes 0/2, "does NOT check anything
students install") — separates "environment broken" from "lab broken".
`e2e.yml`: learner replay, idempotent. Never blend them.
Evidence: deploy-configure `healthcheck.yml` L1-29; operations-rhoai `healthcheck.yml`
L3-36 vs `e2e.yml` L3-31.

### DP-05 — Idempotent QA + honest skips + no flaky tests [RULE]
Check-then-create secrets (never regenerate a DB password); first-run-only restarts
gated on probes; bounded polling with honest names ("retries expected, ~5 min");
`no_log: true` on credential checks. Every exercise not covered by e2e gets an
explicit skip ledger entry with the reason. An exercise that cannot be made reliable
in-environment is downgraded to documented manual testing or deleted — never left
flaky. QA validates the ref the student clones (`repo_ref: "{{ tag | default('main') }}"`).
Evidence: deploy-configure e2e L253-386, header skip ledger, commit 89b36d4.

### DP-05a — Placeholders ⇒ playbooks on the same branch [RULE]
If content renders solve/validate button placeholders (or spec.yaml claims them), the
backing `runtime-automation/module-NN/` playbooks MUST exist on the same branch.
Mechanical check, added to verify-content practice: grep for
`solve-button-placeholder|validate-button-placeholder` → assert matching
`runtime-automation/module-NN/{solve,validate}.yml`.
Evidence (the failure): ph-openshift-autoshift main advertises buttons on all 10
modules + spec.yaml L132 claims them; playbooks only on unmerged
`feat/runtime-automation-solve-validate` — published lab has dead buttons.

### DP-05b — Validate playbook protocol (guided patterns) [HABIT]
Per-module `validate.yml`: read-only probes only (`changed_when: false`,
`failed_when: false`); all conditions folded into ONE `validation_check` emitting a
multi-line ✅/❌ report (the action plugin takes only `pass_msg`/`error_msg` and stops
at the first failure — the JS parser turns each ✅/❌ line into a step chip); modules
that cannot be validated say so honestly instead of faking a check.
Evidence: autoshift branch `runtime-automation/module-01/validate.yml` L42-70;
honest fallback `module-07/validate.yml` L3-14.

---

## B. Automation & deployment

### DP-06 — Two-layer automation: cluster layer + showroom-deployer layer [RULE]
`automation/gitops/` owns desired cluster state; `automation/showroom-deployer/values-<slug>.yaml`
+ Makefile owns delivery. Standard Makefile contract: `SLUG ?= <slug>`,
`NAMESPACE ?= showroom-$(SLUG)`, `VALUES ?= automation/showroom-deployer/values-$(SLUG).yaml`,
`CHART ?= <repo>/showroom-deployer`; targets `repo template deploy status uninstall test`
(`test:` wired to `qa-automation/e2e.yml`). `make template` = no-cluster dry run.
Evidence: ph-operations-rhoai `Makefile` L6-13, `automation/showroom-deployer/README.md`.

### DP-07 — App-of-apps entry chart with per-app documented decisions [RULE]
Cluster layer = one hand-applied entry Application → chart rendering AppProject +
per-workload child Applications, each gated by `{{ if .Values.<workload>.enabled }}`,
sharing YAML-anchor git defaults. syncOptions are engineering decisions documented
inline: client-side apply where SSA field-manager semantics are destructive
(one-directional CEL rules), `ServerSideApply=true` + `SkipDryRunOnMissingResource=true`
+ retry backoff where operator CRD install windows race. Deliberately-degraded
Applications (omit CreateNamespace so ArgoCD enforces the lab's own exercise order)
get a comment saying so.
Evidence: ph-operations-rhoai `cluster/infra/bootstrap/` chart,
`application-rhoai-install.yaml` L24-35, `application-maas-platform.yaml` L32-48,
`application-maas-model.yaml` L17-22.

### DP-08 — Hook Jobs own transitions; Git never flip-flops [RULE]
What ArgoCD cannot express (secret generation, status patches, DSC merge-patches,
federation sequencing) becomes annotated sync-wave hook Jobs inside the same Git
tree, each poll-waiting on its dependency. Fields that hook Jobs patch MUST be absent
from the Git manifest, or ArgoCD and the Jobs will fight forever.
Evidence: ph-operations-rhoai `cluster/maas-platform/templates/job-*.yaml` (waves 0-4),
`values.yaml` L58-61 comment, START_HERE L77-78.

### DP-09 — Secrets: three named mechanisms, zero secrets in git [RULE]
(1) `REPLACE_AT_APPLY` bootstrap values, injected at the one-time apply, never
committed. (2) Hook-Job-generated create-if-absent in-cluster secrets — never
overwritten, so passwords survive syncs. (3) Gitignore-negation idiom for per-cluster
deployer values:
```gitignore
automation/showroom-deployer/values-*.yaml
!automation/showroom-deployer/values-<slug>.yaml
```
(the committed file is the placeholder template; copies with real creds are ignored by
default). Content side: `%PASSWORD%`/`injected-at-runtime` antora attributes;
learner-chosen secrets via env vars (`: "${QUAY_PASSWORD:?set QUAY_PASSWORD …}"`),
never files; agent auth in `~/.config/publishing-house/auth.json`, outside the repo.
Extends §0 below. Evidence: ph-operations-rhoai `bootstrap/values.yaml` L43,66,
`.gitignore` L19-21, `job-postgres-secrets.yaml`; verified clean in all 7 repos.

### DP-05c — QA identity boilerplate [RULE]
In QA/validate plays that call `kubernetes.core.k8s` AND raw `oc` (or nested
ansible-playbook), set the identity three ways or tasks silently fall back to the
pod's in-cluster identity: `module_defaults: group/kubernetes.core.k8s` +
`environment: {KUBECONFIG, K8S_AUTH_KUBECONFIG}`. Document the trap as a comment in
the play itself.
Evidence: ph-openshift-monorepo-example `qa-automation/e2e.yml` L17-37 (comment
explains the nested-playbook case), commit dc93aff.

---

## C. Content house style

### DP-10 — Expected output after every command [RULE]
Every `role="execute"` block is followed by an expected-output block (plain `----`
listing) plus 1-2 sentences of interpretation: what "good" looks like and which
failure the deviation predicts. Screenshots are optional; output blocks are not.
Evidence: ph-openshift-autoshift (zero screenshots, 328 execute blocks, e.g.
`module-01` L192-200 "A RESTARTS count above 0 … indicates the init-container image
resolution failed → Troubleshooting").

### DP-11 — Screenshots: sequential NN- naming, alt text = expected state, click-to-enlarge [RULE]
`image::NN-descriptive-name.png[<alt text restating the expected state>,link=self,window=blank,width=800]`.
Number-as-order; zero orphans; never inside a numbered list without `+` continuation
(place at column 0 outside the item — the silent AsciiDoc regression). When
screenshots are omitted entirely (DP-10 style), that is a valid house choice.
Evidence: deploy-configure 49 PNGs; operations-rhoai 48 with verification alt text.

### DP-12 — Callouts are the debugging-scar ledger [RULE]
Attach the hard-won failure knowledge to the exact manifest line: YAML callouts
carrying silent-failure semantics ("a subscription in any other namespace is
silently ignored", "every call gets 403"), root-cause NOTEs (OMP_NUM_THREADS pin →
30s route timeouts), and workaround notes **with expiry conditions** ("can be dropped
once RHOAI ships a build aligned with COO 1.5.x") in a manifests README ledger.
Version-pin the specifics (RHCL 1.4.2). Follows §1a callout rules.
Evidence: deploy-configure `module-05` L697-710, `module-04` L148-154,
`manifests/README.md` L36-50.

### DP-13 — Canonical module skeleton [RULE]
intro → `== Prerequisites` → `== Learning objectives` → `== Exercise N: <verb title>`
(optional/skipped/reference labeled inline in the heading) → `=== Verify` per
exercise → `== Troubleshooting` → `== Learning Outcomes` (✓ bullets restating the
objectives **with the reasons learned**) → `== Module summary` (What you accomplished
/ Key takeaways / Next steps) → `== References`. Nothing after References.
Evidence: deploy-configure `module-01` L2-355; autoshift 10/10 modules.

### DP-14 — Mechanism before commands; lab-vs-production honesty [RULE]
Explain WHY before the command block ("The order matters. Read the reason before
running any command"). `[IMPORTANT]` names the lab convenience and the production sin
("This is a lab convenience. Do not run a fleet this way."). No marketing superlatives;
no deep links to versioned product docs (URLs churn — link the stable entry point).
Evidence: autoshift `module-01` L115-130, L218-229; `99-conclusion.adoc` L91.

### DP-15 — Transitions stitch the narrative [RULE]
Each exercise opens with why it follows the previous one ("Now that the operator is
running, let's tell it what to do…"); forward references set up later exercises
("intentionally low to demonstrate rate limiting (HTTP 429) in Exercise 7").
Evidence: deploy-configure 13 transitions, `module-01` L75/L244, `module-05` L707.

### DP-16 — Teardown and schedule honesty [RULE]
Any module provisioning billable resources MUST end with a mandatory, verified
teardown exercise ("Teardown is mandatory… billing continues") with
`--ignore-not-found` deletes. Labs >2h ship a schedule-tier table (full / reduced /
core path) with the exact skip list and what still works, plus a pre-flight
checklist where every check maps to a validation command with expected output.
Long-running work is decoupled: declare it, hand the learner a "go on ahead", collect
results in a later module. State-leak traps (Helm not carrying `--set` across
upgrades) get a NOTE at the exact step.
Evidence: autoshift `module-10` L200-291, `02-details.adoc` L20-303, `module-03`
L609-621, `module-08` L528-529.

### DP-17 — UI orientation before exercise 1 [RULE]
`index.adoc` explains the workshop UI itself (panel layout, instructions/split/tabs
modes), tab names in `ui-config.yml` match the prose references, tabs only exist for
things available at lab start, `persist_url_state: true`, honest environment sizing
table ("Not sufficient for this lab" is a valid verdict).
Evidence: deploy-configure `index.adoc` L18-76; autoshift `e98d69d` "rename terminal
tab to match showroom convention", `0b93b73` "remove Argo CD tab".

### DP-18 — Runnability is a decision, not an accident [RULE]
`role="execute"` only where running in the embedded terminal is correct; anything for
the learner's laptop or an external console is plain `[source,bash]` with a sentence
saying why there is no run button. `subs="specialcharacters,attributes"` on any block
that substitutes `{attributes}`.
Evidence: autoshift `module-03` L645-660 (AWS CLI quota — deliberately not runnable);
328 execute blocks total.

---

## D. Multi-variant publishing

### DP-19 — One repo → many variants: gate fragments, not exercises [RULE]
`ifeval::["{rhoai_version}" == "…"]` gates branch **fragments** (a manifest name, a
patch step) inside otherwise-shared exercises. Duplicating whole exercise sets inside
ifeval branches is the anti-pattern (every fix applied twice; drift guaranteed) —
when divergence exceeds ~half a page, split into per-version module files instead.
Manifest twins (`subscription.yaml` / `subscription-35.yaml`) keep both versions
buildable from main; environment differences are directory choices
(`gateway/clusterip/` vs `gateway/loadbalancer/`), not content forks. One attribute
flipped by the root `site.yml` build overlay is the variant knob (attribute defaults
in `content/antora.yml`, overrides in `site.yml`).
Evidence: deploy-configure 38 ifevals + module-05a counter-example (L119-641);
manifests/01-install operator+DSC twins; Makefile L10-13.

---

## E. Attributes & build safety

### DP-20 — antora.yml is the attribute inventory [RULE]
Every environment coordinate is declared once in `content/antora.yml` with a canonical
placeholder (`guid: "%GUID%"`, `user: "%USER%"`, `password: "%PASSWORD%"`,
`https://api.cluster.example.com:6443`) + a comment: "Never put real credentials here:
this file is committed to git." Versions are never hardcoded in content (autoshift
`partials/_attributes.adoc` policy: "Add a version here only if it does not come from
<the product release you clone>"). Runtime-injected attributes
(`{openshift_cluster_admin_username}`, `{openshift_console_url}`, …) are showroom
userdata — whitelist them in review, never "fix" them to repo-local values.
Evidence: operations-rhoai `content/antora.yml` L10-27; autoshift
`partials/_attributes.adoc` L12-15; mistake-note fcce3f74.

### DP-21 — Local Antora extensions only [RULE]
The RHDP content image ships only `@antora/cli` + `@antora/site-generator`; adding an
npm extension builds nothing and 404s every page. All Antora extensions are local
files under `content/lib/*.js` (mermaid injection, button/CSS injection, idempotent
via content guards). The constraint is documented at the point of violation (a
comment block in `site.yml`). Local preview runs the **platform image** via
`podman-compose.yaml` (antora + httpd, SIGWINCH-absorbing wrapper) — same semantics
as production, which is how the constraint gets discovered locally. Guided/nookbag
patterns cannot be previewed this way (document the limit).
Evidence: autoshift `site.yml` L22-42, `content/lib/inject-{mermaid,buttons}.js`,
`podman-compose.yaml` L23-27.

---

## F. PH-frame honesty

### DP-22 — worklog.yaml is the drift journal [HABIT]
Append a dated entry for every known divergence (stale prose, dead manifests,
repo-URL splits, un-run QA) instead of silently fixing or ignoring. Read it FIRST in
every session on an existing repo. The worklog is the only cross-file state that
survives a session boundary. Empty `decisions/`/`reviews/` scaffolds rotted in every
mined repo — do not write to them unless a human asks.
Evidence: ph-operations-rhoai `worklog.yaml` (10 notes, all verified still true) vs
`entries: []` in deploy-configure/monorepo/trio.

### DP-23 — WHY-comments at the point of use [HABIT]
The decision record that survives is the comment colocated with the thing it
explains (`// WHY THIS EXISTS:` — site.yml extension constraint, podman-compose
SIGWINCH trap, _attributes version policy, gitignore-negation threat note). Prefer it
over a separate decision journal; it is versioned with the code and impossible to
skip when editing.

### DP-24 — spec.yaml `development.*` status ritual [HABIT]
Update `spec.yaml` `development.{ansible,gitops,e2e,healthCheck}.status` and
`spec.modules[].status` at the END of each phase, as its own commit. Statuses use the
template vocabulary: `not_started` → `complete` (a directory scaffolded is NOT
"automation written" — distinguish "scaffold placeholder" from "authored"). Never
claim `complete` for e2e/healthCheck before a live-cluster run.
Evidence: deploy-configure commits 712e507/59808ff/35dbb4f; autoshift counter-example
(gitops/ansible `complete` with placeholder-only dirs; e2e `not_started` on a
published lab).

### DP-25 — spec must not lie: spec-vs-reality check [RULE]
Auto-generated `publishing-house/spec/modules/*.md` rot fast and are PUBLISHED as
TechDocs (`mkdocs.yml` `docs_dir: publishing-house/spec`). Before publish: every
manifest path referenced in spec/module outlines must exist; module list in
`spec.yaml` must match shipped pages (no missing modules); frame artifacts must agree
with each other. If outlines cannot be re-synced, banner them as design snapshots.
Evidence (the failure): deploy-configure `spec/modules/module-05-maas-governance.md`
L39-58 references six non-existent paths + wrong CR kinds; module-05a absent from
spec.yaml; mkdocs publishes it.

### DP-26 — Unused machinery gets deleted, not shipped [HABIT]
Dead JS injected into every page, empty frame dirs, leftover spec-speak in learner
prose ("This satisfies the RHOAI Ready precondition"), trailing content after
References — sweep before publish. quality-enrichment owns the unused-asset sweep.
Evidence: deploy-configure anti-patterns §7.3-7.5.

### DP-27 — Migration discipline [RULE]
Check `intake_type` before touching scaffolds; `--migration` mode never wipes
imported pattern dirs; provenance recorded via `ph.rhdp.io/migrated-*` annotations +
`git log --author=Scaffolder`. Post-migration module renames keep the numbering
contract (page stem ⇄ outline stem ⇄ runtime-automation dir).
Evidence: ph-operations-rhoai `scaffold.py` L284-298; PH-MIGRATION-RUNBOOK Route A.

---

## Severity mapping (for verify-content integration)

| Rules | Violation severity |
|---|---|
| DP-01, DP-03, DP-05a, DP-09, DP-25 | Critical (block publish) |
| DP-02, DP-04, DP-06, DP-07, DP-08, DP-10, DP-13, DP-19, DP-20, DP-21, DP-27 | High |
| DP-05, DP-05b, DP-05c, DP-11, DP-12, DP-14, DP-15, DP-16, DP-17, DP-18, DP-22, DP-23, DP-24, DP-26 | Warning |

Maintained by: ph-rhaibu OODA pipeline — mine, don't guess.
