## Install Butane: 

On Bastion, before disconnecting: 
```
curl https://mirror.openshift.com/pub/openshift-v4/clients/butane/latest/butane --output butane
mv butane /usr/bin/
chmod u+x /usr/bin/butane
```

Note:  OpenShift Container Platform supports the addition of a single partition to attach storage to either the /var directory or a subdirectory of /var. For example:

Important Directories: 

| location | purpose | 
========================
| /var/lib/containers | Used to store container imaes and data | 
| /var/lib/etcd | Used to store etcd data | 
| /var | any data that user may want to keep segregated | 

