# Cài đặt CTL 3

## Phần 1: Chuẩn bị

### Phân hoạch

Quy hoạch Network:
- VIP MNGT: 10.10.11.94
- vlan mgnt: eth0: 10.10.11.89
- vlan provider: eth1: 10.10.12.89
- vlan datavm: eth2: 10.10.14.89

### Setup node

```
hostnamectl set-hostname ctl03

echo "Setup IP eth0"
nmcli c modify eth0 ipv4.addresses 10.10.11.89/24
nmcli c modify eth0 ipv4.gateway 10.10.11.1
nmcli c modify eth0 ipv4.dns 8.8.8.8
nmcli c modify eth0 ipv4.method manual
nmcli con mod eth0 connection.autoconnect yes

echo "Setup IP eth1"
nmcli c modify eth1 ipv4.addresses 10.10.12.89/24
nmcli c modify eth1 ipv4.method manual
nmcli con mod eth1 connection.autoconnect yes

echo "Setup IP eth2"
nmcli c modify eth2 ipv4.addresses 10.10.14.89/24
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
```

### Chuẩn bị sysctl
```
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
```

### Cấu hình hostname
```
echo "10.10.11.87 ctl01" >> /etc/hosts
echo "10.10.11.88 ctl02" >> /etc/hosts
echo "10.10.11.89 ctl03" >> /etc/hosts
echo "10.10.11.94 com01" >> /etc/hosts
```

### Setup keypair
```
ssh-keygen -t rsa -f /root/.ssh/id_rsa -q -P ""
ssh-copy-id -o StrictHostKeyChecking=no -i /root/.ssh/id_rsa.pub root@ctl01
ssh-copy-id -o StrictHostKeyChecking=no -i /root/.ssh/id_rsa.pub root@ctl02
```
Lưu ý:
- Snapshot preenv

## Phần 2: Setup Galera

### Setup repo và Cài đặt MariaDB (Trên tất cả CTL)
```
echo '[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.2/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1' >> /etc/yum.repos.d/MariaDB.repo
yum -y update

yum install -y mariadb mariadb-server
yum install -y galera rsync
systemctl stop mariadb
```
### Cấu hình
```
cp /etc/my.cnf.d/server.cnf /etc/my.cnf.d/server.cnf.bak

echo '[server]
[mysqld]
bind-address=10.10.11.89

[galera]
wsrep_on=ON
wsrep_provider=/usr/lib64/galera/libgalera_smm.so
wsrep_cluster_address="gcomm://10.10.11.87,10.10.11.88,10.10.11.89"
binlog_format=row
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
wsrep_cluster_name="galera_cluster"
bind-address=10.10.11.89
wsrep_node_address="10.10.11.89"
wsrep_node_name="ctl03"
wsrep_sst_method=rsync

skip-name-resolve
max_connections = 10240
innodb_locks_unsafe_for_binlog=1
query_cache_size=0
query_cache_type=0
innodb_log_file_size=100M
innodb_file_per_table
innodb_flush_log_at_trx_commit=2
[embedded]
[mariadb]
[mariadb-10.2]
' > /etc/my.cnf.d/server.cnf
```

### Khởi tạo Cluster

Lưu ý:
- Chỉ chạy khi đã khởi tạo galera_new_cluster tại CTL1

```
systemctl start mariadb
systemctl enable mariadb
```

Lưu ý:
- Trở lại CTL1 và chờ tới thao tác tiếp theo

### Cấu hình cho haproxy check mysql
```
yum install rsync xinetd crudini git -y
git clone https://github.com/thaonguyenvan/percona-clustercheck
cp percona-clustercheck/clustercheck /usr/local/bin


cat << EOF >> /etc/xinetd.d/mysqlchk
service mysqlchk
{
      disable = no
      flags = REUSE
      socket_type = stream
      port = 9200
      wait = no
      user = nobody
      server = /usr/local/bin/clustercheck
      log_on_failure += USERID
      only_from = 0.0.0.0/0
      per_source = UNLIMITED
}
EOF
```

### Tạo service
```
echo 'mysqlchk 9200/tcp # MySQL check' >> /etc/services
```

### Bật xinetd
```
systemctl start xinetd
systemctl enable xinetd
```

