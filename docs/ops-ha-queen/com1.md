## Phân hoạch

vlan mgnt: eth0: 10.10.11.94
vlan provider: eth1: 10.10.12.94
vlan datavm: eth2: 10.10.14.94

## Setup node

hostnamectl set-hostname com01

echo "Setup IP eth0"
nmcli c modify eth0 ipv4.addresses 10.10.11.94/24
nmcli c modify eth0 ipv4.gateway 10.10.11.1
nmcli c modify eth0 ipv4.dns 8.8.8.8
nmcli c modify eth0 ipv4.method manual
nmcli con mod eth0 connection.autoconnect yes

echo "Setup IP eth1"
nmcli c modify eth1 ipv4.addresses 10.10.12.94/24
nmcli c modify eth1 ipv4.method manual
nmcli con mod eth1 connection.autoconnect yes

echo "Setup IP eth2"
nmcli c modify eth2 ipv4.addresses 10.10.14.94/24
nmcli c modify eth2 ipv4.method manual
nmcli con mod eth2 connection.autoconnect yes

sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
systemctl stop firewalld
systemctl disable firewalld

curl -Lso- https://raw.githubusercontent.com/nhanhoadocs/scripts/master/Utilities/cmdlog.sh | bash

timedatectl set-timezone Asia/Ho_Chi_Minh
yum -y install chrony
systemctl enable chronyd.service
systemctl restart chronyd.service
chronyc sources

init 6

## Chuẩn bị sysctl

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

## Cấu hình hostname

echo "10.10.11.87 ctl01" >> /etc/hosts
echo "10.10.11.88 ctl02" >> /etc/hosts
echo "10.10.11.89 ctl03" >> /etc/hosts
echo "10.10.11.94 com01" >> /etc/hosts

# Cài đặt các gói cần thiết

yum -y install centos-release-openstack-queens
yum -y install crudini wget vim
yum -y install python-openstackclient openstack-selinux python2-PyMySQL

# Cài đặt Nova

yum install openstack-nova-compute libvirt-client -y

cp /etc/nova/nova.conf  /etc/nova/nova.conf.org
rm -rf /etc/nova/nova.conf

cat << EOF >> /etc/nova/nova.conf 
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:Welcome123@10.10.11.87:5672,openstack:Welcome123@10.10.11.88:5672,openstack:Welcome123@10.10.11.89:5672
my_ip = 10.10.11.94
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver
[api]
auth_strategy = keystone
[api_database]
[barbican]
[cache]
[cells]
[cinder]
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
api_servers = http://10.10.11.93:9292
[guestfs]
[healthcheck]
[hyperv]
[ironic]
[key_manager]
[keystone]
[keystone_authtoken]
auth_url = http://10.10.11.93:5000/v3
memcached_servers = 10.10.11.87:11211,10.10.11.88:11211,10.10.11.89:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = Welcome123
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
os_region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://10.10.11.93:5000/v3
username = placement
password = Welcome123
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
server_proxyclient_address = 10.10.11.94
novncproxy_base_url = http://10.10.11.93:6080/vnc_auto.html
[workarounds]
[wsgi]
[xenserver]
[xvp]
EOF

chown root:nova /etc/nova/nova.conf

systemctl enable libvirtd.service openstack-nova-compute.service
systemctl restart libvirtd.service openstack-nova-compute.service


# Cài Neutron
yum install openstack-neutron openstack-neutron-ml2 \
  openstack-neutron-linuxbridge ebtables -y

#Cau hinh neutron
cp /etc/neutron/neutron.conf /etc/neutron/neutron.conf.org 
rm -rf /etc/neutron/neutron.conf

cat << EOF >> /etc/neutron/neutron.conf
[DEFAULT]
transport_url = rabbit://openstack:Welcome123@10.10.11.87:5672,openstack:Welcome123@10.10.11.88:5672,openstack:Welcome123@10.10.11.89:5672
auth_strategy = keystone
[agent]
[cors]
[database]
[keystone_authtoken]
auth_uri = http://10.10.11.93:5000
auth_url = http://10.10.11.93:35357
memcached_servers = 10.10.11.87:11211,10.10.11.88:11211,10.10.11.89:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = Welcome123
[matchmaker_redis]
[nova]
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
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

# Cau hinh file LB agent

cp /etc/neutron/plugins/ml2/linuxbridge_agent.ini /etc/neutron/plugins/ml2/linuxbridge_agent.ini.org 
rm -rf /etc/neutron/plugins/ml2/linuxbridge_agent.ini

cat << EOF >> /etc/neutron/plugins/ml2/linuxbridge_agent.ini
[DEFAULT]
[agent]
extensions = qos
[linux_bridge]
physical_interface_mappings = provider:eth1 
[network_log]
[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
[vxlan]
enable_vxlan = true
local_ip = 10.10.14.94
l2_population = true
EOF


# Cấu hình dhcp agent

cp /etc/neutron/dhcp_agent.ini /etc/neutron/dhcp_agent.ini.org
rm -rf /etc/neutron/dhcp_agent.ini

cat << EOF >> /etc/neutron/dhcp_agent.ini
[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
force_metadata = True
[agent]
[ovs]
EOF


# Cấu hình metadata agent

cp /etc/neutron/metadata_agent.ini /etc/neutron/metadata_agent.ini.org 
rm -rf /etc/neutron/metadata_agent.ini

cat << EOF >> /etc/neutron/metadata_agent.ini
[DEFAULT]
nova_metadata_host = 10.10.11.93
metadata_proxy_shared_secret = Welcome123
[agent]
[cache]
EOF

# Thêm vào file /etc/nova/nova.conf
[neutron]
url = http://10.10.11.93:9696
auth_url = http://10.10.11.93:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = Welcome123


chown root:neutron /etc/neutron/metadata_agent.ini /etc/neutron/neutron.conf /etc/neutron/dhcp_agent.ini /etc/neutron/plugins/ml2/linuxbridge_agent.ini

systemctl restart libvirtd.service openstack-nova-compute

systemctl enable neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
systemctl restart neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
