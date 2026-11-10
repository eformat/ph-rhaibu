---
name: workshop-act
description: >
  Deploy a scaffolded publishing-house workshop (~/git/ph-<slug>/) to a prelude
  OpenShift cluster, validate it end-to-end, then publish it. Deploys the showroom via
  the zt-showroom-deployer chart with automation/showroom-deployer/values-<slug>.yaml
  (make deploy), builds and serves Antora content, runs browser-based tests using
  playwright-cli, validates each RAC requirement's acceptance criteria, and publishes
  PH-native: push the repo so its own gh-pages.yml deploys to GitHub Pages, run
  publishing-house/tools/ph-check.py / ph-development.py for the PH workflow, then
  /catalog-builder for the RHDP AgnosticV catalog entry. Implements a fix-and-test
  loop: any failed test or unmet requirement triggers another iteration until all
  checks pass. Use when someone wants to "deploy the workshop", "test the workshop",
  "validate workshop requirements", "run workshop on prelude", "workshop smoke test",
  "check workshop quality", "workshop CI loop", "verify the workshop works", "deploy
  to prelude", "test on cluster", "workshop acceptance testing", "workshop fix loop",
  or "publish workshop". This is the fourth and final step of the OODA workshop
  pipeline (Observe -> Orient -> Do -> Act -> Publish). Do NOT use for planning
  (workshop-orient) or scaffolding code (workshop-do).
triggers:
  keywords:
    - "workshop act"
    - "deploy workshop"
    - "test workshop"
    - "validate workshop"
    - "workshop prelude"
    - "ooda act"
    - "workshop smoke test"
    - "workshop acceptance"
  matchMode: any
enabled: true
---

# Workshop Act

Deploy to a prelude cluster, test with browser automation, loop until RAC acceptance
criteria pass, then publish PH-native (repo gh-pages + PH tooling + catalog).

## Skill coordination

- Use **playwright-cli** for all browser automation — opening pages, capturing
  screenshots, clicking, filling forms, assertions.
- Use **openshift-4-22-expert** (4.22 hosts) or **openshift-5-expert** (5.0 hosts) for
  cluster diagnostics when deployments fail.
- Use **openshift-ai-3-5-expert** (GA) or **openshift-ai-3-6-expert** (EA only) for
  RHOAI-specific diagnostics.
- Read RAC requirements from `rac/requirements/` in the workshop repo for acceptance
  criteria to validate.
- Use **workshop-screenshot** for capturing and embedding screenshots into AsciiDoc content.
- Run **verify-content** as a quality gate after content and screenshots are ready.
- Apply the **demo-patterns** catalog (`../docs/DEMO-PATTERNS.md`) at Act: DP-05a
  placeholder⇒playbook gate, DP-03/04/05 QA contracts, DP-09 secrets mechanisms,
  DP-24 final status ritual, DP-25/26 pre-publish sweeps.
- See `../docs/WORKSHOP-COMMON-RULES.md` for shared AsciiDoc, image, security,
  and quality rules.
- See `references/post-enrichment-audit.md` for structured
  post-enrichment audit phases (correctness, deduplication, depth) that slot into
  the fix-and-test loop.

## Prerequisites

Check these before starting. If any are missing, print what's needed and stop.

**Required tools:**
- `git` — state-changing git commands belong to the human git-master; the agent
  prepares files and asks the user to run them
