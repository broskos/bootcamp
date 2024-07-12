
## Complete validated patterns subscription:

Validated Pattern operator subscription was already included in the installation iso file
However, the pattern will not install yet since it doesn't support disconneced environment.  To work around that, we can tempoararily enable proxy configuration on our cluster. 

Prior to do that, lets install proxy server on the host. Exit from bastion (sing "CTRL+]") console, and use the following command at the `[root@hypervisor ~]#` prompt:

```
dnf install -y squid
systemctl start squid
systemctl enable squid
```
While on the host system, note down the IP address it uses, using `ip addr show | grep "bond0$"` . For example: 
```
ip addr show | grep "bond0$"
    inet 147.28.185.223/31 scope global noprefixroute bond0
```

Reconnect to basion using : `virsh console bastion`

Now configure proxy on the cluster using: 
```
oc edit proxy cluster
```
Add the following lines in the .spec section: 

```
status:
  httpProxy: http://147.28.185.223:3128
  httpsProxy: http://147.28.185.223:3128
  noProxy: .cluster.local,.svc,10.128.0.0/14,127.0.0.1,172.30.0.0/16,192.168.125.0/24,api-int.hub.tnc.bootcamp.lab,fd02::/48,localhost,tnc.bootcamp.lab
```

Verify if the patterns operator is properly installed, using `oc get csv` , for example: 

```
oc get csv
NAME                        DISPLAY                       VERSION   REPLACES                    PHASE
patterns-operator.v0.0.52   Validated Patterns Operator   0.0.52    patterns-operator.v0.0.51   Succeeded
```

Note, if the CSV phase shows as "Failed" , then you can remove and readd it by using: 
```
oc delete -f ~/vpattern.yaml
oc delete csv -n openshift-operators patterns-operator.v0.0.52
sleep 10
oc apply -f ~/vpattern.yaml
```

The CSV should get recreated and reach "Succeed" state

The proxy configurion from the cluster can now be removed using `oc edit proxy cluster` and removing the three lines added earlier in this section. 


## Disable default catalog sources: 

```
cat << EOF | oc apply -f -
apiVersion: config.openshift.io/v1
kind: OperatorHub
metadata:
  name: cluster
spec: 
  disableAllDefaultSources: true
EOF
```

## add new catalog source: 

```
for cs_filename in ~/oc-mirror-workspace/result*/catalog*.yaml; do oc apply -f $cs_filename; done
```

Check that the new catalog source is available using: 

`oc get catalogsource -A`

## Check which operators are available using the new catalog source:

Try the following: 
```
oc get packagemanifests.packages.operators.coreos.com
NAME                               CATALOG   AGE
multicluster-engine                          89s
quay-operator                                89s
local-storage-operator                       89s
topology-aware-lifecycle-manager             89s
cluster-logging                              89s
advanced-cluster-management                  89s
openshift-gitops-operator                    89s
patterns-operator                            89s
```

## Using the validated patterns operator: 


To start using the validated pattern operator, lets first create a namespace for it to use: 

```
cat << EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  labels:
    openshift.io/cluster-monitoring: "true"
  name: mgmt-gitops-hub
EOF
```

Now, create secret CR that will be used by the operator's instance to login to Git using the provided credentials: 

```
cat << EOF | oc apply -f -
kind: Secret
apiVersion: v1
metadata:
  name: private-git-repo-creds
  namespace: mgmt-gitops-hub
  labels:
    argocd.argoproj.io/secret-type: repository
data:
  password: aGFzc2Fu
  type: Z2l0
  url: aHR0cDovL2dpdC50bmMuYm9vdGNhbXAubGFiOjMwMDAvc3llZC92cGF0dGVybi5naXQ=
  username: c3llZA==
type: Opaque
EOF
```

Finally, now create the Pattern CR, which will enable this operator to start using Git as the source for the content it should install on the clsuter: 


```
cat << EOF | oc apply -f - 
apiVersion: gitops.hybrid-cloud-patterns.io/v1alpha1
kind: Pattern
metadata:
  labels:
    app.kubernetes.io/managed-by: Helm
  name: mgmt-gitops
  namespace: openshift-operators
spec:
  clusterGroupName: hub
  gitOpsSpec:
    manualSync: true
    operatorChannel: latest
    operatorSource: cs-redhat-operator-index
  gitSpec:
    targetRepo: http://git.tnc.bootcamp.lab:3000/syed/vpattern.git
    targetRevision: main
    tokenSecret: private-git-repo-creds
    tokenSecretNamespace: mgmt-gitops-hub
EOF
```


<!--

## Check the catalog source for version information: 
Use the follwing to find out the version available:
```
oc get packagemanifest advanced-cluster-management -o=jsonpath='{.status.channels[*].name}{"\n"}'
```
Output will show you:

> release-2.10 release-2.9

## ACM:

```
apiVersion: project.openshift.io/v1
kind: Project
metadata:
  name: open-cluster-management
  labels:
    kubernetes.io/metadata.name: open-cluster-management
spec: {}
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: acm
  namespace: open-cluster-management
spec:
  targetNamespaces:
  - open-cluster-management
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: acm-operator-subscription
  namespace: open-cluster-management
spec:
  sourceNamespace: openshift-marketplace
  source: cs-redhat-operator-index
  channel: release-2.9
  installPlanApproval: Automatic
  name: advanced-cluster-management
```

## MCE:

(See: https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.5/html/install/installing#installing-in-a-disconnected-environment)
```
apiVersion: operator.open-cluster-management.io/v1
kind: MultiClusterHub
metadata:
  name: multiclusterhub
  namespace: open-cluster-management
  annotations:
    installer.open-cluster-management.io/mce-subscription-spec: '{"source": "cs-redhat-operator-index"}'
spec: {}
```


## Logging: 

```
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-logging
  annotations:
    openshift.io/node-selector: ""
  labels:
    openshift.io/cluster-monitoring: "true"
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: cluster-logging
  namespace: openshift-logging
spec:
  targetNamespaces:
  - openshift-logging
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cluster-logging
  namespace: openshift-logging
spec:
  channel: "stable-5.9"
  name: cluster-logging
  source: cs-redhat-operator-index
  sourceNamespace: openshift-marketplace
---
apiVersion: "logging.openshift.io/v1"
kind: "ClusterLogging"
metadata:
  name: "instance"
  namespace: "openshift-logging"   
spec:
  managementState: "Managed"
  collection:
    logs:
      type: "vector"
      fluentd: {}
```

## TALM: 

```
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-topology-aware-lifecycle-manager-subscription
  namespace: openshift-operators
spec:
  channel: "stable"
  name: topology-aware-lifecycle-manager
  source: cs-redhat-operator-index
  sourceNamespace: openshift-marketplace
```

## GITOPS
```
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: latest 
  installPlanApproval: Automatic
  name: openshift-gitops-operator 
  source: cs-redhat-operator-index 
  sourceNamespace: openshift-marketplace 
```


For VNC: https://www.ibm.com/support/pages/how-configure-vnc-server-red-hat-enterprise-linux-8
For releease info: `oc adm release info --insecure=true --idms-file=/root/oc-mirror-workspace/results-1714073520/imageContentSourcePolicy.yaml`
-->
