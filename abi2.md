```
sudo -i
dnf update -y
dnf -y install python3.9
# set python stuff (path seem to be needed in RHEL8)
ln -s /bin/pip3 /bin/pip
ln -s /bin/python3.9 /bin/python

parted -s -a optimal /dev/nvme0n1 mklabel gpt mkpart primary 0 3841GB
sleep 20
udevadm settle
parted -s -a optimal /dev/nvme1n1 mklabel gpt mkpart primary 0 3841GB 
sleep 20
udevadm settle
mkfs.xfs /dev/nvme0n1p1
mkfs.xfs /dev/nvme1n1p1
X=`lsblk  /dev/nvme0n1p1 -no UUID`
echo "UUID=$X       /var/lib/containers/storage/   xfs     auto 0       0" >> /etc/fstab
sleep 20
Y=`lsblk  /dev/nvme1n1p1 -no UUID`
echo "UUID=$Y       /var/lib/libvirt   xfs     auto 0       0" >> /etc/fstab
mkdir -p /var/lib/containers/storage/
mkdir -p /var/lib/libvirt/
systemctl daemon-reload
mount -av
restorecon -rF /var/lib/containers/storage/
restorecon -rF /var/lib/libvirt/
```


```
###################
# Step#2: Enable Virtualization:
###################
# enable virtualization: 
dnf -y install libvirt libvirt-daemon-driver-qemu qemu-kvm
usermod -aG qemu,libvirt $(id -un)
newgrp libvirt
systemctl enable libvirtd --now
sleep 10
# verify: systemctl status libvirtd
systemctl is-active libvirtd # should show active

###################
# Step#3: KCLI:
###################
dnf -y copr enable karmab/kcli
dnf -y install kcli bash-completion vim jq tar git ipcalc python3-pip

###################
# Step#4: Install Tools:
###################
dnf -y install podman httpd-tools runc wget nmstate containernetworking-plugins bind-utils

###################
# Step#5: Disable Firewall:
###################
systemctl disable firewalld iptables
systemctl stop firewalld iptables
iptables -F
sleep 30
systemctl restart libvirtd

###################
# Step#6: Enable KShushy:
###################
pip3 install cherrypy 
pip install -U pip setuptools 
pip install pyopenssl
kcli create sushy-service --ssl --port 9000 
systemctl enable ksushy --now
sleep 10
systemctl is-active ksushy

###################
# Step#7: Create Local Net:
###################
# Need to do this before restarting NetworkManager, or else DNS fails as there isn't any 192.168.125.1 existing 
# create cluster: 
kcli create pool -p /var/lib/libvirt/images default 
kcli create network -c 192.168.125.0/24 -P dhcp=false --domain tnc.bootcamp.lab tnc

###################
# Step#8: DNS and SSH:
###################

########### set up DNS:

dnf install dnsmasq

cat << EOF > /etc/NetworkManager/conf.d/dnsmasq.conf
[main]
dns=dnsmasq
[connection]
ipv4.dns-priority=200
ipv6.dns-priority=200
EOF

cat << EOF > /etc/NetworkManager/dnsmasq.d/main.conf
# listen-address=192.168.125.1
server=8.8.8.8
domain=tnc.bootcamp.lab
EOF

cat << EOF > /etc/NetworkManager/dnsmasq.d/hub.conf
address=/api.hub.tnc.bootcamp.lab/192.168.125.100
address=/api-int.hub.tnc.bootcamp.lab/192.168.125.100
address=/.apps.hub.tnc.bootcamp.lab/192.168.125.100
address=/quay.tnc.bootcamp.lab/192.168.125.1
EOF

systemctl reload NetworkManager.service
systemctl restart NetworkManager.service
systemctl is-active NetworkManager

########### set up SSH:

ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/id_rsa

###################
# Step#9: Install OC Client & Mirror
###################
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.14.18/openshift-client-linux-4.14.18.tar.gz
tar -xvf openshift-client-linux-4.14.18.tar.gz 
mv oc /usr/bin/
rm -f openshift-client-linux-4.14.18.tar.gz 
oc completion bash > oc.bash_completion
mv oc.bash_completion /etc/bash_completion.d/
wget https://github.com/quay/mirror-registry/releases/download/v1.3.10/mirror-registry-online.tar.gz
tar -xvf mirror-registry-online.tar.gz
chmod u+x mirror-registry
```

