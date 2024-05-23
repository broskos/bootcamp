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
|--------|-------------|
/var/lib/containers | Used to store container imaes and data | 
/var/lib/etcd | Used to store etcd data | 
/var | any data that user may want to keep segregated | 

## Butane File:

```
variant: openshift
version: 4.15.0
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: 98-var-partition
storage:
  disks:
  - device: /dev/disk/by-path/pci-0000:00:06.0
    partitions:
      - number: 1
        should_exist: true
      - number: 2
        should_exist: true
      - number: 3
        should_exist: true
      - number: 4
        resize: true
        size_mib: 460000
      - label: var-lib-containers
        number: 5
        size_mib: 460000
        start_mib: 0
        wipe_partition_entry: true
      - label: var-lib-etcd
        number: 6
        size_mib: 460000
        start_mib: 0
        wipe_partition_entry: true
      - label: var-lib-prometheus-data
        number: 7
        size_mib: 0
        start_mib: 0
        wipe_partition_entry: true
  filesystems:
    - device: /dev/disk/by-partlabel/var-lib-containers
      format: xfs
      mount_options: [defaults, prjquota]
      path: /var/lib/containers
      wipe_filesystem: true
      with_mount_unit: true
    - device: /dev/disk/by-partlabel/var-lib-etcd
      format: xfs
      mount_options: [default, prjquota]
      path: /var/lib/etcd
      wipe_filesystem: true
      with_mount_unit: true
    - device: /dev/disk/by-partlabel/var-lib-prometheus-data
      format: xfs
      path: /var/lib/prometheus/data
      wipe_filesystem: true
      with_mount_unit: true
      mount_options: [default, prjquota]

# total 0
# drwxr-xr-x. 2 root root 200 May 23 18:29 .
# drwxr-xr-x. 8 root root 160 May 23 18:29 ..
# lrwxrwxrwx. 1 root root   9 May 23 18:29 pci-0000:00:04.0-ata-4 -> ../../sr0
# lrwxrwxrwx. 1 root root   9 May 23 18:29 pci-0000:00:04.0-ata-4.0 -> ../../sr0
# lrwxrwxrwx. 1 root root   9 May 23 18:29 pci-0000:00:06.0 -> ../../vda
# lrwxrwxrwx. 1 root root  10 May 23 18:29 pci-0000:00:06.0-part1 -> ../../vda1
# lrwxrwxrwx. 1 root root   9 May 23 18:29 pci-0000:00:07.0 -> ../../vdb
# lrwxrwxrwx. 1 root root   9 May 23 18:29 virtio-pci-0000:00:06.0 -> ../../vda
# lrwxrwxrwx. 1 root root  10 May 23 18:29 virtio-pci-0000:00:06.0-part1 -> ../../vda1
# lrwxrwxrwx. 1 root root   9 May 23 18:29 virtio-pci-0000:00:07.0 -> ../../vdb
# [root@hub ~]# 
# 

```

Create the machine config: 
```
butane ~/partition.bu -o ~/98-var-partition.yaml
```

In the ABI section:
```
mkdir ~/abi 
cp install-config.yaml ~/abi/
cp agent-config.yaml ~/abi/
cd ~/abi
mkdir openshift
cp ../98-var-partition.yaml openshift/
