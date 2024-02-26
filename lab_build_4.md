## Install Argo Apps:


### Dump the container: 

```
mkdir -p ~/5g-deployment-lab/ztp-pipeline/
podman login infra.5g-deployment.lab:8443 -u admin -p r3dh4t1! --tls-verify=false
podman run --log-driver=none --rm --tls-verify=false infra.5g-deployment.lab:8443/openshift4/ztp-site-generate-rhel8:v4.14.0-71 extract /home/ztp --tar | tar x -C ~/5g-deployment-lab/ztp-pipeline/
```

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
[root@hypervisor 5g-deployment-lab]# oc apply -k ~/5g-deployment-lab/deployment/
namespace/clusters-sub created
namespace/policies-sub created
clusterrolebinding.rbac.authorization.k8s.io/gitops-cluster created
clusterrolebinding.rbac.authorization.k8s.io/gitops-policy created
appproject.argoproj.io/policy-app-project created
appproject.argoproj.io/ztp-app-project created
application.argoproj.io/clusters created
application.argoproj.io/policies created
[root@hypervisor 5g-deployment-lab]# 
[root@hypervisor 5g-deployment-lab]# oc get applications.argoproj.io -A
NAMESPACE          NAME                       SYNC STATUS   HEALTH STATUS
openshift-gitops   clusters                   OutOfSync     Healthy
openshift-gitops   hub-operators-config       Synced        Healthy
openshift-gitops   hub-operators-deployment   Synced        Healthy
openshift-gitops   policies                   Synced        Healthy
openshift-gitops   sno1-deployment            Synced        Healthy
[root@hypervisor 5g-deployment-lab]# oc get policies -A
NAMESPACE      NAME                           REMEDIATION ACTION   COMPLIANCE STATE   AGE
ztp-policies   common-config-policies         inform                                  17s
ztp-policies   common-subscription-policies   inform                                  17s
[root@hypervisor 5g-deployment-lab]# 

```