Kết quả mẫu
```
[root@ctl02 ~]# clustercheck
HTTP/1.1 200 OK
Content-Type: text/plain
Connection: close
Content-Length: 40

Percona XtraDB Cluster Node is synced.
```

Lưu ý:
- Trở lại CTL1 và chờ tới thao tác tiếp theo

## Phần 3: Cài đặt RabbitMQ Cluster 

### Cài đặt môi trường và RabbitMQ (Trên tất cả CTL)
```
yum -y install epel-release
yum update -y
yum -y install erlang socat wget

wget https://www.rabbitmq.com/releases/rabbitmq-server/v3.6.10/rabbitmq-server-3.6.10-1.el7.noarch.rpm
rpm --import https://www.rabbitmq.com/rabbitmq-release-signing-key.asc
rpm -Uvh rabbitmq-server-3.6.10-1.el7.noarch.rpm

systemctl start rabbitmq-server
systemctl enable rabbitmq-server
systemctl status rabbitmq-server

rabbitmq-plugins enable rabbitmq_management
chown -R rabbitmq:rabbitmq /var/lib/rabbitmq/
```

### Join Cluster RabbitMQ
```
chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie
chmod 400 /var/lib/rabbitmq/.erlang.cookie

systemctl restart rabbitmq-server.service

rabbitmqctl stop_app
rabbitmqctl join_cluster rabbit@ctl01
rabbitmqctl start_app
```

Lưu ý:
- Trở lại CTL1 và chờ tới thao tác tiếp theo

## Phần 4: Triển khai PCS

### Chuẩn bị môi trường (Trên tất cả các node)
```
yum install pacemaker corosync haproxy pcs fence-agents-all resource-agents psmisc policycoreutils-python -y

echo Welcome123 | passwd --stdin hacluster

systemctl enable pcsd.service pacemaker.service corosync.service haproxy.service

systemctl start pcsd.service
```

Sau bước này tới CTL2 và CTL3 join cluster RabbitMQ

