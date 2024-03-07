# ToDo:
Work in Progress...

```
kcli create vm -P start=False -P uefi_legacy=true -P plan=hub -P memory=24000 -P numcpus=12 -P disks=[200,200] -P nets=['{"name": "5gdeploymentlab", "mac": "52:54:00:35:aa:80"}'] -P uuid=aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaa9901 -P name=abinode
```


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
baseDomain: 5g-deployment.lab
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
source: quay.io/ocp-release
pullSecret: <<>>
sshKey: |
EOF
echo -n "  " >> install-config.yaml
cat ~/.ssh/id_rsa.pub >> install-config.yaml
```


```
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.14.1/openshift-install-linux-4.14.1.tar.gz
tar -xvf openshift-install-linux-4.14.1.tar.gz
rm -f openshift-install-linux-4.14.1.tar.gz
mv openshift-install /usr/bin
```

```
cat << EOF > /opt/dnsmasq/include.d/abinode.ipv4
host-record=api-int.mgmt.5g-deployment.lab,192.168.125.100
host-record=api.mgmt.5g-deployment.lab,192.168.125.100
address=/apps.mgmt.5g-deployment.lab/192.168.125.100
dhcp-host=52:54:00:35:aa:80,ocp-mgmt,192.168.125.100
EOF
```
```
systemctl restart dnsmasq-virt
```

```
dnf install nmstate -y
```

```
echo "imageContentSources:" >> install-config.yaml
echo "- mirrors:" >> install-config.yaml
echo "  - nfra.5g-deployment.lab:8443/openshift/release" >> install-config.yaml
echo "  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev" >> install-config.yaml
```

```
########## Create HTTP Service
cat << EOF > /etc/systemd/system/podman-webcache.service
[Unit]
Description=Podman container - webcache
After=network.target

[Service]
Type=simple
WorkingDirectory=/root
TimeoutStartSec=300
ExecStartPre=-/usr/bin/podman rm -f webcache
ExecStart=/usr/bin/podman run --name webcache --net host -v /opt/html/data:/usr/local/apache2/htdocs:z quay.io/alosadag/httpd:p8980 
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
```    
########## Copy Agent ISO to HTTP:
cp agent.x86_64.iso /opt/html/data/
```
```
curl -d '{"Image":"http://127.0.0.1:8080/agent.x86_64.iso","Inserted": true}' -H "Content-Type: application/json" -X POST -k https://127.0.0.1:9000/redfish/v1/Managers/local/abinode/VirtualMedia/Cd/Actions/VirtualMedia.InsertMedia
```

```
curl -X PATCH -H 'Content-Type: application/json' -d '{
      "Boot": {
          "BootSourceOverrideTarget": "Cd",
          "BootSourceOverrideMode": "Uefi",
          "BootSourceOverrideEnabled": "Continuous"
      }
    }' -k https://127.0.0.1:9000/redfish/v1/Systems/local/abinode
```

```
#tail /var/log/messages 
#
#Jan 17 13:00:11 Jan17 ksushy[38712]: #033[32mImage agent.x86_64.iso Added#033[0m
#Jan 17 13:00:11 Jan17 ksushy[38712]: #033[36mSwitching iso for vm hubvm to agent.x86_64.iso...#033[0m
#[root@Jan17 temp]# tail /var/log/messages 
#Jan 17 13:00:11 Jan17 ksushy[38712]: #033[36mSetting iso of vm hubvm to http://127.0.0.1:8080/agent.x86_64.iso#033[0m
#Jan 17 13:00:11 Jan17 kvm[39722]: 1 guest now active
#Jan 17 13:00:11 Jan17 ksushy[38712]: #033[36mGrabbing image agent.x86_64.iso from url http://127.0.0.1:8080/agent.x86_64.iso#033[0m
#Jan 17 13:00:11 Jan17 kvm[39723]: 0 guests now active
#Jan 17 13:00:11 Jan17 ksushy[39724]:  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
#Jan 17 13:00:11 Jan17 ksushy[39724]:                                 Dload  Upload   Total   Spent    Left  Speed
#Jan 17 13:00:11 Jan17 podman[39196]: 127.0.0.1 - - [17/Jan/2024:18:00:11 +0000] "GET /agent.x86_64.iso HTTP/1.1" 200 1205862400
#Jan 17 13:00:11 Jan17 ksushy[39724]: #015  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0#015 99 1150M   99 1138M    0     0  2277M      0 --:--:-- --:--:-- --:--:-- 2273M#015100 1150M  100 1150M    0     0  2277M      0 --:--:-- --:--:-- --:--:-- 2277M
#Jan 17 13:00:11 Jan17 ksushy[38712]: #033[32mImage agent.x86_64.iso Added#033[0m
#Jan 17 13:00:11 Jan17 ksushy[38712]: #033[36mSwitching iso for vm hubvm to agent.x86_64.iso...#033[0m
#
```

```
kcli start vm abinode; virsh console --force abinode
```

grub: console=tty0 console=ttyS0

