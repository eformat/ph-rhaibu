---
name: workshop-orient
description: >
  Plan an OpenShift/RHOAI workshop using Requirements As Code (RAC). Takes observations
  from workshop-observe (or freeform input) and produces structured RAC artifacts:
  requirements, decisions, and designs in the rac/ directory of the workshop's
  publishing-house repo (~/git/ph-<slug>/rac/), then synthesizes the publishing-house
  spec artifacts (publishing-house/spec.yaml, spec/design.md, spec/modules/) from them.
  Uses the asdecided-core CLI to scaffold and validate artifacts. Use when someone wants
  to "plan a workshop", "create workshop requirements", "orient on workshop content",
  "define workshop scope", "write workshop RAC", "decide workshop structure", or "design
  a workshop". Also trigger for "workshop planning", "lab design", "module breakdown",
  or "workshop architecture". This is the second step of the OODA workshop pipeline
  (Observe -> Orient -> Do -> Act). Do NOT use for writing actual workshop content (use
  workshop-do) or for deploying workshops (use workshop-act).
triggers:
  keywords:
    - "workshop orient"
    - "workshop plan"
    - "workshop requirements"
    - "workshop design"
    - "orient workshop"
    - "ooda orient"
    - "rac workshop"
    - "workshop scope"
  matchMode: any
enabled: true
---

# Workshop Orient

Plan a workshop by producing machine-validatable RAC artifacts that serve as the
contract between planning and implementation, then bridge them into the
publishing-house spec artifacts the PH tooling consumes.

## Skill coordination

- Follow the **`decided-capture`** skill's approach for artifact creation — interview the
  user and propose, the human ratifies, `decided validate` closes.
- Follow the **`decided-review`** skill's approach for validation — work findings
  worst-first until validation and relationship checks pass. (`decided-artifacts` covers
  authoring/linking conventions.)
- Use `decided schema <type>` to get real artifact schemas. Never invent fields or sections.
- See `../docs/WORKSHOP-COMMON-RULES.md` for shared AsciiDoc, image, security,
  and quality rules.

## Prerequisites

Check these before starting. If any are missing, print what's needed and stop.

**Required tools:**
- `git` — for the workshop repo (state-changing git commands belong to the human
  git-master; the agent prepares files and asks the user to run them)
