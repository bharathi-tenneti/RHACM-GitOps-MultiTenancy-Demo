# Multi-tenancy model example with Red Hat Advanced Cluster Management and OpenShift GitOps operator

This repo provides an example to configure cluster-as-a-service multi-tenancy model using Red Hat Advanced Cluster Management and OpenShift GitOps operator. It extends the per-tenant isolation model with a **fleet-wide GitOps control plane** using the ArgoCD Agent architecture, enabling centralised visibility and platform delivery across all managed clusters.

![image](https://user-images.githubusercontent.com/41969005/159989841-95b5dce8-b678-4cc1-8020-ae9d50a42089.png)

Blue group of users and red group of users share the ACM hub cluster but have access to their own namespaces to manage and view their applications. Each group has separate set of managed clusters where applications are deployed to. ACM group of users has access to both groups' applications and all managed clusters. The application developers pushes application manifests to their Git repos but do not have access to the cluster environment. ACM, blue and red SRE group users can set up GitOps to deploy the applications from Git repos to clusters using OpenShift GitOps ApplicationSets.

## Cluster environment
- ACM hub cluster
- Three managed clusters: `bluecluster1`, `bluecluster2`, `redcluster`

![image](https://user-images.githubusercontent.com/41969005/160040406-bf5e9b3c-ebe9-4e6c-82fa-0fde1071bd03.png)

## Repository structure

```
.
├── kustomization.yaml                          # Root — apply entire repo with: kubectl apply -k .
├── AcmPolicies/
│   ├── kustomization.yaml                      # Aggregates all policy subdirectories
│   ├── InstallGitOpsOperator/                  # ACM Policy: install OpenShift GitOps operator on hub
│   ├── ArgoCDInstances/                        # ACM Policy: per-tenant ArgoCD instances (blue, red)
│   ├── RegisterClustersToArgoCDInstances/      # ACM Policy: register tenant cluster sets to ArgoCD
│   ├── FleetArgoCD/                            # ACM Policy: fleet-wide ArgoCD Principal + Agent
│   └── RegisterAllClustersToFleet/             # ACM Policy: register all clusters to fleet ArgoCD
├── ApplicationSets/
│   ├── blueMobileAppset.yaml                   # Blue tenant app (mobile) → bluecluster1, bluecluster2
│   ├── redGalagaAppset.yaml                    # Red tenant app (galaga) → redcluster
│   └── fleetPlatformAppset.yaml                # Fleet platform infra → all managed clusters
├── PlatformConfig/
│   └── baseline/                               # Kustomize base: LimitRange + NetworkPolicy defaults
└── UsersGroups/                                # Users, Groups, HTPasswd OAuth config
```

Every directory contains a `kustomization.yaml`, so individual layers or the full repo can be rendered with `kubectl kustomize <path>` or applied with `kubectl apply -k <path>`.

---

## Part 1 — Per-tenant multi-tenancy setup (Blue & Red)

### Steps to create multi-tenancy GitOps environment for blue, red and ACM user groups

1. Clone this repo.

2. Log into ACM hub cluster via CLI and create blue, red and ACM SRE and viewer groups and users.

```bash
oc create secret generic htpass-secret --from-file=htpasswd=./UsersGroups/htpasswd -n openshift-config
kubectl apply -k ./UsersGroups
```

![image](https://user-images.githubusercontent.com/41969005/160040354-df18fd29-ff74-463f-b6b1-43045b404e2f.png)

3. Grant ACM groups cluster-wide access.

    a. Grant `acm-sre-group` group admin access cluster-wide. Log into OCP console and go to `User management` > `Groups` > `acm-sre-group`. Go to `Role binding` tab and create a binding. Type `Cluster-wide role binding`, role binding name `acm-sre-group`, role name `cluster-admin`

    b. Grant `acm-viewer-group` group view access cluster-wide. Log into OCP console and go to `User management` > `Groups` > `acm-viewer-group`. Go to `Role binding` tab and create a binding. Type `Cluster-wide role binding`, role binding name `acm-viewer-group`, role name `view`

4. Log into ACM console as an ACM SRE user and create `blueclusterset` cluster set. Add `bluecluster1` and `bluecluster2` clusters to the cluster set.

5. Create `redclusterset` cluster set. Add `redcluster` cluster to the cluster set.

6. In `blueclusterset` cluster set, go to `Access management` tab.

    a. Add `blue-sre-group` group with `Cluster set admin` role. This grants `blue-sre-group` group admin access to `bluecluster1` and `bluecluster2` managed cluster namespaces on ACM hub. This also allows the group admin access to all resources that ACM finds from the remote managed clusters.
    b. Add `blue-viewer-group` group with `Cluster set view` role. This grants `blue-viewer-group` group view access to `bluecluster1` and `bluecluster2` managed cluster namespaces on ACM hub. This also allows the group view access to all resources that ACM finds from the remote managed clusters.

![image](https://user-images.githubusercontent.com/41969005/160016083-83352c70-65d1-4de5-83a1-836e54c51d48.png)

7. In `redclusterset` cluster set, go to `Access management` tab.

    a. Add `red-sre-group` group with `Cluster set admin` role. This grants `red-sre-group` group admin access to `redcluster` managed cluster namespace on ACM hub. This also allows the group admin access to all resources that ACM finds from the remote managed cluster.
    b. Add `red-viewer-group` group with `Cluster set view` role. This grants `red-viewer-group` group view access to `redcluster` managed cluster namespace on ACM hub. This also allows the group view access to all resources that ACM finds from the remote managed cluster.

![image](https://user-images.githubusercontent.com/41969005/160016132-a00c1486-b3c3-4ab0-b30e-f306b3990511.png)

8. Install `Red Hat OpenShift GitOps` operator and wait until all pods in `openshift-gitops` namespace are running.

```bash
kubectl apply -k ./AcmPolicies/InstallGitOpsOperator
```

9. Create one ArgoCD server instance for the blue group in `blueargocd` namespace and another for the red group in `redargocd` namespace. Wait until all pods are running and `applicationset-controller` and `dex-server` pods are present.

```bash
kubectl apply -k ./AcmPolicies/ArgoCDInstances
```

> **Note:** `Red Hat OpenShift GitOps` operator does not need to be installed on managed clusters. The ArgoCD server instance running on the hub cluster connects to target remote clusters to deploy applications defined in the `ApplicationSet`.

10. Register cluster sets to their ArgoCD instances.

```bash
kubectl apply -k ./AcmPolicies/RegisterClustersToArgoCDInstances
```

These operator installation and instance creations are enforced by ACM governance policies.

![image](https://user-images.githubusercontent.com/41969005/160182359-5abfad36-690b-4293-b72c-5bdffa825cd1.png)

11. All blue applications are in `blueargocd` namespace.

    a. Grant `blue-sre-group` admin access to `blueargocd` namespace. Log into OCP console and go to `User management` > `Groups` > `blue-sre-group`. Go to `Role binding` tab and create a binding. Type `Namespace role binding`, role binding name `blue-sre-group`, namespace `blueargocd`, role name `admin`

    b. Grant `blue-viewer-group` view access to `blueargocd` namespace. Same path, role name `view`.

12. All red applications are in `redargocd` namespace.

    a. Grant `red-sre-group` admin access to `redargocd` namespace. Same path, role name `admin`.

    b. Grant `red-viewer-group` view access to `redargocd` namespace. Same path, role name `view`.

13. Edit the blue ArgoCD instance's RBAC:

```bash
oc edit configmap argocd-rbac-cm -n blueargocd
```

```yaml
data:
  policy.csv: |
    g, acm-sre-group, role:admin
    g, acm-viewer-group, role:readonly
    g, blue-sre-group, role:admin
    g, blue-viewer-group, role:readonly
  policy.default: role:''
  scopes: '[groups]'
```

14. Edit the red ArgoCD instance's RBAC:

```bash
oc edit configmap argocd-rbac-cm -n redargocd
```

```yaml
data:
  policy.csv: |
    g, acm-sre-group, role:admin
    g, acm-viewer-group, role:readonly
    g, red-sre-group, role:admin
    g, red-viewer-group, role:readonly
  policy.default: role:''
  scopes: '[groups]'
```

15. Find the blue ArgoCD console URL (OpenShift OAuth via Dex is enabled — ACM and blue group users can log in):

```bash
oc get route blueargocd-server -n blueargocd
```

16. Find the red ArgoCD console URL:

```bash
oc get route redargocd-server -n redargocd
```

> **GitOpsification:** The above steps can be translated to manifest YAMLs with ArgoCD sync-wave and pushed to a Git repository. The ACM-SRE user can then use the default `openshift-gitops` ArgoCD instance to deploy the Git repo as an Argo application to the hub cluster.

![image](https://user-images.githubusercontent.com/41969005/160181938-0bf0f746-706f-471e-9cf0-8b1aa762782f.png)

---

## Part 2 — Fleet-wide GitOps with ArgoCD Agent architecture

This section extends the per-tenant setup with a fleet-wide control plane based on the [ArgoCD Agent architecture](https://medium.com/@tcij1013/fleet-scale-gitops-control-flow-with-red-hat-advanced-cluster-management-0eca855136c3). Instead of a single hub ArgoCD reconciling all clusters directly (which creates a hub bottleneck at scale), this model places a lightweight **Agent** on each spoke that pulls specs from a hub **Principal** over outbound mTLS gRPC — eliminating inbound API exposure and hub-stored spoke credentials.

```
Hub cluster (openshift-gitops namespace)
  └── ArgoCD Principal (fleet-argocd-principal)
        ├── No local reconciler (controller.enabled: false)
        ├── Aggregates sync status from all Agents
        └── Exposes fleet-wide UI + gRPC endpoint

Each spoke cluster (argocd-agent namespace)
  └── ArgoCD Agent (argocd-agent)
        ├── No UI (server.enabled: false)
        ├── Dials Principal via mTLS gRPC (outbound only)
        └── Reconciles Applications locally, streams status back
```

### Prerequisites

- Red Hat OpenShift GitOps operator **≥ 1.10** (required for `argoproj.io/v1beta1` and the `argoCDAgent` spec field)
- Steps 1–10 from Part 1 completed

### Step 17 — Install GitOps operator on all spoke clusters and deploy the fleet ArgoCD Principal

The ArgoCD Agent runs as an ArgoCD CR on each spoke, so the GitOps operator must be present there before the Agent policy is applied. `fleetGitOpsOperatorPolicy.yaml` handles this automatically — it reuses the `all-managed-clusters` PlacementRule and enforces the same GitOps operator Subscription on every spoke.

Apply all three fleet policies together (operator install → Principal → Agent):

```bash
kubectl apply -k ./AcmPolicies/FleetArgoCD
```

Wait for `fleet-gitops-operator-install` to show `Compliant` on all spokes before proceeding, then retrieve the Principal's gRPC/UI route:

```bash
oc get route fleet-argocd-principal-server -n openshift-gitops -o jsonpath='{.spec.host}'
```

### Step 18 — Update the Agent policy with the Principal address

Open `AcmPolicies/FleetArgoCD/fleetArgoCDAgentPolicy.yaml` and replace the placeholder in the `address:` field with the hostname from the previous step:

```yaml
address: "<output-from-step-17>:443"
```

This is the one manual step in the deployment — the Principal must exist before Agents can dial it. In production, use an ACM hub template to auto-populate this value:

```yaml
address: "{{hub (lookup \"v1\" \"Route\" \"openshift-gitops\" \"fleet-argocd-principal-server\").spec.host hub}}:443"
```

### Step 19 — Register all clusters to the fleet ArgoCD

```bash
kubectl apply -k ./AcmPolicies/RegisterAllClustersToFleet
```

This policy creates:
- A `ManagedClusterSetBinding` for the built-in `global` cluster set (contains all managed clusters automatically) in `openshift-gitops`
- A `fleet-placement` Placement selecting all clusters
- A `GitOpsCluster` resource wiring the Placement to the `openshift-gitops` ArgoCD namespace

Verify the PlacementDecision lists all clusters:

```bash
oc get placementdecision -n openshift-gitops
```

### Step 20 — Deploy platform infra to the entire fleet

```bash
kubectl apply -f ./ApplicationSets/fleetPlatformAppset.yaml
```

The `fleet-platform-appset` ApplicationSet reads `fleet-placement` decisions and deploys `PlatformConfig/baseline` (default LimitRanges and NetworkPolicies) to every cluster. ArgoCD automatically uses Kustomize to render the baseline directory because a `kustomization.yaml` is present.

Verify one Application per cluster was created:

```bash
oc get application -n openshift-gitops
```

### Fleet ArgoCD console

```bash
oc get route fleet-argocd-principal-server -n openshift-gitops
```

Open the route in a browser. ACM SRE users can log in via OpenShift OAuth and see sync status for all clusters in a single view.

### Extending `PlatformConfig/baseline`

`PlatformConfig/baseline/kustomization.yaml` is a standard Kustomize base. Add cluster-wide defaults here (e.g. `ResourceQuota`, `PodDisruptionBudget`, admission webhooks). The fleet ApplicationSet will automatically deliver any new resources to all clusters on the next sync.

---

## Managed cluster registration verification in ArgoCD

Log into the blue and red ArgoCD consoles and create an application to verify that the managed clusters are listed as application destinations.

![image](https://user-images.githubusercontent.com/41969005/160016710-38e17da1-f2b4-4400-86ff-ab0b4ed0bf5c.png)

![image](https://user-images.githubusercontent.com/41969005/160016744-31b055ea-38ea-430d-8dab-3620bd96fd70.png)

## RBAC Verifications

Now everything is set up. The `ApplicationSets/blueMobileAppset.yaml` deploys `https://github.com/rokej/BlueApplications/tree/main/mobileApplication` to both remote blue clusters using the `blue-placement` PlacementDecision:

```yaml
- clusterDecisionResource:
    configMapRef: acm-placement
    labelSelector:
      matchLabels:
        cluster.open-cluster-management.io/placement: blue-placement
```

### As a viewer

```bash
kubectl apply -k ./ApplicationSets
```

Expected output — viewers have read-only access to application namespaces:

```
Error from server (Forbidden): error when creating "ApplicationSets/blueMobileAppset.yaml": applicationsets.argoproj.io is forbidden: User "blueviewer1" cannot create resource "applicationsets" in API group "argoproj.io" in the namespace "blueargocd"
```

### As a blue SRE user

```bash
kubectl apply -k ./ApplicationSets
```

Expected output — blue SRE has admin access to `blueargocd` but not `redargocd`:

```
applicationset.argoproj.io/mobile-application-set created
Error from server (Forbidden): ...applicationsets.argoproj.io "galaga-application-set" is forbidden: User "bluesre1" cannot get resource "applicationsets" in API group "argoproj.io" in the namespace "redargocd"
```

### RHACM console multi-tenancy

Blue group users can visualise and manage only blue applications:

![image](https://user-images.githubusercontent.com/41969005/159999028-df152f14-bc68-4720-8506-7ae2abfc2beb.png)

Red group users can visualise and manage only red applications:

![image](https://user-images.githubusercontent.com/41969005/159999168-6a3afc06-ed09-4297-9b4b-7f6083d422b2.png)

ACM group users can visualise and manage all applications:

![image](https://user-images.githubusercontent.com/41969005/160201485-fbe71180-2762-473e-ba47-84f17bb343ef.png)

### Separate ArgoCD consoles

There are two separate ArgoCD consoles — `blueargocd` and `redargocd` — with RBAC configured in steps 13–14, so one group cannot log into the other group's instance.

![image](https://user-images.githubusercontent.com/41969005/159999747-dd55dda5-52e0-48dc-84ab-25a6f4b4772d.png)

![image](https://user-images.githubusercontent.com/41969005/159999791-988724ca-17f2-452e-97b7-985058fce85c.png)

### Managing Argo applications from RHACM console

You can create, view and edit application sets from the RHACM console and launch directly into an ArgoCD console to manage the application.

![image](https://user-images.githubusercontent.com/41969005/160003338-7a5c7450-27fe-457d-b262-dc63643cfe6d.png)
