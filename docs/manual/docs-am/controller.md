echo "172.16.4.200 controller" >> /etc/hosts
echo "172.16.4.201 compute" >> /etc/hosts
echo "172.16.4.202 compute2" >> /etc/hosts

yum -y install centos-release-openstack-queens
yum -y install crudini wget vim
yum -y install python-openstackclient openstack-selinux python2-PyMySQL


yum install -y memcached
sed -i "s/-l 127.0.0.1,::1/-l 172.16.4.200/g" /etc/sysconfig/memcached


systemctl enable memcached.service
systemctl restart memcached.service

yum install mariadb mariadb-server python2-PyMySQL -y

cp /etc/my.cnf.d/server.cnf /etc/my.cnf.d/server.cnf.orig
rm -rf /etc/my.cnf.d/server.cnf

cat << EOF > /etc/my.cnf.d/openstack.cnf
[mysqld]
bind-address = 172.16.4.200
default-storage-engine = innodb
innodb_file_per_table
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
EOF

systemctl enable mariadb.service
systemctl start mariadb.service


mysql_secure_installation

Enter current password for root (enter for none): [enter]
Change the root password? [Y/n]: y
Set root password? [Y/n] y
New password:Welcome123
Re-enter new password:Welcome123
Remove anonymous users? [Y/n]: y
Disallow root login remotely? [Y/n]: y
Remove test database and access to it? [Y/n]: y
Reload privilege tables now? [Y/n]: y

yum -y install rabbitmq-server

systemctl enable rabbitmq-server.service
systemctl start rabbitmq-server.service
rabbitmq-plugins enable rabbitmq_management
systemctl restart rabbitmq-server
curl -O http://localhost:15672/cli/rabbitmqadmin
chmod a+x rabbitmqadmin
mv rabbitmqadmin /usr/sbin/
rabbitmqadmin list users

rabbitmqctl add_user openstack Welcome123
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
rabbitmqctl set_user_tags openstack administrator


Cài đặt các thành phần OpenStack

mysql -u root -pWelcome123
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'Welcome123';
exit

yum install openstack-keystone httpd mod_wsgi -y

cp /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
sed -i -e 's/VirtualHost \*/VirtualHost 172.16.4.200/g' /etc/httpd/conf.d/wsgi-keystone.conf
sed -i -e 's/Listen 5000/Listen 172.16.4.200:5000/g' /etc/httpd/conf.d/wsgi-keystone.conf
sed -i -e 's/Listen 35357/Listen 172.16.4.200:35357/g' /etc/httpd/conf.d/wsgi-keystone.conf
sed -i -e 's/^Listen.*/Listen 172.16.4.200:80/g' /etc/httpd/conf/httpd.conf

cp /etc/keystone/keystone.conf /etc/keystone/keystone.conf.org
rm -rf /etc/keystone/keystone.conf

cat << EOF >> /etc/keystone/keystone.conf
[DEFAULT]
[assignment]
[auth]
[cache]
[catalog]
[cors]
[credential]
[database]
connection = mysql+pymysql://keystone:Welcome123@172.16.4.200/keystone
[domain_config]
[endpoint_filter]
[endpoint_policy]
[eventlet_server]
[federation]
[fernet_tokens]
[healthcheck]
[identity]
[identity_mapping]
[ldap]
[matchmaker_redis]
[memcache]
[oauth1]
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[paste_deploy]
[policy]
[profiler]
[resource]
[revoke]
[role]
[saml]
[security_compliance]
[shadow_users]
[signing]
[token]
provider = fernet
[tokenless_auth]
[trust]
EOF

chown root:keystone /etc/keystone/keystone.conf

su -s /bin/sh -c "keystone-manage db_sync" keystone

keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

keystone-manage bootstrap --bootstrap-password Welcome123 \
  --bootstrap-admin-url http://172.16.4.200:5000/v3/ \
  --bootstrap-internal-url http://172.16.4.200:5000/v3/ \
  --bootstrap-public-url http://172.16.4.200:5000/v3/ \
  --bootstrap-region-id RegionOne

systemctl enable httpd.service
systemctl restart httpd.service


export OS_USERNAME=admin
export OS_PASSWORD=Welcome123
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://172.16.4.200:35357/v3
export OS_IDENTITY_API_VERSION=3

openstack domain create --description "An Example Domain" example
openstack project create --domain default --description "Service Project" service


openstack project create --domain default  --description "Demo Project" demo
openstack user create --domain default --password Welcome123 demo

openstack role create user
openstack role add --project demo --user demo user

unset OS_AUTH_URL OS_PASSWORD