- `decided` CLI — `decided --version` ([install](https://github.com/asdecided/core/releases))

**Optional helper skills:** `decided-capture` / `decided-review` / `decided-artifacts` live
in asdecided-core (`~/git/asdecided-core/.claude/skills/`). They're convenience wrappers over
the same `decided` CLI — this workflow works with just the `decided` CLI on PATH if they
aren't loaded.

**Required state:**
- A workshop repo at `~/git/ph-<slug>/` with a `rac/` corpus and `.decided/config.yaml`
  (created by workshop-observe, or initialized in Step 1 below)
- An observation document in `rac/assets/` (from workshop-observe) or freeform input

## Workflow

### 1. Locate the workshop repo (init if needed)

Determine the workshop slug from `$ARGUMENTS`. If no argument is provided, discover
existing workshop repos:

```bash
ls -d ~/git/ph-*/rac/ 2>/dev/null
```

If exactly one exists, use it. If multiple exist, list them and ask the user to
choose. If none exist, ask the user for the slug.

The repo lives at `~/git/ph-<slug>/`. All `decided` commands in this workflow run
from that directory (the `.decided/` config sits at the repo root; artifacts live in
`rac/`).

If the repo does not exist (workshop-observe was not run first):

```bash
mkdir -p ~/git/ph-<slug>/rac/assets
cd ~/git/ph-<slug>
decided init --key RHAIBU
mkdir -p rac/requirements rac/decisions rac/designs rac/roadmaps rac/prompts
```

Then ask the user to initialize git (`git init && git add -A && git commit`) — git
state changes belong to the human git-master.

If the repo exists, check for `.decided/config.yaml` and initialize if missing.

### 2. Read observations

Look for observation documents in `rac/assets/observations-*.md`.
If found, read and extract: user flows, features, concepts, target audience,
estimated module count. If no observation document exists, accept `$ARGUMENTS` as
inline text describing the workshop concept and interview the user to establish the same.

### 3. Read the real schemas

Before creating any artifact, read the schema for each type you will use:

```bash
decided schema requirement
decided schema decision
decided schema design
```

Use only the section names, frontmatter fields, and controlled values these schemas
define. Never guess or hard-code a field the schema does not list.

**Frontmatter restriction:** `decided new` scaffolds the correct frontmatter fields
(`schema_version`, `id`, `type`). Do not add extra fields like `status` or `category`
to the YAML frontmatter — those are body-level metadata sections (e.g., `## Status`),
not frontmatter keys. The only supported frontmatter fields are: `schema_version`,
`id`, `type`, `relationships`, `tags`.

### 4. Create the workshop-level requirement

Scaffold with `decided new`, then Read the scaffold, then Write the populated content:

```bash
cd ~/git/ph-<slug>
decided new requirement requirements/<workshop-slug>.md
```

After `decided new` creates the file, you MUST Read it before Writing content into it.
`decided new` writes a TODO-placeholder file to disk — Read it to satisfy the
Read-before-Write requirement, then Write the full content.

Fill in:
- **Title**: the workshop name
- **Description**: what the workshop teaches and who it is for
- **Requirements section**: testable acceptance criteria using MUST/SHOULD/MAY:
  - `[REQ-001] Learner MUST be able to log into the OpenShift console`
  - `[REQ-002] Learner MUST deploy a model to RHOAI model serving`
  - `[REQ-003] Learner SHOULD query the model via REST API`
- **Dependencies**: prerequisite knowledge, cluster requirements, operator versions

### 5. Create per-module requirements

For each workshop module identified in observations, scaffold a requirement:

```bash
cd ~/git/ph-<slug>
decided new requirement requirements/<workshop-slug>-module-<N>.md
```

Each module requirement includes:
- Learning objectives (3-5 testable outcomes)
- Hands-on steps outline (ordered list of user actions)
- Estimated duration
- Required cluster state (what must be deployed/running before this module)
- Verification criteria (what the learner checks to confirm success)

Link each module to the parent workshop requirement using a `## Related Requirements`
section. Use bare artifact IDs only — one per line with a list marker. The relationship
resolver matches the full line text against the identifier index, so descriptive
suffixes break resolution:

```markdown
## Related Requirements

- RHAIBU-KXYZ1234ABCD
```

Do NOT write `- RHAIBU-KXYZ1234ABCD: Human Readable Title` — the title suffix
prevents asdecided-core from resolving the reference.

### 6. Create architectural decisions

For each significant technology choice, scaffold a decision:

```bash
cd ~/git/ph-<slug>
decided new decision decisions/<topic-slug>.md
```

At minimum, create decisions for:
- **Content format** — Antora/AsciiDoc showroom under `content/` in the publishing-house
  repo, single Antora component (`content/antora.yml`), built by the root `site.yml`
  (status: decided). Reference the in-repo exemplars `~/git/redhat-ai-inference/`
  and `~/git/ph-deploy-configure-rhoai/`.
- **Repo strategy** — ONE publishing-house repo per workshop (`ph-<slug>/`): planning
  (`rac/` + `publishing-house/`), content (`content/`), and automation (`automation/`)
  live together. No separate showroom/automation/RAC repos (status: decided).
- **Infrastructure approach** — showroom deployment via the reusable `zt-showroom-deployer`
  Helm chart, with per-workshop values at `automation/showroom-deployer/values-<slug>.yaml`
  and `helm` targets in the repo-root `Makefile` (status: decided). Labs that must
  provision cluster-side resources add GitOps layer charts under `automation/gitops/`
  (see `ph-operations-rhoai` for the two-layer pattern: `automation/gitops/cluster/*`
  app-of-apps + deployer values). Chart source: `~/git/zt-showroom-deployer/`
  (published as `eformat/showroom-deployer`).

Add additional decisions only when the workshop requires them (e.g., model serving,
authentication, GPU allocation). Not every workshop needs all decision types.

Include `## Context`, `## Decision`, and `## Consequences` sections per the schema.
Link decisions to the requirements they satisfy using `## Related Requirements`
with bare artifact IDs (same format as requirement links — no descriptive suffixes).

### 7. Create the workshop design

```bash
cd ~/git/ph-<slug>
decided new design designs/<workshop-slug>-architecture.md
```

Document:
- **Module flow** — which modules depend on which (ordered list or diagram)
- **Repo structure** — mapping to the publishing-house layout (content/ under
  `content/modules/ROOT/pages/`, site.yml + ui-config.yml at the repo root,
  automation/ + qa-automation/, rac/ + publishing-house/)
- **Infrastructure requirements** — the showroom-deployer chart values (namespace, git repo,
  domain/apiUrl, wetty — see `references/infra-patterns.md`), plus any cluster-side resources
  the *content* assumes already exist (e.g. RHOAI operator, GPUs, storage). If the lab must
  provision those itself, document the `automation/gitops/` layer here — the workshop deploys
  it via GitOps, not hand-run `oc` commands
- **Parameter inventory** — all `{attribute}` values needed in `content/antora.yml`:
  - `lab_name`, `user`, `password`, `guid`
  - `openshift_api_url`, `openshift_console_url`, `openshift_cluster_ingress_domain`
  - Workshop-specific URLs (model endpoints, dashboards, etc.)
- **Single-repo strategy** — planning, content, and automation in `ph-<slug>/`

Link the design to requirements and decisions using `## Related Requirements` and
`## Related Decisions` sections with bare artifact IDs.

### 8. Validate the corpus

Run validation and fix any findings:

```bash
decided validate ~/git/ph-<slug>/
decided review ~/git/ph-<slug>/
decided relationships ~/git/ph-<slug>/ --validate
```

Work findings in priority order (blocking first, then advisory). Re-validate after
each fix. The work is done only when `decided validate` and
`decided relationships --validate` both exit 0.

### 9. Synthesize the publishing-house spec artifacts

The PH frame (`publishing-house/tools/ph-check.py`, Central API, RHDH TechDocs) reads
its own spec artifacts, not the RAC corpus. Derive them from the ratified RAC — this
is the bridge that keeps `rac/` the source of truth while the PH tooling stays happy.
Follow the shapes in the in-repo exemplars (`~/git/redhat-ai-inference/`,
`~/git/ph-operations-rhoai/`) and the template at `~/git/rhdp-publishing-house-template/`:

- **`publishing-house/spec.yaml`** — machine manifest. Map from RAC:
  - `project.slug` = `ph-<slug>`, `project.content_type: lab`,
    `project.showroom_type: classic`, `project.automation_type: gitops` (or `both` if
    Ansible roles are planned), `project.intake_type: new` (factory-built), the
    workshop description
  - `spec.title` = workshop title, `spec.learning_objectives` = distilled from the
    workshop + module requirements
  - `spec.modules[]` = one entry per module requirement (`id: module-NN`,
    `title`, `duration_min`, `status: planned` — flip to `complete` during Act)
  - `spec.environment` = topology, OCP version, cluster type from the RAC design
  - Leave `jira_ticket`/`workflow_id` as `none`/`null` unless the project is registered
    with the PH Central API
- **`publishing-house/spec/design.md`** — human design doc following
  `publishing-house/spec/design.md` in the template repo (Overview, Target Audience,
  Prerequisites, Learning Objectives, Content Type, Products & Technologies).
  Summarize the RAC design; do not duplicate it — link to `rac/designs/`.
- **`publishing-house/spec/modules/module-NN-<slug>.md`** — one outline per module
  (template: `publishing-house/spec/module-outline-template.md` in the template repo;
  sections: Brief Overview / Audience and Time / Learning Objectives / Lab Structure /
  Detailed Steps / Key Takeaways / Infrastructure Notes). The outline filename stem MUST
  match the future content page stem (`module-NN-<slug>`) — the PH config-reviewer
  enforces outline ↔ page coupling.
- **`publishing-house/worklog.yaml`** — record the Orient session as the first entry.

### 10. Report and handoff

Print a summary table:

| ID | Type | Title | Path |
|----|------|-------|------|
| RHAIBU-REQ-001 | requirement | Workshop Overview | rac/requirements/... |
| RHAIBU-REQ-002 | requirement | Module 1: ... | rac/requirements/... |
| RHAIBU-DEC-001 | decision | Content Format | rac/decisions/... |
| RHAIBU-DES-001 | design | Workshop Architecture | rac/designs/... |

Plus the synthesized PH artifacts:
- `publishing-house/spec.yaml`
- `publishing-house/spec/design.md`
- `publishing-house/spec/modules/module-NN-*.md`

Prompt: "Workshop planning complete. Run `/workshop-do` to scaffold the content and
automation from these requirements."

## Guardrails

- Follow the asdecided-core artifact schemas exactly. Run `decided validate` after every artifact creation.
- Keep acceptance criteria testable and specific. "User understands X" is not testable. "User can run `oc get inferenceservice` and see a READY status" is testable.
- Do not skip decision artifacts — they capture tooling rationale and prevent re-litigation later.
- Reference exemplar repos by path when documenting decisions.
- Use the RHAIBU namespace prefix for all artifact IDs (handled by `decided new`).
- Human ratification is mandatory — present drafts and confirm type, title, and relationships with the user before writing to disk.
- Do not invent requirements the user did not express. Interview to fill gaps.

## Out of scope

- Writing workshop content or AsciiDoc pages — use `workshop-do`
- Analyzing screenshots — use `workshop-observe`
- Deploying or testing — use `workshop-act`
- Recording ownership, sprints, priorities, or due dates — those belong in a work tracker

## Related Skills

- `/workshop-observe` — Analyze demo screenshots for observations
- `/workshop-do` — Scaffold content and automation from these requirements
- `/PH-MIGRATION-RUNBOOK` (see `../docs/`) — Migrate an existing zt-* workshop into this format
