# Infrastructure Patterns Reference

The workshop's automation lives inside the publishing-house repo (`~/git/ph-<slug>/`)
under `automation/`. The showroom is deployed by referencing the reusable
**`zt-showroom-deployer`** Helm chart with per-workshop values at
`automation/showroom-deployer/values-<slug>.yaml`, driven by `helm` targets in the
repo-root `Makefile`. The chart source lives at `~/git/zt-showroom-deployer/` and is
published to a Helm chart repo.

Labs that must provision cluster-side resources themselves add a GitOps layer under
`automation/gitops/` (charts per component, deployed via a bootstrap app-of-apps —
see `~/git/ph-operations-rhoai/automation/gitops/cluster/` for the worked pattern).
Labs that consume pre-provisioned clusters keep only the scaffolded
`automation/gitops/bootstrap-infra/` test-namespace chart.

## Showroom pod anatomy

The chart renders a single `showroom` Deployment (`Recreate`, 1 replica) plus supporting
objects. Knowing what the chart produces makes it clear what the values drive.

**Init containers**
- `cluster-setup` *(optional — `showroom.clusterSetup.enabled`)*: `oc patch` on the default
  IngressController to delete the `X-Frame-Options` and `Content-Security-Policy` response
  headers so the showroom can be iframed. Needs the cluster-setup RBAC below.
- `git-cloner`: clones `showroom.gitRepoUrl` @ `showroom.gitRepoRef` into a shared emptyDir.
- `antora-builder`: builds the Antora site from the cloned content into the shared volume.

**Runtime containers**
- `content` (:8000): serves the built showroom content.
- `nginx` (:8080): reverse proxy. `location /` → content (:8000); `location ^~ /wetty` →
  wetty (:8001) with websocket upgrade (`map $http_upgrade $connection_upgrade`).
- `wetty` (:8001): in-browser SSH terminal, args templated from `showroom.wetty.*`
  (`--base="/wetty/"`, `--port=8001`, `--allow-iframe=true`, `--ssh-host/-port/-user/-auth/-pass`).

**Supporting objects**
- `Namespace`, `ServiceAccount` (`showroom`).
- RBAC (see below): cluster-setup `ClusterRole`/`ClusterRoleBinding` + namespace `edit` `RoleBinding`.
- ConfigMaps: `showroom-proxy-config` (nginx.conf) and `showroom-userdata` (`user_data.yml`).
- `Service` (ClusterIP :8080 → 8080) and `Route` (edge/Redirect, haproxy timeout + tunnel 1h).

**Volumes**: `showroom-files` (emptyDir, shared repo/www), `user-data` (from
`showroom-userdata` CM), `nginx-config` (from `showroom-proxy-config` CM), `nginx-pid`
(emptyDir), `nginx-cache` (emptyDir).

## Chart value contract

`values.yaml` in `zt-showroom-deployer` (defaults are placeholders — override per workshop
in `values-<slug>.yaml`):

```yaml
deployer:
  domain: apps.cluster.example.com          # cluster apps wildcard domain
  apiUrl: https://api.cluster.example.com:6443  # cluster API URL
showroom:
  namespace: showroom                        # target namespace for this showroom
  gitRepoUrl: https://github.com/<org>/ph-<slug>  # this repo (content is at content/)
  gitRepoRef: main                           # branch/tag/sha to build
  antoraPlaybook: site.yml                   # Antora playbook path in the content repo
  ztBundle: https://github.com/rhpds/nookbag/releases/download/nookbag-v0.3.6/nookbag-v0.3.6.zip
  ztUiEnabled: "true"                        # enable the nookbag (zt) UI bundle
  guid: workshop                             # lab GUID surfaced in user_data.yml
  user: user1                                # lab user
  password: openshift                        # lab password
  projectName: demo                          # project name shown to the learner
  clusterSetup:
    enabled: true                            # run the ingress-header-strip init container
  rbac:
    namespaceEdit: true                      # bind ClusterRole/edit to the SA in this ns
  wetty:
    sshHost: ssh.example.com                 # web-terminal SSH target host
    sshPort: "22"                            # SSH port (often a nodePort)
    sshUser: lab-user                        # SSH user
    sshAuth: password                        # auth mode
    sshPass: changeme                        # SSH password (SENSITIVE — see note)
  images:
    oseCli: registry.redhat.io/openshift4/ose-cli:latest
    gitCloner: quay.io/rhpds/git-cloner:v1.1.5
    antora: quay.io/rhpds/antora:v1.3.0
    content: quay.io/rhpds/showroom-content:v1.4.2
    nginx: quay.io/rhpds/nginx:1.25
    wetty: quay.io/rhpds/wetty:v2.7.6
```

