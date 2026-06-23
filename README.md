# RHACM GitOps Multi-Tenancy Demo

Fleet-wide GitOps using Red Hat Advanced Cluster Management and OpenShift GitOps. A single `fleet-argocd` instance on the hub delivers platform config and all tenant applications across all clusters. ArgoCD AppProjects provide tenant isolation — each team sees only their own applications. Platform admins have a single pane of glass.

![image](https://user-images.githubusercontent.com/41969005/159989841-95b5dce8-b678-4cc1-8020-ae9d50a42089.png)

## Prerequisites

- 1 OpenShift hub cluster with RHACM installed
- Managed clusters imported into ACM 
- OpenShift GitOps operator **≥ 1.10** on hub

## Architecture

```
fleet-argocd (fleet-gitops namespace on hub)  — single ArgoCD for all teams
│
├── AppProject: default    → platform-baseline → ALL clusters       (acm-sre-group: admin)
├── AppProject: blue-team  → mobile-app        → blueclusterset     (blue-sre-group: admin)
└── AppProject: red-team   → galaga            → redclusterset      (red-sre-group: admin)

ACM GitOpsCluster + Placements wire clusters to fleet-argocd:
  fleet-placement  (global set)    → all clusters    → platform-baseline
  blue-placement   (blueclusterset) → blue clusters  → mobile-app
  red-placement    (redclusterset)  → red clusters   → galaga
```

All users log into the same `fleet-argocd` URL. AppProjects enforce what each team can see:
- `acmsre1` sees all applications across all clusters
- `bluesre1` sees only `blue-team` project applications (blue clusters)
- `redsre1` sees only `red-team` project applications (red clusters)

## Repository structure

```
.
├── kustomization.yaml                              # Hub resources (UsersGroups + AcmPolicies)
├── AcmPolicies/
│   ├── InstallGitOpsOperator/                      # Install GitOps operator on hub
│   ├── FleetArgoCD/                                # fleet-argocd instance in fleet-gitops namespace
│   ├── RegisterAllClustersToFleet/                 # Global binding + fleet-placement + GitOpsCluster
│   └── ClusterSets/                                # One folder per ClusterSet — add a folder = new team
│       ├── blueclusterset/
│       │   └── gitopsclusterPolicy.yaml            # ManagedClusterSetBinding + blue-placement in fleet-gitops
│       └── redclusterset/
│           └── gitopsclusterPolicy.yaml            # ManagedClusterSetBinding + red-placement in fleet-gitops
├── ApplicationSets/
│   ├── fleet/
│   │   └── fleetPlatformAppset.yaml                # Platform baseline → all clusters (fleet-gitops)
│   ├── blueclusterset/
│   │   ├── blueTeamAppProject.yaml                 # AppProject: blue-team scoped to blue-sre-group
│   │   └── blueMobileAppset.yaml                   # Mobile app → blueclusterset (fleet-gitops, project: blue-team)
│   └── redclusterset/
│       ├── redTeamAppProject.yaml                  # AppProject: red-team scoped to red-sre-group
│       └── redGalagaAppset.yaml                    # Galaga → redclusterset (fleet-gitops, project: red-team)
├── PlatformConfig/baseline/                        # Deployed by fleet-argocd to all clusters
└── UsersGroups/                                    # Users, Groups, HTPasswd OAuth config
```

---

## Setup steps

### Step 1 — Create users and groups on the hub

```bash
oc create secret generic htpass-secret \
  --from-file=htpasswd=./UsersGroups/htpasswd \
  -n openshift-config

oc apply -k ./UsersGroups
```

### Step 2 — Grant ACM group cluster-wide RBAC

```bash
oc adm policy add-cluster-role-to-group cluster-admin acm-sre-group
oc adm policy add-cluster-role-to-group view acm-viewer-group
```

### Step 3 — Install GitOps operator on hub

```bash
oc apply -k ./AcmPolicies/InstallGitOpsOperator
```

Wait until all pods in `openshift-gitops` are `Running`:

```bash
oc get pods -n openshift-gitops
```

---

## Part 1 — Fleet-wide setup

Creates `fleet-argocd` in `fleet-gitops` on the hub. ACM registers all clusters to it via the built-in `global` ManagedClusterSet. `fleet-argocd` delivers platform-level config to every cluster. Platform admins have a single pane of glass.

### Step 4 — Create fleet ArgoCD instance

```bash
oc apply -k ./AcmPolicies/FleetArgoCD
```

Wait for all pods to be `Running`:

```bash
oc get pods -n fleet-gitops
```

### Step 5 — Register all clusters to fleet ArgoCD

```bash
oc apply -k ./AcmPolicies/RegisterAllClustersToFleet
```

Verify PlacementDecisions exist:

```bash
oc get placementdecision -n fleet-gitops
```

### Step 6 — Deploy fleet ApplicationSet

```bash
oc apply -k ./ApplicationSets/fleet
```

This creates `fleet-platform-appset` in `fleet-gitops`, delivering platform baseline to every cluster.

Verify:

```bash
oc get applicationset -n fleet-gitops
oc get applications.argoproj.io -n fleet-gitops
```

Expected:

```
NAME                              SYNC STATUS   HEALTH STATUS
platform-baseline-cluster1        Synced        Healthy
platform-baseline-local-cluster   Synced        Healthy
```

Get the fleet ArgoCD console URL:

```bash
oc get route fleet-argocd-server -n fleet-gitops
```

### Step 7 — Configure fleet ArgoCD RBAC

```bash
oc edit configmap argocd-rbac-cm -n fleet-gitops
```

```yaml
data:
  policy.csv: |
    g, acm-sre-group, role:admin
    g, acm-viewer-group, role:readonly
  policy.default: role:''
  scopes: '[groups]'
```

`acmsre1` (acm-sre-group) now sees all clusters and all applications across every team from one URL.

---

## Part 2 — Per-team ClusterSet onboarding

Each team gets a dedicated ClusterSet. ACM creates a `Placement` in `fleet-gitops` for that ClusterSet, and an AppProject scopes what the team can see in `fleet-argocd`. All teams log into the **same** `fleet-argocd` URL — AppProjects enforce the boundary.

**Adding a new team follows the same pattern every time:**
1. Create a ManagedClusterSet and assign clusters to it
2. Create `AcmPolicies/ClusterSets/<team>/gitopsclusterPolicy.yaml` — registers the ClusterSet binding and Placement into `fleet-gitops`
3. Create `ApplicationSets/<team>/` — AppProject + ApplicationSet (both in `fleet-gitops` namespace)
4. Add both new paths to the root kustomization files

### Step 8 — Create ClusterSets and assign clusters

```bash
oc apply -f - <<'EOF'
apiVersion: cluster.open-cluster-management.io/v1beta2
kind: ManagedClusterSet
metadata:
  name: blueclusterset
---
apiVersion: cluster.open-cluster-management.io/v1beta2
kind: ManagedClusterSet
metadata:
  name: redclusterset
EOF
```

Assign your managed clusters (replace names with your actual cluster names):

```bash
oc label managedcluster <blue-cluster-1> cluster.open-cluster-management.io/clusterset=blueclusterset --overwrite
oc label managedcluster <red-cluster-1>  cluster.open-cluster-management.io/clusterset=redclusterset --overwrite
```

Grant group access to each ClusterSet:

```bash
oc adm policy add-cluster-role-to-group open-cluster-management:managedclusterset:admin:blueclusterset blue-sre-group
oc adm policy add-cluster-role-to-group open-cluster-management:managedclusterset:view:blueclusterset  blue-viewer-group
oc adm policy add-cluster-role-to-group open-cluster-management:managedclusterset:admin:redclusterset  red-sre-group
oc adm policy add-cluster-role-to-group open-cluster-management:managedclusterset:view:redclusterset   red-viewer-group
```

### Step 9 — Register each ClusterSet to fleet-argocd

Each `ClusterSets/<team>/gitopsclusterPolicy.yaml` creates a `ManagedClusterSetBinding` and a team-scoped `Placement` in the `fleet-gitops` namespace. `fleet-argocd` already has cluster secrets for every cluster via the global `fleet-placement` — no new GitOpsCluster is needed. The team Placement is used only as the ApplicationSet generator label selector.

```bash
oc apply -k ./AcmPolicies/ClusterSets/blueclusterset
oc apply -k ./AcmPolicies/ClusterSets/redclusterset
```

Verify Placements and PlacementDecisions exist:

```bash
oc get placement -n fleet-gitops
oc get placementdecision -n fleet-gitops
```

### Step 10 — Deploy team AppProjects and ApplicationSets

Each `ApplicationSets/<team>/` folder contains an AppProject and an ApplicationSet, both in the `fleet-gitops` namespace. The AppProject defines which groups have access, and the ApplicationSet generator uses the team Placement from Step 9.

```bash
oc apply -k ./ApplicationSets/blueclusterset
oc apply -k ./ApplicationSets/redclusterset
```

Verify Applications were generated in `fleet-gitops`:

```bash
oc get applications.argoproj.io -n fleet-gitops
```

Expected:

```
NAME                              SYNC STATUS   HEALTH STATUS
platform-baseline-cluster1        Synced        Healthy
platform-baseline-local-cluster   Synced        Healthy
mobileapp-<blue-cluster>          Synced        Healthy
galaga-<red-cluster>              Synced        Healthy
```

### Step 11 — Verify tenant isolation in fleet-argocd

All users log into the same `fleet-argocd-server` route. AppProjects enforce the scoping — no separate ArgoCD RBAC changes are required per team beyond Step 7.

```bash
oc get route fleet-argocd-server -n fleet-gitops
```

**Access summary — all users log into the same fleet-argocd URL:**

| User | Applications visible in fleet-argocd |
|---|---|
| `acmsre1` (acm-sre-group) | All apps across all clusters (platform + blue + red) |
| `bluesre1` (blue-sre-group) | `mobileapp-*` only — AppProject `blue-team` enforces this |
| `redsre1` (red-sre-group) | `galaga-*` only — AppProject `red-team` enforces this |

---

## RBAC verification

### fleet-argocd — platform admin single pane of glass

`acmsre1` logs into `fleet-argocd-server` and sees all 5 applications across all clusters and both tenant projects:

![fleet-argocd all apps](docs/screenshots/fleet-argocd-all-apps.png)

### blueargocd — blue team isolated view

`bluesre1` logs into `blueargocd-server` and sees only `mobileapp-cluster1` (blue-team project):

![blueargocd tenant view](docs/screenshots/blueargocd-tenant-view.png)

### redargocd — red team isolated view

`redsre1` logs into `redargocd-server` and sees only `galaga-cluster2` (red-team project):

![redargocd tenant view](docs/screenshots/redargocd-tenant-view.png)

---

## References

### Red Hat Advanced Cluster Management (RHACM)

| Topic | Link |
|---|---|
| RHACM documentation home | https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes |
| Governance — ACM Policies | https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.15/html/governance/governance |
| ManagedClusterSet | https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.15/html/clusters/cluster_mce_overview#managedclusterset-intro |
| Placement | https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.15/html/clusters/cluster_mce_overview#placement-intro |
| Registering managed clusters to ArgoCD (GitOpsCluster) | https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.15/html/gitops/gitops-register |

### OpenShift GitOps (ArgoCD)

| Topic | Link |
|---|---|
| OpenShift GitOps documentation home | https://docs.openshift.com/gitops/latest/understanding_openshift_gitops/about-redhat-openshift-gitops.html |
| Installing OpenShift GitOps | https://docs.openshift.com/gitops/latest/installing_gitops/installing-openshift-gitops.html |
| ArgoCD instance (ArgoCD CR) | https://docs.openshift.com/gitops/latest/argocd_instance/setting-up-argocd-instance.html |
| ApplicationSet — Cluster Decision Resource generator | https://docs.openshift.com/gitops/latest/applicationset/applicationset-getting-started.html |

> **Note:** Replace `2.15` in RHACM URLs with your installed version (`oc get csv -n open-cluster-management` to check).