- `oc` CLI — `oc version` (OpenShift client, authenticated as cluster-admin)
- `helm` — `helm version --short`
- `node` / `npm` — `node --version && npm --version`
- `decided` CLI — `decided --version` ([install](https://github.com/asdecided/core/releases))
- `playwright-cli` — for browser automation and screenshot capture

**Required cluster access:**
- `oc` authenticated to a prelude OpenShift cluster (`oc whoami` succeeds)
- RHOAI operator available or pre-installed (only if the workshop content requires it)
- Sufficient resources for the workshop (GPUs if required)

**Required state (from prior pipeline steps):**
- The workshop repo at `~/git/ph-<slug>/` (from workshop-orient + workshop-do) with:
  - a validated `rac/` corpus and synthesized `publishing-house/` spec artifacts
  - `content/` with a successful `make build`
  - `automation/showroom-deployer/values-<slug>.yaml`
  - a `Makefile` with the deployer targets

## Workflow

### 0. Resolve the workshop repo

Accept `$ARGUMENTS` as a workshop slug. If no argument is provided, discover
existing workshop repos:

```bash
ls -d ~/git/ph-*/rac/ 2>/dev/null
```

If exactly one exists, use it. If multiple exist, list them and ask the user to
choose. If none exist, stop — workshop-orient/workshop-do must be run first.

Verify the repo has the required pieces: `rac/`, `content/`,
`automation/showroom-deployer/values-<slug>.yaml`, `Makefile`, `site.yml`.
If content or automation is missing, stop — workshop-do must be run first.

### 1. Pre-flight checks

Verify the environment before deploying:

```bash
oc whoami
oc whoami --show-server
oc get clusterversion -o jsonpath='{.items[0].status.desired.version}'
oc get co --no-headers | grep -v "True.*False.*False" | head -10
helm version --short
decided --version
```

If any check fails, report the issue and stop. Do not deploy to an unhealthy cluster.

Confirm the cluster URL matches the expected prelude cluster. Ask the user to verify
if this is the correct target before proceeding.

### 2. Fill values and deploy the showroom

The showroom is deployed by the `zt-showroom-deployer` chart with values from
`automation/showroom-deployer/values-<slug>.yaml`, driven by the repo-root Makefile
targets. If the lab has a GitOps cluster layer (`automation/gitops/` beyond
bootstrap-infra), deploy that first per its README (bootstrap app-of-apps), then the
showroom.

First fill in the cluster-specific values in
`automation/showroom-deployer/values-<slug>.yaml` (they are placeholders after
workshop-do): `deployer.domain`, `deployer.apiUrl`, and the `showroom.wetty.*` SSH
coordinates. Confirm the repo is pushed to the `showroom.gitRepoUrl` — the chart's
`git-cloner` init container clones it on-cluster.

```bash
cd ~/git/ph-<slug>

# Fill cluster coordinates (or edit the values file directly)
DOMAIN=$(oc get ingress.config cluster -o jsonpath='{.spec.domain}')
API=$(oc whoami --show-server)

make repo                    # helm repo add/update the chart repo
make template                # render locally to review (no cluster changes)
make deploy                  # helm upgrade --install into showroom-<slug>
```

`make deploy` runs `helm upgrade --install showroom <chart> -n showroom-<slug>
--create-namespace -f automation/showroom-deployer/values-<slug>.yaml`. To deploy
against a local chart checkout instead of the published repo, pass
`CHART=~/git/zt-showroom-deployer`.

### 3. Verify showroom readiness

Verify the showroom deployment came up:

```bash
make status
oc rollout status deploy/showroom -n showroom-<slug> --timeout=5m
oc get pods -n showroom-<slug>            # expect the showroom pod Running (all containers ready)
ROUTE=$(oc get route -n showroom-<slug> -o jsonpath='{.items[0].spec.host}')
curl -sk -o /dev/null -w '%{http_code}\n' https://$ROUTE/   # expect 200
```

The chart's init containers (`git-cloner`, `antora-builder`) clone and build the content
on-cluster; `cluster-setup` patches the IngressController so the environment iframe renders.

If the lab has a GitOps cluster layer, verify it synced (ArgoCD applications healthy)
before testing module content that depends on it.

Only if the **workshop content itself** requires cluster-side resources (e.g. RHOAI model
serving), also verify those pre-exist on the cluster — this skill does not install them:
- Operator CSVs `Succeeded`: `oc get csv -A --no-headers | grep -v Succeeded`
- Required CRDs registered (e.g., `InferenceService`)

Map each check back to a RAC requirement's acceptance criteria. Record pass/fail.

### 4. Build and serve workshop content

In the workshop repo:

```bash
cd ~/git/ph-<slug>
make serve &
SERVE_PID=$!
```

Wait for the server to start (poll `http://localhost:8887` until it responds).

Use playwright-cli to open the workshop landing page and verify it renders correctly:
- Page title matches the workshop name
- Navigation sidebar has entries for all modules
- No broken images or missing CSS

### 5. Run acceptance tests

For each RAC requirement, execute its acceptance criteria:

**Content tests** (for each module page):
- Open the page URL with playwright-cli
- Verify the page renders (check for the module title text)
- Verify `[source,role="execute"]` blocks are present (check for code block elements)
- Capture a screenshot and save to `content/modules/ROOT/assets/images/`
  using deterministic filenames

**Infrastructure tests** (for cluster-dependent steps):
- Log into the OpenShift console with playwright-cli
- Navigate to the relevant pages (RHOAI dashboard, model serving, etc.)
- Execute commands from the workshop lab steps in a terminal
- Verify expected outcomes (pods running, models deployed, API responding)
- Capture screenshots of each verified state

**Record results** in a structured format:

| Requirement ID | Test Description | Status | Evidence |
|----------------|------------------|--------|----------|
| RHAIBU-REQ-001 | Login to console | PASS | screenshot: 01-login.png |
| RHAIBU-REQ-002 | Deploy model | FAIL | Error: timeout waiting for pod |

### 5b. Content quality verification

Run the `verify-content` skill (vendored in this repo under `../verify-content/`)
against the content:

```
/verify-content
```

This spawns parallel agents per module, checking against Red Hat quality standards:
- AsciiDoc structure and formatting (missing execute roles, broken blocks)
- Accessibility compliance (image alt text, heading hierarchy)
- Red Hat style guide (product names, acronym expansion)
- Technical accuracy (undefined attributes, broken xrefs)

**Severity handling:**
- **Critical** / **High** — must fix before proceeding to the fix-and-test loop
- **Warning** / **Info** — report in the validation table but do not block

### 5c. PH compliance check

Run the PH tooling against the repo (needs `~/.config/publishing-house/auth.json`;
if absent, record locally and let the user submit later):

```bash
cd ~/git/ph-<slug>
python3 publishing-house/tools/ph-check.py
```

Fix any blocking findings. When the lab content is complete, flip
`spec.modules[].status` to `complete` in `publishing-house/spec.yaml` and, if the
project is registered with the PH Central API, advance the workflow:

```bash
python3 publishing-house/tools/ph-development.py
```

### 5d. Post-enrichment audit (RECOMMENDED)

If the workshop content was generated by an LLM or scaffolded from templates, run the
three-phase post-enrichment audit described in
`references/post-enrichment-audit.md`:

1. **Phase A — Correctness:** Check every command, manifest, and console path against
   product documentation. Ground every fix in doc extractions.
2. **Phase B — Deduplication:** Identify duplicated walkthroughs across modules or
   across workshops in a catalog. Replace with Antora xrefs while maintaining standalone
   coherence.
3. **Phase C — Depth:** Find documented-but-uncovered workflows. Add as depth exercises
   where they fit the RAC requirements. Sync RAC after adding (`decided validate`).

Each phase that produces fixes triggers a rebuild (`make build`) and a re-run of
verify-content before proceeding to the next phase. If no fixes are needed in a phase,
skip the rebuild and move on.

Skip this step for workshops where the author manually wrote and tested every command.

### 6. Fix-and-test loop

If any test fails, identify the failure category and fix:

**Content issue** (AsciiDoc error, wrong instructions, missing page):
1. Fix the `.adoc` file under `content/modules/ROOT/pages/`
2. Rebuild: `make build`
3. Go to step 5

**Infrastructure issue** (showroom pod won't start, wrong values, content build fails):
1. Fix `automation/showroom-deployer/values-<slug>.yaml` (or the content if the
   on-cluster antora build failed), and push content changes so `git-cloner` picks
   them up
2. Redeploy: `make deploy` — or `oc rollout restart deploy/showroom -n showroom-<slug>`
   to re-run the init containers against updated content
3. Wait for the rollout to complete
4. Go to step 3

**Environment issue** (cluster problem, insufficient resources, external dependency):
1. Report the issue to the user with diagnostics
2. Stop the loop — environment issues require human intervention

**Loop bounds:**
- Maximum 5 iterations before escalating to human-in-the-loop
- Track which requirements passed/failed per iteration
- Detect no-progress (same tests failing the same way) and bail out early

### 7. Clean up and report

Stop the local server:

```bash
kill $SERVE_PID 2>/dev/null
```

Print the final results table:

```
=== Workshop Validation Report ===

Requirement ID   | Title                    | Status | Iterations | Evidence
-----------------+--------------------------+--------+------------+---------
RHAIBU-REQ-001   | Console Login            | PASS   | 1          | 01-login.png
RHAIBU-REQ-002   | Model Deployment         | PASS   | 3          | 02-model.png
RHAIBU-REQ-003   | API Query                | FAIL   | 5          | Error: timeout

Passed: 8/10
Failed: 2/10
Iterations: 3

Human review needed for:
- RHAIBU-REQ-003: API endpoint not responding (environment issue)
- RHAIBU-REQ-009: Cannot automate — requires manual verification
```

If all requirements pass:
"Workshop validation complete. All requirements passed. Ready for human-in-the-loop review."

If some failed after max iterations:
"Workshop validation incomplete after 5 iterations. The above requirements need attention."

Prompt the user to commit updated screenshots and any content fixes.

### 8. Publish (PH-native)

Once all requirements pass (or after human sign-off if some needed manual review),
publish through the publishing-house path — there is no separate lessons repo.

**Publishing flow:**

1. **Push the repo.** Git state changes belong to the human git-master — ask the user
   to push `main` to the GitHub remote (`rhpds/ph-<slug>` canonical, or their fork).
   Merging to `main` triggers the repo's own `.github/workflows/gh-pages.yml`, which
   builds with Antora and deploys the site to
   `https://<org>.github.io/ph-<slug>/`. Verify the live site renders after the
   workflow completes (~40s build; check `gh run list` if the site looks stale).
2. **PH workflow submission.** If the project is registered with the PH Central API
   (`catalog-info.yaml` "Central" link + `~/.config/publishing-house/auth.json`),
   submit for review:
   ```bash
   cd ~/git/ph-<slug>
   python3 publishing-house/tools/ph-check.py        # compliance
   python3 publishing-house/tools/ph-development.py  # advance workflow / close tickets
   ```
   Stage transitions are managed by the Central API — never by manual edits.
   If the project is NOT registered, record the compliance result in
   `publishing-house/worklog.yaml` and note the repo is self-published.
3. **Catalog entry.** Run `/catalog-builder` to create the RHDP AgnosticV catalog
   item when the lab is ready for RHPDS provisioning. The showroom repo URL for the
   catalog is this repo (`https://github.com/<org>/ph-<slug>.git`) — the deployer
   chart clones it directly.

Print the published site URL and stop. Human review of the PH submission (if
registered) is the final gate.

## Guardrails

- Never deploy to a production cluster. Verify the cluster URL with the user before
  deploying.
- Never delete namespaces or resources not created by this skill. Use labels
  (`app.kubernetes.io/managed-by: workshop-act`) on test resources.
- Do not modify RAC requirements to make tests pass. If a requirement is wrong or
  untestable, flag it and ask the user to update it in the Orient step.
- Bound the fix-and-test loop at 5 iterations. Infinite loops waste cluster resources.
- Capture screenshots only from the target product version. Never reuse screenshots
  from a different version or fabricate UI states.
- Clean up test artifacts (playwright sessions, background server processes) on exit.
- If the OODA loop finds structural problems (wrong module count, missing concepts),
  recommend going back to Orient rather than patching at the Act level.
- Git push/PR/merge actions belong to the human git-master — prepare and ask.

## Out of scope

- Writing new workshop content from scratch — use `workshop-do`
- Planning the workshop — use `workshop-orient`
- Analyzing demo applications — use `workshop-observe`
- Production deployment or multi-tenant provisioning
- Performance testing or load testing

## Related Skills

- `/workshop-do` — Scaffold content and automation from RAC requirements
- `/workshop-screenshot` — Capture and embed screenshots into AsciiDoc content
- `/verify-content` — Validate content against Red Hat quality standards
- `/catalog-builder` — Create RHDP AgnosticV catalog entry when ready to publish
