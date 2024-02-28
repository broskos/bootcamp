# To Do:
- add verification commands showing details of policies (as they are being applied)
- add GUI view of completed installation and policy application completed (including ztp-done label)

## Install Argo Apps:


### Create the Projects manifests:
```
mkdir ~/5g-deployment-lab/deployment
```
```
cat << EOF > ~/5g-deployment-lab/deployment/cluster-app.yaml 
apiVersion: v1
kind: Namespace
metadata:
  name: clusters-sub
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: clusters
  namespace: openshift-gitops
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: clusters-sub
  project: ztp-app-project
  source:
    path: clusters
    repoURL: http://infra.5g-deployment.lab:3000/student/ztp-repository.git
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF
```

```
cat << EOF > ~/5g-deployment-lab/deployment/policy-app.yaml
apiVersion: v1
kind: Namespace
metadata:
    name: policies-sub
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: policies
  namespace: openshift-gitops
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: policies-sub
  project: policy-app-project
  source:
    path: policies
    repoURL: http://infra.5g-deployment.lab:3000/student/ztp-repository.git
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF
```
```
cat << EOF > ~/5g-deployment-lab/deployment/app-project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: ztp-app-project
  namespace: openshift-gitops
spec:
  clusterResourceWhitelist:
  - group: 'hive.openshift.io'
    kind: ClusterImageSet
  - group: 'cluster.open-cluster-management.io'
    kind: ManagedCluster
  - group: ''
    kind: Namespace
  destinations:
  - namespace: '*'
    server: '*'
  namespaceResourceWhitelist:
  - group: ''
    kind: ConfigMap
  - group: ''
    kind: Namespace
  - group: ''
    kind: Secret
  - group: 'agent-install.openshift.io'
    kind: InfraEnv
  - group: 'agent-install.openshift.io'
    kind: NMStateConfig
  - group: 'extensions.hive.openshift.io'
    kind: AgentClusterInstall
  - group: 'hive.openshift.io'
    kind: ClusterDeployment
  - group: 'metal3.io'
    kind: BareMetalHost
  - group: 'metal3.io'
    kind: HostFirmwareSettings
  - group: 'agent.open-cluster-management.io'
    kind: KlusterletAddonConfig
  - group: 'cluster.open-cluster-management.io'
    kind: ManagedCluster
  - group: 'ran.openshift.io'
    kind: SiteConfig
  sourceRepos:
  - '*'
EOF
```

```
cat << EOF > ~/5g-deployment-lab/deployment/policies-app-project.yaml
---
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: policy-app-project
  namespace: openshift-gitops
spec:
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
  - group: hive.openshift.io
    kind: ClusterImageSet
  destinations:
  - namespace: 'ztp*'
    server: '*'
  - namespace: 'policies-sub'
    server: '*'
  namespaceResourceWhitelist:
  - group: ''
    kind: ConfigMap
  - group: ''
    kind: Namespace
  - group: 'apps.open-cluster-management.io'
    kind: PlacementRule
  - group: 'policy.open-cluster-management.io'
    kind: Policy
  - group: 'policy.open-cluster-management.io'
    kind: PlacementBinding
  - group: 'ran.openshift.io'
    kind: PolicyGenTemplate
  - group: cluster.open-cluster-management.io
    kind: Placement
  - group: policy.open-cluster-management.io
    kind: PolicyGenerator
  - group: policy.open-cluster-management.io
    kind: PolicySet
  - group: cluster.open-cluster-management.io
    kind: ManagedClusterSetBinding
  sourceRepos:
  - '*'
EOF
```

```
cat << EOF > ~/5g-deployment-lab/deployment/gitops-policy-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gitops-policy
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: open-cluster-management:cluster-manager-admin
subjects:
- kind: ServiceAccount
  name: openshift-gitops-argocd-application-controller
  namespace: openshift-gitops
EOF
```

```
cat << EOF > ~/5g-deployment-lab/deployment/gitops-cluster-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gitops-cluster
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: openshift-gitops-argocd-application-controller
  namespace: openshift-gitops
EOF
```

```
cat << EOF > ~/5g-deployment-lab/deployment/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - app-project.yaml
  - policies-app-project.yaml
  - gitops-policy-rolebinding.yaml
  - gitops-cluster-rolebinding.yaml
  - cluster-app.yaml
  - policy-app.yaml
EOF
```

### Verify:

```
ls ~/5g-deployment-lab/deployment/
```

> app-project.yaml&nbsp;&nbsp;gitops-cluster-rolebinding.yaml&nbsp;&nbsp;kustomization.yaml&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;policy-app.yaml<br>
> cluster-app.yaml&nbsp;&nbsp;gitops-policy-rolebinding.yaml&nbsp;&nbsp;&nbsp;policies-app-project.yaml<br>

### Run the applicaitons: 

```
oc apply -k ~/5g-deployment-lab/deployment/
```
> namespace/clusters-sub created<br>
> namespace/policies-sub created<br>
> clusterrolebinding.rbac.authorization.k8s.io/gitops-cluster created<br>
> clusterrolebinding.rbac.authorization.k8s.io/gitops-policy created<br>
> appproject.argoproj.io/policy-app-project created<br>
> appproject.argoproj.io/ztp-app-project created<br>
> application.argoproj.io/clusters created<br>
> application.argoproj.io/policies created<br>

## Verification: 

### Verify the status of application:

```
oc get applications.argoproj.io -A
```

NAMESPACE          NAME                       SYNC STATUS   HEALTH STATUS
openshift-gitops   clusters                   OutOfSync     Healthy
openshift-gitops   hub-operators-config       Synced        Healthy
openshift-gitops   hub-operators-deployment   Synced        Healthy
openshift-gitops   policies                   Synced        Healthy
openshift-gitops   sno1-deployment            Synced        Healthy

### Verify the status of policies:

```
oc get policies -A
```

> NAMESPACE&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;NAME&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;REMEDIATION&nbsp;ACTION&nbsp;&nbsp;&nbsp;COMPLIANCE&nbsp;STATE&nbsp;&nbsp;&nbsp;AGE<br>
> ztp-policies&nbsp;&nbsp;&nbsp;common-config-policies&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;inform&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;17s<br>
> ztp-policies&nbsp;&nbsp;&nbsp;common-subscription-policies&nbsp;&nbsp;&nbsp;inform&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;17s<br>
 
### Continuous Monitoring:

```
while true; do echo "############## echo -n `date` ##############"; echo "---------------- BMH ----------------"; oc get bmh -A; echo "---------------- AgentClusterInstall ----------------"; oc get agentclusterinstall -A; echo "---------------- Clusters ----------------"; oc get managedclusters sno2 --show-labels; echo "---------------- Policies ----------------"; oc get policy -A; echo "---------------- CGU ----------------"; oc get cgu -A; sleep 15; done
```
### After cluster is deployed: 

```
export cluster=sno2
oc get secret -n $cluster $cluster-admin-kubeconfig -o jsonpath='{.data.kubeconfig}' | base64 -d > sno2-kubeconfig
```
```
oc get nodes --kubeconfig ./sno2-kubeconfig 
```
NAME                      STATUS   ROLES                         AGE     VERSION
sno2.5g-deployment.lab    Ready    control-plane,master,worker   5h57m   v1.27.6+f67aeb3