## Bring up VM:

```
###################
# Step#10: Bring up VM for mirroring
###################
cat << EOF > vm1.yaml
vm1:
 pool: default
 rootpassword: redhat
 image: centos9stream
 numcpus: 12
 memory: 32000
 files:
 - path: /etc/motd
   content: Welcome to the cruel world
 nets:
 - name: tnc
   nic: eth0
   ip: 192.168.125.10
   mask: 255.255.255.0
   gateway: 192.168.125.1
 cmds:
 - echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
 - ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/id_rsa
 - systemctl restart sshd
EOF
kcli create plan -f vm1.yaml 
```

## Checking Quay DNS entry:
```
nslookup quay.tnc.bootcamp.lab
Server:		127.0.0.1
Address:	127.0.0.1#53

Name:	quay.tnc.bootcamp.lab
Address: 192.168.125.1

ping quay.tnc.bootcamp.lab
PING quay.tnc.bootcamp.lab (192.168.125.1) 56(84) bytes of data.
64 bytes from hypervisor (192.168.125.1): icmp_seq=1 ttl=64 time=0.022 ms
64 bytes from hypervisor (192.168.125.1): icmp_seq=2 ttl=64 time=0.023 ms
```

## Insert pull secret

** Check if necessary ?? ** 

## Starting the Mirror registry:
Full set of options `https://github.com/quay/mirror-registry` 
```
./mirror-registry install --quayHostname quay.tnc.bootcamp.lab --quayRoot /opt/ --initUser quay --initPassword syed@tnc
```

> INFO[2024-03-28 07:34:08] Quay installed successfully, config data is stored in /opt/ <br>
> INFO[2024-03-28 07:34:08] Quay is available at https://quay.tnc.bootcamp.lab:8443 with credentials (quay, syed@tnc) 

## Setting up the VM for mirroring: 

Add credentials to docker auth file: 
```
podman login https://quay.tnc.bootcamp.lab:8443 --tls-verify=false --authfile .docker/config.json 
```

Push this file to VM: 
```
scp .docker/config.json root@192.168.125.10:~/
```
```
ssh root@192.168.125.10
```
```
curl https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.14.18/oc-mirror.tar.gz -o oc-mirror.tar.gz
tar -xvf oc-mirror.tar.gz 
chmod u+x oc-mirror
mkdir ~/.docker
mv config.json ~/.docker/
```
```
cat <<EOF > imageset-config.yaml
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v1alpha2
storageConfig:
  registry:
    imageURL: quay.tnc.bootcamp.lab:8443/ocp/oc-mirror-metadata
    skipTLS: true
mirror:
  platform:
    channels:
    - name: stable-4.14
      type: ocp
      minVersion: 4.14.18
      maxVersion: 4.14.18
  operators:
  - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.14
    packages:
    - name: cluster-logging
      channels:
      - name: stable-5.8
    - name: advanced-cluster-management 
      channels:
      - name: release-2.10 
    - name: local-storage-operator
      channels:
      - name: stable
    - name: topology-aware-lifecycle-manager
      channels:
      - name: stable
    - name: quay-operator
      channels:
      - name: stable-3.11
    - name: openshift-gitops-operator
      channels:
      - name: latest
  additionalImages:
  - name: registry.redhat.io/ubi8/ubi:latest
  helm: {}
EOF
```

```
for X in {"cluster-logging","advanced-cluster-management","local-storage-operator","topology-aware-lifecycle-manager","quay-operator","openshift-gitops-operator"}; do oc mirror list operators --catalog=registry.redhat.io/redhat/redhat-operator-index:v4.14 --package=$X; done
```


## Mirror: 

```
 oc mirror --config=./imageset-config.yaml docker://quay.tnc.bootcamp.lab:8443 --dest-skip-tls=true --source-skip-tls=true;
```

