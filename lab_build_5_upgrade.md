# Image Upgrade using TALM:

Start by creating a working directly for the upgrade. Lets call this directory `fleet`

```
mkdir ~/5g-deployment-lab/ztp-repository/policies/fleet
```

## Find image pointers: 

The lab image registry has already been populated with 14.4.1 OpenShift image. You can verify this by looking up the repository: 

podman exec -it registry ls -al /registry/docker/registry/v2/repositories/openshift/release-images/_manifests/tags/

total 16
drwxr-xr-x    4 root     root          4096 Feb 28 16:05 .
drwxr-xr-x    4 root     root          4096 Feb 28 16:01 ..
drwxr-xr-x    4 root     root          4096 Feb 28 16:05 4.14.0-x86_64
drwxr-xr-x    4 root     root          4096 Feb 28 16:01 4.14.1-x86_64


```
cat << EOF > ~/5g-deployment-lab/ztp-repository/policies/fleet/zone-europe-upgrade-414-1.yaml
---
apiVersion: ran.openshift.io/v1
kind: PolicyGenTemplate
metadata:
  name: "europe-snos-upgrade"
  namespace: "ztp-policies"
spec:
  bindingRules:
    du-zone: "europe"
    logicalGroup: "active"
  mcp: "master"
  remediationAction: inform
  sourceFiles:
    - fileName: ClusterVersion.yaml
      policyName: "version-414-1"
      metadata:
        name: version
      spec:
        channel: "stable-4.14"
        desiredUpdate:
          force: false
          version: "4.14.1"
          image: "infra.5g-deployment.lab:8443/openshift/release-images@sha256:05ba8e63f8a76e568afe87f182334504a01d47342b6ad5b4c3ff83a2463018bd"
      status:
        history:
          - version: "4.14.1"
            state: "Completed"
EOF
```
```
cat << EOF > ~/5g-deployment-lab/ztp-repository/policies/fleet/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
generators:
- zone-europe-upgrade-414-1.yaml
EOF
```

```
echo "- fleet" >> ~/5g-deployment-lab/ztp-repository/policies/kustomization.yaml
```

```
cd ~/5g-deployment-lab/ztp-repository/
git add .
git commit -m "adds upgrade policy"
git push
```

### Checking Policy Status for Upgrade:

The newly defined upgrade policy will only apply to SNO2.  This is because the policy matches against prense of one of these labels:
>     du-zone: "europe" <br>
>.    logicalGroup: "active"<br>

But the only SNO2 has these labels assosiated with it, as seen here: 

```
oc get managedcluster --show-labels | grep "du-zone\|logicalGroup"
```
> &nbsp;sno2&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;true&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br>
> &nbsp;&nbsp;https://api.sno2.5g-deployment.lab:6443&nbsp;&nbsp;&nbsp;True&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;True&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br>
> &nbsp;&nbsp;8h&nbsp;&nbsp;&nbsp;&nbsp;app.kubernetes.io/instance=<br>
> &nbsp;clusters,cluster.open-cluster-management.io/clusterset=default,clusterID=915a5c43-9236-472c-bf07-ce106655fc90,common=ocp414,du-<br>&nbsp;
> &nbsp;site=sno2,**du-zone=europe**,feature.open-cluster-management.io/addon-cluster-proxy=available,feature.open-cluster-management.io/add<br>
> &nbsp;on-config-policy-controller=available,feature.open-cluster-management.io/addon-governance-policy-framework=available,feature.op<br>
> &nbsp;en-cluster-management.io/addon-work-manager=available,group-du-sno=,**logicalGroup=active**,name=sno2,openshiftVersion-major-minor=<br>
> &nbsp;4.14,openshiftVersion-major=4,openshiftVersion=4.14.0,ztp-done=<br>

Now looking at policy status: 