### Config HAProxy
```
cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.org 
rm -rf /etc/haproxy/haproxy.cfg
cat >> /etc/haproxy/haproxy.cfg << EOF
global
    daemon
    group  haproxy
    log  /dev/log local0
    log /dev/log    local1 notice
    maxconn  16000
    pidfile  /var/run/haproxy.pid
    stats  socket /var/lib/haproxy/stats
    tune.bufsize  32768
    tune.maxrewrite  1024
    user  haproxy

  
defaults
    log  global
    maxconn  8000
    mode  http
    option  redispatch
    option  http-server-close
    option  splice-auto
    retries  3
    timeout  http-request 20s
    timeout  queue 1m
    timeout  connect 10s
    timeout  client 1m
    timeout  server 1m
    timeout  check 10s

listen stats
    bind 10.10.11.93:8080
    mode http
    stats enable
    stats uri /stats
    stats realm HAProxy\ Statistics

listen mysqld 
    bind 10.10.11.93:3306
    balance  leastconn
    mode  tcp
    option  httpchk
    option  tcplog
    option  clitcpka
    option  srvtcpka
    timeout client  28801s
    timeout server  28801s    
    server controller1 10.10.11.87:3306 check port 9200 inter 5s fastinter 2s rise 3 fall 3 
    server controller2 10.10.11.88:3306 check port 9200 inter 5s fastinter 2s rise 3 fall 3 backup
    server controller3 10.10.11.89:3306 check port 9200 inter 5s fastinter 2s rise 3 fall 3 backup


listen keystone-5000
    bind 10.10.11.93:5000 
    option  httpchk
    option  httplog
    option  httpclose
    balance source
    server controller1 10.10.11.87:5000  check inter 5s fastinter 2s downinter 2s rise 3 fall 3
    server controller2 10.10.11.88:5000  check inter 5s fastinter 2s downinter 2s rise 3 fall 3
    server controller3 10.10.11.89:5000  check inter 5s fastinter 2s downinter 2s rise 3 fall 3

listen keystone-35357
    bind 10.10.11.93:35357
    option  httpchk
    option  httplog
    option  httpclose
    balance source
    server controller1 10.10.11.87:35357  check inter 5s fastinter 2s downinter 2s rise 3 fall 3
    server controller2 10.10.11.88:35357  check inter 5s fastinter 2s downinter 2s rise 3 fall 3
    server controller3 10.10.11.89:35357  check inter 5s fastinter 2s downinter 2s rise 3 fall 3


listen nova-api-8774
    bind 10.10.11.93:8774 
    option  httpchk
    option  httplog
    option  httpclose
    timeout server  600s
    server controller1 10.10.11.87:8774  check inter 5s fastinter 2s downinter 3s rise 3 fall 3
    server controller2 10.10.11.88:8774  check inter 5s fastinter 2s downinter 3s rise 3 fall 3
    server controller3 10.10.11.89:8774  check inter 5s fastinter 2s downinter 3s rise 3 fall 3

listen nova-metadata-api
    bind 10.10.11.93:8775 
    option  httpchk
    option  httplog
    option  httpclose
    server controller1 10.10.11.87:8775  check inter 5s fastinter 2s downinter 3s rise 3 fall 3
    server controller2 10.10.11.88:8775  check inter 5s fastinter 2s downinter 3s rise 3 fall 3
    server controller3 10.10.11.89:8775  check inter 5s fastinter 2s downinter 3s rise 3 fall 3

listen nova-novncproxy
	bind 10.10.11.93:6080
    balance  source
    option  httplog
    server controller1 10.10.11.87:6080  check
    server controller2 10.10.11.88:6080  check
    server controller3 10.10.11.89:6080  check
    
listen nova_placement_api
    bind 10.10.11.93:8778
    balance source
    option tcpka
    option tcplog
    http-request del-header X-Forwarded-Proto
    server controller1 10.10.11.87:8778 check inter 2000 rise 2 fall 5
    server controller2 10.10.11.88:8778 check inter 2000 rise 2 fall 5
    server controller3 10.10.11.89:8778 check inter 2000 rise 2 fall 5    
    
listen glance-api
    bind 10.10.11.93:9292
    option  httpchk /versions
    option  httplog
    option  httpclose
    timeout server  11m
    server controller1 10.10.11.87:9292  check inter 5s fastinter 2s downinter 3s rise 3 fall 3
    server controller2 10.10.11.88:9292  check inter 5s fastinter 2s downinter 3s rise 3 fall 3
    server controller3 10.10.11.89:9292  check inter 5s fastinter 2s downinter 3s rise 3 fall 3

listen glance-registry
    bind 10.10.11.93:9191 
    timeout server  11m
    server controller1 10.10.11.87:9191  check
    server controller2 10.10.11.88:9191  check
    server controller3 10.10.11.89:9191  check

listen neutron
    bind 10.10.11.93:9696 
    option  httpchk
    option  httplog
    option  httpclose
    balance source
    server controller1 10.10.11.87:9696  check inter 5s fastinter 2s downinter 3s rise 3 fall 3
    server controller2 10.10.11.88:9696  check inter 5s fastinter 2s downinter 3s rise 3 fall 3
    server controller3 10.10.11.89:9696  check inter 5s fastinter 2s downinter 3s rise 3 fall 3

listen cinder-api
    bind 10.10.11.93:8776 
    option  httpchk
    option  httplog
    option  httpclose
    server controller1 10.10.11.87:8776  check inter 5s fastinter 2s downinter 3s rise 3 fall 3
    server controller2 10.10.11.88:8776 backup check inter 5s fastinter 2s downinter 3s rise 3 fall 3
    server controller3 10.10.11.89:8776 backup check inter 5s fastinter 2s downinter 3s rise 3 fall 3

  
listen horizon
    bind 10.10.11.93:80
    balance  source
    mode  http
    option  forwardfor
    option  httpchk
    option  httpclose
    option  httplog
    stick  on src
    stick-table  type ip size 200k expire 30m
    timeout  client 3h
    timeout  server 3h
    server controller1 10.10.11.87:80  weight 1 check
    server controller2 10.10.11.88:80  weight 1 check
    server controller3 10.10.11.89:80  weight 1 check
EOF
```

Sau bước này trở lại CTL 1

## Phần 5: Cài đặt các gói cần thiết cho OPS (Trên tất cả các node)

```
yum -y install centos-release-openstack-queens
yum -y install crudini wget vim
yum -y install python-openstackclient openstack-selinux python2-PyMySQL
```

Sau bước này trở lại CTL 1

## Phần 6: Cấu hình memcache