What you want to see at the end of this:  (takes about 10-15 mins)

```
Rendering catalog image "quay.tnc.bootcamp.lab:8443/redhat/redhat-operator-index:v4.14" with file-based catalog 
Writing image mapping to oc-mirror-workspace/results-1712158340/mapping.txt
Writing CatalogSource manifests to oc-mirror-workspace/results-1712158340
Writing ICSP manifests to oc-mirror-workspace/results-1712158340
```

## disconenct the bridge 
```
virsh net-edit  tnc
```
remove forwarding section 

```
virsh net-destroy tnc; virsh net-start tnc
virsh destroy vm1; virsh start vm1
```

```
ssh root@192.168.125.10 -- ping 192.168.125.1 -c 3
root@192.168.125.10's password: 
PING 192.168.125.1 (192.168.125.1) 56(84) bytes of data.
64 bytes from 192.168.125.1: icmp_seq=1 ttl=64 time=0.081 ms
64 bytes from 192.168.125.1: icmp_seq=2 ttl=64 time=0.068 ms
64 bytes from 192.168.125.1: icmp_seq=3 ttl=64 time=0.076 ms

ssh root@192.168.125.10 -- ping 8.8.8.8 -c 3
root@192.168.125.10's password: 
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
From 192.168.125.1 icmp_seq=1 Destination Port Unreachable
From 192.168.125.1 icmp_seq=2 Destination Port Unreachable
From 192.168.125.1 icmp_seq=3 Destination Port Unreachable
```
# Install ABI: 

## Installer: 

```
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.14.18/openshift-install-linux-4.14.18.tar.gz
tar -xvf ./openshift-install-linux-4.14.18.tar.gz
rm -f openshift-install-linux-4.14.18.tar.gz
mv openshift-install /usr/bin/
```

## Create install yamls:



NEW:

```
cat << EOF > agent-config.yaml
apiVersion: v1alpha1
metadata:
  name: hub
rendezvousIP: 192.168.125.100
hosts:
  - hostname: sno
    interfaces:
     - name: net0
       macAddress: 52:54:00:35:bb:80
    networkConfig:
      interfaces:
      - name: net0
        state: up
        ipv4:
          address:
          - ip: 192.168.125.100
            prefix-length: 24
          enabled: true
          dhcp: false
      dns-resolver:
        config:
          server:
            - 192.168.125.1
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.125.1
            table-id: 254
            next-hop-interface: net0
EOF
cat << EOF > install-config.yaml
apiVersion: v1
baseDomain: tnc.bootcamp.lab
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  creationTimestamp: null
  name: hub
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 192.168.125.0/24
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
pullSecret: '{"auths":{"cloud.openshift.com":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K29jbV9hY2Nlc3NfZGE0OWY3MDcwODlhNDU0Y2E1NzdiMzJlMTdmMjExODM6MU82R0tFTUtIMlRVNFAyM1FEQlM2TU9LTkk1MVQ4RzlKTVZZM0dDWkxDSEpLOUUwNzQ4Q0tNOVJWNTlMVDk2Ug==","email":"shassan@redhat.com"},"quay.io":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K29jbV9hY2Nlc3NfZGE0OWY3MDcwODlhNDU0Y2E1NzdiMzJlMTdmMjExODM6MU82R0tFTUtIMlRVNFAyM1FEQlM2TU9LTkk1MVQ4RzlKTVZZM0dDWkxDSEpLOUUwNzQ4Q0tNOVJWNTlMVDk2Ug==","email":"shassan@redhat.com"},"registry.connect.redhat.com":{"auth":"fHVoYy1wb29sLTViYzY4YjVhLWU2NmMtNGMxMS1iZDY4LWVkMGI3NjQyZTJmZjpleUpoYkdjaU9pSlNVelV4TWlKOS5leUp6ZFdJaU9pSTJOakUzWldOak16UmpZakEwT0RZNVlXVXdNR00yWkRVeFl6TTBOVEExTlNKOS5RN0FhdERxTmxCOS1vSFBJa0hjWFZJUnhrMnBIOUtQYW11eFRhVDM3ekVZWUpVWFhoek1sTF9zX3ZhVVBzd2lQcXlWVTR6T1JfT3NrZ19lV3dRaEZsU2h5SkVqeDloaVpsdUQyZ2duNS1ueG1sUk5DcFZ2RTJ0YWVkOERJeVNKZEhGa3FSZ3FBRmJGOVJQS1NQUGtHc1R6cHBfT25xYTMxQmgwX1RxTDJ1S1FES1ZHUmFpaHFBanRmbEhMTUIxb3p6UnRfaWRfbFg0TGhtZjNBRE9VYl9XdHR3cXkyRTdaajA5RWdBTHoxZVU5cDEwNlJiSTJOQWMyVGxnR2k3U3BZcVFTUm85YlB3MVpXZVgwN2RHa1ctdy1Kd1lrRTRESzJBNzVwakdRN2xDcGM1ejVFMXZTYldPU3VXMjBNcW1xREpMeGVxN2xSMDBXaktwa2lPNmVrcURENERPYmhORG1mNjNNejg4M3NKaloyQ2c1c0lwSUVVbzRFcGdobFNZUllyZ1dUR2dCQWJuRXpwOHhOcm1jZktKUmRzb2E0NmtOZkhsdXBBeEZuNUJHZ2pkRVlLc2VLNEVySlZVTW9WZnRKeWc3N1FTN1gxdXZLX2EyOW9TM1ltZnBEWnpsOFRYRWlCOWFTdF9NaU9oZGtILVRBVXo5bThwaUszVm50TFJ5ZnVQZGlDcThDalJHZTVJQ0FQYlNKTHBvRFVPbGNwMjl3RHZ1V19KVDNXMXFOMzZLZ28xczdFemQtUGhHWG0tdTlfeExQc0F6Q3JvT20zZmpwbG5idC11MEpEMlM0ZTdUQjZVQnozNWZaVFJPeHhfcTZuc0tKNG40MFFvbFdWem16ZHJoUHRZM3JwRWJSQ3k1ckJvcWw1YWN0ejBLMmRzdFFKMkMxVnR2dTNRTQ==","email":"shassan@redhat.com"},"registry.redhat.io":{"auth":"fHVoYy1wb29sLTViYzY4YjVhLWU2NmMtNGMxMS1iZDY4LWVkMGI3NjQyZTJmZjpleUpoYkdjaU9pSlNVelV4TWlKOS5leUp6ZFdJaU9pSTJOakUzWldOak16UmpZakEwT0RZNVlXVXdNR00yWkRVeFl6TTBOVEExTlNKOS5RN0FhdERxTmxCOS1vSFBJa0hjWFZJUnhrMnBIOUtQYW11eFRhVDM3ekVZWUpVWFhoek1sTF9zX3ZhVVBzd2lQcXlWVTR6T1JfT3NrZ19lV3dRaEZsU2h5SkVqeDloaVpsdUQyZ2duNS1ueG1sUk5DcFZ2RTJ0YWVkOERJeVNKZEhGa3FSZ3FBRmJGOVJQS1NQUGtHc1R6cHBfT25xYTMxQmgwX1RxTDJ1S1FES1ZHUmFpaHFBanRmbEhMTUIxb3p6UnRfaWRfbFg0TGhtZjNBRE9VYl9XdHR3cXkyRTdaajA5RWdBTHoxZVU5cDEwNlJiSTJOQWMyVGxnR2k3U3BZcVFTUm85YlB3MVpXZVgwN2RHa1ctdy1Kd1lrRTRESzJBNzVwakdRN2xDcGM1ejVFMXZTYldPU3VXMjBNcW1xREpMeGVxN2xSMDBXaktwa2lPNmVrcURENERPYmhORG1mNjNNejg4M3NKaloyQ2c1c0lwSUVVbzRFcGdobFNZUllyZ1dUR2dCQWJuRXpwOHhOcm1jZktKUmRzb2E0NmtOZkhsdXBBeEZuNUJHZ2pkRVlLc2VLNEVySlZVTW9WZnRKeWc3N1FTN1gxdXZLX2EyOW9TM1ltZnBEWnpsOFRYRWlCOWFTdF9NaU9oZGtILVRBVXo5bThwaUszVm50TFJ5ZnVQZGlDcThDalJHZTVJQ0FQYlNKTHBvRFVPbGNwMjl3RHZ1V19KVDNXMXFOMzZLZ28xczdFemQtUGhHWG0tdTlfeExQc0F6Q3JvT20zZmpwbG5idC11MEpEMlM0ZTdUQjZVQnozNWZaVFJPeHhfcTZuc0tKNG40MFFvbFdWem16ZHJoUHRZM3JwRWJSQ3k1ckJvcWw1YWN0ejBLMmRzdFFKMkMxVnR2dTNRTQ==","email":"shassan@redhat.com"}}}'
sshKey: |
EOF
echo -n "  " >> install-config.yaml
cat ~/.ssh/id_rsa.pub >> install-config.yaml
```

