# Multi-tenancy model example with Red Hat Advanced Cluster Management and OpenShift GitOps operator

This repo provides an exampl to configure cluster-as-a-service multi-tenancy model using Red Hat Advanced Cluster Management and OpenShift GitOps operator.



![image](https://user-images.githubusercontent.com/41969005/159989841-95b5dce8-b678-4cc1-8020-ae9d50a42089.png)


Blue group of users and red group of users share the ACM hub cluster but have access to their own namespaces to manage and view their applications. Each group has separate set of managed clusters where applications are deployed to. ACM group of users has access to both groups' applications and all managed clusters. The application developers pushes application manifests to their Git repos but do not have access to the cluster environment. ACM, blue and red SRE group users can set up GitOps to deploy the applications from Git repos to clusters using OpenShift GitOps ApplicationSets.

## Cluster environment
- ACM hub cluster
- Three managed clusters: `bluecluster1`, `bluecluster2`, `redcluster`


![image](https://user-images.githubusercontent.com/41969005/160040406-bf5e9b3c-ebe9-4e6c-82fa-0fde1071bd03.png)



## Steps to create multi-tenancy GitOps environment for blue, red and ACM user groups

1. Clone this repo.

2. Log into ACM hub cluster via CLI and create blue, red and ACM SRE and viewer groups and users.

```
oc create secret generic htpass-secret --from-file=htpasswd=./UsersGroups/htpasswd -n openshift-config
oc apply -f ./UsersGroups/htpasswd.yaml
oc apply -f ./UsersGroups/users.yaml
oc apply -f ./UsersGroups/groups.yaml
```

![image](https://user-images.githubusercontent.com/41969005/160040354-df18fd29-ff74-463f-b6b1-43045b404e2f.png)


3. Grant ACM groups cluster-wide access.

    a. Grant `acm-sre-group` group admin access cluster-wide. Log into OCP console and go to `User management` `Groups` `acm-sre-group`. Go to `Role binding` tab and create a binding. Type `Cluster-wide role binding`, role binding name `acm-sre-group`, role name `cluster-admin`

    b. Grant `acm-viewer-group` group view access cluster-wide. Log into OCP console and go to `User management` `Groups` `acm-viewer-group`. Go to `Role binding` tab and create a binding. Type `Cluster-wide role binding`, role binding name `acm-viewer-group`, role name `view`

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

```
oc apply -f ./AcmPolicies/InstallGitOpsOperator
```

9. Run the following command to create one ArgoCD server instance for the blue group in `blueargocd` namespace and another instance for the red group in `redargocd` namespace. Wait until all pods are running in `blueargocd` and `redargocd` namespaces. Also check that `applicationset-controller` and `dex-server` pods are running.

```
oc apply -f ./AcmPolicies/ArgoCDInstances
```

**Note**: `Red Hat OpenShift GitOps` operator does not need to be installed on managed clusters because we are going to use `ApplicationSet` from the hub cluster to `push` applications to managed clusters. The ArgoCD server instance running on the hub cluster connects to target remote clusters to deploy applications defined in the `ApplicationSet`.

10. Register `blueclusterset` cluster set to `blueargocd` ArgoCD instance so that ArgoCD can deploy applications to the clusters in `blueclusterset` cluster set. Register `redclusterset` cluster set to `redargocd` ArgoCD instance so that ArgoCD can deploy applications to the clusters in `redclusterset` cluster set.

```
oc apply -f ./AcmPolicies/RegisterClustersToArgoCDInstances
```

These operator installation and instance creations are enforced by ACM governance policies.

![image](https://user-images.githubusercontent.com/41969005/160182359-5abfad36-690b-4293-b72c-5bdffa825cd1.png)


11. All blue applications are in `blueargocd` namespace.

    a. Grant `blue-sre-group` group admin access to `blueargocd` namespace. Log into OCP console and go to `User management` `Groups` `blue-sre-group`. Go to `Role binding` tab and create a binding. Type `Namespace role binding`, role binding name `blue-sre-group`, namespace `blueargocd`, role name `admin`

    b. Grant `blue-viewer-group` group view access to `blueargocd` namespace. Log into OCP console and go to `User management` `Groups` `blue-viewer-group`. Go to `Role binding` tab and create a binding. Type `Namespace role binding`, role binding name `blue-viewer-group`, namespace `blueargocd`, role name `view`

12. All red applications are in `redargocd` namespace.

    a. Grant `red-sre-group` group admin access to `redargocd` namespace. All blue applications are created in this namespace. Log into OCP console and go to `User management` `Groups` `red-sre-group`. Go to `Role binding` tab and create a binding. Type `Namespace role binding`, role binding name `red-sre-group`, namespace `redargocd`, role name `admin`

    b. Grant `red-viewer-group` group view access to `redargocd` namespace. Log into OCP console and go to `User management` `Groups` `red-viewer-group`. Go to `Role binding` tab and create a binding. Type `Namespace role binding`, role binding name `red-viewer-group`, namespace `redargocd`, role name `view`

13. Edit the blue ArgoCD instance's RBAC to grant `blue-sre-group` `acm-sre-group` admin access and `blue-viewer-group` `acm-viewer-group` read-only access.

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

14. Edit the red ArgoCD instance's RBAC to grant `red-sre-group` `acm-sre-group` admin access and `blue-viewer-group` `acm-viewer-group` read-only access.

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

15. Use the following command to find the blue ArgoCD console URL. Since OpenShift OAuth dex is enabled in ArgoCD instance, you can ACM or blue group users defined in OCP can log into this ArgoCD instance.

```
oc get route blueargocd-server -n blueargocd
```

16. Use the following command to find the blue ArgoCD console URL. Since OpenShift OAuth dex is enabled in ArgoCD instance, you can ACM or red group users defined in OCP can log into this ArgoCD instance.

```
oc get route redargocd-server -n redargocd
```

**GitOpsification**: The above steps can be translated to manifest YAMLs with ArgoCD sync-wave and pushed to a Git repository. Then ACM-SRE user can use the default OpenShift GitOps operator (ArgoCD) instance in `openshift-gitops` namespace, which comes from the GitOps operator installation, to deploy the Git repo as an Argo application to the hub cluster.

![image](https://user-images.githubusercontent.com/41969005/160181938-0bf0f746-706f-471e-9cf0-8b1aa762782f.png)


## Managed cluster registration verification in ArgoCD

Log into the blue and red ArgoCD consoles and create application to verify that the managed clusters are listed as application destination.

![image](https://user-images.githubusercontent.com/41969005/160016710-38e17da1-f2b4-4400-86ff-ab0b4ed0bf5c.png)

![image](https://user-images.githubusercontent.com/41969005/160016744-31b055ea-38ea-430d-8dab-3620bd96fd70.png)


## RBAC Verifications

Now everything is set up! Let's try to create some applications with different users and see what happens. The `./ApplicationSets/blueappset.yaml` creates an ApplicationSet that deploys `https://github.com/rokej/BlueApplications/tree/main/mobileApplication` application to those two remote blue clusters.


```
    - clusterDecisionResource:
        configMapRef: acm-placement
        labelSelector:
          matchLabels:
            cluster.open-cluster-management.io/placement: blue-placement
```

This cluster decision section of the application set uses existing `blue-placement` that was created by step 14. When step 14 creates a `Placement` CR, the placement controller evaluates and creates a `PlacementDecision` CR with `cluster.open-cluster-management.io/placement: blue-placement` label. `PlacementDecision` contains a list of selected clusters.


### As a viewer

If `oc apply -f ./ApplicationSets` to try to create application sets to deploy applications, you should see errors like below because viewers have read-only view access to the application namespace `blueargocd` or `redargocd`.

```
Error from server (Forbidden): error when creating "ApplicationSets/blueappset.yaml": applicationsets.argoproj.io is forbidden: User "blueviewer1" cannot create resource "applicationsets" in API group "argoproj.io" in the namespace "blueargocd"
Error from server (Forbidden): error when retrieving current configuration of:
Resource: "argoproj.io/v1alpha1, Resource=applicationsets", GroupVersionKind: "argoproj.io/v1alpha1, Kind=ApplicationSet"
Name: "galaga-application-set", Namespace: "redargocd"
from server for: "ApplicationSets/redappset.yaml": applicationsets.argoproj.io "galaga-application-set" is forbidden: User "blueviewer1" cannot get resource "applicationsets" in API group "argoproj.io" in the namespace "redargocd"
```

### As a blue SRE user

If `oc apply -f ./ApplicationSets` to try to create application sets to deploy applications, you should see output like below because the bluw SRE user has admin access to `blueargocd` applicaiton namespace but no access to `redargocd` application namespace.

```
applicationset.argoproj.io/mobile-application-set created
Error from server (Forbidden): error when retrieving current configuration of:
Resource: "argoproj.io/v1alpha1, Resource=applicationsets", GroupVersionKind: "argoproj.io/v1alpha1, Kind=ApplicationSet"
Name: "galaga-application-set", Namespace: "redargocd"
from server for: "ApplicationSets/redappset.yaml": applicationsets.argoproj.io "galaga-application-set" is forbidden: User "bluesre1" cannot get resource "applicationsets" in API group "argoproj.io" in the namespace "redargocd"
```

### RHACM console multi-tenancy

The following screenshots depict that blue group users can see only blue applications and red group users can see only red applications.

![image](https://user-images.githubusercontent.com/41969005/159999028-df152f14-bc68-4720-8506-7ae2abfc2beb.png)


![image](https://user-images.githubusercontent.com/41969005/159999168-6a3afc06-ed09-4297-9b4b-7f6083d422b2.png)


### Separate ArgoCD console

There are two separate ArgoCD consoles `blueargocd` `redargocd` with RBAC from step 12 and 13 so that one group cannot log into the other group's ArgoCD instance console to view or manage their applications.

![image](https://user-images.githubusercontent.com/41969005/159999747-dd55dda5-52e0-48dc-84ab-25a6f4b4772d.png)

![image](https://user-images.githubusercontent.com/41969005/159999791-988724ca-17f2-452e-97b7-985058fce85c.png)


### Managing Argo applications from RHACM console

You can create, view and edit application sets from RHACM console. Also you can launch into ArgoCD console from RHACM console to managed the application.

![image](https://user-images.githubusercontent.com/41969005/160003338-7a5c7450-27fe-457d-b262-dc63643cfe6d.png)

You can create, view and edit application sets from RHACM console. Also you can launch into ArgoCD console from RHACM console to managed the application.
