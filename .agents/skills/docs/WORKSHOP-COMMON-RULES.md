# Workshop Common Rules

**Version**: 1.4

Shared contracts for all workshop-* skills in the OODA pipeline. Every workshop skill
MUST reference this file and follow these rules.

---

## 0. Security — No Secrets (MANDATORY)

Skills MUST NOT produce, accept, or store sensitive information in any file.

**Prohibited content:**
- Real passwords, API keys, tokens, secrets, or credentials
- Internal hostnames, VPN endpoints, bastion IPs, or non-public URLs
- SSH keys, certificates, kubeconfig contents, or private key material

**Required patterns:**
- Use `{attribute}` placeholders for all environment-specific values (e.g., `{password}`, `{openshift_console_url}`)
- Use `example.com`, `192.0.2.x` (RFC 5737), and dummy UUIDs in examples
- Never commit auth state files (playwright state, `.auth-state.json`, kubeconfig)

**Skill behavior:**
- Never ask users to paste credentials into the conversation
- Never write credentials to generated files
- Never echo back real credentials provided by the user

**Secret mechanisms (DP-09 in `DEMO-PATTERNS.md`):** every credential path gets a
named mechanism — `REPLACE_AT_APPLY` bootstrap values injected at the one-time apply;
hook-Job-generated create-if-absent in-cluster secrets (never overwritten); the
gitignore-negation idiom for per-cluster deployer values (`values-*.yaml` ignored,
`!values-<slug>.yaml` placeholder template re-included). Content consumes
`%PASSWORD%`/`injected-at-runtime` antora attributes; learner-chosen secrets ride env
vars, never files; agent auth lives in `~/.config/`, outside the repo.

---

## 1. AsciiDoc Code Blocks (REQUIRED)

### Executable Commands

Use `[source,role="execute"]` for every command the learner should run. This enables
the Showroom UI copy/execute button.

```asciidoc
[source,role="execute"]
----
oc get pods -n my-project
----
```

When attribute substitution is needed:

```asciidoc
[source,bash,role="execute",subs="attributes"]
----
oc login {openshift_api_url} -u {user} -p {password}
----
```

**Do NOT use `[.copypaste]`** — it is the older nookbag pattern and is deprecated.

### Non-executable Blocks

- Config/data blocks: `[source,yaml]`, `[source,json]` — no `role="execute"`
- Expected output: plain `----` listing with no source declaration
- AsciiDoc examples: `[source,asciidoc]`

### List Continuation

The `+` on its own line continues a list item (keeps numbering intact):

```asciidoc
. Run the command:
+
[source,role="execute"]
----
oc get pods
----
```

---

## 1a. YAML/Config Callouts (RECOMMENDED)

Non-executable `[source,yaml]` / `[source,json]` blocks that represent manifests or config
applied to the cluster SHOULD use AsciiDoc numbered callouts to annotate important fields.

Full reference with examples: `skills/openshift-workshop-builder/references/yaml-callouts.md`.

### When to use

- Block is a manifest the learner applies (`oc apply`, `oc create`, `oc process`)
- Block has fields the learner must understand to succeed in later exercises
- Field names are opaque without context (e.g., `modelFormat.name`, `runtime`)

### When NOT to use

- Executable blocks (`[source,role="execute"]`) — callout markers would be copied to terminal
- Expected output or trivial config (1–2 obvious fields)
- Surrounding prose already explains every field

### Rules

- Maximum **5 callouts per block** — more is noise
- Each callout: one sentence — `<1> *Bold label* — explanation.`
- Use `[source,yaml,subs="attributes,callouts"]` when the block also has `{attribute}` placeholders
- Markers must be sequential (`<1>`, `<2>`, `<3>`) and match between inline and explanation list

```asciidoc
[source,yaml]
----
spec:
  predictor:
    model:
      modelFormat:
        name: vLLM                       # <1>
      resources:
        limits:
          nvidia.com/gpu: "1"            # <2>
----
<1> *Serving runtime* — selects vLLM; other options: `ovms`, `caikit`.
<2> *GPU limit* — requires exactly one GPU; `0` falls back to CPU.
```

---

## 2. Image Conventions (REQUIRED)

All images go in `content/modules/ROOT/assets/images/`.

### Required Syntax

```asciidoc
image::filename.png[Alt text,link=self,window=blank,width=700]
```

- `link=self` — click opens full-size image
- `window=blank` — opens in new tab
- `width=700` — constrain display width (500-800px typical)
- Alt text is the first positional parameter — make it meaningful for accessibility

### Content Correctness (REQUIRED)