openstack --os-auth-url http://172.16.4.200:35357/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name admin --os-username admin token issue
Password: Welcome123

cat << EOF >> admin-openrc
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=Welcome123
export OS_AUTH_URL=http://172.16.4.200:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOF

cat << EOF >> demo-openrc
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=Welcome123
export OS_AUTH_URL=http://172.16.4.200:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOF

. admin-openrc
openstack token issue

mysql -u root -pWelcome123
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'Welcome123';
exit

openstack user create --domain default --password Welcome123 glance
openstack role add --project service --user glance admin
openstack service create --name glance --description "OpenStack Image" image

openstack endpoint create --region RegionOne image public http://172.16.4.200:9292
openstack endpoint create --region RegionOne image admin http://172.16.4.200:9292
openstack endpoint create --region RegionOne image internal http://172.16.4.200:9292
 
yum install -y openstack-glance

cp /etc/glance/glance-api.conf /etc/glance/glance-api.conf.org 
rm -rf /etc/glance/glance-api.conf


cat << EOF >> /etc/glance/glance-api.conf
[DEFAULT]
bind_host = 172.16.4.200
registry_host = 172.16.4.200
[cors]
[database]
connection = mysql+pymysql://glance:Welcome123@172.16.4.200/glance
[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
[image_format]
[keystone_authtoken]
auth_uri = http://172.16.4.200:5000
auth_url = http://172.16.4.200:5000
memcached_servers = 172.16.4.200:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = Welcome123
[matchmaker_redis]
[oslo_concurrency]
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[paste_deploy]
flavor = keystone
[profiler]
[store_type_location_strategy]
[task]
[taskflow_executor]
EOF

cp /etc/glance/glance-registry.conf /etc/glance/glance-registry.conf.org
rm -rf /etc/glance/glance-registry.conf

cat << EOF >> /etc/glance/glance-registry.conf
[DEFAULT]
bind_host = 172.16.4.200
[database]
connection = mysql+pymysql://glance:Welcome123@172.16.4.200/glance
[keystone_authtoken]
auth_uri = http://172.16.4.200:5000
auth_url = http://172.16.4.200:5000
memcached_servers = 172.16.4.200
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = Welcome123
[matchmaker_redis]
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_policy]
[paste_deploy]
flavor = keystone
[profiler]
EOF

chown root:glance /etc/glance/glance-api.conf
chown root:glance /etc/glance/glance-registry.conf

su -s /bin/sh -c "glance-manage db_sync" glance

systemctl enable openstack-glance-api.service openstack-glance-registry.service
systemctl start openstack-glance-api.service openstack-glance-registry.service

wget http://download.cirros-cloud.net/0.3.5/cirros-0.3.5-x86_64-disk.img
openstack image create "cirros" \
  --file cirros-0.3.5-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --public

mysql -u root -pWelcome123
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'Welcome123';
exit

openstack user create --domain default --password Welcome123 nova
openstack role add --project service --user nova admin
openstack service create --name nova --description "OpenStack Compute" compute


openstack endpoint create --region RegionOne compute public http://172.16.4.200:8774/v2.1
openstack endpoint create --region RegionOne compute admin http://172.16.4.200:8774/v2.1
openstack endpoint create --region RegionOne compute internal http://172.16.4.200:8774/v2.1

openstack user create --domain default --password Welcome123 placement
openstack role add --project service --user placement admin
openstack service create --name placement --description "Placement API" placement

openstack endpoint create --region RegionOne placement public http://172.16.4.200:8778
openstack endpoint create --region RegionOne placement admin http://172.16.4.200:8778
openstack endpoint create --region RegionOne placement internal http://172.16.4.200:8778

yum install -y openstack-nova-api openstack-nova-conductor openstack-nova-console openstack-nova-novncproxy openstack-nova-scheduler openstack-nova-placement-api

cp /etc/nova/nova.conf /etc/nova/nova.conf.org 
rm -rf /etc/nova/nova.conf

