sudo systemctl disable NetworkManager
sudo systemctl stop NetworkManager
sudo systemctl enable network
sudo systemctl start network

echo "172.16.4.200 controller" >> /etc/hosts
echo "172.16.4.201 compute" >> /etc/hosts
echo "172.16.4.202 compute2" >> /etc/hosts

yum -y install centos-release-openstack-queens
yum -y install crudini wget vim
yum -y install python-openstackclient openstack-selinux python2-PyMySQL


yum -y install chrony
VIP_MGNT_IP='172.16.4.200'
sed -i '/server/d' /etc/chrony.conf
echo "server $VIP_MGNT_IP iburst" >> /etc/chrony.conf
systemctl enable chronyd.service
systemctl restart chronyd.service

sed -i 's|baseurl=http:\/\/mirror.centos.org\/$contentdir\/$releasever\/virt\/$basearch\/kvm-common\/|baseurl=http:\/\/mirror.centos.org\/centos\/7\/virt\/x86_64\/kvm-common\/|g' /etc/yum.repos.d/CentOS-QEMU-EV.repo

yum install openstack-nova-compute libvirt-client -y

cp /etc/nova/nova.conf  /etc/nova/nova.conf.org
rm -rf /etc/nova/nova.conf

cat << EOF >> /etc/nova/nova.conf 
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:Welcome123@172.16.4.200:5672
my_ip = 172.16.4.202
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
api_servers = http://172.16.4.200:9292
[guestfs]
[healthcheck]
[hyperv]
[ironic]
[key_manager]
[keystone]
[keystone_authtoken]
auth_url = http://172.16.4.200:5000/v3
memcached_servers = 172.16.4.200:11211
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
auth_url = http://172.16.4.200:5000/v3
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
server_proxyclient_address = 172.16.4.202
novncproxy_base_url = http://172.16.4.200:6080/vnc_auto.html
[workarounds]
[wsgi]
[xenserver]
[xvp]
EOF

chown root:nova /etc/nova/nova.conf

systemctl enable libvirtd.service openstack-nova-compute.service
systemctl start libvirtd.service openstack-nova-compute.service

yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-openvswitch ebtables -y

cp /etc/neutron/neutron.conf /etc/neutron/neutron.conf.org 
rm -rf /etc/neutron/neutron.conf

cat << EOF >> /etc/neutron/neutron.conf
[DEFAULT]
transport_url = rabbit://openstack:Welcome123@172.16.4.200:5672
auth_strategy = keystone
[agent]
[cors]
[database]
[keystone_authtoken]
auth_uri = http://172.16.4.200:5000
auth_url = http://172.16.4.200:35357
memcached_servers = 172.16.4.200:11211
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

cp /etc/neutron/plugins/ml2/openvswitch_agent.ini /etc/neutron/plugins/ml2/openvswitch_agent.ini.org 
rm -rf /etc/neutron/plugins/ml2/openvswitch_agent.ini

cat << EOF >> /etc/neutron/plugins/ml2/openvswitch_agent.ini
[DEFAULT]
[agent]
tunnel_types = vxlan
l2_population = True
[network_log]
[ovs]
bridge_mappings = provider:br-provider
local_ip = 10.0.3.10
[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
[xenapi]
EOF

# Cấu hình dhcp agent
cp /etc/neutron/dhcp_agent.ini /etc/neutron/dhcp_agent.ini.org
rm -rf /etc/neutron/dhcp_agent.ini

cat << EOF >> /etc/neutron/dhcp_agent.ini
[DEFAULT]
interface_driver = openvswitch
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = True
force_metadata = True
[agent]
[ovs]
EOF

cp /etc/neutron/metadata_agent.ini /etc/neutron/metadata_agent.ini.org 
rm -rf /etc/neutron/metadata_agent.ini

cat << EOF >> /etc/neutron/metadata_agent.ini
[DEFAULT]
nova_metadata_host = 172.16.4.200
metadata_proxy_shared_secret = Welcome123
[agent]
[cache]
EOF

[neutron]
url = http://172.16.4.200:9696
auth_url = http://172.16.4.200:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = Welcome123

chown root:neutron /etc/neutron/metadata_agent.ini /etc/neutron/neutron.conf /etc/neutron/dhcp_agent.ini /etc/neutron/plugins/ml2/openvswitch_agent.ini

systemctl enable openvswitch
systemctl restart openvswitch

ovs-vsctl add-br br-provider
ovs-vsctl add-port br-provider ens224

rm -rf /etc/sysconfig/network-scripts/ifcfg-ens224
cat << EOF >> /etc/sysconfig/network-scripts/ifcfg-ens224
DEVICE=ens224
NAME=ens224
DEVICETYPE=ovs
TYPE=OVSPort
OVS_BRIDGE=br-provider
ONBOOT=yes
BOOTPROTO=none
NM_CONTROLLED=no
EOF

cat << EOF >> /etc/sysconfig/network-scripts/ifcfg-br-provider
ONBOOT=yes
IPADDR=172.16.9.12
NETMASK=255.255.255.0
DEVICE=br-provider
NAME=br-provider
DEVICETYPE=ovs
OVSBOOTPROTO=none
TYPE=OVSBridge
EOF

systemctl restart network

systemctl restart openstack-nova-compute

systemctl enable neutron-openvswitch-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
systemctl restart neutron-openvswitch-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service

---------------
cp /etc/neutron/metadata_agent.ini /etc/neutron/metadata_agent.ini.org 
rm -rf /etc/neutron/metadata_agent.ini

cat << EOF >> /etc/neutron/metadata_agent.ini
[DEFAULT]
nova_metadata_host = 172.16.4.200
metadata_proxy_shared_secret = Welcome123
[agent]
[cache]
EOF 

edit: /etc/nova/nova.conf

[neutron]
url = http://172.16.4.200:9696
auth_url = http://172.16.4.200:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = Welcome123

chown root:neutron /etc/neutron/metadata_agent.ini /etc/neutron/neutron.conf /etc/neutron/dhcp_agent.ini /etc/neutron/plugins/ml2/openvswitch_agent.ini

systemctl enable openvswitch
systemctl restart openvswitch