An embedded image must show what its alt text and surrounding step text describe.
File existence is not correctness: verify every capture visually after taking it
(final URL/title plus a read of the image). 404 pages, OAuth error pages, welcome
tour modals, loading spinners, and empty states that contradict the step are worse
than no image — they teach the wrong thing. If the described state does not exist
in the current UI build, remove the embed and leave a `// TODO(<reason>)` comment
instead of shipping a mismatched capture. A capture file shared between workshops
may be correct in one and wrong in another — audit each embed.

### Filename Conventions

- Use deterministic names so re-captures replace existing files: `01-keycloak-login.png`, `06-workbench-creation-form.png`
- Use 2-digit prefix for ordering within a module
- Descriptive slug after the prefix — no `image1.png`

### Placeholders

If an image doesn't exist yet:

```asciidoc
// TODO: capture screenshot
image::create-workbench-form.png[Workbench creation form,link=self,window=blank,width=700]
```

---

## 3. Attribute Substitution (REQUIRED)

Never hardcode cluster-specific values. Use Antora attributes defined in
`content/antora.yml`:

```yaml
asciidoc:
  attributes:
    openshift_console_url: https://console-openshift-console.{deployer_domain}
    user: user1
    password: openshift
    project_name: demo
```

Use `subs="attributes"` on code blocks that contain `{attribute}` values. Without it,
curly-brace placeholders render literally.

---

## 4. RAC Integration (REQUIRED)

Workshop artifacts live in `rac/` inside the workshop's publishing-house repo
(`~/git/ph-<slug>/rac/`). The publishing-house spec artifacts
(`publishing-house/spec.yaml`, `publishing-house/spec/`) are synthesized from RAC by
workshop-orient; the RAC corpus remains the source of truth.

**Read from:**
- `rac/requirements/` — acceptance criteria, learning objectives, module structure
- `rac/decisions/` — architectural decisions (content format, infra approach)
- `rac/designs/` — parameter inventory, module flow

**Write to:**
- `rac/assets/` only — observation documents, screenshot evidence, validation reports

**Never modify:**
- `rac/requirements/`, `rac/decisions/`, `rac/designs/` — if a requirement is wrong,
  flag it and ask the user to update it via workshop-orient

---

## 5. Quality Gate (REQUIRED)

Before declaring content ready, run the `verify-content` skill (vendored in this repo
under `../verify-content/`).

```
/verify-content
```

This launches parallel agents per module, checking against Red Hat quality standards:
- AsciiDoc structure and formatting
- Accessibility compliance
- Red Hat style guide
- Technical accuracy

**Severity handling:**
- **Critical** / **High** — must fix before proceeding
- **Warning** — report but do not block
- **Info** / **Recommendation** — optional improvement

---

## 6. Related Skills Convention (REQUIRED)

Every SKILL.md ends with a `## Related Skills` section listing connected skills:

```markdown
## Related Skills

- `/workshop-orient` — Plan the workshop from observations
- `/workshop-do` — Scaffold content and infrastructure
- `/verify-content` — Validate content against Red Hat standards
```

---

## 7. Skill Coordination Pattern

Skills in the OODA pipeline follow a sequential handoff:

```
Observe → Orient → Do → Quality → Act
                    ↑       ↑        ↑
              screenshot    |   verify-content
                     quality-enrichment
```

Each skill reads from the previous phase's output and writes to its own output
location. Skills communicate through:
- RAC artifacts (rac/requirements, rac/decisions, rac/designs)
- The single publishing-house repo (content/, automation/, qa-automation/)
- Structured reports (validation tables, evidence maps)

---

## 8. Prose Style (REQUIRED)

Every module page MUST follow these four prose rules. Full AsciiDoc examples with bad/good
side-by-sides are in
`skills/openshift-workshop-builder/references/prose-style.md`.

### P.1 — Module summary format

End every module page with `== Module summary` containing three bold-label sections:

```asciidoc
== Module summary

**What you accomplished:**

* Past-tense action (3 bullets)

**Key takeaways:**

* Present-tense declarative fact (3 bullets)

**Next steps:**

One or two sentences of prose linking to the next module — never bullets.
```

The conclusion module omits **Next steps**.

### P.2 — Exercise transitions

Every `== Exercise N` boundary must have at least one bridging sentence (after `=== Verify`,
before the next heading). Three interchangeable formulas:

- **Callback-then-pivot**: "Now that you've [done X], let's [do Y]."
- **Problem-then-purpose**: State the gap, then what the next exercise closes.
- **Forward reference**: "You will explore this in Exercise N."

### P.3 — Workaround handling

