```
mkdir ~/5g-deployment-lab/ztp-repository/policies/fleet
```
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
> nbsp;sno2 nbsp; nbsp; nbsp; nbsp; nbsp; nbsp; nbsp; nbsp; nbsp; nbsp; nbsp; nbsp;true nbsp; nbsp; nbsp; nbsp; nbsp; nbsp; nbsp; nbsp; nbsp; nbsp; nbsp;https://api.sno2.5g-deployment.lab:6443 nbsp; nbsp; nbsp;True nbsp; nbsp; nbsp; nbsp; nbsp;True nbsp; nbsp; nbsp; nbsp; nbsp; nbsp; nbsp; nbsp;8h nbsp; nbsp; nbsp; nbsp;app.kubernetes.io/instance=<br>
> nbsp;clusters,cluster.open-cluster-management.io/clusterset=default,clusterID=915a5c43-9236-472c-bf07-ce106655fc90,common=ocp414,du-<br> nbsp;
> nbsp;site=sno2du-zone=europe,feature.open-cluster-management.io/addon-cluster-proxy=available,feature.open-cluster-management.io/add<br>
> nbsp;on-config-policy-controller=available,feature.open-cluster-management.io/addon-governance-policy-framework=available,feature.op<br>
> nbsp;en-cluster-management.io/addon-work-manager=available,group-du-sno=,logicalGroup=active,name=sno2,openshiftVersion-major-minor=<br>
> nbsp;4.14,openshiftVersion-major=4,openshiftVersion=4.14.0,ztp-done=<br>

Now looking at policy status: 

```
oc get policy -A
```
NAMESPACE      NAME                                             REMEDIATION ACTION   COMPLIANCE STATE   AGE
sno2           ztp-policies.common-config-policies              inform               Compliant          8h
sno2           ztp-policies.common-subscription-policies        inform               Compliant          8h
sno2           ztp-policies.europe-snos-upgrade-version-414-1   inform               NonCompliant       116s
ztp-policies   common-config-policies                           inform               Compliant          8h
ztp-policies   common-subscription-policies                     inform               Compliant          8h
ztp-policies   europe-snos-upgrade-version-414-1                inform               NonCompliant       116s

Lets check the reason for non-compliance of this policy: 

```
oc get policy -n sno2 ztp-policies.europe-snos-upgrade-version-414-1 -o jsonpath={.status.details} | jq
```

[
  {
    "compliant": "NonCompliant",
    "history": [
      {
        "eventName": "ztp-policies.europe-snos-upgrade-version-414-1.17b7dbafebcf958b",
        "lastTimestamp": "2024-02-27T23:22:12Z",
        "message": "NonCompliant; violation - clusterversions [version] found but not as specified"
      }
    ],
    "templateMeta": {
      "creationTimestamp": null,
      "name": "europe-snos-upgrade-version-414-1-config"
    }
  }
]

This is because the version found on the cluster is not matching 14.4.1. Thats expected. 

