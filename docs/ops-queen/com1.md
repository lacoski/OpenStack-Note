### COM1

```
yum install -y epel-release
yum install -y wget byobu git vim

hostnamectl set-hostname com01

echo "Setup IP eth0"
nmcli c modify eth0 ipv4.addresses 10.10.30.71/24
nmcli c modify eth0 ipv4.gateway 10.10.30.1
nmcli c modify eth0 ipv4.dns 8.8.8.8
nmcli c modify eth0 ipv4.method manual
nmcli con mod eth0 connection.autoconnect yes

echo "Setup IP eth1"
nmcli c modify eth1 ipv4.addresses 10.10.31.71/24
nmcli c modify eth1 ipv4.method manual
nmcli con mod eth1 connection.autoconnect yes

echo "Setup IP eth2"
nmcli c modify eth2 ipv4.addresses 10.10.32.71/24
nmcli c modify eth2 ipv4.method manual
nmcli con mod eth2 connection.autoconnect yes

systemctl disable NetworkManager
systemctl stop NetworkManager
systemctl start network
systemctl enable network

sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
systemctl stop firewalld
systemctl disable firewalld

curl -Lso- https://raw.githubusercontent.com/nhanhoadocs/scripts/master/Utilities/cmdlog.sh | bash

timedatectl set-timezone Asia/Ho_Chi_Minh
yum -y install chrony
NTP_SERVER_IP='10.10.30.15'
sed -i '/server/d' /etc/chrony.conf
echo "server $NTP_SERVER_IP iburst" >> /etc/chrony.conf
systemctl enable chronyd.service
systemctl restart chronyd.service
chronyc sources


echo 'net.ipv4.conf.all.arp_ignore = 1'  >> /etc/sysctl.conf
echo 'net.ipv4.conf.all.arp_announce = 2'  >> /etc/sysctl.conf
echo 'net.ipv4.conf.all.rp_filter = 2'  >> /etc/sysctl.conf
echo 'net.netfilter.nf_conntrack_tcp_be_liberal = 1'  >> /etc/sysctl.conf

cat << EOF >> /etc/sysctl.conf
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.tcp_keepalive_time = 6
net.ipv4.tcp_keepalive_intvl = 3
net.ipv4.tcp_keepalive_probes = 6
net.ipv4.ip_forward = 1
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
EOF
sysctl -p

echo "10.10.30.70 ctl01" >> /etc/hosts
echo "10.10.30.71 com01" >> /etc/hosts
echo "10.10.30.72 com02" >> /etc/hosts

init 6
```

## Cài đặt các gói cần thiết cho OPS
```
yum install -y centos-release-openstack-queens \
   open-vm-tools python2-PyMySQL vim telnet wget curl 
yum install -y python-openstackclient openstack-selinux 
```

Lưu ý, nếu lỗi update gói Ceph và Queen Openstack:
- Sửa lại base url nếu update lỗi

```
[root@ctl01 ~]# cd /etc/yum.repos.d

[root@ctl01 yum.repos.d]# cat CentOS-OpenStack-queens.repo 
# CentOS-OpenStack-queens.repo
#
# Please see http://wiki.centos.org/SpecialInterestGroup/Cloud for more
# information

[centos-openstack-queens]
name=CentOS-7 - OpenStack queens
#baseurl=http://mirror.centos.org/$contentdir/$releasever/cloud/$basearch/openstack-queens/
baseurl=http://mirror.centos.org/centos-7/7.9.2009/cloud/x86_64/openstack-queens/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Cloud
exclude=sip,PyQt4

[centos-openstack-queens-test]
name=CentOS-7 - OpenStack queens Testing
baseurl=https://buildlogs.centos.org/centos/7/cloud/$basearch/openstack-queens/
gpgcheck=0
enabled=0
exclude=sip,PyQt4

[centos-openstack-queens-debuginfo]
name=CentOS-7 - OpenStack queens - Debug
baseurl=http://debuginfo.centos.org/centos/7/cloud/$basearch/
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Cloud
exclude=sip,PyQt4

[centos-openstack-queens-source]
name=CentOS-7 - OpenStack queens - Source
baseurl=http://vault.centos.org/centos/7/cloud/Source/openstack-queens/
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Cloud
exclude=sip,PyQt4

[rdo-trunk-queens-tested]
name=OpenStack queens Trunk Tested
baseurl=https://trunk.rdoproject.org/centos7-queens/current-passed-ci/
gpgcheck=0
enabled=0


[root@ctl01 yum.repos.d]# cat CentOS-Ceph-Luminous.repo 
# CentOS-Ceph-Luminous.repo
#
# Please see http://wiki.centos.org/SpecialInterestGroup/Storage for more
# information

[centos-ceph-luminous]
name=CentOS-$releasever - Ceph Luminous
#baseurl=http://mirror.centos.org/$contentdir/$releasever/storage/$basearch/ceph-luminous/
baseurl=http://mirror.centos.org/centos-7/7.9.2009/storage/x86_64/ceph-luminous/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Storage

[centos-ceph-luminous-test]
name=CentOS-$releasever - Ceph Luminous Testing
baseurl=http://buildlogs.centos.org/centos/$releasever/storage/$basearch/ceph-luminous/
gpgcheck=0
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Storage

[centos-ceph-luminous-source]
name=CentOS-$releasever - Ceph Luminous Source
baseurl=http://vault.centos.org/$contentdir/$releasever/storage/Source/ceph-luminous/
gpgcheck=0
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Storage
```