Any workaround or known deviation: NOTE/WARNING **before** the command → "This is expected."
→ one sentence why → then the command block. Never explain after the command.
Complex optional depth → `[%collapsible]` block.

### P.4 — Admonition type

| Admonition | Use for |
|---|---|
| `TIP` | Orientation, persona framing, practical shortcuts |
| `NOTE` | Non-determinism, expected friction, key conceptual asides |
| `IMPORTANT` | Structural, safety, or ordering constraints |
| `WARNING` | Destructive or irreversible operations |

**Severity handling (verify-content check IDs P.1–P.4):**
- P.1 missing or malformed → Warning
- P.2 missing transition → Warning
- P.3 explanation after command → Warning
- P.4 wrong admonition type → Info

---

---

## 9. Feature Maturity Depth Rules (REQUIRED)

Workshop content depth MUST vary based on the feature's maturity level. Define
the maturity in `content/antora.yml` and adjust content accordingly.

### M.1 — GA (Generally Available)

Default mode. Full hands-on exercises with real commands, `=== Verify` blocks
after each exercise, YAML callouts on applied manifests. No maturity banner
needed.

### M.2 — TP (Technology Preview)

Same hands-on depth as GA, plus:

- A WARNING admonition at the top of `index.adoc` rendered conditionally:

```asciidoc
ifeval::["{feature_maturity}" == "TP"]
WARNING: This feature is a *Technology Preview*. Technology Preview features are
not supported with Red Hat production service-level agreements (SLAs) and might
not be functionally complete.
endif::[]
```

- NOTE admonitions before any steps whose API or manifest schema may change
  between releases.

### M.3 — DP (Developer Preview)

Guided-tour mode — descriptive prose, observe/explore steps, fewer execute
blocks. No full end-to-end exercises.

- A WARNING admonition at the top of `index.adoc` rendered conditionally:

```asciidoc
ifeval::["{feature_maturity}" == "DP"]
WARNING: This feature is a *Developer Preview*. Developer Preview features are
provided as-is with no support and no guarantee of future availability.
endif::[]
```

### M.4 — Maturity attribute

`content/antora.yml` MUST define:

```yaml
asciidoc:
  attributes:
    feature_maturity: "GA"   # or "TP" or "DP"
```

This attribute enables conditional banners via `ifeval` and allows verify-content
to check maturity-specific rules.

**Severity handling (verify-content check IDs M.1):**
- Missing maturity banner when TP/DP → Warning
- Missing `feature_maturity` attribute → Info

---

## 10. Documentation Grounding (RECOMMENDED)

Product documentation is the source of truth for all commands, manifests, console
navigation paths, and API calls. Workshop content SHOULD be grounded in
extracted documentation to prevent fabrication.

### G.1 — Doc extraction

When product documentation is available as PDFs, extract text:

```bash
pdftotext <product-docs>.pdf /tmp/<slug>-docs/<name>.txt
```

The text files serve as grepable source material for all workshop content. Store
extraction references in the RAC corpus: `~/git/ph-<slug>/rac/assets/doc-extracts/`.

### G.2 — Grounded commands

Every `oc` command, YAML manifest, and console navigation path in a workshop
SHOULD be traceable to the product documentation. If the command comes from
upstream docs, community guides, or the author's experience, note the source.

### G.3 — NOT-IN-DOCS reporting

When content cannot be found in the product documentation, mark the section:

```asciidoc
// NOT-IN-DOCS: <feature-name> — <reason this content has no doc backing>
```

This marker signals that reviewers should assess whether the gap is a product
documentation issue or a content error. The verify-content skill checks for
orphaned NOT-IN-DOCS markers.

### G.4 — Version coupling

Doc extractions MUST be tagged with the product version (e.g., RHOAI 3.5). When
the product version changes, extractions must be refreshed and affected workshop
content re-audited against the new docs.

**Severity handling (verify-content check ID G.3):**
- Orphaned `// NOT-IN-DOCS:` markers → Info

---

## 11. Cross-Workshop Navigation (RECOMMENDED)

Workshops that are part of a multi-workshop catalog SHOULD include navigation
links to related and prerequisite workshops.

### N.1 — "Where next in the catalog"

Conclusion pages SHOULD include a `== Where next in the catalog` section after
the resources list, containing 1–3 Antora cross-component xrefs:

```asciidoc
== Where next in the catalog

Continue your learning journey with these related workshops:

* xref:sibling-slug::index.adoc[Sibling Workshop Title]
* xref:another-slug::index.adoc[Another Workshop Title]
```

### N.2 — Prerequisite xrefs

