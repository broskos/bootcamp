[TOC]

# Installing a Disconnected Cluster using Agent Based Installation method: 

This lab guide uses a bare metal server to demonstrate the deployment of a fully disconnected clsuter using Agent Based Installer (ABI) methodology. 
The lab involves the following steps. 
1. Parepare the host environemnt. This installs the applications needed, and disk partitions for performing the main lab tasks. 
2. Create the networking environment for the environment
3. Create the Vritual Machines to emulate Bastion and Cluster Node(s)
- Installing the mirror-registry 
- Mirroring Openshift image and operators to the local image registry 
- Genering an ISO file and booting the OpenShift Node(s) with the ISO

## Prepare the host server: 
The lab uses m3.large.x86 metal server from Equinix. This can be requested through [RHDP](https://demo.redhat.com/catalog?search=equinix+metal+baremetal+blank)
Use the following to set up the server: 

```
###################
# Step#1: Basic Config, Add-ons and Disk Partitiions:
###################
sudo -i
dnf update -y
# install python and cockpit (for VM console later)
dnf -y install python3.9 cockpit cockpit-machines
systemctl enable --now cockpit.socket
# set python stuff (path needed in RHEL8)
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
sleep 20
########### set up SSH:
ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/id_rsa
sleep 10
###################
# Step#2: Enable Virtualization:
###################
# enable virtualization: 
dnf -y install libvirt libvirt-daemon-driver-qemu qemu-kvm
usermod -aG qemu,libvirt $(id -un)
newgrp libvirt
systemctl enable libvirtd --now
sleep 10
# verify: systemctl status libvirtd. Output should show that its active. 
systemctl is-active libvirtd # should show active
sleep 10
#
###################
# Step#3: Install KCLI Tool to manage virtual environment
###################
dnf -y copr enable karmab/kcli
dnf -y install kcli bash-completion vim jq tar git ipcalc python3-pip
#
###################
# Step#4: Install other tools:
###################
dnf -y install podman httpd-tools runc wget nmstate containernetworking-plugins bind-utils
#
###################
# Step#5: Disable Firewall:
###################
systemctl disable firewalld iptables
systemctl stop firewalld iptables
iptables -F
sleep 30
systemctl restart libvirtd
```

### Installing Ksushy for Redfish API usage: 

The basic tools are installed at this point. Next we will install the Ksushy tool, as it allows use of Redfish APIs to manage the state of the VMs. This will primarily be used to mount the bootable media using Redfish API calls later. 

```
###################
# Step#6: Enable KShushy, to use Redfish API for VM management:
###################
pip3 install cherrypy 
pip install -U pip setuptools 
pip install pyopenssl
kcli create sushy-service --ssl --port 9000 
systemctl enable ksushy --now
sleep 10
systemctl is-active ksushy
```
## Creating Networking:

### Creating Virtual Network Bridges: 
The virual environment will comprise a virtual machine that will act as our jumphost server (or Bastion, as its commonly called), and one or more VMs that will be used to install the OpenShift Cluster. 

The VM(s) for OpenShift cluster have to be fully disconencted from the internet. Hence they will require their own network. In a real world, that may ba L2/L3 VPN (typically L2 vpn, as its same subnet). In the lab, we will emulate this by creating a virtal bridge called `tnc` 

The Bastion VM will also need connectivity to this L2 bridge, to be able to communicate with the server, make API calls etc. But the Bastion server will also need access to the internet to be able to download installation images etc. So it will connect to `tnc` bridge, as well as another bridge called `tnc-connected` that can reach internet by using NAT. 

```
###################
# Step#7: Create Local Net:
###################
# Need to do this before restarting NetworkManager, or else DNS fails as there isn't any 192.168.125.1 existing
# create cluster:
kcli create pool -p /var/lib/libvirt/images default
kcli create network -c 192.168.125.0/24 -P forward_mode=route -P dhcp=false --domain tnc.bootcamp.lab tnc
kcli create network -c 192.168.126.0/24 -P dhcp=false --domain tnc.bootcamp.lab tnc-connected
```

### Creating DNS Service: 

DNS service is a pre-requisite for cluster deployment and usage. The service can run anywhere that is reachable by the cluster (as well as by Bastion). The best place to set up this service in the lab environment is on the host operating system. This is done using the following: 

```
###################
# Step#8: DNS:
###################
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
# Check if service is active: 
systemctl is-active NetworkManager
```

Note that the DNS records for the "hub" SNO cluster have been created as well. 

### Install Openshift Client:

Even though we will primarily use Bastion for making API calls and run OC commands, it doesn't hurt to install this tool on the host as well. This is optional:

```
###################
# Step#9: Install OC Client 
###################
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.14.18/openshift-client-linux-4.14.18.tar.gz
tar -xvf openshift-client-linux-4.14.18.tar.gz 
mv oc /usr/bin/
rm -f openshift-client-linux-4.14.18.tar.gz 
oc completion bash > oc.bash_completion
mv oc.bash_completion /etc/bash_completion.d/
```

## Create and configure needed VMs: 

### Configure and Bringup Bastion VM:

```
###################
# Step#10: Configure, and Bring up VM for mirroring
###################
cat << EOF > bastion.yaml
bastion:
 pool: default
 rootpassword: redhat
 image: centos9stream
 numcpus: 6
 memory: 16000
 files:
 - path: /etc/motd
   content: Welcome to the cruel world
 nets:
 - name: tnc
   nic: eth0
   ip: 192.168.125.10
   mask: 255.255.255.0
   gateway: 192.168.125.1
 - name: tnc-connected
   nic: eth1
   ip: 192.168.126.10
   mask: 255.255.255.0
   gateway: 192.168.126.1
 cmds:
 - echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
 - ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/id_rsa
 - systemctl restart sshd
 - dnf install -y podman bind-utils nmstate httpd
EOF
```

Start the VM:

```
kcli create plan -f bastion.yaml
```

Verify if the VM has been created and is up:

```
kcli list vm
+---------+--------+----------------+---------------+----------------+---------+
|   Name  | Status |       Ip       |     Source    |      Plan      | Profile |
+---------+--------+----------------+---------------+----------------+---------+
| bastion |   up   | 192.168.125.10 | centos9stream | romantic-goiko |  kvirt  |
+---------+--------+----------------+---------------+----------------+---------+
```

### Configure Password-less SSH for Bastion access:
```
ssh-copy-id root@192.168.126.10 -f
```
Also make the `id_rsa` file to tbe the primary key used: 
```
echo "IdentityFile /root/.ssh/id_rsa" >> ~/.ssh/config
```
### Create VM for SNO Cluster: 

```
cat << EOF > ~/hub.yaml
hub:
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
   mac: 52:54:00:35:bb:80
   ip: 192.168.125.100
   mask: 255.255.255.0
   gateway: 192.168.125.1
EOF
kcli create plan -f hub.yaml
```

## Creating the Mirror registry:

The mirror registry can be any Any Docker v2.2 compliant registry. Some examples include Red Hat Quay, JFrog Artifactory, Harbor etc. More information can be found [here](https://docs.openshift.com/container-platform/4.14/installing/disconnected_install/installing-mirroring-installation-images.html#installation-about-mirror-registry_installing-mirroring-installation-images). 

**Note** OpenShift's built-in image registry can't be used for this purpose due to its limition of not being able to not being able to push images without tag

We will use the Red Hat Quay as mirror registry. The installation of this involves the following requirements:
* configuring a pull secret. This is needed to pull the registry images from Red Hat. We will take care of this in upcoming section
* 2 or more vCPU & 8GB of RAM. 
* FQDN already defined for the Quay registry. This was taken care of by the earlier step where DNS was configured. 
* About 12GB for the OCP 14 images. If including all operators, this may require upto 358GB. This is taken care by allocating a large disk partition to `/var/lib/containers/storage/` in the initial configurations we did. 

More details about the requirements and other steps can be found [here](https://docs.openshift.com/container-platform/4.14/installing/disconnected_install/installing-mirroring-creating-registry.html)

### Pull Secret:

For this step, you will need a pull secret from Red Hat. You can download your pull secret [here](https://console.redhat.com/openshift/downloads) (scroll all the way down to the "Token" section)
```
export PULL_SECRET="<<paste your pull secret here>>"
```

```
###################
# Step#11_a: Mirroring - Add Pull Secret
###################
mkdir ~/.docker
echo $PULL_SECRET > ~/.docker/config.json
```
### Install the Mirror Registry: 

First, download the installer using the following: 

```
wget https://github.com/quay/mirror-registry/releases/download/v1.3.10/mirror-registry-online.tar.gz
tar -xvf mirror-registry-online.tar.gz
chmod u+x mirror-registry
```
Run the installer: 

```
###################
# Step#11_b: Installing Mirror Registry
###################
~/mirror-registry install --quayHostname quay.tnc.bootcamp.lab --quayRoot /opt/ --initUser quay --initPassword syed@redhat
```

The insaller shall create a few pods for the mirror registry, and end with the following lines: 

```
INFO[2024-04-05 13:06:26] Quay installed successfully, config data is stored in /opt/
INFO[2024-04-05 13:06:26] Quay is available at https://quay.tnc.bootcamp.lab:8443 with credentials (quay, syed@redhat)
```

To furhter verify if the installation is successful, check the pods it must have created: 
```
podman ps
CONTAINER ID  IMAGE                                                    COMMAND         CREATED         STATUS         PORTS                   NAMES
6da068577f02  registry.access.redhat.com/ubi8/pause:8.7-6              infinity        12 minutes ago  Up 12 minutes  0.0.0.0:8443->8443/tcp  2ee3e8b0d965-infra
321acd27dc91  registry.redhat.io/rhel8/postgresql-10:1-203.1669834630  run-postgresql  12 minutes ago  Up 12 minutes  0.0.0.0:8443->8443/tcp  quay-postgres
6494e91da547  registry.redhat.io/rhel8/redis-6:1-92.1669834635         run-redis       11 minutes ago  Up 11 minutes  0.0.0.0:8443->8443/tcp  quay-redis
2ab76abfdb27  registry.redhat.io/quay/quay-rhel8:v3.8.14               registry        11 minutes ago  Up 11 minutes  0.0.0.0:8443->8443/tcp  quay-app
```
### Add Registry Credentials:

To add the newly deployed registry credentials in your locally saved pull secret (by default, its `~/.docker/config.json`), use the following: 
```
###################
# Step#11_c: Adding Mirror Registry Secret
###################
podman login https://quay.tnc.bootcamp.lab:8443 --tls-verify=false --authfile .docker/config.json -u quay -p syed@redhat
```

This will result in: 
> Login Succeeded!

## Preparing the Bastion VM:

The Bastion VM was already brought up. To make it useful, few tools will need to be installed on it. 

### Copying pull secret & Quay cert to Bastion VM:

```
###################
# Step#12_a: Adding Pull Secret & Quay Cert to Bastion
###################
scp .docker/config.json root@192.168.126.10:~/
scp /opt/quay-rootCA/rootCA.pem root@192.168.126.10:~/
```
The rest of the configuration will be done from inside the VM. You can connect to it using the following: 
```
virsh console bastion
```
The credentials to use are `root` and `redhat` 

### Installing OpenShift Client, Installer  and Mirror Plugin

To perform the mirroring to the registry, the `oc mirror` command will be used. This requires installation of `openshift client` as well as `mirroring pluging` for the openshift client. 
Additionally, to build the ISO image for Agent Based Installer, the `openshift-install` binary is needed. 

The following steps install all three of these items. Before doing that, however, lets temporarily disconenct the Bastion VM from the `tnc` bridge, to ensrue that there aren't any routing issues created due to its dual connectivity. Use the following commands: 
```
nmcli con down "System eth0"
sleep 5
sed -i 's/^nameserver 192.168.125.1/#nameserver 192.168.125.1/g' /etc/resolv.conf
```
** HERE** 

Now lets install the tools:

```
###################
# Step#12_b: Installing OpenShift Client, Installer  and Mirror Plugin
###################
curl https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.14.18/openshift-client-linux-4.14.18.tar.gz -o openshift-client-linux-4.14.18.tar.gz
tar -xvf openshift-client-linux-4.14.18.tar.gz
mv oc /usr/bin/
rm -f openshift-client-linux-4.14.18.tar.gz
oc completion bash > oc.bash_completion
mv oc.bash_completion /etc/bash_completion.d/
#
curl https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.14.18/oc-mirror.tar.gz -o oc-mirror.tar.gz
tar -xvf oc-mirror.tar.gz
chmod u+x oc-mirror
#
curl https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.14.18/openshift-install-linux-4.14.18.tar.gz -o openshift-install-linux-4.14.18.tar.gz
tar -xvf ./openshift-install-linux-4.14.18.tar.gz
rm -f openshift-install-linux-4.14.18.tar.gz
mv openshift-install /usr/bin/
#
mv oc-mirror /usr/bin
mkdir ~/.docker
mv config.json ~/.docker/
```
## Performing the Mirroring:

For the mirroring process, we will need to first identify which images we want to mirror to the registry. This is done using `imageSetConfiguration`.  Use the following snippet to create this manifest, in which only 4.14.18 images are being identified, in addition to the apprprpriate versions of the operators we will need: 

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
      - name: stable-5.9
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

Note that the `default` channel of the operators always need to be included. In the above manifest, we have ensured to always include the default (also happens to be latest in the current case, but doesn't have to be). This is true at the time of creating these instructions, but may change over time. To find out the available channel versions and default channel version, the following snippet would be handy: 

```
for X in {"cluster-logging","advanced-cluster-management","local-storage-operator","topology-aware-lifecycle-manager","quay-operator","openshift-gitops-operator"}; do oc mirror list operators --catalog=registry.redhat.io/redhat/redhat-operator-index:v4.14 --package=$X; done
```

### Mirroring:
Now the stage is set to start the mirroring process. Use the following command to start that: 
```
oc mirror --config=./imageset-config.yaml docker://quay.tnc.bootcamp.lab:8443 --dest-skip-tls=true --source-skip-tls=true
```

The output will look like the following: 
```
Logging to .oc-mirror.log
Checking push permissions for quay.tnc.bootcamp.lab:8443
Creating directory: oc-mirror-workspace/src/publish
Creating directory: oc-mirror-workspace/src/v2
Creating directory: oc-mirror-workspace/src/charts
Creating directory: oc-mirror-workspace/src/release-signatures
No metadata detected, creating new workspace
wrote mirroring manifests to oc-mirror-workspace/operators.1712338271/manifests-redhat-operator-index

To upload local images to a registry, run:

        oc adm catalog mirror file://redhat/redhat-operator-index:v4.14 REGISTRY/REPOSITORY
<<SNIP>>
info: Mirroring completed in 4m3.52s (176.5MB/s)
Rendering catalog image "quay.tnc.bootcamp.lab:8443/redhat/redhat-operator-index:v4.14" with file-based catalog
Writing image mapping to oc-mirror-workspace/results-1712338799/mapping.txt
Writing CatalogSource manifests to oc-mirror-workspace/results-1712338799
Writing ICSP manifests to oc-mirror-workspace/results-1712338799
Fri Apr  5 01:40:18 PM EDT 2024
[root@bastion ~]#

```

**NOTE** Takes almost 10 mins (not 4). If it fails, re-run

In case the script ends with something like:
> error: one or more errors occurred while uploading images

Then just rerun the command one more time. This tends to happen in the virtual environment with virtual linux bridges. Re-run will download the items that couldn't be downloaded in previous attempt. 

## Disconnected Environment: 

At this point all the tasks that require internet access for Bastion node are completed. We can go ahead and disconect it completely from the internet, using the following: 

```
nmcli con up "System eth0"
sleep 10
nmcli con down "System eth1"
sed -i 's/^#nameserver 192.168.125.1/nameserver 192.168.125.1/g' /etc/resolv.conf
sed -i 's/^nameserver 192.168.126.1/#nameserver 192.168.126.1/g' /etc/resolv.conf
```

Check to ensure that DNS can still be reached: 
```
nslookup quay.tnc.bootcamp.lab
Server:         192.168.125.1
Address:        192.168.125.1#53

Name:   quay.tnc.bootcamp.lab
Address: 192.168.125.1
```

## Creating the Agent Installer manifests: 

There are two manifests required by Agent Instaler. Those will be created now. 

### Creating Install-Config:

```
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
pullSecret: '{"auths":{"quay.tnc.bootcamp.lab:8443":{"auth":"cXVheTpzeWVkQHRuYw=="}}}'
sshKey: |
EOF
```

Note that the pull secret here is for the locally created mirror registry. We don't need the pull secret to reach redhat.com any more. 

Add the ssh key to be able to access the deployed clsuter node (if needed at some point for debugging):

```
echo -n "  " >> install-config.yaml
cat ~/.ssh/id_rsa.pub >> install-config.yaml
```

Since this is a disconnected environment, the public urls need to be `mapped` to the local registry. This is done through use of `imageContentSourcePolicy`. The manifest for this was already created by the `oc mirror` commmand, and saved in the `~/oc-mirror-workspace/results-X` directory (X is an encoded number). Since our cluster is not yet deployed, we can't apply this manifest to it (the information will be needed at the time of deployment). So this information is plugged into the `install-config.yaml` instead. The following will extract the information from the yaml manifest and save it to a local file called `mirror` :

```
cat ~/oc-mirror-workspace/$(ls ~/oc-mirror-workspace/ | grep result)/imageContentSourcePolicy.yaml | grep -A2 mirrors: | grep -v "\-\-" | sed 's/^  //g' > mirrors
```

You can inspect this file to understand the information bettter. Now lets go ahead and push this information into the `install-config.yaml` file: 

```
echo "imageContentSources:" >> install-config.yaml
cat mirrors >> install-config.yaml
```

Another thing needed to access the local registry is the certificate for accessing Quay. This was created at the time when Quay was deployed, and saved on the host device's `/opt/quay/` directory. In a previous step, we had copied over that file to the home directory of the VM, so now all thats needed is to plug that information into the install-config.yaml: 

```
echo "additionalTrustBundle: |" >> install-config.yaml
cat ~/rootCA.pem | sed 's/^/  /g' >> install-config.yaml
```

### Creating Agent-Config manifest:

The second manifest needed by the installer is `agent-config.yaml`. It will be created using the following: 

```
cat << EOF > agent-config.yaml
apiVersion: v1alpha1
metadata:
  name: hub
rendezvousIP: 192.168.125.100
hosts:
  - hostname: sno
    interfaces:
     - name: eth0
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
```
## Creating & Mounting the ISO file:

All the pieces are ready to start the building of ISO file. Use the following command for that: 

**NOTE** This process removes the two manifests we created earlier. IF those have to be preserved for future observation, make a copy of those at some other location. 

```
mkdir ~/abi 
cp install-config.yaml ~/abi/
cp agent-config.yaml ~/abi/
cd ~/abi
openshift-install agent create image --dir=./ --log-level=debug
```

The logs will end with the following: 
```
DEBUG   Reusing previously-fetched ClusterDeployment Config
DEBUG Generating Kubeconfig Admin Client...
DEBUG Fetching Kubeadmin Password...
DEBUG Reusing previously-fetched Kubeadmin Password
```
An ISO would now have been created. We can go ahead and copy that to the default home directory for the HTTP server, and also enable the HTTP server to start listening on the default http port: 

```
cp agent.x86_64.iso /var/www/html/
systemctl enable httpd --now
sleep 10
systemctl is-active httpd
# should show "active"
```

You can now go ahead and mount this ISO the the VM (called `hub`) that was created in an earlier step: 

```
curl -d '{"Image":"http:/192.168.125.10/agent.x86_64.iso","Inserted": true}' -H "Content-Type: application/json" -X POST -k https://192.168.125.1:9000/redfish/v1/Managers/local/hub/VirtualMedia/Cd/Actions/VirtualMedia.InsertMedia
```

This command calls the sushy api (installed during host buildup, and listening on port 9000). 

To verify that the mounting was successful, exit the VM, and verify by running the following command on the host machine: 

```
[root@hypervisor ~]# virsh dumpxml hub | grep source
      <source file='/var/lib/libvirt/images/hub_0.img'/>
      <source file='/var/lib/libvirt/images/hub_1.img'/>
      <source file='/var/lib/libvirt/images/agent.x86_64.iso'/>
      <source network='tnc'/>
[root@hypervisor ~]#
```

This command shows the availble boot sources for the vm called `hub`. As you can see, the ISO file is listed amount those sources. Its not the first source, but since the other sources don't contain any boot image, its safe to leave the boot operation as-is. (in a production deployment, the boot sequence should be changed to avoid any confusion due to some preexisting image on the local disks)

# TODO:
- add GUI for start and monitor of install
- show progrsss (use belwo) via client
- show oc get nodes 
- Check status of ICSP and CS ...add operators

## Perform and Monitor the Installation:


### Monitor install progress: 

Once the VM is started, use the following commnad (from inside the Bastion VM, run from the ~/abi/ directory) to monitor progress:

```
openshift-install agent wait-for bootstrap-complete --log-debug=debug
```
The output will end with the following:
```
INFO Host: sno, reached installation stage Writing image to disk: 6% 
INFO Host: sno, reached installation stage Writing image to disk: 20% 
INFO Host: sno, reached installation stage Writing image to disk: 27% 
INFO Host: sno, reached installation stage Writing image to disk: 41% 
INFO Host: sno, reached installation stage Writing image to disk: 48% 
INFO Host: sno, reached installation stage Writing image to disk: 57% 
INFO Host: sno, reached installation stage Writing image to disk: 67% 
INFO Host: sno, reached installation stage Writing image to disk: 78% 
INFO Host: sno, reached installation stage Writing image to disk: 84% 
INFO Host: sno, reached installation stage Writing image to disk: 90% 
INFO Host: sno, reached installation stage Writing image to disk: 100% 
INFO Bootstrap configMap status is complete       
INFO cluster bootstrap is complete  
```






=======
=======
=======
=======
=======
=======
=======
=======
=======
=======
=======
=======
=======
=======
=======
==============
=======
=======
=======

























# OLD ================

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
 - dnf install -y nmstate
EOF
kcli create plan -f vm1.yaml 
```

Configure keyless ssh to VM1

```
ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/vm1
ssh-copy-id -i ~/.ssh/vm1.pub root@192.168.125.10
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
./mirror-registry install --quayHostname quay.tnc.bootcamp.lab --quayRoot /opt/ --initUser quay --initPassword syed@redhat
```

> INFO[2024-03-28 07:34:08] Quay installed successfully, config data is stored in /opt/ <br>
> INFO[2024-03-28 07:34:08] Quay is available at https://quay.tnc.bootcamp.lab:8443 with credentials (quay, syed@redhat) 

## Setting up the VM for mirroring: 

Add credentials to docker auth file: 
```
podman login https://quay.tnc.bootcamp.lab:8443 --tls-verify=false --authfile .docker/config.json 
```

Push this file to VM: 
```
scp .docker/config.json root@192.168.126.10:~/
```
```
ssh root@192.168.126.10
```
```
curl https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.14.18/openshift-client-linux-4.14.18.tar.gz -o openshift-client-linux-4.14.18.tar.gz
tar -xvf openshift-client-linux-4.14.18.tar.gz 
mv oc /usr/bin/
rm -f openshift-client-linux-4.14.18.tar.gz 
oc completion bash > oc.bash_completion
mv oc.bash_completion /etc/bash_completion.d/
curl https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.14.18/oc-mirror.tar.gz -o oc-mirror.tar.gz
tar -xvf oc-mirror.tar.gz 
chmod u+x oc-mirror
mv oc-mirror /usr/bin
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
      - name: stable-5.9
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
curl https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.14.18/openshift-install-linux-4.14.18.tar.gz -o openshift-install-linux-4.14.18.tar.gz
tar -xvf ./openshift-install-linux-4.14.18.tar.gz
rm -f openshift-install-linux-4.14.18.tar.gz
mv openshift-install /usr/bin/
```

## Create install yamls:



NEW:

```
mkdir ~/abi 
cd ~/abi
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
  replicas: 1
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
pullSecret: '{"auths":{"quay.tnc.bootcamp.lab:8443":{"auth":"cXVheTpzeWVkQHRuYw=="}}}'
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

