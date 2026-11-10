# ph-rhaibu — OODA Workshop Factory (publishing-house format)

Workshop creation pipeline based on the OODA loop. Each phase is a skill in
`.agents/skills/` (agent-neutral layout — discovered natively by OpenCode from this
repo; see the Skills section below).

## Pipeline

```
Observe → Orient → Do → Quality → Act → Publish
(images)   (RAC)   (code) (enrich)  (deploy+test)  (gh-pages + PH tooling + catalog)
```

**Each workshop is ONE publishing-house repo** at `~/git/ph-<slug>/` — planning
(`rac/` + `publishing-house/`), content (`content/`), automation (`automation/`), and
QA (`qa-automation/`) live together.

| Phase | Skill | Input | Output |
|-------|-------|-------|--------|
| Observe | `workshop-observe` | Demo screenshots/keyframes | `~/git/ph-<slug>/rac/assets/observations-<slug>.md` |
| Orient | `workshop-orient` | Observation doc or freeform | `rac/requirements|decisions|designs/` + synthesized `publishing-house/spec.yaml`, `spec/design.md`, `spec/modules/` |
| Do | `workshop-do` | RAC artifacts | `content/`, `automation/`, `qa-automation/`, root files in `~/git/ph-<slug>/` |
| Act | `workshop-act` | Scaffolded repo + RAC requirements | Deployed workshop on a prelude cluster |
| Publish | `workshop-act` (step 8) | Validated repo | Push → repo gh-pages site; `ph-check.py`/`ph-development.py`; `/catalog-builder` |
| Quality | `quality-enrichment` | Enriched workshops + RAC + code repos | Per-workshop quality report + content fixes |

### Workshop repo structure

```
~/git/ph-<slug>/
  rac/                          # decided corpus — the planning contract
    requirements/ decisions/ designs/ roadmaps/ prompts/ assets/
  publishing-house/             # PH frame
    spec.yaml                   # machine manifest (project/spec/modules/environment)
    spec/design.md  spec/modules/module-NN-<slug>.md
    tools/ph-*.py  worklog.yaml  decisions/  reviews/
  content/                      # Antora component (name: modules)
    antora.yml  modules/ROOT/{nav.adoc,pages/,assets/images/}
    supplemental-ui/  lib/
  automation/
    gitops/                     # cluster layer (only if the lab provisions resources)
    showroom-deployer/values-<slug>.yaml
  qa-automation/                # healthcheck.yml + e2e.yml
  manifests/                    # optional learner-facing kustomize
  catalog-info.yaml  CLAUDE.md  Makefile  mkdocs.yml  package.json
  site.yml  ui-config.yml  scaffold.py  .scaffolds/  hooks/
  .github/workflows/gh-pages.yml
```

Publishing is PH-native: pushing `main` triggers the repo's own gh-pages workflow
(`https://<org>.github.io/ph-<slug>/`); PH compliance runs via
`publishing-house/tools/ph-check.py` / `ph-development.py`; the RHPDS catalog entry
comes from `/catalog-builder`. There is no separate lessons repo.

This factory repo (`ph-rhaibu`) holds only skills and tooling — no per-workshop
state. Pipeline changes are developed in branches of this repo.

## Exemplar Repos (read, don't clone)

- **Hybrid with RAC**: `~/git/redhat-ai-inference/` — canonical PH frame + `rac/` corpus
- **Two-layer automation**: `~/git/ph-operations-rhoai/` — `automation/gitops/cluster/*`
  app-of-apps + `automation/showroom-deployer/values-<slug>.yaml` + Makefile deployer targets
- **Classic content + manifests**: `~/git/ph-deploy-configure-rhoai/` — content/, manifests/,
  qa-automation conventions
- **Template**: `~/git/rhdp-publishing-house-template/` — scaffold.py + .scaffolds/ +
  publishing-house/tools frame (Mike's fork of rhpds/rhdp-publishing-house-template)
- **Deployer chart**: `~/git/zt-showroom-deployer/` (published as `eformat/showroom-deployer`
  at https://eformat.github.io/helm-charts) — the per-workshop values live at
  `automation/showroom-deployer/values-<slug>.yaml`

## Skill Coordination

- **openshift-workshop-builder** — handles Antora/showroom content scaffolding (called by workshop-do)
- **playwright-cli** — browser automation for testing (called by workshop-act)
- **openshift-4-21-expert** — OpenShift console flows and cluster admin steps
- **openshift-ai-3-3-expert** — RHOAI workbenches, model serving, pipelines
- **verify-content** (`/verify-content`) — validates showroom content against Red Hat quality standards; quality gate in workshop-act. Vendored from the RHDP skills marketplace and registered as a slash command.
- **catalog-builder** (`/catalog-builder`) — builds an RHDP AgnosticV catalog entry at publish time. Vendored from the RHDP skills marketplace and registered as a slash command.
- **quality-enrichment** (`/quality-enrichment`) — audits and uplifts bulk-scaffolded workshops in the ralf-wiggum monorepo. Validates RAC coverage, audits screenshots, enriches with code from companion repos (`red-hat-ai-examples`, `trustyai-llm-demo`, etc.), and fixes content quality defects. Operates post-enrichment, before Act.

## RAC

Requirements As Code via [`asdecided-core`](https://github.com/asdecided-core/releases). Assumes `decided` is on PATH.

Artifact ID prefix: `RHAIBU-`. Key CLI commands: `decided schema`, `decided new`, `decided validate`, `decided review`, `decided relationships`.

Each workshop's RAC corpus lives INSIDE its publishing-house repo (`~/git/ph-<slug>/rac/`)
with the `.decided/` config at the repo root. Skills scaffold it with `decided init
--key RHAIBU` during Observe/Orient.

In the workshop repo the RAC corpus is:

```bash
mkdir -p ~/git/ph-<slug>/rac/{requirements,decisions,designs,roadmaps,prompts,assets}
```

The bridge to the PH frame: workshop-orient synthesizes `publishing-house/spec.yaml`,
`publishing-house/spec/design.md`, and `publishing-house/spec/modules/` from the
ratified RAC corpus. The PH tooling (`ph-check.py`, Central API, TechDocs) reads the
spec artifacts; the RAC corpus stays the source of truth.

## Migration

Legacy `zt-*` three-repo workshops (zt-<slug>-rac / -showroom / -automation) migrate
into this format via `.agents/skills/docs/PH-MIGRATION-RUNBOOK.md`.

## Skills — discovery

Skills live in `.agents/skills/<skill-id>/SKILL.md` (ID = directory name, kebab-case;
`description` frontmatter drives discovery). OpenCode discovers them **only when the
session workspace is this factory repo** — no config, global registration, or
symlinks. Run sessions from `~/git/ph-rhaibu/` and operate on workshop repos by path.

Paths inside a skill body are **relative to the skill's directory** — both OpenCode
and Claude Code provide that base directory when the skill loads:

- `references/<file>` — the skill's own supporting files
- `../docs/<file>` — shared rules (`WORKSHOP-COMMON-RULES.md`, `PH-MIGRATION-RUNBOOK.md`)
- `../<other-skill>/...` — another factory skill's files

## Future

Skills will integrate with hermes kanban for independent step/stage processing with LLM-as-judge validation.
