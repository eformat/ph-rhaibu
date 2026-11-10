---
name: workshop-do
description: >
  Scaffold workshop content and automation code from RAC requirements into the
  workshop's single publishing-house repo (~/git/ph-<slug>/). Seeds the repo from the
  rhdp-publishing-house-template fork (scaffold.py + .scaffolds/ + publishing-house/
  frame), lays down the Antora content component under content/, the automation layer
  (automation/gitops/ charts + automation/showroom-deployer/values-<slug>.yaml), and
  qa-automation/. Use when someone wants to "scaffold a workshop", "generate workshop
  code", "build workshop content", "create showroom content", "create workshop
  infrastructure", "implement the workshop", "write the workshop code", "antora
  scaffold", "showroom scaffold", "gitops workshop", "helm charts for workshop", or
  "workshop implementation". This is the third step of the OODA workshop pipeline
  (Observe -> Orient -> Do -> Act). Do NOT use for analyzing demos (workshop-observe),
  planning workshops (workshop-orient), or deploying/testing (workshop-act).
triggers:
  keywords:
    - "workshop do"
    - "scaffold workshop"
    - "workshop content"
    - "workshop infrastructure"
    - "showroom scaffold"
    - "workshop implementation"
    - "ooda do"
    - "generate workshop"
  matchMode: any
enabled: true
---

# Workshop Do

Scaffold content, automation, and QA into the workshop's single publishing-house repo
`~/git/ph-<slug>/` — one repo holds planning (`rac/`, `publishing-house/`), content
(`content/`), automation (`automation/`), and tests (`qa-automation/`).

## Skill coordination

- **Delegate content scaffolding** to the `openshift-workshop-builder` skill. It owns
  the Antora/showroom structure, file templates, AsciiDoc patterns, and build process.
  This skill orchestrates it with the right parameters, then adds workshop-specific content.
