```
sudo -i
dnf update -y
dnf -y install python3.9
# set python stuff (path seem to be needed in RHEL8)
ln -s /bin/pip3 /bin/pip
ln -s /bin/python3.9 /bin/python

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
PING quay.tnc.bootcamp.lab (139.178.70.133) 56(84) bytes of data.
64 bytes from hypervisor (139.178.70.133): icmp_seq=1 ttl=64 time=0.023 ms
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








## Installer: 
https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.14.18/openshift-install-linux-4.14.18.tar.gz