Thử lại
```
yum clean all
yum install -y centos-release-openstack-queens \
   open-vm-tools python2-PyMySQL vim telnet wget curl 
yum install -y python-openstackclient openstack-selinux 
```

## Nova

yum install -y openstack-nova-compute libvirt-client
mv /etc/nova/nova.{conf,conf.bk}

cat << EOF >> /etc/nova/nova.conf 
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:passla123@10.10.30.70:5672
my_ip = 10.10.30.71
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver
[api]
auth_strategy = keystone
[api_database]
[barbican]
[cache]
[cells]
[cinder]
os_region_name = RegionOne
[compute]
[conductor]
[console]
[consoleauth]
[cors]
[crypto]
[database]
[devices]
[ephemeral_storage_encryption]
[filter_scheduler]
[glance]
api_servers = http://10.10.30.70:9292
[guestfs]
[healthcheck]
[hyperv]
[ironic]
[key_manager]
[keystone]
[keystone_authtoken]
auth_url = http://10.10.30.70:5000/v3
memcached_servers = 10.10.30.70:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = passla123
region_name = RegionOne
[libvirt]
virt_type = qemu
[matchmaker_redis]
[metrics]
[mks]
[neutron]
[notifications]
[osapi_v21]
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
#driver = messagingv2
[oslo_messaging_rabbit]
rabbit_ha_queues = true
rabbit_retry_interval = 1
rabbit_retry_backoff = 2
amqp_durable_queues= true
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[pci]
[placement]
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://10.10.30.70:5000/v3
username = placement
password = passla123
os_region_name = RegionOne
[quota]
[rdp]
[remote_debug]
[scheduler]
[serial_console]
[service_user]
[spice]
[upgrade_levels]
[vault]
[vendordata_dynamic_auth]
[vmware]
[vnc]
enabled = True
server_listen = 0.0.0.0
server_proxyclient_address = 10.10.30.71
novncproxy_base_url = http://10.10.30.70:6080/vnc_auto.html
[workarounds]
[wsgi]
[xenserver]
[xvp]
EOF

chown root:nova /etc/nova/nova.conf
systemctl enable libvirtd.service openstack-nova-compute.service
systemctl start libvirtd.service openstack-nova-compute.service

## Neutron

yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables -y

mv /etc/neutron/neutron.{conf,conf.bk}

cat << EOF >> /etc/neutron/neutron.conf
[DEFAULT]
transport_url = rabbit://openstack:passla123@10.10.30.70:5672
auth_strategy = keystone
[agent]
[cors]
[database]
[keystone_authtoken]
auth_uri = http://10.10.30.70:5000
auth_url = http://10.10.30.70:35357
memcached_servers = 10.10.30.70:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = passla123
region_name = RegionOne
[matchmaker_redis]
[nova]
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
#driver = messagingv2
[oslo_messaging_rabbit]
rabbit_ha_queues = true
rabbit_retry_interval = 1
rabbit_retry_backoff = 2
amqp_durable_queues= true
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[quotas]
[ssl]
EOF

chown root:neutron /etc/neutron/neutron.conf

mv /etc/neutron/plugins/ml2/linuxbridge_agent.{ini,ini.bk}

cat << EOF >> /etc/neutron/plugins/ml2/linuxbridge_agent.ini
[DEFAULT]
[agent]
[linux_bridge]
physical_interface_mappings = provider:eth2
[network_log]
[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
[vxlan]
# enable_vxlan = true
## network dataVM
local_ip = 10.10.31.71
# l2_population = true
EOF

chown root:neutron /etc/neutron/plugins/ml2/linuxbridge_agent.ini

mv /etc/neutron/dhcp_agent.{ini,ini.bk}
cat << EOF >> /etc/neutron/dhcp_agent.ini
[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
force_metadata = True
[agent]
[ovs]
EOF

chown root:neutron /etc/neutron/dhcp_agent.ini

mv /etc/neutron/metadata_agent.{ini,ini.bk}

cat << EOF >> /etc/neutron/metadata_agent.ini
[DEFAULT]
nova_metadata_host = 10.10.30.70
metadata_proxy_shared_secret = passla123
[agent]
[cache]
EOF

chown root:neutron /etc/neutron/metadata_agent.ini

## Chỉnh sửa bổ sung cấu hình [neutron] trong file /etc/nova/nova.conf

[neutron]
url = http://10.10.30.70:9696
auth_url = http://10.10.30.70:35357
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = passla123
region_name = RegionOne

systemctl restart openstack-nova-compute.service libvirtd.service 

systemctl enable neutron-linuxbridge-agent.service \
neutron-dhcp-agent.service neutron-metadata-agent.service

systemctl start neutron-linuxbridge-agent.service \
neutron-dhcp-agent.service neutron-metadata-agent.service
