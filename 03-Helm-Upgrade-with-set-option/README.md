# Helm Upgrade with `--set`

## Step-01: Introduction

In this lab, we will deploy **Grafana** using Helm and then upgrade the existing Helm release by changing a chart value with the `--set` option.

The primary goal is to understand:

```text
Helm Chart
    ↓
Chart Values
    ↓
--set override
    ↓
Helm renders templates
    ↓
Kubernetes manifests
    ↓
Helm Release
    ↓
Kubernetes resources
    ↓
Grafana
    ↓
Web UI
```

We will use Grafana because it gives us a real Web UI where we can observe the application after making configuration changes.

### Helm commands covered

```text
helm repo
helm search repo
helm show
helm install
helm template
helm upgrade
helm list
helm history
helm status
helm get values
helm get manifest
helm rollback
helm uninstall
```

We will also learn how Helm works with:

* OCI registries
* Traditional HTTP Helm chart repositories
* `values.yaml`
* `--set`
* Release revisions
* Upgrade and rollback

---

# Step-02: Helm Chart Distribution

Helm charts can be distributed in more than one way.

Two important mechanisms are:

1. **OCI-based registries**
2. **Traditional HTTP Helm chart repositories**

---

## Step-02-01: OCI Registry

Helm supports storing and distributing charts through OCI-compliant registries.

