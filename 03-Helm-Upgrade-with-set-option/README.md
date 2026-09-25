# Helm Upgrade with `--set` Option

## Step-01: Introduction

In this demo, we will upgrade an existing **WordPress Helm release** using the `helm upgrade` command.

We will specifically use the `--set` option to override the Docker image tag during the upgrade:

```bash
helm upgrade wordpress helmforge/wordpress --set "image.tag=7.0.2-apache"
```

### Helm commands used in this demo

* `helm repo`
* `helm search repo`
* `helm install`
* `helm upgrade`
* `helm list`
* `helm history`
* `helm status`
* `helm uninstall`

### What we are learning

The important idea in this demo is:

```text
Existing Helm Release
        |
        | helm upgrade
        | --set image.tag=<new-tag>
        v
Helm renders the chart again
        |
        v
Kubernetes resources are updated
        |
        v
WordPress runs with the new image
```

`--set` allows us to override chart values directly from the command line. Helm gives these command-line overrides higher priority than values coming from the chart's default values.

---

# Step-02: Add and Explore the Helm Repository

## Step-02-01: Add the Helm Repository

Add the Helm repository that contains the WordPress chart:

```bash
# Add Helm repository
helm repo add helmforge https://repo.helmforge.dev

# Update local repository information
helm repo update
```

### Verify the repository

```bash
helm repo list
```

You should see the repository:

```text
NAME        URL
helmforge   https://repo.helmforge.dev
```

`helm repo add` registers a chart repository with your local Helm client, while `helm repo update` refreshes the locally cached information about charts available from configured repositories.

---

## Step-02-02: Search for the WordPress Chart

Once the repository has been added, search the repositories configured on your machine:

```bash
helm search repo wordpress
```

You should see the WordPress chart:

```text
helmforge/wordpress
```

### Important distinction

There are two commonly used Helm search commands:

```bash
helm search repo wordpress
```

Searches the **repositories already added to your local Helm client**.

Whereas:

```bash
helm search hub wordpress
```

searches **Artifact Hub** for publicly available charts.

---

# Step-03: Inspect the WordPress Chart

Before installing a chart, it is useful to understand the values that the chart exposes.

### Display the chart's default values

```bash
helm show values helmforge/wordpress
```

This shows the chart's `values.yaml`.

Look for the image configuration, for example:

```yaml
image:
  repository: ...
  tag: ...
```

The exact structure depends on the chart version.

### Inspect a specific chart version

```bash
helm show values helmforge/wordpress --version <CHART-VERSION>
```

For example:

```bash
helm show values helmforge/wordpress --version 3.0.3
```

### Why this matters

The important Helm concept is:

```text
Chart's values.yaml
        +
Your overrides
        |
        v
Final values used by Helm
        |
        v
Templates
        |
        v
Kubernetes manifests
```

We will override the image tag using `--set`.

---

# Step-04: Install WordPress

Install the WordPress chart:

```bash
helm install wordpress helmforge/wordpress
```

Here:

```text
wordpress
    |
    +-- Release name

helmforge/wordpress
    |
    +-- Chart reference
```

The release name is **`wordpress`**.

Check the installed release:

```bash
helm list
```

You should see something similar to:

```text
NAME       STATUS     CHART
wordpress  deployed   wordpress-<version>
```

---

# Step-05: List Kubernetes Resources and Access WordPress

### List Pods

```bash
kubectl get pods
```

### List Services

```bash
kubectl get svc
```

You should find the WordPress Service.

For example:

```text
NAME        TYPE        CLUSTER-IP      PORT(S)
wordpress   ClusterIP   10.x.x.x        80/TCP
```

### Access WordPress using port-forward

If the Service exposes port `80`:

```bash
kubectl port-forward svc/wordpress 8080:80
```

Then open:

```text
http://localhost:8080
```

### What is happening?

`kubectl port-forward` creates a temporary connection:

```text
Browser
   |
   | localhost:8080
   v
kubectl port-forward
   |
   | Kubernetes connection
   v
wordpress Service :80
   |
   v
WordPress Pod
```

This avoids requiring a NodePort or external LoadBalancer just to access the application locally.

---

# Step-06: Upgrade WordPress Using `--set`

Now we will change the WordPress container image tag.

First, identify the image configuration exposed by the chart:

```bash
helm show values helmforge/wordpress
```

Suppose the chart contains:

```yaml
image:
  repository: ...
  tag: 6.0.0-apache
```

We can override only the tag:

```bash
helm upgrade wordpress helmforge/wordpress \
  --set "image.tag=7.0.2-apache"
```

### Understand the command

```text
helm upgrade
    |
    +-- wordpress
    |      |
    |      +-- Existing release name
    |
    +-- helmforge/wordpress
           |
           +-- Chart

--set "image.tag=7.0.2-apache"
           |
           +-- Override chart value
```

The important point is that **we are not editing the chart's `values.yaml`**.

We are saying:

> "For this upgrade, use `7.0.2-apache` for the `image.tag` value."

Helm supports `--set` specifically for supplying value overrides from the command line.

---

# Step-07: Observe the Helm Revision

After the upgrade:

```bash
helm list
```

The release should now have a newer revision.

You can inspect the revision history:

```bash
helm history wordpress
```

You should see something similar to:

```text
REVISION   STATUS
1          superseded
2          deployed
```

### Important concept: Revision

