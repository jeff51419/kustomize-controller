# Kustomization

The `Kustomization` API defines a pipeline for fetching, decrypting, building, validating and applying Kubernetes manifests.

## Specification

A **Kustomization** object defines the source of Kubernetes manifests by referencing an object
managed by [source-controller](https://github.com/fluxcd/source-controller),
the path to the kustomization file within that source,
and the interval at which the kustomize build output is applied on the cluster.

```go
type KustomizationSpec struct {
	// DependsOn may contain a dependency.CrossNamespaceDependencyReference slice
	// with references to Kustomization resources that must be ready before this
	// Kustomization can be reconciled.
	// +optional
	DependsOn []dependency.CrossNamespaceDependencyReference `json:"dependsOn,omitempty"`

	// Decrypt Kubernetes secrets before applying them on the cluster.
	// +optional
	Decryption *Decryption `json:"decryption,omitempty"`

	// The interval at which to apply the kustomization.
	// +required
	Interval metav1.Duration `json:"interval"`

	// The KubeConfig for reconciling the Kustomization on a remote cluster.
	// +optional
	KubeConfig *KubeConfig `json:"kubeConfig,omitempty"`

	// Path to the directory containing the kustomization file.
	// +kubebuilder:validation:Pattern="^\\./"
	// +required
	Path string `json:"path"`

	// Enables garbage collection.
	// +required
	Prune bool `json:"prune"`

	// A list of resources to be included in the health assessment.
	// +optional
	HealthChecks []CrossNamespaceObjectReference `json:"healthChecks,omitempty"`

	// The Kubernetes service account used for applying the kustomization.
	// +optional
	ServiceAccount *ServiceAccount `json:"serviceAccount,omitempty"`

	// Reference of the source where the kustomization file is.
	// +required
	SourceRef CrossNamespaceSourceReference `json:"sourceRef"`

	// This flag tells the controller to suspend subsequent kustomize executions,
	// it does not apply to already started executions. Defaults to false.
	// +optional
	Suspend bool `json:"suspend,omitempty"`

	// TargetNamespace sets or overrides the namespace in the
	// kustomization.yaml file.
	// +optional
	TargetNamespace string `json:"targetNamespace,omitempty"`

	// Timeout for validation, apply and health checking operations.
	// Defaults to 'Interval' duration.
	// +optional
	Timeout *metav1.Duration `json:"timeout,omitempty"`

	// Validate the Kubernetes objects before applying them on the cluster.
	// The validation strategy can be 'client' (local dry-run) or 'server' (APIServer dry-run).
	// +kubebuilder:validation:Enum=client;server
	// +optional
	Validation string `json:"validation,omitempty"`
}
```

The decryption section defines how decryption is handled for Kubernetes manifests:

```go
type Decryption struct {
	// Provider is the name of the decryption engine.
	// +kubebuilder:validation:Enum=sops
	// +required
	Provider string `json:"provider"`

	// The secret name containing the private OpenPGP keys used for decryption.
	// +optional
	SecretRef *corev1.LocalObjectReference `json:"secretRef,omitempty"`
}
```

KubeConfig references a Kubernetes Secret for applying to another cluster.
This can be used with Cluster API:

```go
type KubeConfig struct {
	// SecretRef holds the name to a secret that contains a 'value' key with
	// the kubeconfig file as the value. It must be in the same namespace as
	// the Kustomization.
	// It is recommended that the kubeconfig is self-contained, and the secret
	// is regularly updated if credentials such as a cloud-access-token expire.
	// Cloud specific `cmd-path` auth helpers will not function without adding
	// binaries and credentials to the Pod that is responsible for reconciling
	// the Kustomization.
	// +required
	SecretRef corev1.LocalObjectReference `json:"secretRef,omitempty"`
}
```

The status sub-resource records the result of the last reconciliation:

```go
type KustomizationStatus struct {
	// ObservedGeneration is the last reconciled generation.
	// +optional
	ObservedGeneration int64 `json:"observedGeneration,omitempty"`

	// +optional
	Conditions []Condition `json:"conditions,omitempty"`

	// The last successfully applied revision.
	// The revision format for Git sources is <branch|tag>/<commit-sha>.
	// +optional
	LastAppliedRevision string `json:"lastAppliedRevision,omitempty"`

	// LastAttemptedRevision is the revision of the last reconciliation attempt.
	// +optional
	LastAttemptedRevision string `json:"lastAttemptedRevision,omitempty"`

	// LastHandledReconcileAt is the last manual reconciliation request (by
	// annotating the Kustomization) handled by the reconciler.
	// +optional
	LastHandledReconcileAt string `json:"lastHandledReconcileAt,omitempty"`

	// The last successfully applied revision metadata.
	// +optional
	Snapshot *Snapshot `json:"snapshot"`
}
```

Status condition types:

```go
const (
	// ReadyCondition is the name of the condition that
	// records the readiness status of a Kustomization.
	ReadyCondition string = "Ready"
)
```

Status condition reasons:

```go
const (
	// ReconciliationSucceededReason represents the fact that the
	// reconciliation of the Kustomization has succeeded.
	ReconciliationSucceededReason string = "ReconciliationSucceeded"

	// ReconciliationFailedReason represents the fact that the
	// reconciliation of the Kustomization has failed.
	ReconciliationFailedReason string = "ReconciliationFailed"

	// ProgressingReason represents the fact that the
	// reconciliation of the Kustomization is underway.
	ProgressingReason string = "Progressing"

	// SuspendedReason represents the fact that the
	// reconciliation of the Kustomization has been suspended.
	SuspendedReason string = "Suspended"

	// DependencyNotReady represents the fact that
	// one of the dependencies of the Kustomization is not ready.
	DependencyNotReadyReason string = "DependencyNotReady"

	// PruneFailedReason represents the fact that the
	// pruning of the Kustomization failed.
	PruneFailedReason string = "PruneFailed"

	// ArtifactFailedReason represents the fact that the
	// artifact download of the kustomization failed.
	ArtifactFailedReason string = "ArtifactFailed"

	// BuildFailedReason represents the fact that the
	// kustomize build of the Kustomization failed.
	BuildFailedReason string = "BuildFailed"

	// HealthCheckFailedReason represents the fact that
	// one of the health checks of the Kustomization failed.
	HealthCheckFailedReason string = "HealthCheckFailed"

	// ValidationFailedReason represents the fact that the
	// validation of the Kustomization manifests has failed.
	ValidationFailedReason string = "ValidationFailed"
)
```

## Source reference

The kustomization `spec.sourceRef` is a reference to an object managed by
[source-controller](https://github.com/fluxcd/source-controller). When the source
[revision](https://github.com/fluxcd/source-controller/blob/master/docs/spec/v1beta1/common.md#source-status)
changes, it generates a Kubernetes event that triggers a kustomize build and apply.

Source supported types:

* [GitRepository](https://github.com/fluxcd/source-controller/blob/master/docs/spec/v1beta1/gitrepositories.md)
* [Bucket](https://github.com/fluxcd/source-controller/blob/master/docs/spec/v1beta1/buckets.md)

> **Note** that the source should contain the kustomization.yaml and all the
> Kubernetes manifests and configuration files referenced in the kustomization.yaml.
> If your Git repository or S3 bucket contains only plain manifests,
> then a kustomization.yaml will be automatically generated.

## Generate kustomization.yaml

If your repository contains plain Kubernetes manifests, the `kustomization.yaml`
file is automatically generated for all the Kubernetes manifests
in the `spec.path` and sub-directories.

If the `spec.prune` is enable, the controller generates a label transformer to enable
[garbage collection](#garbage-collection).

## Reconciliation

The kustomization `spec.interval` tells the controller at which interval to fetch the
Kubernetes manifest for the source, build the kustomization and apply it on the cluster.
The interval time units are `s`, `m` and `h` e.g. `interval: 5m`, the minimum value should be over 60 seconds.

The kustomization execution can be suspended by setting `spec.suspend` to `true`.

The controller can be told to reconcile the kustomization outside of the specified interval
by annotating the kustomization object with:

```go
const (
	// ReconcileAtAnnotation is the annotation used for triggering a
	// reconciliation outside of the defined schedule.
	ReconcileAtAnnotation string = "reconcile.fluxcd.io/requestedAt"
)
```

On-demand execution example:

```bash
kubectl annotate --overwrite kustomization/podinfo reconcile.fluxcd.io/requestedAt="$(date +%s)"
```

List all Kubernetes objects reconciled from a Kustomization:

```sh
kubectl get all --all-namespaces \
-l=kustomize.toolkit.fluxcd.io/name="<Kustomization name>" \
-l=kustomize.toolkit.fluxcd.io/namespace="<Kustomization namespace>"
```

## Garbage collection

To enable garbage collection, set `spec.prune` to `true`.

Garbage collection means that the Kubernetes objects that were previously applied on the cluster
but are missing from the current source revision, are removed from cluster automatically.
Garbage collection is also performed when a Kustomization object is deleted,
triggering a removal of all Kubernetes objects previously applied on the cluster.

To keep track of the Kubernetes objects reconciled from a Kustomization, the following labels 
are injected into the manifests:

```yaml
labels:
  kustomize.toolkit.fluxcd.io/name: "<Kustomization name>"
  kustomize.toolkit.fluxcd.io/namespace: "<Kustomization namespace>"
  kustomize.toolkit.fluxcd.io/checksum: "<manifests checksum>"
```

The checksum label value is updated if the content of `spec.path` changes.
When pruning is disabled, the checksum label is omitted. 

## Health assessment

A kustomization can contain a series of health checks used to determine the
[rollout status](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#deployment-status)
of the deployed workloads and the ready status of custom resources.

A health check entry can reference one of the following types:

* Kubernetes builtin kinds: Deployment, DaemonSet, StatefulSet, PersistentVolumeClaim, Pod, PodDisruptionBudget, Job, CronJob, Service, Secret, ConfigMap, CustomResourceDefinition
* Toolkit kinds: HelmRelease, HelmRepository, GitRepository, etc
* Custom resources that are compatible with [kstatus](https://github.com/kubernetes-sigs/cli-utils/tree/master/pkg/kstatus)

Assuming the kustomization source contains a Kubernetes Deployment named `backend`,
a health check can be defined as follows:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: backend
  namespace: default
spec:
  interval: 5m
  path: "./webapp/backend/"
  prune: true
  sourceRef:
    kind: GitRepository
    name: webapp
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: backend
      namespace: dev
  timeout: 2m
```

After applying the kustomize build output, the controller verifies if the rollout completed successfully.
If the deployment was successful, the kustomization ready condition is marked as `true`,
if the rollout failed, or if it takes more than the specified timeout to complete, then the
kustomization ready condition is set to `false`. If the deployment becomes healthy on the next
execution, then the kustomization is marked as ready.

When a Kustomization contains HelmRelease objects, instead of checking the underling Deployments, you can
define a health check that waits for the HelmReleases to be reconciled with:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: webapp
  namespace: default
spec:
  interval: 15m
  path: "./releases/"
  prune: true
  sourceRef:
    kind: GitRepository
    name: webapp
  healthChecks:
    - apiVersion: helm.toolkit.fluxcd.io/v1beta1
      kind: HelmRelease
      name: frontend
      namespace: dev
    - apiVersion: helm.toolkit.fluxcd.io/v1beta1
      kind: HelmRelease
      name: backend
      namespace: dev
  timeout: 5m
```

If all the HelmRelease objects are successfully installed or upgraded, then the Kustomization will be marked as ready.

## Kustomization dependencies

When applying a kustomization, you may need to make sure other resources exist before the
workloads defined in your kustomization are deployed.
For example, a namespace must exist before applying resources to it.

With `spec.dependsOn` you can specify that the execution of a kustomization follows another.
When you add `dependsOn` entries to a kustomization, that kustomization is applied
only after all of its dependencies are ready. The readiness state of a kustomization is determined by
its last apply status condition.

Assuming two kustomizations:
* `common` - contains a namespace and service accounts definitions
* `backend` - contains the workloads to be deployed in that namespace

You can instruct the controller to apply the `common` kustomization before `backend`:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: common
  namespace: default
spec:
  interval: 5m
  path: "./webapp/common/"
  prune: true
  sourceRef:
    kind: GitRepository
    name: webapp
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: backend
  namespace: default
spec:
  dependsOn:
    - name: common
  interval: 5m
  path: "./webapp/backend/"
  prune: true
  sourceRef:
    kind: GitRepository
    name: webapp
```

When combined with health assessment, a kustomization will run after all its dependencies health checks are passing.
For example, a service mesh proxy injector should be running before deploying applications inside the mesh.

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: istio
  namespace: istio-system
spec:
  interval: 5m
  path: "./profiles/default/"
  sourceRef:
    kind: GitRepository
    name: istio
  healthChecks:
    - kind: Deployment
      name: istiod
      namespace: istio-system
  timeout: 2m
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: backend
  namespace: default
spec:
  dependsOn:
    - name: common
    - name: istio
      namespace: istio-system
  interval: 5m
  path: "./webapp/backend/"
  prune: true
  sourceRef:
    kind: GitRepository
    name: webapp
```

> **Note** that circular dependencies between kustomizations must be avoided, otherwise the
> interdependent kustomizations will never be applied on the cluster.

## Role-based access control

By default, a kustomization apply runs under the cluster admin account and can create, modify, delete
cluster level objects (namespaces, CRDs, etc) and namespeced objects (deployments, ingresses, etc).
For certain kustomizations a cluster admin may wish to control what types of Kubernetes objects can
be reconciled and under which namespaces.
To restrict a kustomization, one can assign a service account under which the reconciliation is performed.

Assuming you want to restrict a group of kustomizations to a single namespace, you can create an account
with a role binding that grants access only to that namespace:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: webapp
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: webapp-reconciler
  namespace: webapp
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: webapp-reconciler
  namespace: webapp
rules:
  - apiGroups: ['*']
    resources: ['*']
    verbs: ['*']
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: webapp-reconciler
  namespace: webapp
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: webapp-reconciler
subjects:
- kind: ServiceAccount
  name: webapp-reconciler
  namespace: webapp
```

> **Note** that the namespace, RBAC and service account manifests should be
> placed in a Git source and applied with a kustomization. The kustomizations that
> are running under that service account should depend-on the one that contains the account.

Create a kustomization that prevents altering the cluster state outside of the `webapp` namespace:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: backend
  namespace: webapp
spec:
  dependsOn:
    - name: common
  serviceAccount:
    name: webapp-reconciler
    namespace: webapp
  interval: 5m
  path: "./webapp/backend/"
  prune: true
  sourceRef:
    kind: GitRepository
    name: webapp
```

When the controller reconciles the `frontend-webapp` kustomization, it will impersonate the `webapp-reconciler`
account. If the kustomization contains cluster level objects like CRDs or objects belonging to a different
namespace, the reconciliation will fail since the account it runs under has no permissions to alter objects
outside of the `webapp` namespace.

## Remote Clusters / Cluster-API

If the `kubeConfig` field is set, objects will be applied, health-checked, pruned, and deleted for the default
cluster specified in that KubeConfig instead of using the in-cluster ServiceAccount.

The secret defined in the `kubeConfig.SecretRef` must exist in the same namespace as the Kustomization.
On every reconciliation, the KubeConfig bytes will be loaded from the `values` key of the secret's data, and
the secret can thus be regularly updated if cluster-access-tokens have to rotate due to expiration.

This composes well with Cluster API bootstrap providers such as CAPBK (kubeadm) as well as the CAPA (AWS) EKS
integration.

To reconcile a Kustomization to a CAPI controlled cluster, put the `Kustomization` in the same namespace as your
`Cluster` object, and set the `kubeConfig.secretRef.name` to `<cluster-name>-kubeconfig`:

```yaml
apiVersion: cluster.x-k8s.io/v1alpha3
kind: Cluster
metadata:
  name: stage  # the kubeconfig Secret will contain the Cluster name
  namespace: capi-stage
spec:
  clusterNetwork:
    pods:
      cidrBlocks:
      - 10.100.0.0/16
    serviceDomain: stage-cluster.local
    services:
      cidrBlocks:
      - 10.200.0.0/12
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1alpha3
    kind: KubeadmControlPlane
    name: stage-control-plane
    namespace: capi-stage
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1alpha3
    kind: DockerCluster
    name: stage
    namespace: capi-stage
---
# ... unrelated Cluster API objects omitted for brevity ...
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: cluster-addons
  namespace: capi-stage
spec:
  interval: 5m
  path: "./config/addons/"
  prune: true
  sourceRef:
    kind: GitRepository
    name: cluster-addons
  kubeConfig:
    secretRef:
      name: stage-kubeconfig  # Cluster API creates this for the matching Cluster
```

The Cluster and Kustomization can be created at the same time.
The Kustomization will eventually reconcile once the cluster is available.

If you wish to target clusters created by other means than CAPI, you can create a ServiceAccount
on the remote cluster, generate a KubeConfig for that account, and then create a secret on the
cluster where kustomize-controller is running e.g.:

```sh
kubectl create secret generic prod-kubeconfig \
    --from-file=value=./kubeconfig
```

> **Note** that the KubeConfig should be self-contained and not rely on binaries, environment,
> or credential files from the kustomize-controller Pod.
> This matches the constraints of KubeConfigs from current Cluster API providers.
> KubeConfigs with `cmd-path` in them likely won't work without a custom,
> per-provider installation of kustomize-controller.

## Secrets decryption

In order to store secrets safely in a public or private Git repository,
you can use [Mozilla SOPS](https://github.com/mozilla/sops)
and encrypt your Kubernetes Secrets data with OpenPGP keys.

Generate a GPG key **without passphrase** using [gnupg](https://www.gnupg.org/)
then use sops to encrypt a Kubernetes secret:

```sh
sops --pgp=FBC7B9E2A4F9289AC0C1D4843D16CEE4A27381B4 \
--encrypt --encrypted-regex '^(data|stringData)$' --in-place my-secret.yaml
```

Commit and push the encrypted file to Git.

> **Note** that you should encrypt only the `data` section, encrypting the Kubernetes secret
> metadata, kind or apiVersion is not supported by kustomize-controller.

Create a secret in the `default` namespace with the OpenPGP private key:

```sh
gpg --export-secret-keys --armor FBC7B9E2A4F9289AC0C1D4843D16CEE4A27381B4 |
kubectl -n default create secret generic sops-gpg \
--from-file=sops.asc=/dev/stdin
```

Configure decryption by referring the private key secret:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: my-secrets
  namespace: default
spec:
  interval: 5m
  path: "./"
  sourceRef:
    kind: GitRepository
    name: my-secrets
  decryption:
    provider: sops
    secretRef:
      name: sops-pgp
```

## Status

When the controller completes a kustomization apply, reports the result in the `status` sub-resource.

A successful reconciliation sets the ready condition to `true` and updates the revision field:

```yaml
status:
  conditions:
  - lastTransitionTime: "2020-09-17T19:28:48Z"
    message: "Applied revision: master/a1afe267b54f38b46b487f6e938a6fd508278c07"
    reason: ReconciliationSucceeded
    status: "True"
    type: Ready
  lastAppliedRevision: master/a1afe267b54f38b46b487f6e938a6fd508278c07
  lastAttemptedRevision: master/a1afe267b54f38b46b487f6e938a6fd508278c07
```

You can wait for the kustomize controller to complete a reconciliation with:

```bash
kubectl wait kustomization/backend --for=condition=ready
```

The controller logs the Kubernetes objects:

```json
{
  "level": "info",
  "ts": "2020-09-17T07:27:11.921Z",
  "logger": "controllers.Kustomization",
  "msg": "Kustomization applied in 1.436096591s",
  "kustomization": "default/backend",
  "output": {
    "service/backend": "created",
    "deployment.apps/backend": "created",
    "horizontalpodautoscaler.autoscaling/backend": "created"
  }
}
```

A failed reconciliation sets the ready condition to `false`:

```yaml
status:
  conditions:
  - lastTransitionTime: "2020-09-17T07:26:48Z"
    message: "The Service 'backend' is invalid: spec.type: Unsupported value: 'Ingress'"
    reason: ValidationFailed
    status: "False"
    type: Ready
  lastAppliedRevision: master/a1afe267b54f38b46b487f6e938a6fd508278c07
  lastAttemptedRevision: master/7c500d302e38e7e4a3f327343a8a5c21acaaeb87
```

> **Note** that the last applied revision is updated only on a successful reconciliation.

When a reconciliation fails, the controller logs the error and issues a Kubernetes event:

```json
{
  "level": "error",
  "ts": "2020-09-17T07:27:11.921Z",
  "logger": "controllers.Kustomization",
  "kustomization": "default/backend",
  "error": "The Service 'backend' is invalid: spec.type: Unsupported value: 'Ingress'"
}
```
