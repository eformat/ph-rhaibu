# PH Migration Runbook

Migrate a legacy three-repo zt-* workshop into the single publishing-house repo
format. Reference implementations (read, don't modify): `~/git/redhat-ai-inference/`
(hybrid with rac/), `~/git/ph-operations-rhoai/` (two-layer automation),
`~/git/ph-deploy-configure-rhoai/` (classic + manifests).

## Source layout (legacy)

```
~/git/zt-<slug>-rac/           # planning: decided corpus (requirements/decisions/designs/assets)
~/git/zt-<slug>-showroom/      # content: site.yml, ui-config.yml, content/, Makefile, gh-pages
~/git/zt-<slug>-automation/    # infra: values-<slug>.yaml + Makefile, or cluster/ gitops charts
```

Some legacy RAC repos are named without the `-rac` suffix (e.g. `zt-rhoai-deploy-configure`).

## Target layout

```
~/git/ph-<slug>/
├── rac/                                  # ← zt-<slug>-rac copied wholesale (incl. .decided/, assets/)
├── publishing-house/                     # ← template frame + synthesized spec artifacts
│   ├── spec.yaml  spec/design.md  spec/modules/module-NN-<slug>.md
│   ├── tools/ph-*.py  worklog.yaml  decisions/.gitkeep  reviews/.gitkeep
├── content/                              # ← zt-<slug>-showroom/content/ (antora.yml, modules/, supplemental-ui/, lib/)
├── automation/
│   ├── gitops/                           # ← zt-<slug>-automation cluster/ charts (if any) + scaffolded bootstrap-infra/
│   └── showroom-deployer/values-<slug>.yaml   # ← zt-<slug>-automation/values-<slug>.yaml
├── qa-automation/                        # ← template stubs, filled from existing test scripts if any
├── manifests/                            # ← only if the showroom shipped learner-facing manifests
├── catalog-info.yaml  CLAUDE.md  Makefile  mkdocs.yml  package.json
├── site.yml  ui-config.yml               # ← from zt-<slug>-showroom root
├── scaffold.py  .scaffolds/  hooks/  setup-claude-hooks.sh  .claude.settings.template.json
└── .github/workflows/gh-pages.yml        # ← from zt-<slug>-showroom
```

## Steps

### 1. Create the repo from the template fork

```bash
TEMPLATE=~/git/rhdp-publishing-house-template
REPO=~/git/ph-<slug>
mkdir -p $REPO
cp -R $TEMPLATE/scaffold.py $TEMPLATE/.scaffolds $TEMPLATE/publishing-house \
      $TEMPLATE/hooks $TEMPLATE/setup-claude-hooks.sh \
      $TEMPLATE/.claude.settings.template.json $TEMPLATE/.gitignore $REPO/
```

### 2. Import the RAC corpus

```bash
cp -R ~/git/zt-<slug>-rac $REPO/rac     # carries .decided/config.yaml, requirements/, assets/
```

If the legacy RAC repo kept `.decided/` at its root, move it to `$REPO/.decided/`
(decided commands run from the repo root).

### 3. Import content

```bash
cp -R ~/git/zt-<slug>-showroom/content $REPO/content
cp ~/git/zt-<slug>-showroom/{site.yml,ui-config.yml,package.json} $REPO/
mkdir -p $REPO/.github/workflows
cp ~/git/zt-<slug>-showroom/.github/workflows/gh-pages.yml $REPO/.github/workflows/
```

Fixes after import:
- `site.yml`: set `site.url` to `https://<org>.github.io/ph-<slug>`; keep
  `content.sources[0] = {url: ., start_path: content}` unchanged. If `start_page`
  references the component by name (`<name>::index.adoc`), keep the pair consistent.
- `content/antora.yml`: normalize the component `name:` to `modules` (the PH
  convention — all migrated ph- repos use it) and update `start_page` to
  `modules::index.adoc` to match; KEEP all per-environment attribute placeholders
  (`cluster.example.com`, `%password%`, `%GUID%`) exactly — never hardcode live
  cluster URLs.

### 4. Run the pattern scaffolder FIRST (before importing the gitops layer)

```bash
cd $REPO && python3 scaffold.py --pattern agd-open --automation gitops --migration
```

`--migration` never overwrites existing imported files, BUT if `automation/gitops/`
already exists it asks "Re-scaffolding will clear and recreate them. Continue? [y/N]"
and answering yes DELETES the imported cluster/ layer. So: run the scaffolder BEFORE
importing the cluster layer. Pass `--topology shared-cluster` in the same invocation
if the lab is shared-cluster.

### 5. Import automation

```bash
mkdir -p $REPO/automation/showroom-deployer
cp ~/git/zt-<slug>-automation/values-*.yaml $REPO/automation/showroom-deployer/
cp -R ~/git/zt-<slug>-automation/cluster $REPO/automation/gitops/cluster   # if present
cp -R ~/git/zt-<slug>-automation/tenant $REPO/automation/gitops/tenant     # if present
cp ~/git/zt-<slug>-automation/bootstrap.yaml $REPO/automation/gitops/bootstrap.yaml  # if present
```

- Update `repoURL`/`path:` inside `bootstrap.yaml` to point at this repo and the new
  `automation/gitops/cluster/...` path.
- Legacy `values-<slug>.yaml`: update `showroom.gitRepoUrl` to point at the ph-<slug>
  repo. Flag any file containing real cluster coordinates/wetty SSH as
  environment-specific (gitignore or placeholder per the repo's convention).
- Renamed environment values files (e.g. `values-<slug>-<env>.yaml`) keep their names.
- If the legacy repo shipped its own in-cluster showroom chart (e.g.
  `cluster/platform/bootstrap/showroom/`), keep it for bootstrap compatibility and
  note in `automation/showroom-deployer/README.md` that the deployer values supersede it.

### 6. Add the root files the template doesn't ship

- **`catalog-info.yaml`** — copy the shape from `~/git/ph-operations-rhoai/`; set
  `metadata.name: ph-<slug>`, `ph.rhdp.io/github-user`, `ph.rhdp.io/owner`, and the
  `ph.rhdp.io/migrated-repo` / `ph.rhdp.io/migrated-automation-repo` annotations
  pointing at the legacy showroom/automation GitHub URLs.
- **`Makefile`** — combined build + deployer targets (see
  `.agents/skills/workshop-do/SKILL.md` Step 7 or `~/git/ph-operations-rhoai/Makefile`);
  `VALUES ?= automation/showroom-deployer/values-$(SLUG).yaml`.
- **`mkdocs.yml`** — TechDocs nav over `publishing-house/spec` (one entry per module outline).
- **`CLAUDE.md`** — start from the template's `CLAUDE.md`, add a `## RAC` section
  (decided CLI, RHAIBU key, `decided validate .`) — see
  `~/git/redhat-ai-inference/CLAUDE.md` for the hybrid.
- **`qa-automation/`** — template stubs arrive via scaffold.py; port any existing
  legacy test scripts into `e2e.yml` module-by-module.

### 7. Synthesize the publishing-house spec artifacts

Reverse-engineer from the RAC corpus + imported content (same as workshop-orient
Step 9, or the PH `migrate` skill's approach):

- `publishing-house/spec.yaml` — `project.intake_type: migration`, `project.slug:
  ph-<slug>`, `spec.title`/`learning_objectives` from the workshop requirement,
  `spec.modules[]` from the module requirements, `spec.environment` from the design.
- `publishing-house/spec/design.md` — summarize `rac/designs/` (Overview, Target
  Audience, Prerequisites, Learning Objectives, Content Type, Products & Technologies).
- `publishing-house/spec/modules/module-NN-<slug>.md` — one outline per module
  requirement; filename stem MUST match the imported page stem
  (`NN-module-MM-<slug>.adoc` → `module-MM-<slug>.md`).
- `publishing-house/worklog.yaml` — first entry records the migration.

### 8. Validate

```bash
cd ~/git/ph-<slug>
decided validate .                                   # RAC corpus (config at repo root)
make build                                           # Antora build green
helm template showroom ~/git/zt-showroom-deployer \
  -n showroom-<slug> -f automation/showroom-deployer/values-<slug>.yaml
python3 publishing-house/tools/ph-check.py           # if Central auth available
grep -rn "AKIA\|aws_secret\|hosted_zone" . || true   # secrets scan
```

### 9. Hand off to git

Git state changes belong to the human git-master. Ask the user to: `git init &&
git add -A && git commit`, create the GitHub repo, push, and (optionally) register
the project with the PH Central API.

## Sanity checklist

- [ ] `rac/` corpus present, `decided validate .` exits 0
- [ ] `content/` builds via `make build`
- [ ] `site.yml` content source is `url: .` + `start_path: content`
- [ ] No hardcoded cluster URLs in `content/antora.yml` or templates
- [ ] `automation/showroom-deployer/values-<slug>.yaml` points at the ph-<slug> repo
- [ ] `catalog-info.yaml` annotations complete (incl. migrated-repo for migrations)
- [ ] `publishing-house/spec.yaml` module outline stems match page stems
- [ ] Secrets scan clean