```
yum install -y memcached

sed -i "s/-l 127.0.0.1,::1/-l 10.10.11.89/g" /etc/sysconfig/memcached

systemctl enable memcached.service
systemctl restart memcached.service
```

Sau bước này trở lại CTL 1

## Phần 7: Cài đặt Keystone

### Cài packages
```
yum install openstack-keystone httpd mod_wsgi -y
```

### Cấu hình bind port

```
cp /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
sed -i -e 's/VirtualHost \*/VirtualHost 10.10.11.89/g' /etc/httpd/conf.d/wsgi-keystone.conf
sed -i -e 's/Listen 5000/Listen 10.10.11.89:5000/g' /etc/httpd/conf.d/wsgi-keystone.conf
sed -i -e 's/Listen 35357/Listen 10.10.11.89:35357/g' /etc/httpd/conf.d/wsgi-keystone.conf
sed -i -e 's/^Listen.*/Listen 10.10.11.89:80/g' /etc/httpd/conf/httpd.conf
```

### Cấu hình dịch vụ
```
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
connection = mysql+pymysql://keystone:Welcome123@10.10.11.93/keystone
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
chown -R keystone:keystone /etc/keystone/credential-keys /etc/keystone/fernet-keys

systemctl enable httpd.service
systemctl start httpd.service
```

Sau bước này trở lại CTL 1

## Phần 8: Cài đặt Glance

### Cài packages (Thực hiện trên tất cả CTL)
```
yum install -y openstack-glance
```

### Cấu hình glance-api

```
cp /etc/glance/glance-api.conf /etc/glance/glance-api.conf.org 
rm -rf /etc/glance/glance-api.conf

cat << EOF >> /etc/glance/glance-api.conf
[DEFAULT]
bind_host = 10.10.11.89
registry_host = 10.10.11.93
[cors]
[database]
connection = mysql+pymysql://glance:Welcome123@10.10.11.93/glance
[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
[image_format]
[keystone_authtoken]
auth_uri = http://10.10.11.93:5000
auth_url = http://10.10.11.93:5000
memcached_servers = 10.10.11.87:11211,10.10.11.88:11211,10.10.11.89:11211
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
```

### Cấu hình glance-registry
```
cp /etc/glance/glance-registry.conf /etc/glance/glance-registry.conf.org
rm -rf /etc/glance/glance-registry.conf
cat << EOF >> /etc/glance/glance-registry.conf
[DEFAULT]
bind_host = 10.10.11.89
[database]
connection = mysql+pymysql://glance:Welcome123@10.10.11.93/glance
[keystone_authtoken]
auth_uri = http://10.10.11.93:5000
auth_url = http://10.10.11.93:5000
memcached_servers = 10.10.11.87:11211,10.10.11.88:11211,10.10.11.89:11211
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
```

### Phân quyền
```
chown root:glance /etc/glance/glance-api.conf
chown root:glance /etc/glance/glance-registry.conf
```

### Enable và start dịch vụ
Lưu ý: Yêu cầu thực hiện xong bước sync db trên node controller1

```
systemctl enable openstack-glance-api.service \
  openstack-glance-registry.service

systemctl start openstack-glance-api.service \
  openstack-glance-registry.service
```

Lưu ý: Thực hiện sau khi up image sau khi scp Image giữa các CTL
```
chown -R glance:glance /var/lib/glance/images
```

Sau bước này trở lại CTL 1

## Phần 9: Cài đặt Nova

### Cài gói (Thực hiện trên tất cả CTL)
```
yum install -y openstack-nova-api openstack-nova-conductor \
  openstack-nova-console openstack-nova-novncproxy \
  openstack-nova-scheduler openstack-nova-placement-api
```

### Cấu hình nova

Lưu ý: Yêu cầu thực hiện xong bước tạo endpoint trên node controller1