OCI support became generally available in Helm 3.8.0 and is enabled by default. Helm supports commands such as `helm install`, `helm upgrade`, `helm show`, `helm template`, `helm pull`, and `helm push` with OCI chart references. ([helm.sh](https://docs.helm.sh/docs/v3/topics/registries/?utm_source=chatgpt.com))

An OCI chart reference looks like:

```text
oci://registry.example.com/path/chart
```

For example:

```bash
helm install <RELEASE-NAME> \
  oci://ghcr.io/grafana-community/helm-charts/grafana
```

The important difference is that we **do not need `helm repo add`** for an OCI registry.

Compare:

```text
Traditional repository:

helm repo add
      ↓
repository metadata
      ↓
helm search repo
      ↓
chart
```

with:

```text
OCI registry:

oci://...
      ↓
Helm accesses the chart directly
```

Helm's official documentation describes OCI registries as a supported mechanism for storing and sharing Helm charts. ([helm.sh](https://docs.helm.sh/docs/v3/topics/registries/?utm_source=chatgpt.com))

### Versioned OCI installation

It is good practice to specify the chart version when reproducibility matters:

```bash
helm install <RELEASE-NAME> \
  oci://ghcr.io/grafana-community/helm-charts/grafana \
  --version <CHART-VERSION>
```

An OCI chart version is represented by the OCI tag.

Helm requires OCI chart references to use the chart's name as the basename and the chart version as the tag. ([helm.sh](https://docs.helm.sh/docs/v3/topics/registries/?utm_source=chatgpt.com))

---

# Step-03: Traditional HTTP Helm Repository

The traditional Helm repository mechanism uses an HTTP server containing an `index.yaml` file and packaged charts.

Grafana's official documentation currently provides this installation method for the Grafana Helm chart. ([grafana.com](https://grafana.com/docs/grafana/latest/setup-grafana/installation/helm/?utm_source=chatgpt.com))

Add the Grafana repository:

```bash
helm repo add grafana-community \
  https://grafana-community.github.io/helm-charts
```

Update the local repository metadata:

```bash
helm repo update
```

Verify:

```bash
helm repo list
```

Search for the Grafana chart:

```bash
helm search repo grafana-community/grafana
```

The important distinction is:

```text
helm search repo
        ↓
Searches repositories registered with Helm
```

Whereas:

```bash
helm search hub grafana
```

searches Artifact Hub.

---

# Step-04: OCI vs HTTP Helm Repository

For this lab, understand the difference rather than memorizing two installation commands.

### OCI

```bash
helm install grafana \
  oci://ghcr.io/grafana-community/helm-charts/grafana
```

No `helm repo add` is required.

### HTTP chart repository

```bash
helm repo add grafana-community \
  https://grafana-community.github.io/helm-charts

helm repo update

helm install grafana \
  grafana-community/grafana
```

### Mental model

```text
OCI
│
└── Registry directly stores/distributes chart artifacts

HTTP Chart Repository
│
└── HTTP server
      ├── index.yaml
      └── packaged charts
```

Both are valid ways of obtaining Helm charts.

For this lab, we will primarily use **OCI**, because it gives us experience with the modern OCI-based distribution model.

---

# Step-05: Inspect the Grafana Chart

Before installing a chart, understand what you are installing.

For the OCI chart:

```bash
helm show chart \
  oci://ghcr.io/grafana-community/helm-charts/grafana
```

View the chart's default values:

```bash
helm show values \
  oci://ghcr.io/grafana-community/helm-charts/grafana
```

If you want a specific chart version:

```bash
helm show values \
  oci://ghcr.io/grafana-community/helm-charts/grafana \
  --version <CHART-VERSION>
```

### Why `helm show values` matters

The chart's `values.yaml` defines the configuration interface exposed by the chart.

For example, Grafana's chart exposes configuration for things such as:

```text
Image
Persistence
Service
Ingress
Resources
Admin credentials
Plugins
Grafana configuration
```

The exact values should always be checked against the chart version you are using.

Do not assume that a value exists merely because another chart uses the same name.

---

# Step-06: Understand Helm Values

A chart has default values.

Conceptually:

```text
Chart
 │
 └── values.yaml
       │
       ├── image
       ├── service
       ├── persistence
       └── ...
```

We can override those defaults using:

### Values file

```bash
-f my-values.yaml
```

or:

### Command-line value

```bash
--set key=value
```

For example:

```bash
--set image.tag=<TAG>
```

Conceptually:

```yaml
image:
  tag: <TAG>
```

The important point:

> `--set` does not modify the chart's `values.yaml`.

It supplies an override to Helm when Helm processes the chart.

---

# Step-07: Render the Chart Before Installing

Before installing anything, render the chart locally:

```bash
helm template grafana \
  oci://ghcr.io/grafana-community/helm-charts/grafana
```

Save the result:

```bash
helm template grafana \
  oci://ghcr.io/grafana-community/helm-charts/grafana \
  > rendered.yaml
```

Now inspect:

```bash
less rendered.yaml
```

This is one of the most important Helm learning exercises.

You are seeing the Kubernetes manifests generated by Helm.

The relationship is:

```text
Chart
  +
Values
  ↓
Helm template engine
  ↓
Rendered Kubernetes YAML
```

`helm template` renders locally without installing the release into the Kubernetes cluster.

---

# Step-08: Create a Namespace

Grafana's official documentation recommends installing Grafana into a dedicated namespace rather than relying on the default namespace. ([grafana.com](https://grafana.com/docs/grafana/latest/setup-grafana/installation/helm/?utm_source=chatgpt.com))

Create:

```bash
kubectl create namespace monitoring
```

Verify:

```bash
kubectl get namespace monitoring
```

---

# Step-09: Install Grafana from the OCI Registry

Install Grafana:

```bash
helm install grafana \
  oci://ghcr.io/grafana-community/helm-charts/grafana \
  --namespace monitoring \
  --create-namespace \
  --wait
```

Here:

```text
grafana
   ↑
Release name

oci://ghcr.io/grafana-community/helm-charts/grafana
   ↑
OCI chart reference

--namespace monitoring
   ↑
Kubernetes namespace
```

The release name is:

```text
grafana
```

The chart is:

```text
oci://ghcr.io/grafana-community/helm-charts/grafana
```

---

# Step-10: Verify the Helm Release

List releases:

```bash
helm list -n monitoring
```

You should see something similar to:

```text
NAME      NAMESPACE    REVISION    STATUS
grafana   monitoring   1           deployed
```

The first installation creates:

```text
Revision 1
```

### Important

This is a **Helm release revision**.

It is not the Grafana application version.

For example:

```text
Helm revision = 1
Grafana app version = 12.x
```

These represent different things.

---

# Step-11: Verify Kubernetes Resources

List the resources:

```bash
kubectl get all -n monitoring
```

Check Pods:

```bash
kubectl get pods -n monitoring
```

Check Services:

```bash
kubectl get svc -n monitoring
```

Check the Deployment:

```bash
kubectl get deployment -n monitoring
```

---

# Step-12: Access Grafana

Grafana's official documentation provides instructions for obtaining the generated admin password and accessing Grafana through port forwarding. ([grafana.com](https://grafana.com/docs/grafana/latest/setup-grafana/installation/helm/?utm_source=chatgpt.com))

First inspect the chart notes:

```bash
helm get notes grafana -n monitoring
```

The notes provide chart-specific instructions.

Get the generated admin password:

```bash
kubectl get secret \
  --namespace monitoring \
  grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

The username is:

```text
admin
```

Now identify the Grafana Pod:

```bash
kubectl get pods -n monitoring
```

You can port-forward the Service directly:

```bash
kubectl port-forward \
  -n monitoring \
  svc/grafana \
  3000:80
```

Open:

```text
http://localhost:3000
```

Log in with:

```text
Username: admin
Password: <decoded password>
```

---

# Step-13: Inspect the Installed Release

Now we start investigating what Helm actually installed.

### Current status

```bash
helm status grafana -n monitoring
```

### Values

```bash
helm get values grafana -n monitoring
```

### All computed values

```bash
helm get values grafana \
  --namespace monitoring \
  --all
```

### Kubernetes manifests

```bash
helm get manifest grafana -n monitoring
```

### Chart notes

```bash
helm get notes grafana -n monitoring
```

These commands let us inspect different parts of the Helm release.

---

# Step-14: Find the Image Configuration

Before changing the image tag, inspect the chart's values:

```bash
helm show values \
  oci://ghcr.io/grafana-community/helm-charts/grafana
```

Find the relevant image configuration.

For example, depending on the chart version, you may see an image section containing a tag.

Do **not** blindly assume a particular key.

The chart's `values.yaml` is the source of truth for the values exposed by that chart version.

---

# Step-15: Perform a Helm Upgrade Using `--set`

Suppose the chart exposes:

```yaml
image:
  tag: <CURRENT-TAG>
```

We can override the tag:

```bash
helm upgrade grafana \
  oci://ghcr.io/grafana-community/helm-charts/grafana \
  --namespace monitoring \
  --set "image.tag=<NEW-TAG>" \
  --wait
```

### What does `--set` mean?

It means:

> Override this chart value for this Helm operation.

It does **not** modify the chart.

It does **not** modify the original `values.yaml`.

It supplies a value to Helm while Helm calculates the configuration used to render the chart.

---

# Step-16: What Happens During `helm upgrade`?

This is the most important conceptual part of this lab.

Suppose the current release is:

```text
Revision 1
```

We execute:

```bash
helm upgrade grafana \
  oci://ghcr.io/grafana-community/helm-charts/grafana \
  --namespace monitoring \
  --set "image.tag=<NEW-TAG>"
```

Conceptually:

```text
Existing Helm Release
        │
        ├── Chart
        │
        └── Existing configuration
                 │
                 │
           New --set value
                 │
                 ↓
          Computed values
                 │
                 ↓
          Helm templates
                 │
                 ↓
       Rendered Kubernetes YAML
                 │
                 ↓
          Kubernetes API
                 │
                 ↓
      Kubernetes reconciles changes
                 │
                 ↓
          Grafana workload
                 │
                 ↓
             Revision 2
```

The important thing is:

> **Helm does not simply change a Docker image string inside a Pod.**

Helm renders the chart again using the new configuration and performs the upgrade of the release.

Kubernetes then handles the resulting workload changes.

---

# Step-17: Verify the New Revision

Run:

```bash
helm list -n monitoring
```

The revision should now be:

```text
2
```

Again:

```text
Revision 2 ≠ Grafana version 2
```

It means:

> This is the second revision of the `grafana` Helm release.

---

# Step-18: Verify the Actual Kubernetes Image

Do not rely only on the Grafana UI.

First:

```bash
kubectl get pods -n monitoring
```

Then:

```bash
kubectl get pod <POD-NAME> \
  -n monitoring \
  -o jsonpath='{.spec.containers[*].image}'
```

You can also inspect the Deployment:

```bash
kubectl get deployment -n monitoring
```

Then:

```bash
kubectl get deployment <DEPLOYMENT-NAME> \
  -n monitoring \
  -o jsonpath='{.spec.template.spec.containers[*].image}'
```

This gives you direct evidence that the value ultimately affected the Kubernetes workload.

---

# Step-19: Helm History

View the release history:

```bash
helm history grafana -n monitoring
```

You should see something similar to:

```text
REVISION    STATUS
1           superseded
2           deployed
```

The history answers:

```text
What revisions exist?

What was deployed?

Which revision is currently deployed?

What chart/app version was associated with each revision?
```

Helm's `helm history` command retrieves historical revisions for a release. ([helm.sh](https://docs.helm.sh/docs/helm/helm_history/?utm_source=chatgpt.com))

---

# Step-20: Perform Another Upgrade

Perform another change using a valid value from the chart's current `values.yaml`.

For example:

```bash
helm upgrade grafana \
  oci://ghcr.io/grafana-community/helm-charts/grafana \
  --namespace monitoring \
  --set "image.tag=<ANOTHER-TAG>" \
  --wait
```

Then:

```bash
helm history grafana -n monitoring
```

You should now see:

```text
Revision 1
Revision 2
Revision 3
```

---

# Step-21: `--set` vs `-f values.yaml`

This distinction is fundamental.

Suppose we create:

```yaml
# my-values.yaml

image:
  tag: <VERSION-A>
```

We can upgrade with:

```bash
helm upgrade grafana \
  oci://ghcr.io/grafana-community/helm-charts/grafana \
  -n monitoring \
  -f my-values.yaml
```

Or:

```bash
helm upgrade grafana \
  oci://ghcr.io/grafana-community/helm-charts/grafana \
  -n monitoring \
  -f my-values.yaml \
  --set "image.tag=<VERSION-B>"
```

When both specify the same value, the command-line `--set` value has higher precedence than the value supplied through the file. ([helm.sh](https://docs.helm.sh/docs/helm/helm_upgrade/?utm_source=chatgpt.com))

Conceptually:

```text
Chart defaults
      ↓
values file
      ↓
--set
      ↓
Final value
```

---

# Step-22: Multiple `--set` Values

You can specify multiple values:

```bash
helm upgrade grafana \
  oci://ghcr.io/grafana-community/helm-charts/grafana \
  -n monitoring \
  --set "image.tag=<VERSION>" \
  --set "service.type=ClusterIP"
```

You can also provide multiple assignments:

```bash
--set "image.tag=<VERSION>,service.type=ClusterIP"
```

If the same key is specified multiple times, the right-most value takes precedence. ([helm.sh](https://docs.helm.sh/docs/helm/helm_upgrade/?utm_source=chatgpt.com))

Example:

```bash
--set image.tag=1.0 \
--set image.tag=2.0
```

results in:

```yaml
image:
  tag: 2.0
```

---

# Step-23: `--reuse-values`

Helm also provides:

```bash
--reuse-values
```

This tells Helm to reuse the existing release's values and merge new values supplied through `--set` or `-f` into them. ([helm.sh](https://docs.helm.sh/docs/helm/helm_upgrade/?utm_source=chatgpt.com))

For example:

```bash
helm upgrade grafana \
  oci://ghcr.io/grafana-community/helm-charts/grafana \
  -n monitoring \
  --reuse-values \
  --set "image.tag=<NEW-TAG>"
```

Conceptually:

```text
Previous release values
        +
new --set values
        ↓
new computed values
```

This is an important option to understand, but don't use it blindly in production. You should understand exactly which previous values you are intentionally carrying forward.

---

# Step-24: Helm Rollback

Suppose the release history looks like:

```text
Revision 1 → Working
Revision 2 → Working
Revision 3 → Problem
```

Inspect the history:

```bash
helm history grafana -n monitoring
```

Then roll back to revision 2:

```bash
helm rollback grafana 2 -n monitoring
```

Verify:

```bash
helm history grafana -n monitoring
```

You will now have another revision.

For example:

```text
Revision 1 → Initial installation
Revision 2 → First upgrade
Revision 3 → Bad upgrade
Revision 4 → Rollback to revision 2
```

### Critical concept

Rollback does **not** make revision 2 become the current revision.

It creates a **new revision** representing the rollback operation.

---

# Step-25: Helm Status

Check the current release:

```bash
helm status grafana -n monitoring
```

You can also inspect a particular revision:

```bash
helm status grafana \
  --namespace monitoring \
  --revision 2
```

Think of the commands this way:

```text
helm history
    ↓
"What revisions exist?"

helm status
    ↓
"What is the release's status?"

helm get values
    ↓
"What values are associated with it?"

helm get manifest
    ↓
"What manifests are stored for it?"
```

---

# Step-26: Uninstall the Grafana Release

When you are finished:

```bash
helm uninstall grafana -n monitoring
```

This removes the Helm release and the Kubernetes resources associated with that release.

Helm's official documentation uses `helm uninstall` for removing a release. By default, the release history is also removed; `--keep-history` can be used when you want to retain the release history after uninstalling. ([helm.sh](https://helm.sh/docs/intro/quickstart/?utm_source=chatgpt.com))

Verify:

```bash
helm list -n monitoring
```

Then:

```bash
kubectl get all -n monitoring
```

If this namespace was created only for this lab:

```bash
kubectl delete namespace monitoring
```

### Important: Helm release vs persistent data

Do not assume that:

```bash
helm uninstall grafana
```

always means:

> "Every piece of application data is permanently deleted."

Persistent resources such as PVCs require deliberate consideration.

For a learning environment where you want a completely clean lab, inspect:

```bash
kubectl get pvc -n monitoring
```

before deleting the namespace or storage.

---

# Step-27: Complete Helm Command Reference

These are the commands you should be comfortable with after this lab.

## 1. `helm repo add`

```bash
helm repo add <NAME> <URL>
```

Adds a traditional HTTP Helm chart repository to your local Helm configuration.

Example:

```bash
helm repo add grafana-community \
  https://grafana-community.github.io/helm-charts
```

Use this for traditional chart repositories.

---

## 2. `helm repo update`

```bash
helm repo update
```

Downloads the latest repository metadata for repositories you've added.

Think:

```text
Remote repository
       ↓
helm repo update
       ↓
Local repository metadata
```

---

## 3. `helm repo list`

```bash
helm repo list
```

Lists traditional Helm repositories configured on your local machine.

Important:

> OCI registries don't require `helm repo add`, so they don't appear here merely because you've installed a chart from an OCI registry.

---

## 4. `helm search repo`

```bash
helm search repo <KEYWORD>
```

Searches charts in repositories you've added locally.

Example:

```bash
helm search repo grafana
```

---

## 5. `helm search hub`

```bash
helm search hub <KEYWORD>
```

Searches Artifact Hub.

This is a **chart discovery** mechanism, not the same thing as searching your locally added repositories.

---

## 6. `helm show chart`

```bash
helm show chart <CHART>
```

Shows chart metadata.

Useful for understanding:

```text
Chart name
Chart version
Application version
Description
```

Example:

```bash
helm show chart \
  oci://ghcr.io/grafana-community/helm-charts/grafana
```

---

## 7. `helm show values`

```bash
helm show values <CHART>
```

Shows the chart's default `values.yaml`.

This is one of the most important commands when learning or configuring a chart.

---

## 8. `helm template`

```bash
helm template <RELEASE-NAME> <CHART>
```

Renders the chart locally.

It does **not** install the release.

Think:

> "Show me the Kubernetes YAML Helm would generate."

Example:

```bash
helm template grafana \
  oci://ghcr.io/grafana-community/helm-charts/grafana
```

---

## 9. `helm install`

```bash
helm install <RELEASE-NAME> <CHART>
```

Creates a new Helm release.

Example:

```bash
helm install grafana \
  oci://ghcr.io/grafana-community/helm-charts/grafana \
  -n monitoring \
  --create-namespace
```

This is where:

```text
Chart + Values
      ↓
New Helm Release
```

---

## 10. `helm list`

```bash
helm list
```

Lists Helm releases.

For a namespace:

```bash
helm list -n monitoring
```

It commonly shows:

```text
NAME
NAMESPACE
REVISION
STATUS
CHART
APP VERSION
```

---

## 11. `helm upgrade`

```bash
helm upgrade <RELEASE-NAME> <CHART>
```

Upgrades an existing release using a chart and new configuration.

Example:

```bash
helm upgrade grafana \
  oci://ghcr.io/grafana-community/helm-charts/grafana \
  -n monitoring \
  --set "image.tag=<NEW-TAG>"
```

This is the central command of this lesson.

---

## 12. `helm get values`

```bash
helm get values <RELEASE-NAME>
```

Shows USER-SUPPLIED VALUES. the ones you passed brother using your extra values.yaml, --set etc.

Use:

```bash
helm get values grafana -n monitoring
```

For all computed values:

```bash
helm get values grafana \
  -n monitoring \
  --all
```

---

## 13. `helm get manifest`

```bash
helm get manifest <RELEASE-NAME>
```

Shows the Kubernetes manifests stored for the release.

Example:

```bash
helm get manifest grafana -n monitoring
```

This is extremely useful when debugging:

```text
"What Kubernetes YAML did Helm generate for this release?"
```

---

## 14. `helm get notes`

```bash
helm get notes <RELEASE-NAME>
```

Shows the chart's post-install instructions.

For Grafana:

```bash
helm get notes grafana -n monitoring
```

This is often useful for discovering how the chart expects you to access the application.

---

## 15. `helm status`

```bash
helm status <RELEASE-NAME>
```

Shows the current status/details of a release.

Example:

```bash
helm status grafana -n monitoring
```

---

## 16. `helm history`

```bash
helm history <RELEASE-NAME>
```

Shows release revisions.

Example:

```bash
helm history grafana -n monitoring
```

Think:

> "Show me the timeline of this Helm release."

---

## 17. `helm rollback`

```bash
helm rollback <RELEASE-NAME> <REVISION>
```

Restores a previous release state.

Example:

```bash
helm rollback grafana 2 -n monitoring
```

Remember:

```text
Rollback to revision 2
        ↓
Creates a new revision
```

It does not erase the history.

---

## 18. `helm uninstall`

```bash
helm uninstall <RELEASE-NAME>
```

Removes a Helm release.

Example:

```bash
helm uninstall grafana -n monitoring
```

If you need to retain release history:

```bash
helm uninstall grafana \
  -n monitoring \
  --keep-history
```

---

# Final Mental Model

Don't memorize Helm as a collection of commands.

Think of it as a lifecycle:

```text
                 CHART
                   │
                   ↓
            helm show values
                   │
                   ↓
              VALUES
          ┌────────┴────────┐
          │                 │
       -f file            --set
          │                 │
          └────────┬────────┘
                   ↓
             HELM RENDERING
                   │
                   ↓
           Kubernetes YAML
                   │
                   ↓
             helm install
                   │
                   ↓
             RELEASE v1
                   │
                   ↓
             helm upgrade
                   │
                   ↓
             RELEASE v2
                   │
                   ↓
             helm history
                   │
             ┌─────┴─────┐
             ↓           ↓
        helm status   helm rollback
                         │
                         ↓
                    RELEASE v3
```

### The core idea

> **A Helm chart is the package/template. Values configure the chart. Helm renders the chart into Kubernetes manifests and manages those manifests as a versioned release. `helm upgrade` creates a new revision of that release using the new configuration.**

That is the mental model I want you to carry forward into the next Helm topic.

One small but important correction to your requested uninstall section: I'd teach **`helm uninstall` rather than `helm delete`**. `helm delete` is an alias, but `helm uninstall` is the clearer current terminology in the official Helm docs. Also, `helm uninstall` removes release history by default; `--keep-history` changes that behavior. ([helm.sh][1])

For the next `.md` you send, I'll keep this exact standard: **official-doc verification first, then correction, then deeper mental model, then hands-on commands, then a detailed command/concept reference at the end.**

[1]: https://helm.sh/docs/intro/quickstart/?utm_source=chatgpt.com "Quickstart Guide | Helm"