- Use the **OpenShift 4.21 Expert** skill for OpenShift-specific lab steps.
- Use the **OpenShift AI 3.3 Expert** skill for RHOAI-specific lab steps.
- Reference the in-repo exemplars for the target shape (read, don't clone):
  - `~/git/redhat-ai-inference/` — hybrid repo with `rac/` + PH frame
  - `~/git/ph-operations-rhoai/` — `automation/gitops/cluster/*` two-layer pattern +
    deployer values + Makefile deployer targets
  - `~/git/ph-deploy-configure-rhoai/` — manifests/ + qa-automation conventions
- The chart value contract, showroom pod anatomy, and `automation/` layout are
  documented in `references/infra-patterns.md`.
- See `../docs/WORKSHOP-COMMON-RULES.md` for shared AsciiDoc, image, security,
  and quality rules (Sections 9–11 cover maturity depth, doc grounding, and
  cross-workshop navigation).
- See `references/doc-grounding.md` for the documentation
  grounding workflow, self-check checklist, and maturity-specific scaffolding patterns.
- See `references/cross-workshop-nav.md` for Antora
  cross-component xref patterns when the workshop is part of a multi-workshop catalog.

## Prerequisites

Check these before starting. If any are missing, print what's needed and stop.

**Required tools:**
- `git` — state-changing git commands belong to the human git-master; the agent
  prepares files and asks the user to run them
- `node` / `npm` — `node --version && npm --version` (for Antora build)
- `helm` — `helm version --short` (for chart validation)
- `decided` CLI — `decided --version` ([install](https://github.com/asdecided/core/releases))

**Required template repo** (clone if missing):
- `~/git/rhdp-publishing-house-template/` — Mike's fork of
  `rhpds/rhdp-publishing-house-template`; supplies `scaffold.py`, `.scaffolds/`,
  `publishing-house/tools/`, `hooks/`, `setup-claude-hooks.sh`,
  `.claude.settings.template.json`, `publishing-house/spec/` design + outline templates.

**Required chart:**
- The reusable `zt-showroom-deployer` Helm chart. Values reference it either from
  the published Helm repo (`https://eformat.github.io/helm-charts`) or a local path
  fallback to `~/git/zt-showroom-deployer/`.

**Required state:**
- The workshop repo at `~/git/ph-<slug>/` (from workshop-orient) containing a validated
  `rac/` corpus (requirements with acceptance criteria, decisions, at least one design
  with a parameter inventory) and the synthesized `publishing-house/spec.yaml` +
  `publishing-house/spec/` artifacts.

## Workflow

### 1. Read RAC requirements

Accept `$ARGUMENTS` as a workshop slug. If no argument is provided, discover
existing workshop repos:

```bash
ls -d ~/git/ph-*/rac/ 2>/dev/null
```

If exactly one exists, use it. If multiple exist, list them and ask the user to
choose. If none exist, stop — workshop-orient must be run first.

The repo is at `~/git/ph-<slug>/`.
Read all requirement artifacts from `rac/requirements/`. Extract:
- Module list with ordering
- Acceptance criteria per module
- Parameter inventory from the design artifact
- Infrastructure requirements (operators, CRDs, GPUs, storage — and whether the lab
  must provision them itself via GitOps, or only consume pre-provisioned ones)

### 2. Seed the repo from the template fork

If the repo already has the PH frame (`scaffold.py`, `.scaffolds/`, `publishing-house/tools/`),
skip this step. Otherwise copy the frame in from
`~/git/rhdp-publishing-house-template/` (working-tree copy — do not git-clone into it):

```bash
TEMPLATE=~/git/rhdp-publishing-house-template
REPO=~/git/ph-<slug>
cp -R $TEMPLATE/scaffold.py $TEMPLATE/.scaffolds $TEMPLATE/publishing-house \
      $TEMPLATE/hooks $TEMPLATE/setup-claude-hooks.sh \
      $TEMPLATE/.claude.settings.template.json $REPO/
```

The template intentionally does NOT ship: `publishing-house/spec.yaml` (Orient
synthesizes it), `catalog-info.yaml`, `Makefile`, `mkdocs.yml`, `package.json`,
`site.yml` at root, `ui-config.yml`, `content/`, `automation/`, `qa-automation/`,
`.github/`. Those come from the steps below and from `scaffold.py`'s pattern copy.

Then run the pattern scaffolder once to lay down the common Antora + QA stubs (our
labs are the `agd-open`/classic pattern):

```bash
cd $REPO
python3 scaffold.py --pattern agd-open --automation gitops
```

- `--pattern agd-open` — classic showroom (rhdp_showroom_theme, no nookbag). All
  current RHAIBU labs use this.
- `--automation gitops` — creates `automation/gitops/bootstrap-infra/` (and
  `bootstrap-tenant/` only with `--topology shared-cluster`). Pass `--topology
  shared-cluster` in the SAME invocation if the RAC design says shared-cluster —
  `scaffold.py` deletes `.scaffolds/` when it finishes, so automation scaffolding
  cannot be added later.
- The script also sets `showroom_type` / `infrastructure` in `publishing-house/spec.yaml`.

If the repo already has content (migration case), run `python3 scaffold.py --pattern
agd-open --automation gitops --migration` instead — it never overwrites existing
imported files.

### 3. Scaffold the content component

Follow the `openshift-workshop-builder` workflow to create/complete the showroom
content under `content/`. The component lives at `content/antora.yml` with modules in
`content/modules/ROOT/` — the repo-root `site.yml` (from the template's
`.scaffolds/common/`) already points at it (`start_path: content`).

**Required files** (all must exist for `make build` to succeed):

- `package.json` — Antora dependencies (`@antora/cli`, `@antora/site-generator`)
- `Makefile` — repo root; use the combined template in Step 7
- `site.yml` — Antora playbook at the repo root (template provides it). MUST use the
  RHDP theme bundle and `output: dir: ./www` — the gh-pages workflow uploads `path: www`.
- `content/antora.yml` — component descriptor with all `{attribute}` defaults from
  the RAC parameter inventory. MUST include:
  ```yaml
  asciidoc:
    attributes:
      source-highlighter: highlight.js
      experimental: true
      page-pagination: true
      feature_maturity: "GA"   # or "TP" or "DP"
  ```
  and MUST use `version: ~` (a pinned version disables the theme's pagination nav).
  Per-environment attributes stay as placeholders (`cluster.example.com`,
  `%password%`, `%GUID%`) — the Showroom platform injects real values at deploy time.
  Never hardcode live cluster URLs.
- `content/modules/ROOT/nav.adoc` — navigation tree with entries for each module
- `.gitignore` — exclude `node_modules/`, `www/`, `.cache/`
- `content/supplemental-ui/` and `content/lib/inject-buttons.js` — **copy** from an
  existing workshop repo (e.g. `~/git/redhat-ai-inference/content/`), do not recreate

**Content pages** under `content/modules/ROOT/pages/`:

- `index.adoc` — welcome page with: title, what you'll learn, who this is for,
  prerequisites, estimated time, link to first module. For TP/DP features, add
  `ifeval` maturity banners after the page title (see
  `references/doc-grounding.md` for the exact blocks).
  Adjust content depth by maturity: GA = full hands-on, TP = same + API-unstable
  warnings, DP = guided-tour mode with fewer execute blocks
- `getting-connected.adoc` — login steps using `{user}` / `{password}` attributes,
  project setup, environment verification
- `NN-module-MM-<slug>.adoc` — one page per module requirement (filename stem MUST
  match the `publishing-house/spec/modules/module-NN-<slug>.md` outline), with:
  - `:navtitle:` and `:toc: macro` attributes
  - `== Learning objectives` section (bulleted list from RAC requirement)
  - Numbered `== Exercise N: Title` sections with `[source,role="execute"]` code blocks
  - `=== Verify` section after each exercise
  - `== Module summary` with key takeaways
- `NN-conclusion.adoc` — wrap-up with per-module summary, resources, thank you

**AsciiDoc rules:**
- Every command uses `[source,role="execute"]` for the Showroom copy/execute button
- Use `[source,bash,role="execute",subs="attributes"]` for blocks containing `{attribute}` values
- Use `{attribute}` substitution for all environment-specific values — never hardcode
- Page naming: `NN-descriptive-name.adoc` with 2-digit prefix for ordering

**Optional files** (add if the workshop needs interactive features):
- `ui-config.yml` — showroom tabs and terminal configuration
- `manifests/` — learner-applied kustomize manifests, module by module
  (see `ph-deploy-configure-rhoai/manifests/` for the README-mapping convention)

**Always include** (copy from exemplar):
- `.github/workflows/gh-pages.yml` — GitHub Pages CI/CD. Triggers on pushes to
  `content/**`, `site.yml`, or the workflow file itself. Builds with Antora (Node 22),
  uploads `www/` as the pages artifact, and deploys on merge to `main`. Copy verbatim
  from `~/git/redhat-ai-inference/.github/workflows/gh-pages.yml` — do not recreate.

### 4. Scaffold the automation layer

Read `references/infra-patterns.md` for the chart value contract
and layout. Two optional layers, driven by the RAC design:

**a. Showroom deployer values (always):**

```
automation/showroom-deployer/
  README.md             # what deploys, chart ref, how to override
  values-<slug>.yaml    # per-workshop values for the zt-showroom-deployer chart
```

Values contract (see `references/infra-patterns.md` for the full list): set
`showroom.gitRepoUrl` (the GitHub URL of this repo — `git-cloner` clones it
on-cluster), `deployer.domain`/`apiUrl`, `showroom.namespace`, lab identity
(`guid`/`user`/`password`/`projectName`), `showroom.wetty.*` (placeholders if unknown
at scaffold time — filled during workshop-act), and image pins. Do **not** hardcode
cluster values — placeholders only; they live per-environment in this values file.

**b. GitOps cluster layer (only if the RAC design says the lab provisions cluster
resources itself):** charts under `automation/gitops/` following the
`ph-operations-rhoai` two-layer pattern — e.g. `automation/gitops/cluster/<component>/
{Chart.yaml,values.yaml,templates/}` for rhoai-install, kueue, maas-platform, etc.,
plus a bootstrap app-of-apps chart. Conventions (from the PH gitops docs):
sync-waves (namespaces -2, RBAC/SA -1, workloads 0, CRs/Routes 1+),
`SkipDryRunOnMissingResource=true` for operator CRDs, no hardcoded domains (construct
URLs from `deployer.domain`), no ApplicationSets in bootstrap-infra, tenant resources
only in tenant namespaces. If the lab only consumes pre-provisioned clusters, keep
just the scaffolded `automation/gitops/bootstrap-infra/` test-namespace chart.

### 5. Seed qa-automation

The template's `.scaffolds/common/qa-automation/` (copied by `scaffold.py`) provides
`healthcheck.yml` and `e2e.yml` stubs. Fill them to the lab:

- `healthcheck.yml` — validates the base environment the lab assumes (< 60s,
  non-destructive, exit 0 = ready). Scope: only what provisioning deploys, not what
  students install.
- `e2e.yml` — replays the student commands module by module as the automated "solve"
  (task naming `M<N> | <description>`, one VERIFY task per module). Build it out
  during workshop-act against a live cluster; scaffold the module skeletons here.

### 6. Root files

The template does not ship these — create them:

- **`catalog-info.yaml`** — Backstage Component (copy the shape from
  `~/git/ph-operations-rhoai/catalog-info.yaml`): `metadata.name: ph-<slug>`,
  `ph.rhdp.io/content-type: lab`, `ph.rhdp.io/deployment-mode: rhdp_published`,
  `ph.rhdp.io/github-user` / `ph.rhdp.io/owner`, `backstage.io/techdocs-ref: dir:.`,
  `github.com/project-slug`, `ph.rhdp.io/created-at`, tags `publishing-house`/`lab`.
  New (non-migration) repos omit the `ph.rhdp.io/migrated-*` annotations.
- **`Makefile`** — repo root, combined build + deployer targets (Step 7 template).
- **`mkdocs.yml`** — TechDocs nav over `publishing-house/spec` (copy the shape from
  `~/git/ph-operations-rhoai/mkdocs.yml`; `docs_dir: publishing-house/spec`, nav lists
  design.md + one entry per module outline).
- **`CLAUDE.md`** — hybrid instructions: start from the template's `CLAUDE.md` (PH
  session-start, scaffolding, content paths) and add a `## RAC` section documenting
  the `rac/` corpus (decided CLI, RHAIBU key, validate commands) and the deployer
  targets. See `~/git/redhat-ai-inference/CLAUDE.md` for a working hybrid.
- **`.gitignore`** — `node_modules/`, `www/`, `.cache/`, `*.tgz`, `charts/`, `.claude/`

### 7. The Makefile

```makefile
PORT ?= 8887
DOCS_DIR := $(shell pwd)
SITE_DIR := $(DOCS_DIR)/www

# Showroom deployer (workshop UI deployment via the zt-showroom-deployer chart)
SLUG        ?= <slug>
NAMESPACE   ?= showroom-$(SLUG)
RELEASE     ?= showroom
VALUES      ?= automation/showroom-deployer/values-$(SLUG).yaml
# Published chart ref; override CHART=~/git/zt-showroom-deployer for local source.
CHART_REPO_NAME ?= eformat
CHART_REPO  ?= https://eformat.github.io/helm-charts
CHART       ?= $(CHART_REPO_NAME)/showroom-deployer

.PHONY: install build serve clean test repo template deploy status uninstall

install:
	cd $(DOCS_DIR) && npm install

build: install
	rm -rf $(SITE_DIR)
	cd $(DOCS_DIR) && npx antora site.yml --stacktrace
	touch $(SITE_DIR)/.nojekyll

serve: build
	@echo "Serving at http://localhost:$(PORT)"
	python3 -m http.server $(PORT) --directory $(SITE_DIR)

test:
	ansible-playbook qa-automation/e2e.yml -v

clean:
	rm -rf $(SITE_DIR)

# --- Showroom deployer (workshop UI) ---

repo:
	helm repo add $(CHART_REPO_NAME) $(CHART_REPO) || true
	helm repo update $(CHART_REPO_NAME)

template:
	helm template $(RELEASE) $(CHART) -n $(NAMESPACE) -f $(VALUES)

deploy:
	helm upgrade --install $(RELEASE) $(CHART) \
	  -n $(NAMESPACE) --create-namespace -f $(VALUES)

status:
	helm status $(RELEASE) -n $(NAMESPACE)

uninstall:
	helm uninstall $(RELEASE) -n $(NAMESPACE)
```

Key rules: port 8887 (not 8080), output to `www/` (not `docs/`), call `npx antora`
directly, `rm -rf` before each build, `.nojekyll` for GitHub Pages, deployer values
live under `automation/showroom-deployer/`.

### 8. Initialize git and test build

Antora requires at least one commit — without it the site builds but contains zero
pages (only a warn: `No matching references found for content source entry`). Git
state changes belong to the human git-master — **ask the user** to run:

```bash
cd ~/git/ph-<slug> && git init && git add -A && git commit -m "Initial workshop scaffold"
```

Then build:

```bash
cd ~/git/ph-<slug> && make build
```

If the build fails, fix errors and retry.

### 9. Validate and report

```bash
cd ~/git/ph-<slug>
decided validate .                          # RAC corpus still passes
helm template showroom ~/git/zt-showroom-deployer \
  -n showroom-<slug> -f automation/showroom-deployer/values-<slug>.yaml   # render check
python3 publishing-house/tools/ph-check.py  # PH compliance (needs Central auth; skip gracefully offline)
grep -rn "AKIA\|aws_secret\|hosted_zone" --include="*" . || true   # secrets scan
```

Also check:
- `feature_maturity` is set in `content/antora.yml`
- Every `[source,...]` block containing `{...}` attribute placeholders has `subs="attributes"`
- TP/DP features have the corresponding `ifeval` banner in `index.adoc`
- `publishing-house/spec/modules/` outline stems match the `NN-module-MM-<slug>.adoc`
  page stems

Print a summary:
- Files created (count by type: .adoc, .yaml, .json, .yml)
- Content build status (pass/fail)
- `helm template` render status against `zt-showroom-deployer` (pass/fail)
- `decided validate` + `ph-check` status

Prompt: "Workshop scaffolded. Run `/workshop-act` to deploy and test on a cluster."

## Guardrails

- Never hardcode cluster-specific values in content. Use `{attribute}` substitution;
  per-environment attributes stay as placeholders in `content/antora.yml`.
- Never hardcode cluster-specific values in automation. Per-environment coordinates
  live only in `automation/showroom-deployer/values-<slug>.yaml`.
- Follow the `openshift-workshop-builder` patterns exactly for content structure.
- One Antora component per repo (`content/`), `site.yml` immutable in its content-source
  settings (`url: .`, `start_path: content`).
- Run `scaffold.py` BEFORE removing `.scaffolds/` (the script removes it itself) and
  pass `--automation` in the same invocation — it cannot be added later.
- Every command in a module must use `[source,role="execute"]` blocks.
- Git init/commit/push belongs to the human git-master — prepare files, then ask.
- Do not create screenshots — that requires a running cluster and is deferred to
  `workshop-act`.

## Out of scope

- Screenshot capture (requires a running cluster — deferred to workshop-act)
- Deployment to a cluster (workshop-act)
- Workshop planning or requirement gathering (workshop-orient)
- Analyzing demo applications (workshop-observe)

## Related Skills

- `/workshop-orient` — Plan the workshop and create RAC requirements
- `/workshop-act` — Deploy and test the workshop end-to-end
- `/workshop-screenshot` — Capture and embed screenshots into content
- `/openshift-workshop-builder` — AsciiDoc/Antora scaffolding patterns
- `/verify-content` — Validate content against Red Hat quality standards