OLD:

```
cat << EOF > agent-config.yaml
apiVersion: v1alpha1
metadata:
  name: mgmt				# tag, doesn't seem to mean anything
rendezvousIP: 192.168.125.100
hosts:
  - hostname: sno			# hostname, but not necessarily node name
    interfaces:
     - name: eth0			# enp1s0
       macAddress: 52:54:00:35:bb:80
    networkConfig:
      interfaces:
      - name: eth0
        state: up
        ipv4:
          address:
          - ip: 192.168.125.100
            prefix-length: 24
          enabled: true
          dhcp: false
      dns-resolver:
        config:
          server:
            - 192.168.125.1
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.125.1
            table-id: 254
            next-hop-interface: eth0
EOF
cat << EOF > install-config.yaml
apiVersion: v1
baseDomain: tnc.bootcamp.lab
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  replicas: 1
metadata:
  creationTimestamp: null
  name: hub
networking:
  # dual-stack is supported only on baremetal
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 192.168.125.0/24
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
pullSecret: '{"auths":{"cloud.openshift.com":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K29jbV9hY2Nlc3NfZGE0OWY3MDcwODlhNDU0Y2E1NzdiMzJlMTdmMjExODM6MU82R0tFTUtIMlRVNFAyM1FEQlM2TU9LTkk1MVQ4RzlKTVZZM0dDWkxDSEpLOUUwNzQ4Q0tNOVJWNTlMVDk2Ug==","email":"shassan@redhat.com"},"quay.io":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K29jbV9hY2Nlc3NfZGE0OWY3MDcwODlhNDU0Y2E1NzdiMzJlMTdmMjExODM6MU82R0tFTUtIMlRVNFAyM1FEQlM2TU9LTkk1MVQ4RzlKTVZZM0dDWkxDSEpLOUUwNzQ4Q0tNOVJWNTlMVDk2Ug==","email":"shassan@redhat.com"},"registry.connect.redhat.com":{"auth":"fHVoYy1wb29sLTViYzY4YjVhLWU2NmMtNGMxMS1iZDY4LWVkMGI3NjQyZTJmZjpleUpoYkdjaU9pSlNVelV4TWlKOS5leUp6ZFdJaU9pSTJOakUzWldOak16UmpZakEwT0RZNVlXVXdNR00yWkRVeFl6TTBOVEExTlNKOS5RN0FhdERxTmxCOS1vSFBJa0hjWFZJUnhrMnBIOUtQYW11eFRhVDM3ekVZWUpVWFhoek1sTF9zX3ZhVVBzd2lQcXlWVTR6T1JfT3NrZ19lV3dRaEZsU2h5SkVqeDloaVpsdUQyZ2duNS1ueG1sUk5DcFZ2RTJ0YWVkOERJeVNKZEhGa3FSZ3FBRmJGOVJQS1NQUGtHc1R6cHBfT25xYTMxQmgwX1RxTDJ1S1FES1ZHUmFpaHFBanRmbEhMTUIxb3p6UnRfaWRfbFg0TGhtZjNBRE9VYl9XdHR3cXkyRTdaajA5RWdBTHoxZVU5cDEwNlJiSTJOQWMyVGxnR2k3U3BZcVFTUm85YlB3MVpXZVgwN2RHa1ctdy1Kd1lrRTRESzJBNzVwakdRN2xDcGM1ejVFMXZTYldPU3VXMjBNcW1xREpMeGVxN2xSMDBXaktwa2lPNmVrcURENERPYmhORG1mNjNNejg4M3NKaloyQ2c1c0lwSUVVbzRFcGdobFNZUllyZ1dUR2dCQWJuRXpwOHhOcm1jZktKUmRzb2E0NmtOZkhsdXBBeEZuNUJHZ2pkRVlLc2VLNEVySlZVTW9WZnRKeWc3N1FTN1gxdXZLX2EyOW9TM1ltZnBEWnpsOFRYRWlCOWFTdF9NaU9oZGtILVRBVXo5bThwaUszVm50TFJ5ZnVQZGlDcThDalJHZTVJQ0FQYlNKTHBvRFVPbGNwMjl3RHZ1V19KVDNXMXFOMzZLZ28xczdFemQtUGhHWG0tdTlfeExQc0F6Q3JvT20zZmpwbG5idC11MEpEMlM0ZTdUQjZVQnozNWZaVFJPeHhfcTZuc0tKNG40MFFvbFdWem16ZHJoUHRZM3JwRWJSQ3k1ckJvcWw1YWN0ejBLMmRzdFFKMkMxVnR2dTNRTQ==","email":"shassan@redhat.com"},"registry.redhat.io":{"auth":"fHVoYy1wb29sLTViYzY4YjVhLWU2NmMtNGMxMS1iZDY4LWVkMGI3NjQyZTJmZjpleUpoYkdjaU9pSlNVelV4TWlKOS5leUp6ZFdJaU9pSTJOakUzWldOak16UmpZakEwT0RZNVlXVXdNR00yWkRVeFl6TTBOVEExTlNKOS5RN0FhdERxTmxCOS1vSFBJa0hjWFZJUnhrMnBIOUtQYW11eFRhVDM3ekVZWUpVWFhoek1sTF9zX3ZhVVBzd2lQcXlWVTR6T1JfT3NrZ19lV3dRaEZsU2h5SkVqeDloaVpsdUQyZ2duNS1ueG1sUk5DcFZ2RTJ0YWVkOERJeVNKZEhGa3FSZ3FBRmJGOVJQS1NQUGtHc1R6cHBfT25xYTMxQmgwX1RxTDJ1S1FES1ZHUmFpaHFBanRmbEhMTUIxb3p6UnRfaWRfbFg0TGhtZjNBRE9VYl9XdHR3cXkyRTdaajA5RWdBTHoxZVU5cDEwNlJiSTJOQWMyVGxnR2k3U3BZcVFTUm85YlB3MVpXZVgwN2RHa1ctdy1Kd1lrRTRESzJBNzVwakdRN2xDcGM1ejVFMXZTYldPU3VXMjBNcW1xREpMeGVxN2xSMDBXaktwa2lPNmVrcURENERPYmhORG1mNjNNejg4M3NKaloyQ2c1c0lwSUVVbzRFcGdobFNZUllyZ1dUR2dCQWJuRXpwOHhOcm1jZktKUmRzb2E0NmtOZkhsdXBBeEZuNUJHZ2pkRVlLc2VLNEVySlZVTW9WZnRKeWc3N1FTN1gxdXZLX2EyOW9TM1ltZnBEWnpsOFRYRWlCOWFTdF9NaU9oZGtILVRBVXo5bThwaUszVm50TFJ5ZnVQZGlDcThDalJHZTVJQ0FQYlNKTHBvRFVPbGNwMjl3RHZ1V19KVDNXMXFOMzZLZ28xczdFemQtUGhHWG0tdTlfeExQc0F6Q3JvT20zZmpwbG5idC11MEpEMlM0ZTdUQjZVQnozNWZaVFJPeHhfcTZuc0tKNG40MFFvbFdWem16ZHJoUHRZM3JwRWJSQ3k1ckJvcWw1YWN0ejBLMmRzdFFKMkMxVnR2dTNRTQ==","email":"shassan@redhat.com"}}}'
sshKey: |
EOF
echo -n "  " >> install-config.yaml
cat ~/.ssh/id_rsa.pub >> install-config.yaml
```