A Helm **revision** represents a version of the release's deployment history.

For example:

```text
Revision 1
    |
    | helm install
    v
WordPress image = old version

Revision 2
    |
    | helm upgrade --set image.tag=7.0.2-apache
    v
WordPress image = 7.0.2-apache

Revision 3
    |
    | another helm upgrade
    v
WordPress image = newer version
```

`helm history` displays the historical revisions of a release.

---

# Step-08: Verify the WordPress Upgrade

Check the Pods:

```bash
kubectl get pods
```

You can also inspect the image currently used by the Pod:

```bash
kubectl get pod <POD-NAME> \
  -o jsonpath='{.spec.containers[*].image}'
```

You should see the new image tag.

You can also inspect the Deployment:

```bash
kubectl get deployment wordpress \
  -o jsonpath='{.spec.template.spec.containers[*].image}'
```

### Access WordPress again

If the port-forward is still running:

```text
http://localhost:8080
```

The application should now be running with the upgraded image.

---

# Step-09: Perform More WordPress Upgrades

For practice, perform additional upgrades by choosing valid image tags supported by the chart/application.

For example:

```bash
helm upgrade wordpress helmforge/wordpress \
  --set "image.tag=<NEW-TAG>"
```

Then verify:

```bash
helm history wordpress
```

and:

```bash
kubectl get pods
```

You should see the Helm revision increasing:

```text
Revision 1 → Initial installation
Revision 2 → First upgrade
Revision 3 → Second upgrade
Revision 4 → Third upgrade
```

This is a good way to understand that **each successful `helm upgrade` creates a new release revision**.

---

# Step-10: Helm History

`helm history` displays the historical revisions of a release.

```bash
helm history wordpress
```

Example:

```text
REVISION   UPDATED                  STATUS       CHART
1          ...                      superseded   wordpress-...
2          ...                      superseded   wordpress-...
3          ...                      deployed     wordpress-...
```

### Why is this useful?

It allows you to answer questions such as:

* What happened to this release?
* How many upgrades have occurred?
* Which revision is currently deployed?
* Which revision was deployed before the current one?
* What chart version was used for each revision?

You can also limit the number of revisions displayed:

```bash
helm history wordpress --max 5
```

---

# Step-11: Helm Status

`helm status` shows the current status of a release, including its state, revision, namespace, description, resources, and chart-provided notes.

```bash
helm status wordpress
```

### Show the release description

```bash
helm status wordpress --show-desc
```

### Show resources belonging to the release

```bash
helm status wordpress --show-resources
```

### Show a specific revision

```bash
helm status wordpress --revision 2
```

For example:

```bash
helm status wordpress --revision 2
```

This lets you inspect the status information associated with revision 2.

---

# Step-12: Inspect the Values Used by the Release

An extremely useful command when learning Helm is:

```bash
helm get values wordpress
```

This shows the values that were supplied for the release.

To see all computed values, including chart defaults:

```bash
helm get values wordpress --all
```

This is useful for understanding the difference between:

```text
Chart defaults
      +
User-supplied values
      +
--set overrides
      |
      v
Computed values
```

---

# Step-13: Inspect the Kubernetes Manifests Generated by Helm

You can inspect the manifests stored for the release:

```bash
helm get manifest wordpress
```

This is one of the most useful commands for understanding Helm.

The relationship becomes:

```text
values.yaml
     +
--set image.tag=...
     |
     v
Helm template rendering
     |
     v
Kubernetes YAML
     |
     v
Kubernetes API Server
     |
     v
Pods / Services / Deployments / etc.
```

This lets you connect the **Helm values** you change with the **actual Kubernetes resources** created or updated by Helm.

---

# Step-14: Uninstall the WordPress Release

When finished with the demo:

```bash
helm uninstall wordpress
```

This removes the Helm release and the Kubernetes resources managed by that release.

Verify:

```bash
helm list
```

and:

```bash
kubectl get pods
```

---

# Key Takeaways

### 1. `helm install` creates the initial release

```bash
helm install wordpress helmforge/wordpress
```

This creates **revision 1**.

### 2. `helm upgrade` changes an existing release

```bash
helm upgrade wordpress helmforge/wordpress \
  --set "image.tag=7.0.2-apache"
```

This creates a **new revision**.

### 3. `--set` overrides chart values

```bash
--set "image.tag=7.0.2-apache"
```

You don't need to modify the chart's `values.yaml`.

### 4. `helm history` shows the release's revisions

```bash
helm history wordpress
```

### 5. `helm status` shows the current release state

```bash
helm status wordpress
```

### 6. `helm get values` shows release values

```bash
helm get values wordpress
```

### 7. `helm get manifest` shows the Kubernetes manifests

```bash
helm get manifest wordpress
```

### The complete picture

```text
                   Helm Chart
                       |
                 values.yaml
                       |
              +--------+--------+
              |                 |
        chart defaults      --set override
              |                 |
              +--------+--------+
                       |
                 Final Values
                       |
                 Helm Rendering
                       |
                 Kubernetes YAML
                       |
              Kubernetes API Server
                       |
          +------------+------------+
          |            |            |
      Deployment     Service       ...
          |
        Pod
          |
      WordPress
```

This is the core idea behind the demo: **`helm upgrade` takes an existing release, renders the chart with the new values, and applies the resulting changes to Kubernetes.**
