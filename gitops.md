# OpenShift GitOps Lab

## Check if GitOps Operator is installed:

## Get Routes:
```
oc get -n openshift-gitops routes openshift-gitops-server
```
Output will be somehting like: 
> NAME                      HOST/PORT                                                             PATH   SERVICES                  PORT    TERMINATION            WILDCARD<br>
> openshift-gitops-server   openshift-gitops-server-openshift-gitops.apps.hub.5g-deployment.lab          openshift-gitops-server   https   passthrough/Redirect   None

## Get Secret: 
```
oc get -n openshift-gitops secrets openshift-gitops-cluster -o jsonpath='{.data.admin\.password}' | base64 -d
```
output will be:
> bWHpNqcBdt6UDXPMQo1Z3xAYGKE8J4mS[root@hypervisor ~]#

## Check GitOps deployments are health:
```
oc get deployment -n openshift-gitops
```

output should show all deployments are running successfully 


