yum install openswan openstack-neutron-vpnaas


# Config /etc/neutron/neutron.conf
service_plugins = ...,vpnaas

# Config /etc/neutron/neutron_vpnaas.conf
[service_providers]
service_provider = VPN:openswan:neutron_vpnaas.services.vpn.service_drivers.ipsec.IPsecVPNDriver:default

# Config /etc/neutron/l3_agent.ini
[AGENT]
extensions = ...,vpnaas

[vpnagent]
vpn_device_driver = neutron_vpnaas.services.vpn.device_drivers.libreswan_ipsec.LibreSwanDriver

# Populate database
neutron-db-manage --subproject neutron-vpnaas upgrade head

# Restart neutron-server & l3-agent
systemctl restart neutron-server
systemctl restart neutron-l3-agent

# Tạo Network + Router + Binding

## Example

[root@ctl01 ~(admin-openrc-r1)]$ yum install openswan openstack-neutron-vpnaas -y

# /etc/neutron/neutron.conf

[root@ctl01 ~(admin-openrc-r1)]$ cat /etc/neutron/neutron.conf
[DEFAULT]
bind_host = 10.10.30.70
core_plugin = ml2
service_plugins = router, vpnaas                          ##### CONFIG TẠI ĐÂY
transport_url = rabbit://openstack:passla123@10.10.30.70
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
allow_overlapping_ips = True
dhcp_agents_per_network = 2
[agent]
[cors]
[database]
connection = mysql+pymysql://neutron:passla123@10.10.30.70/neutron
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
auth_url = http://10.10.30.70:35357
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = passla123
region_name = RegionOne
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
#driver = messagingv2
[oslo_messaging_rabbit]
rabbit_retry_interval = 1
rabbit_retry_backoff = 2
amqp_durable_queues = true
rabbit_ha_queues = true
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[quotas]
[ssl]

### /etc/neutron/neutron_vpnaas.conf

[root@ctl01 ~(admin-openrc-r1)]$ cat /etc/neutron/neutron_vpnaas.conf
[DEFAULT]

[service_providers]
service_provider = VPN:openswan:neutron_vpnaas.services.vpn.service_drivers.ipsec.IPsecVPNDriver:default

### /etc/neutron/l3_agent.ini

[root@ctl01 ~(admin-openrc-r1)]$ cat /etc/neutron/l3_agent.ini
[DEFAULT]
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
[agent]
[ovs]
[AGENT]
extensions = vpnaas

[vpnagent]
vpn_device_driver = neutron_vpnaas.services.vpn.device_drivers.libreswan_ipsec.LibreSwanDriver

## Populate database

neutron-db-manage --subproject neutron-vpnaas upgrade head
systemctl restart neutron-server
systemctl restart neutron-l3-agent

## Config site 2 site

# Create policy

openstack vpn ike policy create ikepolicy --auth-algorithm sha1 --encryption-algorithm aes-256 --ike-version v1 --lifetime value=86400 --phase1-negotiation-mode main --pfs group2
openstack vpn ipsec policy create ipsecpolicy --auth-algorithm sha1 --encryption-algorithm aes-256 --lifetime value=3600 --transform-protocol esp  --pfs group2

# R1

Router R1: 014d25b1-042e-47b0-872a-3d8f5a40361e
Sub R1: 6d4c1508-4a6b-4442-b3c1-d6bd1db3c268

openstack vpn service create vpn-r1 \
  --router 014d25b1-042e-47b0-872a-3d8f5a40361e \
  --subnet 6d4c1508-4a6b-4442-b3c1-d6bd1db3c268

openstack vpn ipsec site connection create conn-r1 \
  --vpnservice vpn-r1 \
  --ikepolicy ikepolicy \
  --ipsecpolicy ipsecpolicy \
  --peer-address 10.10.32.75 \
  --peer-id 10.10.32.75 \
  --peer-cidr 192.168.1.0/24 \
  --psk your_secret

# R2

Router R2: d1310774-6a8a-463c-a3d2-b02781d808d6
Sub R2: 9b532453-b2af-4b5f-acad-ae2c9287e8bc


openstack vpn service create vpn-r2 \
  --router d1310774-6a8a-463c-a3d2-b02781d808d6 \
  --subnet 9b532453-b2af-4b5f-acad-ae2c9287e8bc

openstack vpn ipsec site connection create conn-r2 \
  --vpnservice vpn-r2 \
  --ikepolicy ikepolicy \
  --ipsecpolicy ipsecpolicy \
  --peer-address 10.10.32.76 \
  --peer-id 10.10.32.76 \
  --peer-cidr 192.168.0.0/24 \
  --psk your_secret

# Some troubleshooting
openstack vpn service show vpn-r1
openstack vpn service show vpn-r2
openstack vpn ipsec site connection show conn-r1
openstack vpn ipsec site connection show conn-r2

openstack vpn ike policy list
openstack vpn ipsec policy list
openstack vpn service list
openstack vpn ipsec site connection list