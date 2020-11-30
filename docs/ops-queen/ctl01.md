# Hưỡng dẫn cài Openstack Queen với CentOS 7

---

## Phần 1: Chuẩn bị

### Mô hình

ctl01
- vlan mgnt: eth0: 10.10.30.70
- vlan provider: eth1: 10.10.31.70
- vlan datavm: eth2: 10.10.32.70

com01
- vlan mgnt: eth0: 10.10.30.71
- vlan provider: eth1: 10.10.31.71
- vlan datavm: eth2: 10.10.32.71

com02
- vlan mgnt: eth0: 10.10.30.72
- vlan provider: eth1: 10.10.31.72
- vlan datavm: eth2: 10.10.32.72

### CTL 1

```
yum install -y epel-release
yum install -y wget byobu git vim

hostnamectl set-hostname ctl01

echo "Setup IP eth0"
nmcli c modify eth0 ipv4.addresses 10.10.30.70/24
nmcli c modify eth0 ipv4.gateway 10.10.30.1
nmcli c modify eth0 ipv4.dns 8.8.8.8
nmcli c modify eth0 ipv4.method manual
nmcli con mod eth0 connection.autoconnect yes

echo "Setup IP eth1"
nmcli c modify eth1 ipv4.addresses 10.10.31.70/24
nmcli c modify eth1 ipv4.method manual
nmcli con mod eth1 connection.autoconnect yes

echo "Setup IP eth2"
nmcli c modify eth2 ipv4.addresses 10.10.32.70/24
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


echo "10.10.30.70 ctl01" >> /etc/hosts
echo "10.10.30.71 com01" >> /etc/hosts
echo "10.10.30.72 com02" >> /etc/hosts

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


init 6
```

## Phần 1: Cài đặt keystone


yum install mariadb mariadb-server -y 
cat << EOF >> /etc/my.cnf.d/openstack.cnf 
[mysqld]
bind-address = 10.10.30.70
        
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
EOF

systemctl enable mariadb.service
systemctl start mariadb.service


mysql_secure_installation <<EOF

y
passla123
passla123
y
y
y
y
EOF


## RabbitMQ

yum install rabbitmq-server -y
rabbitmq-plugins enable rabbitmq_management
systemctl restart rabbitmq-server
curl -O http://localhost:15672/cli/rabbitmqadmin
chmod a+x rabbitmqadmin
mv rabbitmqadmin /usr/sbin/
rabbitmqadmin list users


systemctl start rabbitmq-server
systemctl enable rabbitmq-server
systemctl status rabbitmq-server


rabbitmqctl add_user openstack passla123
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
systemctl enable rabbitmq-server.service
rabbitmqctl set_user_tags openstack administrator

rabbitmqadmin list users


## Cài đặt Memcached (Chỉ cấu hình trên Node Controller)


yum install memcached python-memcached -y 
sed -i "s/-l 127.0.0.1,::1/-l 10.10.30.70/g" /etc/sysconfig/memcached
systemctl enable memcached.service
systemctl start memcached.service


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

## Cài đặt Keystone (Service Identity) (Chỉ cấu hình trên Node Controller)

mysql -uroot -ppassla123
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost'  IDENTIFIED BY 'passla123';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'passla123';
FLUSH PRIVILEGES;
exit;


yum install -y openstack-keystone httpd mod_wsgi
mv /etc/keystone/keystone.{conf,conf.bk}

cat << EOF >> /etc/keystone/keystone.conf
[DEFAULT]
[assignment]
[auth]
[cache]
[catalog]
[cors]
[credential]
[database]
connection = mysql+pymysql://keystone:passla123@10.10.30.70/keystone
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
#driver = messagingv2
[oslo_messaging_rabbit]
#rabbit_retry_interval = 1
#rabbit_retry_backoff = 2
#amqp_durable_queues = true
#rabbit_ha_queues = true
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

keystone-manage bootstrap --bootstrap-password passla123 \
--bootstrap-admin-url http://10.10.30.70:5000/v3/ \
--bootstrap-internal-url http://10.10.30.70:5000/v3/ \
--bootstrap-public-url http://10.10.30.70:5000/v3/ \
--bootstrap-region-id RegionOne