### Create VM:

```
cat << EOF > ~/hub.yaml
hub-m1:
 pool: default
 uefi: true
 start: false
 numcpus: 16
 memory: 48000
 disks: 
 - size: 200
 - size: 300
 nets:
 - name: tnc
   nic: eth0
   mac: 52:54:00:35:bc:20
   ip: 192.168.125.20
   mask: 255.255.255.0
   gateway: 192.168.125.1
hub-m2:
 pool: default
 uefi: true
 start: false
 numcpus: 16
 memory: 48000
 disks: 
 - size: 200
 - size: 300
 nets:
 - name: tnc
   nic: eth0
   mac: 52:54:00:35:bc:21
   ip: 192.168.125.21
   mask: 255.255.255.0
   gateway: 192.168.125.1
hub-m3:
 pool: default
 uefi: true
 start: false
 numcpus: 16
 memory: 48000
 disks: 
 - size: 200
 - size: 300
 nets:
 - name: tnc
   nic: eth0
   mac: 52:54:00:35:bc:22
   ip: 192.168.125.22
   mask: 255.255.255.0
   gateway: 192.168.125.1
EOF
```
```
kcli create plan -f ~/hub.yaml 
```
```
kcli list vm
+--------+--------+----------------+---------------+-----------------------+---------+
|  Name  | Status |       Ip       |     Source    |          Plan         | Profile |
+--------+--------+----------------+---------------+-----------------------+---------+
| hub-m1 |  down  | 192.168.125.20 |               |   suspicious-amazigh  |  kvirt  |
| hub-m2 |  down  | 192.168.125.21 |               |   suspicious-amazigh  |  kvirt  |
| hub-m3 |  down  | 192.168.125.22 |               |   suspicious-amazigh  |  kvirt  |
|  vm1   |   up   | 192.168.125.10 | centos9stream | compassionate-goodall |  kvirt  |
+--------+--------+----------------+---------------+-----------------------+---------+
```

