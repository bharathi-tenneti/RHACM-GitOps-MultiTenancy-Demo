# Multi-tenancy model example with Red Hat Advanced Cluster Management and OpenShift GitOps operator


![image](https://user-images.githubusercontent.com/41969005/159989841-95b5dce8-b678-4cc1-8020-ae9d50a42089.png)


## Cluster environment
- ACM hub cluster
- Three managed clusters: `bluecluster1`, `bluecluster2`, `redcluster`

## Steps to create multi-tenancy GitOps environment for blue, red and ACM user groups

1. Clone this repo.

2. Log into ACM hub cluster via CLI and create blue, red and ACM SRE and viewer groups and users.

```
    oc create secret generic htpass-secret --from-file=htpasswd=./UsersGroups/htpasswd -n openshift-config
    oc apply -f ./UsersGroups/htpasswd.yaml
    oc apply -f ./UsersGroups/users.yaml
    oc apply -f ./UsersGroups/groups.yaml
```

3. Log into ACM console and create `blueclusterset` cluster set. Add `bluecluster1` and `bluecluster2` clusters to the cluster set.

4. Create `redclusterset` cluster set. Add `redcluster` cluster to the cluster set.

5. In `blueclusterset` cluster set, go to `Access management` tab.

    a. Add `blue-sre-group` group with `Cluster set admin` role. This grants `blue-sre-group` group admin access to `bluecluster1` and `bluecluster2` managed cluster namespaces on ACM hub. This also allows the group admin access to all resources that ACM finds from the remote managed clusters.
    b. Add `blue-viewer-group` group with `Cluster set view` role. This grants `blue-viewer-group` group view access to `bluecluster1` and `bluecluster2` managed cluster namespaces on ACM hub. This also allows the group view access to all resources that ACM finds from the remote managed clusters.

6. In `redclusterset` cluster set, go to `Access management` tab.

    a. Add `red-sre-group` group with `Cluster set admin` role. This grants `red-sre-group` group admin access to `redcluster` managed cluster namespace on ACM hub. This also allows the group admin access to all resources that ACM finds from the remote managed cluster.
    b. Add `red-viewer-group` group with `Cluster set view` role. This grants `red-viewer-group` group view access to `redcluster` managed cluster namespace on ACM hub. This also allows the group view access to all resources that ACM finds from the remote managed cluster.

7. Install `Red Hat OpenShift GitOps` operator and wait until all pods in `openshift-gitops` namespace are running.

```
    oc apply -f ./InstallGitOpsOperator
```

8. Run the following command to create one ArgoCD server instance for the blue group in `blueargocd` namespace and another instance for the red group in `redargocd` namespace. Wait until all pods are running in `blueargocd` and `redargocd` namespaces. Also check that `applicationset-controller` and `dex-server` pods are running.

```
    oc apply -f ./ArgoCDInstances
```

9. All blue applications are in `blueargocd` namespace.

    a. Grant `blue-sre-group` group admin access to `blueargocd` namespace. Log into OCP console and go to `User management` `Groups` `blue-sre-group`. Go to `Role binding` tab and create a binding. Type `Namespace role binding`, role binding name `blue-sre-group`, namespace `blueargocd`, role name `admin`

    b. Grant `blue-viewer-group` group view access to `blueargocd` namespace. Log into OCP console and go to `User management` `Groups` `blue-viewer-group`. Go to `Role binding` tab and create a binding. Type `Namespace role binding`, role binding name `blue-viewer-group`, namespace `blueargocd`, role name `view`

10. All red applications are in `redargocd` namespace.

    a. Grant `red-sre-group` group admin access to `redargocd` namespace. All blue applications are created in this namespace. Log into OCP console and go to `User management` `Groups` `red-sre-group`. Go to `Role binding` tab and create a binding. Type `Namespace role binding`, role binding name `red-sre-group`, namespace `redargocd`, role name `admin`

    b. Grant `red-viewer-group` group view access to `redargocd` namespace. Log into OCP console and go to `User management` `Groups` `red-viewer-group`. Go to `Role binding` tab and create a binding. Type `Namespace role binding`, role binding name `red-viewer-group`, namespace `redargocd`, role name `view`

11. Grant ACM groups cluster-wide access.

    a. Grant `acm-sre-group` group admin access cluster-wide. Log into OCP console and go to `User management` `Groups` `acm-sre-group`. Go to `Role binding` tab and create a binding. Type `Cluster-wide role binding`, role binding name `acm-sre-group`, role name `admin`

    b. Grant `acm-viewer-group` group view access cluster-wide. Log into OCP console and go to `User management` `Groups` `acm-viewer-group`. Go to `Role binding` tab and create a binding. Type `Cluster-wide role binding`, role binding name `acm-viewer-group`, role name `view`

12. Edit the blue ArgoCD instance's RBAC to grant `blue-sre-group` `acm-sre-group` admin access and `blue-viewer-group` `acm-viewer-group` read-only access.

```
    oc edit configmap argocd-rbac-cm -n blueargocd
```


```
data:
  policy.csv: |
    g, acm-sre-group, role:admin
    g, acm-viewer-group, role:readonly
    g, blue-sre-group, role:admin
    g, blue-viewer-group, role:readonly
  policy.default: role:readonly
  scopes: '[groups]'
```

13. Edit the red ArgoCD instance's RBAC to grant `red-sre-group` `acm-sre-group` admin access and `blue-viewer-group` `acm-viewer-group` read-only access.

```
    oc edit configmap argocd-rbac-cm -n redargocd
```

```
data:
  policy.csv: |
    g, acm-sre-group, role:admin
    g, acm-viewer-group, role:readonly
    g, red-sre-group, role:admin
    g, red-viewer-group, role:readonly
  policy.default: role:readonly
  scopes: '[groups]'
```

14. Register `blueclusterset` cluster set to `blueargocd` ArgoCD instance so that ArgoCD can deploy applications to the clusters in `blueclusterset` cluster set. Register `redclusterset` cluster set to `redargocd` ArgoCD instance so that ArgoCD can deploy applications to the clusters in `redclusterset` cluster set.

```
    oc apply -f ./RegisterClustersToArgoCDInstances
```

15. Use the following command to find the blue ArgoCD console URL.

```
    oc get route blueargocd-server -n blueargocd
```

16. Use the following command to find the blue ArgoCD console URL.

```
    oc get route redargocd-server -n redargocd
```