sed -i 's|#ServerName www.example.com:80|ServerName 10.10.30.70|g' /etc/httpd/conf/httpd.conf 

ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
ls /etc/httpd/conf.d/

systemctl enable httpd.service
systemctl restart httpd.service
systemctl status httpd.service


cat << EOF >> admin-openrc
export export OS_REGION_NAME=RegionOne
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=passla123
export OS_AUTH_URL=http://10.10.30.70:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
export PS1='[\u@\h \W(admin-openrc-r1)]\$ '
EOF


cat << EOF >> demo-openrc
export export OS_REGION_NAME=RegionOne
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=passla123
export OS_AUTH_URL=http://10.10.30.70:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
export PS1='[\u@\h \W(demo-openrc-r1)]\$ '
EOF


source admin-openrc 
openstack project create --domain default --description "Service Project" service
openstack project create --domain default --description "Demo Project" demo
openstack user create --domain default --password passla123 demo
openstack role create user
openstack role add --project demo --user demo user


unset OS_AUTH_URL OS_PASSWORD
openstack --os-auth-url http://10.10.30.70:5000/v3 --os-project-domain-name Default \
--os-user-domain-name Default --os-project-name admin --os-username admin token issue


openstack --os-auth-url http://10.10.30.70:5000/v3 --os-project-domain-name default \
--os-user-domain-name default --os-project-name demo --os-username demo token issue

source admin-openrc 

openstack token issue


## Cài đặt Glance

mysql -uroot -ppassla123
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost'  IDENTIFIED BY 'passla123';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'passla123';
FLUSH PRIVILEGES;
exit;

source admin-openrc

openstack user create --domain default --password passla123 glance

openstack role add --project service --user glance admin
openstack role list --user glance --project service
openstack service create --name glance --description "OpenStack Image" image


openstack endpoint create --region RegionOne image public http://10.10.30.70:9292
openstack endpoint create --region RegionOne image internal http://10.10.30.70:9292
openstack endpoint create --region RegionOne image admin http://10.10.30.70:9292


yum install -y openstack-glance
mv /etc/glance/glance-api.{conf,conf.bk}