```
cp /etc/nova/nova.conf /etc/nova/nova.conf.org 
rm -rf /etc/nova/nova.conf

cat << EOF >> /etc/nova/nova.conf
[DEFAULT]
my_ip = 10.10.11.89
enabled_apis = osapi_compute,metadata
use_neutron = True
osapi_compute_listen=10.10.11.89
metadata_host=10.10.11.89
metadata_listen=10.10.11.89
metadata_listen_port=8775
firewall_driver = nova.virt.firewall.NoopFirewallDriver
transport_url = rabbit://openstack:Welcome123@10.10.11.87:5672,openstack:Welcome123@10.10.11.88:5672,openstack:Welcome123@10.10.11.89:5672
[api]
auth_strategy = keystone
[api_database]
connection = mysql+pymysql://nova:Welcome123@10.10.11.93/nova_api
[barbican]
[cache]
backend = oslo_cache.memcache_pool
enabled = true
memcache_servers = 10.10.11.87:11211,10.10.11.88:11211,10.10.11.89:11211
[cells]
[cinder]
[compute]
[conductor]
[console]
[consoleauth]
[cors]
[crypto]
[database]
connection = mysql+pymysql://nova:Welcome123@10.10.11.93/nova
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
discover_hosts_in_cells_interval = 300
[serial_console]
[service_user]
[spice]
[upgrade_levels]
[vault]
[vendordata_dynamic_auth]
[vmware]
[vnc]
novncproxy_host=10.10.11.89
enabled = true
#vncserver_listen = 10.10.11.89
#vncserver_proxyclient_address = 10.10.11.89
novncproxy_base_url = http://10.10.11.93:6080/vnc_auto.html
[workarounds]
[wsgi]
[xenserver]
[xvp]
EOF
```

### Thêm vào file 00-nova-placement-api.conf 
```
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
```

### Cấu hình bind port cho nova-placement
```
sed -i -e 's/VirtualHost \*/VirtualHost 10.10.11.89/g' /etc/httpd/conf.d/00-nova-placement-api.conf
sed -i -e 's/Listen 8778/Listen 10.10.11.89:8778/g' /etc/httpd/conf.d/00-nova-placement-api.conf
systemctl restart httpd
```

### Enable và start
```
systemctl enable openstack-nova-api.service \
  openstack-nova-scheduler.service openstack-nova-consoleauth.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service

systemctl start openstack-nova-api.service \
  openstack-nova-scheduler.service openstack-nova-consoleauth.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service
```

Sau bước này trở về CTL 1

## Phần 10: Cài neutron

### Cài packages (Thực hiện trên tất cả CTL)
```
yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables -y
```

DHCP agent và metadata agent được chạy trên node compute

### Cấu hình
```
cp /etc/neutron/neutron.conf /etc/neutron/neutron.conf.org
rm -rf /etc/neutron/neutron.conf

cat << EOF >> /etc/neutron/neutron.conf
[DEFAULT]
bind_host = 10.10.11.89
core_plugin = ml2
transport_url = rabbit://openstack:Welcome123@10.10.11.87:5672,openstack:Welcome123@10.10.11.88:5672,openstack:Welcome123@10.10.11.89:5672
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
allow_overlapping_ips = True
dhcp_agents_per_network = 2
service_plugins = neutron.services.qos.qos_plugin.QoSPlugin

[agent]
[cors]
[database]
connection = mysql+pymysql://neutron:Welcome123@10.10.11.93/neutron
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
auth_url = http://10.10.11.93:35357
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
```

### Cấu hình file ml2
```
cp /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/ml2_conf.ini.org
rm -rf /etc/neutron/plugins/ml2/ml2_conf.ini

cat << EOF >> /etc/neutron/plugins/ml2/ml2_conf.ini
[DEFAULT]
[l2pop]
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population
extension_drivers = port_security,qos
[ml2_type_flat]
flat_networks = provider
[ml2_type_geneve]
[ml2_type_gre]
[ml2_type_vlan]
#network_vlan_ranges = provider
[ml2_type_vxlan]
vni_ranges = 1:1000
[securitygroup]
enable_ipset = True
EOF
```

### Cấu hình file LB agent
```
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
local_ip = 10.10.14.89
l2_population = true
EOF
```

### Cấu hình trên file l3 agent

```
cp /etc/neutron/l3_agent.ini /etc/neutron/l3_agent.ini.org
rm -rf /etc/neutron/l3_agent.ini

cat << EOF >> /etc/neutron/l3_agent.ini
[DEFAULT]
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
[agent]
[ovs]
EOF
```

### Chỉnh sửa file /etc/nova/nova.conf

```
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
service_metadata_proxy = true
metadata_proxy_shared_secret = Welcome123
```

