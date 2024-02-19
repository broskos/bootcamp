```
sudo -i 
dnf install -y tree jq
# kcli list vm
cp /opt/dnsmasq/include.d/sno2.ipv4 /opt/dnsmasq/include.d/sno3.ipv4
sed -i s/sno2/sno3/g /opt/dnsmasq/include.d/sno3.ipv4
sed -i s/40/41/g /opt/dnsmasq/include.d/sno3.ipv4
kcli create vm -P start=False -P uefi_legacy=true -P plan=hub -P memory=24000 -P numcpus=12 -P disks=[200,200] -P nets=['{"name": "5gdeploymentlab", "mac": "aa:aa:aa:aa:04:01"}'] -P uuid=aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaa0401 -P name=sno3
## kcli list vm
## +----------------+--------+----------------+----------------------------------------------------+------+---------+
## |      Name      | Status |       Ip       |                       Source                       | Plan | Profile |
## +----------------+--------+----------------+----------------------------------------------------+------+---------+
## | hub-ctlplane-0 |   up   | 192.168.125.20 | rhcos-414.92.202310170514-0-openstack.x86_64.qcow2 | hub  |  kvirt  |
## | hub-ctlplane-1 |   up   | 192.168.125.21 | rhcos-414.92.202310170514-0-openstack.x86_64.qcow2 | hub  |  kvirt  |
## | hub-ctlplane-2 |   up   | 192.168.125.22 | rhcos-414.92.202310170514-0-openstack.x86_64.qcow2 | hub  |  kvirt  |
## |      sno1      |   up   | 192.168.125.30 |                                                    | hub  |  kvirt  |
## |      sno2      |  down  |                |                                                    | hub  |  kvirt  |
## |      sno3      |  down  |                |                                                    | hub  |  kvirt  |
## +----------------+--------+----------------+----------------------------------------------------+------+---------+

## kcli list plan
## +------+-------------------------------------------------------------+
## | Plan |                             Vms                             |
## +------+-------------------------------------------------------------+
## | hub  | hub-ctlplane-0,hub-ctlplane-1,hub-ctlplane-2,sno1,sno2,sno3 |
## +------+-------------------------------------------------------------+
```