cat << EOF >> /etc/nova/nova.conf
[DEFAULT]
my_ip = 172.16.4.200
enabled_apis = osapi_compute,metadata
use_neutron = True
osapi_compute_listen=172.16.4.200
metadata_host=172.16.4.200
metadata_listen=172.16.4.200
metadata_listen_port=8775
firewall_driver = nova.virt.firewall.NoopFirewallDriver
transport_url = rabbit://openstack:Welcome123@172.16.4.200:5672
[api]
auth_strategy = keystone
[api_database]
connection = mysql+pymysql://nova:Welcome123@172.16.4.200/nova_api
[barbican]
[cache]
backend = oslo_cache.memcache_pool
enabled = true
memcache_servers = 172.16.4.200:11211
[cells]
[cinder]
[compute]
[conductor]
[console]
[consoleauth]
[cors]
[crypto]
[database]
connection = mysql+pymysql://nova:Welcome123@172.16.4.200/nova
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
discover_hosts_in_cells_interval = 300
[serial_console]
[service_user]
[spice]
[upgrade_levels]
[vault]
[vendordata_dynamic_auth]
[vmware]
[vnc]
novncproxy_host=172.16.4.200
enabled = true
vncserver_listen = 172.16.4.200
vncserver_proxyclient_address = 172.16.4.200
novncproxy_base_url = http://172.16.4.200:6080/vnc_auto.html
[workarounds]
[wsgi]
[xenserver]
[xvp]
EOF


cat << 'EOF' >> /etc/httpd/conf.d/00-nova-placement-api.conf

<Directory /usr/bin>
   <IfVersion >= 2.4>
      Require all granted
   </IfVersion>
   <IfVersion < 2.4>
      Order allow,deny
      Allow from all
   </IfVersion>
</Directory>
EOF

sed -i -e 's/VirtualHost \*/VirtualHost 172.16.4.200/g' /etc/httpd/conf.d/00-nova-placement-api.conf
sed -i -e 's/Listen 8778/Listen 172.16.4.200:8778/g' /etc/httpd/conf.d/00-nova-placement-api.conf


systemctl restart httpd

su -s /bin/sh -c "nova-manage api_db sync" nova
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
su -s /bin/sh -c "nova-manage db sync" nova

systemctl enable openstack-nova-api.service openstack-nova-scheduler.service openstack-nova-consoleauth.service openstack-nova-conductor.service openstack-nova-novncproxy.service
systemctl start openstack-nova-api.service openstack-nova-scheduler.service openstack-nova-consoleauth.service openstack-nova-conductor.service openstack-nova-novncproxy.service

openstack compute service list

mysql -u root -pWelcome123
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'Welcome123';
exit

openstack user create --domain default --password Welcome123 neutron
openstack role add --project service --user neutron admin
openstack service create --name neutron --description "OpenStack Networking" network

openstack endpoint create --region RegionOne network public http://172.16.4.200:9696
openstack endpoint create --region RegionOne network internal http://172.16.4.200:9696
openstack endpoint create --region RegionOne network admin http://172.16.4.200:9696

yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-openvswitch ebtables -y

cp /etc/neutron/neutron.conf /etc/neutron/neutron.conf.org
rm -rf /etc/neutron/neutron.conf

cat << EOF >> /etc/neutron/neutron.conf
[DEFAULT]
bind_host = 172.16.4.200
core_plugin = ml2
service_plugins = router
transport_url = rabbit://openstack:Welcome123@172.16.4.200:5672
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
allow_overlapping_ips = True
dhcp_agents_per_network = 2
[agent]
[cors]
[database]
connection = mysql+pymysql://neutron:Welcome123@172.16.4.200/neutron
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
auth_url = http://172.16.4.200:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = Welcome123
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
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
EOF

cp /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/ml2_conf.ini.org
rm -rf /etc/neutron/plugins/ml2/ml2_conf.ini

cat << EOF >> /etc/neutron/plugins/ml2/ml2_conf.ini
[DEFAULT]
[l2pop]
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = openvswitch,l2population
extension_drivers = port_security
[ml2_type_flat]
flat_networks = provider
[ml2_type_geneve]
[ml2_type_gre]
[ml2_type_vlan]
[ml2_type_vxlan]
vni_ranges = 1:1000
[securitygroup]
enable_ipset = True
EOF

cp /etc/neutron/plugins/ml2/openvswitch_agent.ini /etc/neutron/plugins/ml2/openvswitch_agent.ini.org 
rm -rf /etc/neutron/plugins/ml2/openvswitch_agent.ini

cat << EOF >> /etc/neutron/plugins/ml2/openvswitch_agent.ini
[DEFAULT]
interface_driver = openvswitch
[agent]
tunnel_types=vxlan
l2_population = True
[network_log]
[ovs]
bridge_mappings = provider:br-provider
local_ip=172.16.4.200
[securitygroup]
enable_security_group = True
firewall_driver = iptables_hybrid
[xenapi]
EOF

cp /etc/neutron/l3_agent.ini /etc/neutron/l3_agent.ini.org
rm -rf /etc/neutron/l3_agent.ini

cat << EOF >> /etc/neutron/l3_agent.ini
[DEFAULT]
interface_driver = openvswitch
external_network_bridge = 
[agent]
[ovs]
EOF


/etc/nova/nova.conf

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
service_metadata_proxy = true
metadata_proxy_shared_secret = Welcome123