### Restart lại dv nova
```
systemctl restart openstack-nova-api.service openstack-nova-scheduler.service openstack-nova-consoleauth.service openstack-nova-conductor.service openstack-nova-novncproxy.service
```

### Phần quyền
```
chown -R root:neutron /etc/neutron/
```

### Tạo liên kết
```
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```

### Enable và start dịch vụ 
```
systemctl enable neutron-server.service neutron-linuxbridge-agent.service neutron-l3-agent.service
systemctl restart neutron-server.service neutron-linuxbridge-agent.service neutron-l3-agent.service
```

Sau bước này trở lại CTL 1

## Phần 11: Cấu hình horizon

### Tải packages (Trên tất cả các CTL)
```
yum install openstack-dashboard -y
```

### Tạo file direct
```
filehtml=/var/www/html/index.html
touch $filehtml
cat << EOF >> $filehtml
<html>
<head>
<META HTTP-EQUIV="Refresh" Content="0.5; URL=http://10.10.11.93/dashboard">
</head>
<body>
<center> <h1>Redirecting to OpenStack Dashboard</h1> </center>
</body>
</html>
EOF
```

### Kiểm tra file /etc/openstack-dashboard/local_settings

Bảo đảm nội dùng giống cấu hình trên CTL 1

### Phân quyền cho cấu hình mới
Lưu ý: Yêu cầu thực hiện xong bước chuyển cấu hình qua 2 node còn lại trên node controller1
```
chown root:apache /etc/openstack-dashboard/local_settings
```
### Thêm vào file /etc/httpd/conf.d/openstack-dashboard.conf
```
echo "WSGIApplicationGroup %{GLOBAL}" >> /etc/httpd/conf.d/openstack-dashboard.conf
```
### Restart lại httpd
```
systemctl restart httpd.service memcached.service
```

## Phần 12: Cấu hình Cinder

- HA cho Cinder cần có Ceph

### Tải package (Trên tất cả CTL)
yum install openstack-cinder -y

### Sửa cấu hình cinder

cp /etc/cinder/cinder.conf /etc/cinder/cinder.conf.bak 
rm -rf /etc/cinder/cinder.conf

cat << EOF >> /etc/cinder/cinder.conf
[DEFAULT]
my_ip = 10.10.11.89
transport_url = rabbit://openstack:Welcome123@10.10.11.87:5672,openstack:Welcome123@10.10.11.88:5672,openstack:Welcome123@10.10.11.89:5672
auth_strategy = keystone
osapi_volume_listen = 10.10.11.89
[backend]
[backend_defaults]
[barbican]
[brcd_fabric_example]
[cisco_fabric_example]
[coordination]
[cors]
[database]
connection = mysql+pymysql://cinder:Welcome123@10.10.11.93/cinder
[fc-zone-manager]
[healthcheck]
[key_manager]
[keystone_authtoken]
auth_uri = http://10.10.11.93:5000
auth_url = http://10.10.11.93:35357
memcached_servers = 10.10.11.87:11211,10.10.11.88:11211,10.10.11.89:11211
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
EOF

### Phân quyền

chown root:cinder /etc/cinder/cinder.conf

### Chỉnh sửa file /etc/nova/nova.conf

[cinder]
os_region_name = RegionOne

### Restart lại dịch vụ nova api

systemctl restart openstack-nova-api.service

### Enable và start dịch vụ

systemctl enable openstack-cinder-api.service openstack-cinder-volume.service openstack-cinder-scheduler.service
systemctl start openstack-cinder-api.service openstack-cinder-volume.service openstack-cinder-scheduler.service

Trở về CTL 1 và thực hiện
```
[root@ctl01 ~]# openstack volume service list
+------------------+-------+------+---------+-------+----------------------------+
| Binary           | Host  | Zone | Status  | State | Updated At                 |
+------------------+-------+------+---------+-------+----------------------------+
| cinder-scheduler | ctl01 | nova | enabled | up    | 2020-11-12T17:41:48.000000 |
| cinder-scheduler | ctl02 | nova | enabled | up    | 2020-11-12T17:41:45.000000 |
| cinder-scheduler | ctl03 | nova | enabled | up    | 2020-11-12T17:41:40.000000 |
+------------------+-------+------+---------+-------+----------------------------+
```