### Create ISO:

```
openshift-install agent create image --dir=./ --log-level=debug
```

### Install HTTP Server:

```
cat << EOF > /etc/systemd/system/podman-webcache.service
[Unit]
Description=Podman container - webcache
After=network.target

[Service]
Type=simple
WorkingDirectory=/root
TimeoutStartSec=300
ExecStartPre=-/usr/bin/podman rm -f webcache
ExecStart=/usr/bin/podman run --name webcache --net host -v /opt/webcache/data:/usr/local/apache2/htdocs:z quay.io/alosadag/httpd:p8080 
ExecStop=-/usr/bin/podman rm -f webcache
Restart=always
RestartSec=30s
StartLimitInterval=60s
StartLimitBurst=99

[Install]
WantedBy=multi-user.target
EOF

mkdir -p /opt/webcache/data
systemctl daemon-reload
systemctl enable podman-webcache --now
```

### Copy iso and mount:

```
########## Copy Agent ISO to HTTP:
cp agent.x86_64.iso /opt/webcache/data/

########## Mount the Agent ISO to the VM: 
curl -d '{"Image":"http://127.0.0.1:8080/agent.x86_64.iso","Inserted": true}' -H "Content-Type: application/json" -X POST -k https://127.0.0.1:9000/redfish/v1/Managers/local/hubvm/VirtualMedia/Cd/Actions/VirtualMedia.InsertMedia

```
```
virsh dumpxml hub-m1 | grep source
      <source file='/var/lib/libvirt/images/hub-m1_0.img'/>
      <source file='/var/lib/libvirt/images/hub-m1_1.img'/>
      <source file='/var/lib/libvirt/images/agent.x86_64.iso'/>
      <source network='tnc'/>
```