systemctl restart openstack-nova-api.service


chown -R root:neutron /etc/neutron/

ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini

su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron

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
IPADDR=172.16.9.10
NETMASK=255.255.255.0
DEVICE=br-provider
NAME=br-provider
DEVICETYPE=ovs
OVSBOOTPROTO=none
TYPE=OVSBridge
EOF

systemctl restart network

systemctl enable neutron-server.service neutron-openvswitch-agent.service neutron-l3-agent.service
systemctl start neutron-server.service neutron-openvswitch-agent.service neutron-l3-agent.service


yum install openstack-dashboard -y


filehtml=/var/www/html/index.html
touch $filehtml
cat << EOF >> $filehtml
<html>
<head>
<META HTTP-EQUIV="Refresh" Content="0.5; URL=http://172.16.4.200/dashboard">
</head>
<body>
<center> <h1>Redirecting to OpenStack Dashboard</h1> </center>
</body>
</html>
EOF

cp /etc/openstack-dashboard/local_settings /etc/openstack-dashboard/local_settings.org


/etc/openstack-dashboard/local_settings

ALLOWED_HOSTS = ['*',]
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 2,
}
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'Default'

SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': ['172.16.4.200:11211',]
    }
}

OPENSTACK_HOST = "172.16.4.200"
OPENSTACK_KEYSTONE_URL = "http://172.16.4.200:5000/v3"
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

OPENSTACK_NEUTRON_NETWORK = {
    'enable_router': False,
    'enable_quotas': False,
    'enable_ipv6': False,
    'enable_distributed_router': False,
    'enable_ha_router': False,
    'enable_fip_topology_check': False,
}

TIME_ZONE = "Asia/Ho_Chi_Minh"

echo "WSGIApplicationGroup %{GLOBAL}" >> /etc/httpd/conf.d/openstack-dashboard.conf


mysql -u root -pWelcome123
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'Welcome123';  
exit

openstack user create --domain default --password Welcome123 cinder
openstack role add --project service --user cinder admin
openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2
openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3

openstack endpoint create --region RegionOne volumev2 public http://172.16.4.200:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionOne volumev2 internal http://172.16.4.200:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionOne volumev2 admin http://172.16.4.200:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 public http://172.16.4.200:8776/v3/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 internal http://172.16.4.200:8776/v3/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 admin http://172.16.4.200:8776/v3/%\(project_id\)s


yum install openstack-cinder targetcli python-keystone -y

yum install -y lvm2

systemctl enable lvm2-lvmetad.service
systemctl start lvm2-lvmetad.service

edit: /etc/lvm/lvm.conf
devices {
...
filter = [ "a/sdb/", "r/.*/"]

cp /etc/cinder/cinder.conf /etc/cinder/cinder.conf.bak 
rm -rf /etc/cinder/cinder.conf


cat << EOF >> /etc/cinder/cinder.conf
[DEFAULT]
my_ip = 172.16.4.200
transport_url = rabbit://openstack:Welcome123@172.16.4.200:5672
auth_strategy = keystone
osapi_volume_listen = 172.16.4.200
enabled_backends = lvm
[backend]
[backend_defaults]
[barbican]
[brcd_fabric_example]
[cisco_fabric_example]
[coordination]
[cors]
[database]
connection = mysql+pymysql://cinder:Welcome123@172.16.4.200/cinder
[fc-zone-manager]
[healthcheck]
[key_manager]
[keystone_authtoken]
auth_uri = http://172.16.4.200:5000
auth_url = http://172.16.4.200:35357
memcached_servers = 172.16.4.200:11211
auth_type = password
project_domain_id = default
user_domain_id = default
project_name = service
username = cinder
password = Welcome123
[matchmaker_redis]
[nova]
[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
rabbit_retry_interval = 1
rabbit_retry_backoff = 2
amqp_durable_queues = true
rabbit_ha_queues = true
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[oslo_reports]
[oslo_versionedobjects]
[profiler]
[service_user]
[ssl]
[vault]
[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
iscsi_protocol = iscsi
iscsi_helper = lioadm
volume_backend_name = lvm
EOF 

chown root:cinder /etc/cinder/cinder.conf

su -s /bin/sh -c "cinder-manage db sync" cinder

edit:  /etc/nova/nova.conf
[cinder]
os_region_name = RegionOne

systemctl restart openstack-nova-api.service

systemctl enable openstack-cinder-api.service openstack-cinder-volume.service openstack-cinder-scheduler.service
systemctl start openstack-cinder-api.service openstack-cinder-volume.service openstack-cinder-scheduler.service