Per-key notes:
- `deployer.domain` / `deployer.apiUrl` feed the `showroom-userdata` ConfigMap
  (`login_command`, console/api/ingress/dashboard URLs).
- `showroom.gitRepoUrl` / `gitRepoRef` drive the `git-cloner` init container — point at
  this publishing-house repo (`ph-<slug>`); the chart builds `site.yml` from it directly.
- `showroom.clusterSetup.enabled` toggles both the `cluster-setup` init container and the
  cluster-scoped RBAC that lets it patch the IngressController.
- `showroom.rbac.namespaceEdit` toggles the `edit-showroom-sa` RoleBinding (matches the live
  reference deployment).
- `showroom.wetty.sshPass` is **sensitive** — the chart renders it as a container arg to
  match the live reference. Hardening to a `Secret` is a future improvement, out of scope now.

### RBAC produced by the chart

- Cluster-setup (gated by `showroom.clusterSetup.enabled`): a `ClusterRole` +
  `ClusterRoleBinding` named `showroom-cluster-setup-<namespace>` (namespace-suffixed to
  avoid collisions across multiple showroom deployments) granting `get`/`patch` on
  `operator.openshift.io` `ingresscontrollers`.
- Namespace edit (gated by `showroom.rbac.namespaceEdit`): a `RoleBinding` `edit-showroom-sa`
  binding ClusterRole `edit` to the `showroom` ServiceAccount in the target namespace.

## Automation layout inside the publishing-house repo

```
~/git/ph-<slug>/
  Makefile                          # repo root: build targets + helm deployer targets
  automation/
    gitops/                         # cluster layer (optional — lab-provisioned resources)
      bootstrap-infra/              # scaffolded by scaffold.py (test-namespace chart)
      cluster/<component>/          # e.g. rhoai-install, kueue, maas-platform (ph-operations pattern)
    showroom-deployer/
      README.md
      values-<slug>.yaml            # per-workshop overrides for the showroom-deployer chart
```

### Makefile deployer targets (repo root)

```makefile
SLUG        ?= <slug>
NAMESPACE   ?= showroom-$(SLUG)
RELEASE     ?= showroom
VALUES      ?= automation/showroom-deployer/values-$(SLUG).yaml
# Published chart ref; override CHART=~/git/zt-showroom-deployer for local source.
CHART_REPO_NAME ?= eformat
CHART_REPO  ?= https://eformat.github.io/helm-charts
CHART       ?= $(CHART_REPO_NAME)/showroom-deployer

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

`make template CHART=~/git/zt-showroom-deployer` renders against the local chart source with
no cluster required — use it as the scaffold-time validation in workshop-do Step 9.

## Screenshot Manifest Integration

The `workshop-screenshot` skill can consume a `capture/screenshot-manifest.yaml` in
the showroom repo. The manifest follows the pattern from `openshift-workshop-builder`:

```yaml
workshop:
  slug: workbench-create
  title: RHOAI Workbench Creation
  product: rhai-3.3

shots:
  - name: login-page
    url: "{{ console_url }}"
    output: content/modules/ROOT/assets/images/01-keycloak-login.png
    caption: Keycloak login page
    wait_for:
      - "getByRole('heading', { name: /log in/i })"
    steps: []
```

When a manifest exists, `workshop-screenshot` uses it as the authoritative shot list
instead of scanning AsciiDoc `image::` references. The showroom `values.yaml`
`deployer.domain` and credential values are substituted into `{{ }}` placeholders.