```
oc get policy -A
```
> NAMESPACE&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;NAME&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;REMEDIATION&nbsp;ACTION&nbsp;&nbsp;&nbsp;COMPLIANCE&nbsp;STATE&nbsp;&nbsp;&nbsp;AGE<br>   
> sno2&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ztp-policies.common-config-policies&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;inform&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Compliant&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;8h<br>
> sno2&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ztp-policies.common-subscription-policies&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;inform&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Compliant&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;8h<br>
> **sno2&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ztp-policies.europe-snos-upgrade-version-414-1&nbsp;&nbsp;&nbsp;inform&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;NonCompliant&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;116s**<br>
> ztp-policies&nbsp;&nbsp;&nbsp;common-config-policies&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;inform&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Compliant&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;8h<br>
> ztp-policies&nbsp;&nbsp;&nbsp;common-subscription-policies&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;inform&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Compliant&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;8h<br>
> ztp-policies&nbsp;&nbsp;&nbsp;europe-snos-upgrade-version-414-1&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;inform&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;NonCompliant&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;116s<br>


Lets check the reason for non-compliance of this policy: 

```
oc get policy -n sno2 ztp-policies.europe-snos-upgrade-version-414-1 -o jsonpath={.status.details} | jq
```

> [<br>
> &nbsp;&nbsp;{<br>
> &nbsp;&nbsp;&nbsp;&nbsp;"compliant":&nbsp;"NonCompliant",<br>
> &nbsp;&nbsp;&nbsp;&nbsp;"history":&nbsp;[<br>
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;{<br>
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"eventName":&nbsp;"ztp-policies.europe-snos-upgrade-version-414-1.17b7dbafebcf958b",<br>  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"lastTimestamp":&nbsp;"2024-02-27T23:22:12Z",<br>
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**"message":&nbsp;"NonCompliant;&nbsp;violation&nbsp;-&nbsp;clusterversions&nbsp;[version]&nbsp;found&nbsp;but&nbsp;not&nbsp;as&nbsp;specified"**<br>
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>
> &nbsp;&nbsp;&nbsp;&nbsp;],<br>
> &nbsp;&nbsp;&nbsp;&nbsp;"templateMeta":&nbsp;{<br>
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"creationTimestamp":&nbsp;null,<br>
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"name":&nbsp;"europe-snos-upgrade-version-414-1-config"<br>
> &nbsp;&nbsp;&nbsp;&nbsp;}<br>
> &nbsp;&nbsp;}<br>
> ]<br>

This is because the version found on the cluster is not matching 14.4.1. Thats expected. 

This nonciiance is also visible on the GUI: 

![image1](upgrade_1.png)

### Include SNO1 for upgrade

Add one (or all) of the lables to sno1 as well, so its also marked for upgrade: 

```
oc label managedcluster sno1 logicalGroup=active
```
> managedcluster.cluster.open-cluster-management.io/sno1 labeled

You can now view the policies, and find SNO1 included in the list: 

```
```oc get policy -A 
```
> NAMESPACE&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;NAME&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;REMEDIATION&nbsp;ACTION&nbsp;&nbsp;&nbsp;COMPLIANCE&nbsp;STATE&nbsp;&nbsp;&nbsp;AGE<br>   
> sno1&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ztp-policies.europe-snos-upgrade-version-414-1&nbsp;&nbsp;&nbsp;inform&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2m1s<br>
> sno2&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ztp-policies.common-config-policies&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;inform&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Compliant&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;12h<br>
> sno2&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ztp-policies.common-subscription-policies&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;inform&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Compliant&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;12h<br>
> sno2&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ztp-policies.europe-snos-upgrade-version-414-1&nbsp;&nbsp;&nbsp;inform&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;NonCompliant&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3h43m<br>
> ztp-policies&nbsp;&nbsp;&nbsp;common-config-policies&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;inform&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Compliant&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;12h<br>
> ztp-policies&nbsp;&nbsp;&nbsp;common-subscription-policies&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;inform&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Compliant&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;12h<br>
> ztp-policies&nbsp;&nbsp;&nbsp;europe-snos-upgrade-version-414-1&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;inform&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;NonCompliant&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3h43m<br>

### Apply Precaching and CGU:

[root@hypervisor ~]# oc get cgu -A
NAMESPACE      NAME                 AGE    STATE        DETAILS
ztp-install    local-cluster        13h    Completed    All clusters already compliant with the specified managed policies
ztp-install    sno1                 4h1m   Completed    All clusters already compliant with the specified managed policies
ztp-install    sno2                 12h    Completed    All clusters are compliant with all the managed policies
ztp-policies   update-europe-snos   3s     InProgress   Precaching in progress for 2 clusters

> [NOTE] Pre-cache job can take up to 5m to be created.

