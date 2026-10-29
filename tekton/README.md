# Tekton Pipeline for Manifest Hydration and OCI Artifact Publishing

A set of Tekton Tasks and Pipelines that hydrate dry Kubernetes manifests (via Helm or Kustomize), apply optional post-render JSONPath patches, and push the rendered output as OCI artifacts to a container registry. The resulting artifacts are directly consumable by Argo CD using its native `oci://` source support.

This project is inspired by [Kokumi](https://github.com/kokumi-dev/kokumi), which provides a Kubernetes controller-based approach to the same workflow. This implementation replaces the controller with Tekton CI pipelines, making it suitable for teams that already have Tekton installed and prefer a CI-driven approach over a custom operator.

## Architecture

### Git Source Pipeline

```
Git Repo (dry manifests)
      |
      v
[git-clone]              Clone repo, output commit SHA
      |
      v
[check-drift]            Compute config hash, check if artifact exists
      |                  (skip downstream if unchanged)
      v
[hydrate-manifests]      helm template / kustomize build
      |
      v
[patch-manifests]        Apply JSONPath patches (optional)
      |                  + YAML normalization for deterministic hashing
      v
[oci-push]               tar+gzip -> oras push -> registry
      |                  + content-addressable tag
      |                  + parent digest chain
      |                  + renderer versioning
      |
      +------+------+
      |             |
      v             v
[cosign-sign]   [argocd-promote]
(optional)      (optional, with sync policy)
```

### OCI-to-OCI Pipeline

```
OCI Registry (source)    registry.example.com/charts/app:v1.0
      |
      v
[oci-pull]               Pull and extract source artifact
      |
      v
[hydrate-manifests]      helm template / kustomize build
      |
      v
[patch-manifests]        Apply JSONPath patches (optional)
      |                  + YAML normalization
      v
[oci-push]               Push hydrated artifact to destination
      |
      +------+------+
      |             |
      v             v
[cosign-sign]   [argocd-promote]
(optional)      (optional, with sync policy)
```

## Directory Structure

```
tekton/
├── README.md
├── kustomization.yaml                  # Top-level Kustomize base (includes tasks/ + pipeline/)
├── tasks/
│   ├── kustomization.yaml              # Kustomize base for all tasks
│   ├── task-hydrate.yaml               # Renders manifests using Helm or Kustomize
│   ├── task-patch.yaml                 # Applies JSONPath patches + YAML normalization
│   ├── task-oci-pull.yaml              # Pulls an OCI artifact as pipeline source
│   ├── task-oci-push.yaml              # Packages and pushes manifests as OCI artifact
│   ├── task-check-drift.yaml           # Checks if config hash already exists (drift detection)
│   ├── task-sign.yaml                  # Signs the artifact with cosign
│   └── task-promote.yaml               # Updates Argo CD Application targetRevision + sync policy
├── pipeline/
│   ├── kustomization.yaml              # Kustomize base for all pipelines
│   ├── pipeline-hydrate-and-push.yaml  # Git source -> full pipeline
│   └── pipeline-oci-hydrate-and-push.yaml  # OCI source -> full pipeline
└── examples/
    ├── pipelinerun-helm-example.yaml       # Helm with patches, signing, promotion, sync policy
    ├── pipelinerun-kustomize-example.yaml  # Kustomize (minimal)
    ├── pipelinerun-oci-source-example.yaml # OCI-to-OCI with signing and promotion
    ├── argocd-application.yaml             # Argo CD Application consuming the artifact
    └── registry-credentials.yaml           # ServiceAccount, Secrets, RBAC setup
```

## Tasks

### `hydrate-manifests`

Renders dry manifests using Helm or Kustomize.

| Parameter | Default | Description |
|-----------|---------|-------------|
| `HYDRATION_METHOD` | *(required)* | `helm` or `kustomize` |
| `HELM_CHART_PATH` | `.` | Path to Helm chart within the repo |
| `HELM_RELEASE_NAME` | `release` | Helm release name |
| `HELM_NAMESPACE` | `default` | Target namespace for `helm template` |
| `HELM_VALUES_FILE` | `""` | Path to a values file |
| `HELM_VALUES` | `""` | Inline YAML values |
| `HELM_INCLUDE_CRDS` | `"true"` | Include CRDs in Helm output (`"true"` or `"false"`) |
| `HELM_SKIP_TESTS` | `"true"` | Skip Helm test hooks in output (`"true"` or `"false"`) |
| `KUSTOMIZE_PATH` | `.` | Path to kustomization directory |
| `OUTPUT_DIR` | `_rendered` | Output subdirectory for rendered manifests |

**Image:** `alpine/k8s:1.30.2` (includes helm, kustomize, kubectl)

**Results:** `MANIFEST_SHA`

### `patch-manifests`

Applies JSONPath-based patches to rendered manifests. Compatible with the [Kokumi patch format](https://github.com/kokumi-dev/kokumi). Includes optional YAML normalization for deterministic content hashing.

| Parameter | Default | Description |
|-----------|---------|-------------|
| `PATCHES` | `[]` | JSON array of patches (see format below) |
| `RENDERED_DIR` | `_rendered` | Directory containing manifests to patch |
| `NORMALIZE` | `"true"` | Normalize YAML formatting for deterministic SHA output |

**Patch format:**

```json
[
  {
    "target": { "kind": "Deployment", "name": "nginx", "namespace": "default" },
    "set": {
      ".spec.replicas": "3",
      ".spec.template.spec.containers[0].image": "nginx:1.25"
    }
  }
]
```

- `target.kind` and `target.name` are required; `target.namespace` is optional
- `set` maps YAML paths (with leading `.`) to values
- Numeric and boolean values are auto-detected; everything else is treated as a string
- Pass `"[]"` or `""` to skip patching (no-op passthrough)
- When `NORMALIZE` is `"true"`, YAML is normalized via `yq eval-all '.' -P` even when no patches are applied, ensuring consistent content hashes regardless of renderer whitespace differences

**Image:** `mikefarah/yq:4`

**Results:** `PATCH_COUNT`, `MANIFEST_SHA`

### `oci-pull`

Pulls an OCI artifact from a container registry and extracts it into a workspace.

| Parameter | Default | Description |
|-----------|---------|-------------|
| `SOURCE_IMAGE_REF` | *(required)* | OCI image reference to pull |

**Image:** `ghcr.io/oras-project/oras:v1.2.0`

**Results:** `digest`, `source-ref`

### `oci-push`

Packages rendered manifests as a single-layer `tar+gzip` OCI artifact and pushes it. Produces artifacts compatible with Argo CD's OCI source consumption.

| Parameter | Default | Description |
|-----------|---------|-------------|
| `IMAGE_REF` | *(required)* | Full OCI image reference with tag |
| `MEDIA_TYPE` | `application/vnd.oci.image.layer.v1.tar+gzip` | OCI layer media type |
| `RENDERED_DIR` | `_rendered` | Directory containing manifests |
| `EXTRA_TAGS` | `""` | Comma-separated additional tags (e.g., `latest,stable`) |
| `ANNOTATION_SOURCE_REPO` | `""` | Source repo URL annotation |
| `ANNOTATION_SOURCE_REVISION` | `""` | Source commit SHA annotation |
| `ANNOTATION_MANIFEST_SHA` | `""` | Manifest content SHA annotation |
| `ANNOTATION_PARENT_DIGEST` | `""` | Previous artifact digest (revision chain) |
| `ANNOTATION_COMMIT_MESSAGE` | `""` | Human-readable description |
| `ANNOTATION_RENDERER_IMAGE` | `""` | Renderer container image (e.g., `alpine/k8s:1.30.2`) |
| `ANNOTATION_RENDERER_VERSION` | `""` | Renderer tool version (e.g., `helm` or `kustomize`) |
| `ANNOTATION_RENDERER_DIGEST` | `""` | Renderer image digest (e.g., `sha256:abc123...`) |
| `ANNOTATION_RENDERER_TYPE` | `""` | Formal renderer type: `helm`, `kustomize`, or `manifest` |

**Image:** `ghcr.io/oras-project/oras:v1.2.0`

**Results:** `IMAGE_DIGEST`, `IMAGE_REF`, `CONTENT_TAG`

### `check-drift`

Computes a config hash from pipeline inputs and checks whether an OCI artifact with that hash already exists. If it exists, downstream tasks can be skipped via Tekton `when` expressions.

| Parameter | Default | Description |
|-----------|---------|-------------|
| `IMAGE_BASE` | *(required)* | Base OCI image path (without tag) |
| `HYDRATION_METHOD` | *(required)* | Hydration method |
| `HELM_CHART_PATH` | `.` | Path to Helm chart |
| `HELM_RELEASE_NAME` | `release` | Helm release name |
| `HELM_NAMESPACE` | `default` | Helm namespace |
| `HELM_VALUES_FILE` | `""` | Values file path |
| `HELM_VALUES` | `""` | Inline values |
| `KUSTOMIZE_PATH` | `.` | Kustomization path |
| `PATCHES` | `[]` | Patches JSON |
| `SOURCE_REVISION` | `""` | Source revision (git SHA or OCI digest) |

**Image:** `ghcr.io/oras-project/oras:v1.2.0`

**Results:** `CHANGED` (`true`/`false`), `CONFIG_HASH`, `EXISTING_TAG`

### `cosign-sign`

Signs an OCI artifact using cosign. Supports key-based and keyless (Sigstore Fulcio/Rekor) signing.

| Parameter | Default | Description |
|-----------|---------|-------------|
| `IMAGE_REF` | *(required)* | OCI image reference to sign |
| `IMAGE_DIGEST` | *(required)* | Digest of the artifact to sign |
| `SIGNING_MODE` | `keyless` | `key` or `keyless` |
| `COSIGN_KEY_FILE` | `cosign.key` | Private key filename (for key mode) |
| `REKOR_URL` | `https://rekor.sigstore.dev` | Rekor URL (for keyless mode) |
| `FULCIO_URL` | `https://fulcio.sigstore.dev` | Fulcio URL (for keyless mode) |

**Workspace (optional):** `cosign-key` -- contains the cosign private key file

**Image:** `gcr.io/projectsigstore/cosign:v2.4.1`

**Results:** `SIGNED` (`true`/`false`)

### `argocd-promote`

Updates an Argo CD Application's `targetRevision` to point to a newly pushed artifact. Can also configure the Application's sync policy for automated deployments.

| Parameter | Default | Description |
|-----------|---------|-------------|
| `APP_NAME` | *(required)* | Argo CD Application name |
| `APP_NAMESPACE` | `argocd` | Argo CD Application namespace |
| `TARGET_REVISION` | *(required)* | New targetRevision to set |
| `PROMOTE` | `true` | Set to `false` to skip (manual promotion) |
| `SYNC_AFTER_PROMOTE` | `false` | Trigger sync after updating |
| `SET_SYNC_POLICY` | `false` | Configure automated sync policy on the Application |
| `SYNC_POLICY_PRUNE` | `true` | Enable automated pruning (when `SET_SYNC_POLICY` is `true`) |
| `SYNC_POLICY_SELF_HEAL` | `true` | Enable self-healing (when `SET_SYNC_POLICY` is `true`) |

**Image:** `bitnami/kubectl:1.30`

**Results:** `PROMOTED` (`true`/`false`), `PREVIOUS_REVISION`

**RBAC:** The ServiceAccount needs `get` and `patch` permissions on `applications.argoproj.io`. See `registry-credentials.yaml` for the required ClusterRole.

## Pipelines

### `hydrate-and-push` (Git source)

Full pipeline: `git-clone` -> `check-drift` -> `hydrate` -> `patch` -> `push` -> `sign` + `promote`

- **Drift detection:** Set `skip-on-no-drift: "true"` to skip hydrate/patch/push when the config hash already exists in the registry
- **Signing:** Set `sign: "true"` to sign the artifact with cosign after push
- **Promotion:** Set `promote: "true"` to update the Argo CD Application's targetRevision after push
- **Sync policy:** Set `set-sync-policy: "true"` to configure automated sync on the Application
- **Signing and promotion run in parallel** after push completes

Uses the [Tekton Catalog git-clone task](https://github.com/tektoncd/catalog/tree/main/task/git-clone/0.9). Install it before using this pipeline:

```bash
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.9/git-clone.yaml
```

### `oci-hydrate-and-push` (OCI source)

Full pipeline: `oci-pull` -> `hydrate` -> `patch` -> `push` -> `sign` + `promote`

Same as above but pulls the source from an OCI registry instead of git.

## Quick Start

### 1. Install Tasks and Pipelines

```bash
# Install the Tekton Catalog git-clone task
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.9/git-clone.yaml

# Install custom tasks and pipelines (using Kustomize)
kubectl apply -k tekton/

# Or install individually
kubectl apply -f tekton/tasks/
kubectl apply -f tekton/pipeline/
```

### 2. Configure Credentials and RBAC

Edit `tekton/examples/registry-credentials.yaml` with your registry credentials, then apply:

```bash
kubectl apply -f tekton/examples/registry-credentials.yaml
```

This creates:
- Registry credentials Secret (for OCI push/pull)
- ServiceAccount with registry credentials attached
- ClusterRole + ClusterRoleBinding for Argo CD Application promotion
- Argo CD repository Secret (for OCI artifact pull)

### 3. Run a Pipeline

```bash
# Helm example with all features (drift detection, signing, promotion, sync policy)
kubectl create -f tekton/examples/pipelinerun-helm-example.yaml

# Kustomize example (minimal, no signing or promotion)
kubectl create -f tekton/examples/pipelinerun-kustomize-example.yaml

# OCI-to-OCI example with signing and promotion
kubectl create -f tekton/examples/pipelinerun-oci-source-example.yaml
```

### 4. Verify the OCI Artifact

```bash
# Check the artifact manifest (should show exactly one tar+gzip layer)
oras manifest fetch registry.example.com/manifests/my-app:v1.2.3 | jq .

# List all tags (user tag + content-addressable sha- tag + config cfg- tag)
oras repo tags registry.example.com/manifests/my-app

# Check annotations (source, revision, parent digest, renderer, commit message)
oras manifest fetch registry.example.com/manifests/my-app:v1.2.3 | jq .annotations

# Verify cosign signature
cosign verify registry.example.com/manifests/my-app:v1.2.3
```

### 5. Deploy with Argo CD

If auto-promotion is enabled, the pipeline updates the Application automatically. Otherwise:

```bash
# Register the OCI repo in Argo CD
argocd repo add oci://registry.example.com/manifests/my-app \
  --type oci --username <user> --password <pass>

# Create the Application
kubectl apply -f tekton/examples/argocd-application.yaml
```

## Argo CD OCI Compatibility

The OCI artifacts produced by this pipeline are designed to be consumed by Argo CD natively:

- **Single content layer:** Argo CD requires exactly one content layer (validated in `util/oci/client.go`)
- **Accepted media type:** `application/vnd.oci.image.layer.v1.tar+gzip` (default in Argo CD repo-server)
- **Plain YAML directory:** Argo CD extracts the layer and processes it as a directory source
- **ORAS v1.2.0:** Same version used by Argo CD's own E2E tests

Example Argo CD Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  source:
    repoURL: oci://registry.example.com/manifests/my-app
    targetRevision: "v1.2.3"    # or use the content tag: sha-<12chars>
    path: "."
  destination:
    server: https://kubernetes.default.svc
    namespace: production
```

## Packaging and Distribution

This pipeline can be distributed and consumed in several ways:

### OCI Bundles (Recommended)

Package tasks and pipelines as OCI artifacts using Tekton Bundles. This is the Tekton-endorsed best practice.

```bash
# Push individual tasks
tkn bundle push ghcr.io/your-org/tekton-hydrator/tasks/hydrate-manifests:0.2 \
  -f tekton/tasks/task-hydrate.yaml

tkn bundle push ghcr.io/your-org/tekton-hydrator/tasks/oci-push:0.4 \
  -f tekton/tasks/task-oci-push.yaml

# Push a pipeline
tkn bundle push ghcr.io/your-org/tekton-hydrator/pipelines/hydrate-and-push:0.4 \
  -f tekton/pipeline/pipeline-hydrate-and-push.yaml

# Sign the bundle (optional)
cosign sign ghcr.io/your-org/tekton-hydrator/tasks/hydrate-manifests:0.2
```

Consumers reference via the bundle resolver (no pre-installation needed):

```yaml
taskRef:
  resolver: bundles
  params:
    - name: bundle
      value: ghcr.io/your-org/tekton-hydrator/tasks/hydrate-manifests:0.2
    - name: name
      value: hydrate-manifests
    - name: kind
      value: Task
```

### Git Resolver

Reference tasks directly from this repository without any packaging:

```yaml
taskRef:
  resolver: git
  params:
    - name: url
      value: https://github.com/argoproj/argo-cd.git
    - name: revision
      value: v2.14.0
    - name: pathInRepo
      value: tekton/tasks/task-hydrate.yaml
```

### Kustomize

Install all tasks and pipelines using the included Kustomize bases:

```bash
# Install everything
kubectl apply -k https://github.com/argoproj/argo-cd.git/tekton/?ref=v2.14.0

# Install tasks only
kubectl apply -k https://github.com/argoproj/argo-cd.git/tekton/tasks/?ref=v2.14.0

# Overlay for customization
# In your kustomization.yaml:
resources:
  - https://github.com/argoproj/argo-cd.git/tekton/?ref=v2.14.0
```

### Raw kubectl apply

```bash
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.14.0/tekton/tasks/task-hydrate.yaml
```

### Comparison

| Approach | Versioning | Air-gap | Runtime Resolution | Signing |
|----------|-----------|---------|-------------------|---------|
| **OCI Bundles** | Tags + digests | Yes | Yes (bundle resolver) | cosign |
| **Git Resolver** | Tags/commits | No | Yes | No |
| **Kustomize** | Git refs | Partial | No (pre-install) | No |
| **kubectl apply** | URL path | No | No (pre-install) | No |

## Feature Comparison: Kokumi vs Tekton Pipeline

### Detailed Feature Matrix

| # | Kokumi Feature | Description | Status | Notes |
|---|---------------|-------------|--------|-------|
| 1 | **Helm rendering** | Render Helm charts with values, release name, namespace | Covered | `task-hydrate.yaml` with `HYDRATION_METHOD=helm` |
| 2 | **Helm CRD inclusion** | Toggle CRD inclusion in rendered output | Covered | `HELM_INCLUDE_CRDS` param (default `true`) |
| 3 | **Helm test hook exclusion** | Skip test hooks in rendered output | Covered | `HELM_SKIP_TESTS` param (default `true`) |
| 4 | **Kustomize rendering** | Render Kustomize overlays | Covered | `task-hydrate.yaml` with `HYDRATION_METHOD=kustomize` |
| 5 | **Post-render patches** | JSONPath-based mutations targeting resources by kind/name | Covered | `task-patch.yaml` using yq, Kokumi-compatible format |
| 6 | **YAML normalization** | Normalize YAML formatting for deterministic content hashing | Covered | `NORMALIZE` param in `task-patch.yaml` (default `true`) |
| 7 | **OCI source pull** | Pull source artifacts from OCI registries (OCI-to-OCI) | Covered | `task-oci-pull.yaml` + `pipeline-oci-hydrate-and-push.yaml` |
| 8 | **OCI artifact push** | Push hydrated manifests as single-layer OCI artifacts | Covered | `task-oci-push.yaml` with Argo CD-compatible format |
| 9 | **Content-addressable tags** | Immutable revision tags based on content hash | Covered | `sha-<12chars>` tag pushed alongside user tag |
| 10 | **Parent digest chaining** | Link artifacts in a revision chain for rollback | Covered | `dev.kokumi.parent.digest` OCI annotation |
| 11 | **Commit messages** | Human-readable audit trail on artifacts | Covered | `dev.kokumi.commit.message` OCI annotation |
| 12 | **Extra tags** | Apply additional tags (latest, stable) | Covered | `EXTRA_TAGS` param with `oras tag` |
| 13 | **Provenance annotations** | Source repo, commit SHA, manifest SHA, timestamp | Covered | Standard OCI annotations on every push |
| 14 | **Renderer versioning** | Track which renderer image and version produced the artifact | Covered | `dev.tekton.renderer.image` and `dev.tekton.renderer.version` annotations |
| 15 | **Renderer type annotation** | Formal render type (Helm/Kustomize/Manifest) | Covered | `dev.tekton.renderer.type` OCI annotation |
| 16 | **Renderer digest** | Digest of the renderer container image for exact reproducibility | Covered | `dev.tekton.renderer.digest` OCI annotation |
| 17 | **Drift detection** | Skip re-rendering when config hash hasn't changed | Covered | `task-check-drift.yaml` + `skip-on-no-drift` pipeline param with Tekton `when` expressions |
| 18 | **Artifact signing** | Cryptographic signature on artifacts | Covered | `task-sign.yaml` using cosign with key-based or keyless (Sigstore) signing. Exceeds Kokumi which only has a boolean flag |
| 19 | **Serving (activation)** | Point Argo CD at a specific immutable artifact as the "active" version | Covered | `task-promote.yaml` patches the Argo CD Application's `targetRevision` |
| 20 | **AutoDeploy** | Automatically activate new artifacts when ready | Covered | `promote: "true"` pipeline param triggers `argocd-promote` task after push |
| 21 | **Sync policy configuration** | Configure automated sync with prune and self-heal | Covered | `SET_SYNC_POLICY`, `SYNC_POLICY_PRUNE`, `SYNC_POLICY_SELF_HEAL` params in `task-promote.yaml` |
| 22 | **Menu templates** | Reusable delivery configurations with override policies | N/A | Kokumi uses a `Menu` CRD; not applicable to CI. Use parameterized Pipelines or Trigger Templates instead |
| 23 | **Edits (UI-driven patches)** | Post-patch modifications from UI, not inherited from templates | N/A | Functionally identical to patches. The distinction is a UI/CRD concern not applicable to CI |
| 24 | **Override policies** | Restrict which Helm values and patch targets consumers can customize | N/A | Kokumi enforces `All/Restricted/None` policies via Menu CRD. Use Tekton RBAC or an admission webhook instead |

### Tekton Extras (Features Not in Kokumi)

| Feature | Notes |
|---------|-------|
| **Kustomize rendering** | Kokumi only supports Helm and pre-rendered manifests |
| **Full cosign signing** | Kokumi only has a boolean `signed` flag with no actual signing implementation |
| **Git source pipeline** | Kokumi only supports OCI sources |
| **Values file support** | Kokumi only supports inline JSON values |
| **Extra tags** | Kokumi does not support `latest`/`stable` additional tags |
| **Sync-after-promote trigger** | Kokumi relies on ArgoCD auto-sync; Tekton can explicitly trigger refresh |
| **Drift detection via registry** | Kokumi checks CRD status; Tekton checks the OCI registry directly (stateless) |
| **Kustomize bases** | Distributed via Kustomize for easy installation with overlays |
| **OCI Bundle packaging** | Distributable as Tekton Bundles for runtime resolution |

### Key Architectural Differences

| Aspect | Kokumi | Tekton Pipeline |
|--------|--------|-----------------|
| **Execution model** | Kubernetes controller (continuous reconciliation) | CI pipeline (triggered per run) |
| **State management** | CRDs (Order, Preparation, Serving) | Stateless; state lives in OCI annotations and registry tags |
| **Rollback** | Re-point Serving to an older Preparation CRD | Re-deploy with an older content-addressable tag (`sha-<12chars>`) |
| **Drift detection** | Built-in via ConfigHash comparison | Config hash checked against registry tags (`cfg-<hash>`) |
| **Multi-tenancy** | Menu CRDs with override policies | Tekton RBAC and Pipeline parameterization |
| **Promotion** | Serving CRD with Automatic/Manual policy | `argocd-promote` task patches the Application directly |
| **Helm version** | Helm v4 SDK | Helm v3 CLI (via `alpine/k8s:1.30.2`) |
| **Preview/dry-run** | Built-in `PreviewOrder()` API | Run pipeline up to patch step and inspect workspace |

### Items Marked N/A

Items 22-24 (Menu templates, UI-driven edits, override policies) are fundamentally controller/CRD concepts that don't map to a CI pipeline architecture. They require persistent state, continuous reconciliation, and a custom API surface -- all of which are what Kokumi's operator provides. In a Tekton-based workflow, equivalent capabilities come from:

- **Menu templates** -> Parameterized Pipelines, Trigger Templates, or a pipeline-of-pipelines
- **UI-driven edits** -> The `patches` param already supports arbitrary mutations; a UI can construct the JSON
- **Override policies** -> Kubernetes RBAC on PipelineRun creation, OPA/Gatekeeper admission webhooks, or Tekton Pipelines-as-Code approval gates

### Known Differences

These are documented behavioral differences between Kokumi and this Tekton implementation that are worth noting:

| Difference | Impact | Mitigation |
|-----------|--------|------------|
| **Helm v4 vs v3** | Kokumi uses Helm v4 SDK; Tekton uses Helm v3 CLI. Rendering differences are possible for edge-case charts | Update the `alpine/k8s` image when a Helm v4-compatible version is available |
| **Config hash algorithm** | Kokumi YAML-serializes the spec; Tekton builds a pipe-delimited string. Different hashes for the same config | Both produce deterministic hashes. Only matters if switching between Kokumi and Tekton for the same artifacts |
| **Application lifecycle** | Kokumi creates/deletes the ArgoCD Application; Tekton only patches an existing one | Pre-create the Application manually or via a separate pipeline |
| **OCI source media type validation** | Kokumi validates that the source is a Helm chart by media type; Tekton does not | If the source is not a valid Helm chart, `helm template` will fail with an error |
| **Default destination** | Kokumi defaults to `oci://<in-cluster-registry>/<ns>/<name>`; Tekton requires explicit `image-ref` | Always specify `image-ref` in PipelineRun params |

## OCI Annotation Reference

The following annotations are set on pushed artifacts:

| Annotation | Source | Description |
|-----------|--------|-------------|
| `org.opencontainers.image.source` | `ANNOTATION_SOURCE_REPO` | Git repository or OCI source URL |
| `org.opencontainers.image.revision` | `ANNOTATION_SOURCE_REVISION` | Git commit SHA or OCI source digest |
| `org.opencontainers.image.created` | Auto-generated | ISO 8601 creation timestamp |
| `dev.tekton.manifests.sha256` | `ANNOTATION_MANIFEST_SHA` | SHA-256 of rendered manifest content |
| `dev.kokumi.parent.digest` | `ANNOTATION_PARENT_DIGEST` | Previous artifact digest (revision chain) |
| `dev.kokumi.commit.message` | `ANNOTATION_COMMIT_MESSAGE` | Human-readable description |
| `dev.tekton.renderer.image` | `ANNOTATION_RENDERER_IMAGE` | Container image used for rendering |
| `dev.tekton.renderer.version` | `ANNOTATION_RENDERER_VERSION` | Renderer tool name/version |
| `dev.tekton.renderer.digest` | `ANNOTATION_RENDERER_DIGEST` | Renderer image digest |
| `dev.tekton.renderer.type` | `ANNOTATION_RENDERER_TYPE` | Formal type: `helm`, `kustomize`, or `manifest` |