Index pages SHOULD list prerequisite workshops in the `== Prerequisites` section:

```asciidoc
* Recommended prerequisite: complete the xref:prereq-slug::index.adoc[Prerequisite Workshop] workshop first
```

### N.3 — Standalone coherence

Each workshop MUST remain standalone-buildable and understandable even with
cross-references. When a workshop references content from a sibling, include
a bridge sentence:

> If you have not completed the <sibling> workshop, the summary below covers
> what you need.

Cross-component xrefs only resolve when both workshops are registered in the
same Antora playbook (`antora-playbook.yml`). Workshops that may be deployed
standalone should treat cross-references as optional enhancements.

**Severity handling:**
- N.1–N.3 are RECOMMENDED, not REQUIRED. Skip for workshops not part of a catalog.

---

## 12. Content ↔ QA Coupling (REQUIRED)

The lab's QA is the lab's solve — mined from the live ph-* corpus (full catalog:
`DEMO-PATTERNS.md`, rules DP-01…DP-05c). Summary contract:

- **DP-01** Every `== Exercise` has a `=== Verify` with a literal expected value.
- **DP-02** Negative tests included (HTTP 401; `oc auth can-i --as=… --as-group=…`
  impersonation for RBAC — no persona passwords needed).
- **DP-03** `qa-automation/e2e.yml` replays the student's commands; task names embed
  the exercise (`"M5-Ex2 | Polling for RHCL CSV"`); header carries the contributor
  contract (Purpose/Idempotency/Scope/Runtime/Retry/"Adding modules for your lab").
- **DP-04** `healthcheck.yml` (provisioning-only, <30 s, exit codes) is a different
  job from `e2e.yml` (learner replay, idempotent). Never blend.
- **DP-05** Idempotent QA; honest skip ledger for uncovered exercises; an unreliable
  check is downgraded to documented manual testing or deleted — never left flaky.
- **DP-05a** Solve/validate placeholders in content ⇒ `runtime-automation/module-NN/`
  playbooks on the SAME branch (the autoshift dead-button failure).
- **DP-05c** QA plays calling `kubernetes.core.k8s` + raw `oc` set identity three ways
  (module_defaults + KUBECONFIG + K8S_AUTH_KUBECONFIG).

---

## 13. Build Safety & Local Extensions (REQUIRED)

- **DP-21** The RHDP content image ships only `@antora/cli` + `@antora/site-generator`.
  npm Antora extensions are FATAL (empty site, every page 404s). All extensions are
  local `content/lib/*.js`; the constraint is documented as a comment in `site.yml`
  at the point of violation.
- **DP-21** Local preview = `podman-compose.yaml` running the platform image
  (same extension semantics as production). Guided/nookbag patterns cannot be
  previewed this way — document the limit where relevant.
- **DP-20** `content/antora.yml` is the single attribute inventory with canonical
  placeholders (`%GUID%`, `%USER%`, `%PASSWORD%`, `cluster.example.com`). Versions are
  never hardcoded in content. Runtime-injected `{openshift_*}` attributes are
  showroom userdata — whitelist them in review, never "fix" them.
- **DP-18** `role="execute"` only where running in the embedded terminal is correct;
  anything for the learner's laptop/external console stays plain `[source,bash]` with
  a sentence saying why there is no run button.

---

## 14. Multi-Variant & Frame Honesty (REQUIRED)

- **DP-19** One repo → many variants: `ifeval` gates FRAGMENTS (manifest name, patch
  step), never whole exercise sets (duplication = double maintenance + drift).
  Divergence >~half a page ⇒ separate per-version module files. Manifest twins
  (`-34`/`-35`) keep both versions buildable from main; environment differences are
  directory choices (`clusterip/` vs `loadbalancer/`).
- **DP-22** `publishing-house/worklog.yaml` is the drift journal: append dated entries
  for known divergences; read it FIRST on existing repos. Empty `decisions/`/
  `reviews/` scaffolds are not decision records.
- **DP-23** The surviving decision mechanism is the WHY-comment at the point of use.
- **DP-24** Update `spec.yaml development.*` + `spec.modules[].status` at phase end
  (template vocabulary `not_started` → `complete`; scaffolded-dir ≠ authored;
  e2e/healthCheck `complete` only after a live-cluster run).
- **DP-25** Spec must not lie: before publish, every manifest path referenced in
  `spec/modules/*.md` exists, module list matches shipped pages, frame artifacts
  agree. Outlines are published as TechDocs — stale outlines are confidently wrong
  in public.

---

**Maintained by:** ph-rhaibu OODA pipeline