cat << EOF >> /etc/glance/glance-api.conf
[DEFAULT]
bind_host = 10.10.30.70
registry_host = 10.10.30.70
[cors]
[database]
connection = mysql+pymysql://glance:passla123@10.10.30.70/glance
[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
[image_format]
[keystone_authtoken]
auth_uri = http://10.10.30.70:5000
auth_url = http://10.10.30.70:5000
memcached_servers = 10.10.30.70:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = passla123
region_name = RegionOne
[matchmaker_redis]
[oslo_concurrency]
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
#driver = messagingv2
[oslo_messaging_rabbit]
#rabbit_ha_queues = true
#rabbit_retry_interval = 1
#rabbit_retry_backoff = 2
#amqp_durable_queues= true
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


chown root:glance /etc/glance/glance-api.conf
mv /etc/glance/glance-registry.{conf,conf.bk}

cat << EOF >> /etc/glance/glance-registry.conf
[DEFAULT]
bind_host = 10.10.30.70
[database]
connection = mysql+pymysql://glance:passla123@10.10.30.70/glance
[keystone_authtoken]
auth_uri = http://10.10.30.70:5000
auth_url = http://10.10.30.70:5000
memcached_servers = 10.10.30.70
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = passla123
region_name = RegionOne
[matchmaker_redis]
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
#driver = messagingv2
[oslo_messaging_rabbit]
#rabbit_ha_queues = true
#rabbit_retry_interval = 1
#rabbit_retry_backoff = 2
#amqp_durable_queues= true
[oslo_messaging_zmq]
[oslo_policy]
[paste_deploy]
flavor = keystone
[profiler]
EOF


chown root:glance /etc/glance/glance-registry.conf
su -s /bin/sh -c "glance-manage db_sync" glance

systemctl enable openstack-glance-api.service openstack-glance-registry.service
systemctl start openstack-glance-api.service openstack-glance-registry.service

wget http://download.cirros-cloud.net/0.3.5/cirros-0.3.5-x86_64-disk.img

openstack image create "cirros" --file cirros-0.3.5-x86_64-disk.img \
--disk-format qcow2 --container-format bare --public

openstack image list

## Nova

mysql -u root -ppassla123
CREATE DATABASE nova;
CREATE DATABASE nova_api;
CREATE DATABASE nova_cell0;
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost'  IDENTIFIED BY 'passla123';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'passla123';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost'  IDENTIFIED BY 'passla123';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'passla123';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost'  IDENTIFIED BY 'passla123';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'passla123';
FLUSH PRIVILEGES;
exit;

source admin-openrc
openstack user create --domain default --password passla123 nova
openstack role add --project service --user nova admin
openstack service create --name nova --description "OpenStack Compute" compute
openstack endpoint create --region RegionOne compute public http://10.10.30.70:8774/v2.1
openstack endpoint create --region RegionOne compute internal http://10.10.30.70:8774/v2.1
openstack endpoint create --region RegionOne compute admin http://10.10.30.70:8774/v2.1

openstack user create --domain default --password passla123 placement
openstack role add --project service --user placement admin
openstack service create --name placement --description "Placement API" placement

openstack endpoint create --region RegionOne placement public http://10.10.30.70:8778
openstack endpoint create --region RegionOne placement internal http://10.10.30.70:8778
openstack endpoint create --region RegionOne placement admin http://10.10.30.70:8778

yum install -y openstack-nova-api openstack-nova-conductor openstack-nova-console \
openstack-nova-novncproxy openstack-nova-scheduler openstack-nova-placement-api

mv /etc/nova/nova.{conf,conf.bk}

cat << EOF >> /etc/nova/nova.conf
[DEFAULT]
my_ip = 10.10.30.70
enabled_apis = osapi_compute,metadata
use_neutron = True
osapi_compute_listen=10.10.30.70
metadata_host=10.10.30.70
metadata_listen=10.10.30.70
metadata_listen_port=8775    
firewall_driver = nova.virt.firewall.NoopFirewallDriver
allow_resize_to_same_host=True
notify_on_state_change = vm_and_task_state
transport_url = rabbit://openstack:passla123@10.10.30.70:5672
[api]
auth_strategy = keystone
[api_database]
connection = mysql+pymysql://nova:passla123@10.10.30.70/nova_api
[barbican]
[cache]
backend = oslo_cache.memcache_pool
enabled = true
memcache_servers = 10.10.30.70:11211
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
connection = mysql+pymysql://nova:passla123@10.10.30.70/nova
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
[matchmaker_redis]
[metrics]
[mks]
[neutron]
region_name = RegionOne
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
discover_hosts_in_cells_interval = 300
[serial_console]
[service_user]
[spice]
[upgrade_levels]
[vault]
[vendordata_dynamic_auth]
[vmware]
[vnc]
novncproxy_host=10.10.30.70
enabled = true
vncserver_listen = 10.10.30.70
vncserver_proxyclient_address = 10.10.30.70
novncproxy_base_url = http://10.10.30.70:6080/vnc_auto.html
[workarounds]
[wsgi]
[xenserver]
[xvp]
EOF


chown root:nova /etc/nova/nova.conf

cp /etc/httpd/conf.d/00-nova-placement-api.{conf,conf.bk}

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

sed -i -e 's/VirtualHost \*/VirtualHost 10.10.30.70/g' /etc/httpd/conf.d/00-nova-placement-api.conf
sed -i -e 's/Listen 8778/Listen 10.10.30.70:8778/g' /etc/httpd/conf.d/00-nova-placement-api.conf

systemctl restart httpd 

su -s /bin/sh -c "nova-manage api_db sync" nova
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova 
su -s /bin/sh -c "nova-manage db sync" nova

nova-manage cell_v2 list_cells

systemctl enable openstack-nova-api.service openstack-nova-consoleauth.service \
openstack-nova-scheduler.service openstack-nova-conductor.service \
openstack-nova-novncproxy.service
systemctl start openstack-nova-api.service openstack-nova-consoleauth.service \
openstack-nova-scheduler.service openstack-nova-conductor.service \
openstack-nova-novncproxy.service

openstack compute service list

## Cài đặt Neutron (Service Network)

mysql -uroot -ppassla123
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'passla123';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'passla123';
FLUSH PRIVILEGES;
exit;

source admin-openrc
openstack user create --domain default --password passla123 neutron
openstack role add --project service --user neutron admin
openstack service create --name neutron --description "OpenStack Networking" network
openstack endpoint create --region RegionOne network public http://10.10.30.70:9696
openstack endpoint create --region RegionOne network internal http://10.10.30.70:9696
openstack endpoint create --region RegionOne network admin http://10.10.30.70:9696

yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables -y
mv /etc/neutron/neutron.{conf,conf.bk}

cat << EOF >> /etc/neutron/neutron.conf
[DEFAULT]
bind_host = 10.10.30.70
core_plugin = ml2
service_plugins = router
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
EOF

chown root:neutron /etc/neutron/neutron.conf

mv /etc/neutron/plugins/ml2/ml2_conf.{ini,ini.bk}


cat << EOF >> /etc/neutron/plugins/ml2/ml2_conf.ini
[DEFAULT]
[l2pop]
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
# mechanism_drivers = linuxbridge,l2population
mechanism_drivers = linuxbridge
extension_drivers = port_security
[ml2_type_flat]
flat_networks = provider
[ml2_type_geneve]
[ml2_type_gre]
[ml2_type_vlan]
# network_vlan_ranges = provider
[ml2_type_vxlan]
vni_ranges = 1:1000
[securitygroup]
enable_ipset = true
EOF

chown root:neutron /etc/neutron/plugins/ml2/ml2_conf.ini
mv /etc/neutron/plugins/ml2/linuxbridge_agent.{ini,init.bk}

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
local_ip = 10.10.31.70
# l2_population = true
EOF

chown root:neutron /etc/neutron/plugins/ml2/linuxbridge_agent.ini

mv /etc/neutron/l3_agent.{ini,ini.bk}

cat << EOF >> /etc/neutron/l3_agent.ini
[DEFAULT]
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
[agent]
[ovs]
EOF

chown root:neutron /etc/neutron/l3_agent.ini

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
service_metadata_proxy = true
metadata_proxy_shared_secret = passla123
region_name = RegionOne

## Cấu hình

ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
--config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron

systemctl restart openstack-nova-api.service openstack-nova-scheduler.service \
openstack-nova-consoleauth.service openstack-nova-conductor.service \
openstack-nova-novncproxy.service

systemctl enable neutron-server.service neutron-linuxbridge-agent.service \
neutron-l3-agent.service

systemctl start neutron-server.service neutron-linuxbridge-agent.service \
neutron-l3-agent.service

## 3.6 Cài đặt Horizon (Service Dashboard) (Chỉ cấu hình trên Node Controller)

yum install -y openstack-dashboard

filehtml=/var/www/html/index.html
touch $filehtml
cat << EOF >> $filehtml
<html>
<head>
<META HTTP-EQUIV="Refresh" Content="0.5; URL=http://10.10.30.70/dashboard">
</head>
<body>
<center> <h1>Redirecting to OpenStack Dashboard</h1> </center>
</body>
</html>
EOF

cp /etc/openstack-dashboard/{local_settings,local_settings.bk}

vi /etc/openstack-dashboard/local_settings
# line 38
ALLOWED_HOSTS = ['*',]
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 2,
}
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'Default'
    
# line 171 add
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': ['10.10.30.70:11211',]
    }
}

#line 205
OPENSTACK_HOST = "10.10.30.70"
OPENSTACK_KEYSTONE_URL = "http://10.10.30.70:5000/v3"
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

#line 481
TIME_ZONE = "Asia/Ho_Chi_Minh"

echo "WSGIApplicationGroup %{GLOBAL}" >> /etc/httpd/conf.d/openstack-dashboard.conf
systemctl restart httpd.service memcached.service

http://10.10.30.70
user: admin
password: passla123
