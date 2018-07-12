# Cài đặt OpenStack manual (Queen)
---
## Phần 1: Chuẩn bị các node
Các phần:
- Cấu hình Hostname, static interface
- Cấu hình host file
- Tắt Firewalld
- Tắt SELinux
- Cài gói mặc định
- Test ping các node

### Bước 1: Cấu hình hostname, static interface
#### Tại CONTROLLER
Cấu hình hostname
```
hostnamectl set-hostname controller
```

Cấu hình ip tĩnh
```
# Config interface eno16777984
nmcli c modify eno16777984 ipv4.addresses 172.16.4.200/20
nmcli c modify eno16777984 ipv4.gateway 172.16.10.1
nmcli c modify eno16777984 ipv4.dns 8.8.8.8
nmcli c modify eno16777984 ipv4.method manual
nmcli con mod eno16777984 connection.autoconnect yes

# Config interface eno33557248
nmcli c modify eno33557248 ipv4.addresses 10.0.3.10/24
nmcli c modify eno33557248 ipv4.method manual
nmcli con mod eno33557248 connection.autoconnect yes

# ens224
nmcli c modify ens224 ipv4.addresses 172.16.9.10/24
nmcli c modify ens224 ipv4.method manual
nmcli con mod ens224 connection.autoconnect yes
```

#### Tại COMPUTE1

Cấu hình hostname
```
hostnamectl set-hostname compute1
```

Cấu hình ip tĩnh
```
# Config interface eno16777984
nmcli c modify eno16777984 ipv4.addresses 172.16.4.201/20
nmcli c modify eno16777984 ipv4.gateway 172.16.10.1
nmcli c modify eno16777984 ipv4.dns 8.8.8.8
nmcli c modify eno16777984 ipv4.method manual
nmcli con mod eno16777984 connection.autoconnect yes

# Config interface eno33557248
nmcli c modify eno33557248 ipv4.addresses 10.0.3.11/24
nmcli c modify eno33557248 ipv4.method manual
nmcli con mod eno33557248 connection.autoconnect yes

# ens224
nmcli c modify ens224 ipv4.addresses 172.16.9.11/24
nmcli c modify ens224 ipv4.method manual
nmcli con mod ens224 connection.autoconnect yes
```

#### Tại COMPUTE2

Cấu hình hostname
```
hostnamectl set-hostname compute2
```

Cấu hình ip tĩnh
```
# Config interface eno16777984
nmcli c modify eno16777984 ipv4.addresses 172.16.4.202/20
nmcli c modify eno16777984 ipv4.gateway 172.16.10.1
nmcli c modify eno16777984 ipv4.dns 8.8.8.8
nmcli c modify eno16777984 ipv4.method manual
nmcli con mod eno16777984 connection.autoconnect yes

# Config interface eno33557248
nmcli c modify eno33557248 ipv4.addresses 10.0.3.12/24
nmcli c modify eno33557248 ipv4.method manual
nmcli con mod eno33557248 connection.autoconnect yes

# ens224
nmcli c modify ens224 ipv4.addresses 172.16.9.12/24
nmcli c modify ens224 ipv4.method manual
nmcli con mod ens224 connection.autoconnect yes
```

### Bước 2: Cấu hình Firewalld, SELinux
> Tại tất cả các host

Disable firewalld, selinux
```
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
systemctl stop firewalld
systemctl disable firewalld
```


### Cấu hình hostfile
> Tại tất cả các host

Lưu ý:
- Hoạt động giám sát, cài đặt gói thông qua đường MNGT

Nội dung
```
# controller1
172.16.4.200       controller1

# compute1
172.16.4.201       compute1

# compute2
172.16.4.202       compute2
```

Lệnh thực thi
```
echo '
# controller1
172.16.4.200       controller1

# compute1
172.16.4.201       compute1

# compute2
172.16.4.202       compute2' >> /etc/hosts
```

Kiểm tra
```
fping 172.16.4.200 172.16.4.201 172.16.4.202
fping 10.0.3.10 10.0.3.11 10.0.3.12
fping 172.16.9.10 172.16.9.11 172.16.9.12
fping controller1 compute1 compute2
```

### Bước 3: Cài đặt các gói cần thiết
```
yum install -y python-setuptools
sudo yum install -y wget crudini fping
yum install -y epel-release
sudo yum install -y byobu
```

## Phần 2: Cấu hình NTP tại các node
### Tại controller
Cài đặt gói
```
yum install chrony -y
```

Thiết lập config
```
sed -i "s/server 0.centos.pool.ntp.org iburst/server vn.pool.ntp.org iburst/g" /etc/chrony.conf
sed -i 's/server 1.centos.pool.ntp.org iburst/#server 1.centos.pool.ntp.org iburst/g' /etc/chrony.conf
sed -i 's/server 2.centos.pool.ntp.org iburst/#server 2.centos.pool.ntp.org iburst/g' /etc/chrony.conf
sed -i 's/server 3.centos.pool.ntp.org iburst/#server 3.centos.pool.ntp.org iburst/g' /etc/chrony.conf
sed -i 's/#allow 192.168.0.0\/16/allow 172.16.0.0\/20/g' /etc/chrony.conf
```

Chạy, thiết lạp service
```
systemctl enable chronyd.service
systemctl start chronyd.service
```
chronyc sources
### Tại compute
Cài đặt gói
```
yum install chrony -y
```

Thiết lập config
```
sed -i "s/server 0.centos.pool.ntp.org iburst/server 172.16.4.200 iburst/g" /etc/chrony.conf
sed -i 's/server 1.centos.pool.ntp.org iburst/#server 1.centos.pool.ntp.org iburst/g' /etc/chrony.conf
sed -i 's/server 2.centos.pool.ntp.org iburst/#server 2.centos.pool.ntp.org iburst/g' /etc/chrony.conf
sed -i 's/server 3.centos.pool.ntp.org iburst/#server 3.centos.pool.ntp.org iburst/g' /etc/chrony.conf
```

Thiết lập, chạy service
```
systemctl enable chronyd.service
systemctl start chronyd.service
```

Đồng bộ thời gian
```
chronyc sources
```

## Phần 3: Cài đặt các gói cần thiết OpenStack
> Tại tất cả các node

Cài đặt OPS Queen package
```
yum install centos-release-openstack-queens -y
yum upgrade -y
yum install python-openstackclient -y
yum install openstack-selinux -y
```

## Phần 4: Thiết lập DB SQL
> Thiết lập trên Controller

Lưu ý:
- Thông thường DB được chạy trên node controller
- Có thể chạy trên node riêng phục vụ nhu cầu

Cài đặt gói
```
yum install mariadb mariadb-server python2-PyMySQL -y
```

Tạo backup `/etc/my.cnf.d/openstack.cnf` nếu có:
```
cp /etc/my.cnf.d/openstack.cnf /etc/my.cnf.d/openstack.cnf.bak
```

Chỉnh sửa sql config
> Dường dẫn `/etc/my.cnf.d/openstack.cnf`

```
cat << EOF > /etc/my.cnf.d/openstack.cnf
[mysqld]
bind-address = 172.16.4.200
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
EOF
```
Lưu ý:
- `bind-address` là IP controller

Khởi động lại dịch vụ
```
systemctl enable mariadb.service
systemctl start mariadb.service
```

Thiết lập mật khẩu DB
```
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
```

## Phần 5: Thiết lập Message Queue
> Thiết lập trên node `CONTROLLER`

> Message queue service thường chạy trên `controller`

Note: 
- Openstack hỗ trợ 1 số loại queue bao gồm: 
 - RabbitMQ
 - Qpid
 - ZeroMQ

- Bài lab hiện tại sẽ sử dụng RabbitMQ message queue service, phiên bản được sử dụng rộng rãi nhất cho OPS.

Cài đặt gói
```
yum install rabbitmq-server -y
```

Chạy dịch vụ
```
systemctl enable rabbitmq-server.service
systemctl start rabbitmq-server.service
```

Khởi tạo và phân quyền user
```
rabbitmqctl add_user openstack Welcome123
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

## Phần 6: Thiết lập Memcached
> Thiết lập trên node `CONTROLLER`

> Memcached service chạy trên `controller` 

Cài đặt gói
```
yum install memcached python-memcached -y
```

Thiết lập cấu hình
> File `/etc/sysconfig/memcached`

```
sed -i 's/OPTIONS=\"-l 127.0.0.1,::1\"/OPTIONS=\"-l 172.16.4.200,::1\"/g' /etc/sysconfig/memcached
```

Khởi động dịch vụ
```
systemctl enable memcached.service
systemctl start memcached.service
```


## Thiết lập cơ bản nhất cho OPENSTACK QUEEN
### Tổng quản
QpenStack bao gồm rất nhiều thành phân riêng biệt. Chúng làm việc với nhau để tạo thành dịch vụ cloud.

Các thành phần sẽ triển khai bao gồm:
- Compute - Nova
- Identity - KeyStone
- Networking - Neutron
- Image - Glance
- Block Storage - Cinder
- Dashboard - Horizon

## Phần 1: Cài đặt KeyStone Keystone
### Tổng quan
- Là dịch vụ cung cấp OpenStack Identity service.  
- Quản trị định danh, chứng thực, quản lý truy cập.

## Cài đặt tại Controller

### Tạo database dịch vụ
```
mysql -u root -pWelcome123 

CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'Welcome123';
exit
```

### Cài đặt keystone
```
yum install openstack-keystone httpd mod_wsgi -y
```

### Tạo bản backup
```
cp /etc/keystone/keystone.conf /etc/keystone/keystone.conf.bak
```

### Cấu hình File
```
cat << EOF > /etc/keystone/keystone.conf
[DEFAULT]
[application_credential]
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
[unified_limit]
EOF
```

Lưu ý cấu hình
```
[database]
connection = mysql+pymysql://keystone:Welcome123@172.16.4.200/keystone
[token]
provider = fernet
```

### Đồng bộ Identity service database:
```
su -s /bin/sh -c "keystone-manage db_sync" keystone
```

### Khởi tạo Fernet key repositories:
```
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

### Khởi tạo Bootstrap Identity service:
```
keystone-manage bootstrap --bootstrap-password Welcome123 \
  --bootstrap-admin-url http://172.16.4.200:5000/v3/ \
  --bootstrap-internal-url http://172.16.4.200:5000/v3/ \
  --bootstrap-public-url http://172.16.4.200:5000/v3/ \
  --bootstrap-region-id RegionOne
```

### Cấu hình Apache HTTP server

Sao lưu file /etc/httpd/conf/httpd.conf
```
cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd.conf.bak
```

Chỉnh sửa `/etc/httpd/conf/httpd.conf` file
```
vi /etc/httpd/conf/httpd.conf
```
- Nội dung
```
ServerName controller1
```

Tạo đường dẫn
```
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
```

Khởi động dịch vụ
```
systemctl enable httpd.service
systemctl start httpd.service
```

Khởi tạo biên môi trường
```
export OS_USERNAME=admin
export OS_PASSWORD=Welcome123
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://172.16.4.200:35357/v3
export OS_IDENTITY_API_VERSION=3
```

### Tạo domain, project, user, role

Tạo new domain:
```
openstack domain create --description "An Example Domain" example
```

Tạo service project:
```
openstack project create --domain default --description "Service Project" service
```

Tạo user demo.
- Tạo demo project:
```
openstack project create --domain default --description "Demo Project" demo
```

- Tạo demo user:
```
openstack user create --domain default --password-prompt demo
User Password: Welcome123
Repeat User Password:Welcome123
```

- Tạo user role:
```
openstack role create user
```

- Thêm user role tới demo project, user:
```
openstack role add --project demo --user demo user
```

Sao lưu file /etc/keystone/keystone-paste.ini
```
cp /etc/keystone/keystone-paste.ini /etc/keystone/keystone-paste.ini.bak
```

### Kiểm tra hoạt động

Hủy biến môi trường OS_AUTH_URL, OS_PASSWORD environment:
```
unset OS_AUTH_URL OS_PASSWORD
```

Với admin user, yêu cầu chứng thực (authentication token):
```
openstack --os-auth-url http://172.16.4.200:35357/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name admin --os-username admin token issue

Password: Welcome123  

```

Với demo user, yêu cầu chứng thực:
```
openstack --os-auth-url http://172.16.4.200:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name demo --os-username demo token issue

Password: Welcome123
```

Tạo biến môi trường
- Với User admin
```
vi admin-openrc

export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=Welcome123
export OS_AUTH_URL=http://172.16.4.200:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

- Với User demo
```
vi demo-openrc

export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=Welcome123
export OS_AUTH_URL=http://172.16.4.200:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

Sử dụng script:
- Khởi tạo môi trường dựa trên script admin-openrc
```
. admin-openrc

openstack token issue
```

## Phần 2: Cài đặt Glance - Image service
>  Cấu hình trên `Controller`

To create the database, complete these steps:
```
mysql -u root -pWelcome123 
Enter password:  Welcome123
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'Welcome123';

exit
```

Source the admin credentials to gain access to admin-only CLI commands:
```
. admin-openrc
```

To create the service credentials, complete these steps:
- Create the glance user:
```
openstack user create --domain default --password-prompt glance
User Password: Welcome123
Repeat User Password: Welcome123
```

- Add the admin role to the glance user and service project:
```
openstack role add --project service --user glance admin
```

- Create the glance service entity:
```
openstack service create --name glance --description "OpenStack Image" image
```

- Create the Image service API endpoints:
```
openstack endpoint create --region RegionOne image public http://172.16.4.200:9292
openstack endpoint create --region RegionOne image internal http://172.16.4.200:9292
openstack endpoint create --region RegionOne image admin http://172.16.4.200:9292

```

Install and configure components

yum install openstack-glance -y

Sao lưu cấu hình file config glance-api
```
cp /etc/glance/glance-api.conf /etc/glance/glance-api.conf.bak
```

### Edit the /etc/glance/glance-api.conf file and complete the following actions:
```
[database]
connection = mysql+pymysql://glance:Welcome123@172.16.4.200/glance

[keystone_authtoken]
# ...
auth_uri = http://172.16.4.200:5000
auth_url = http://172.16.4.200:5000
memcached_servers = 172.16.4.200:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = Welcome123

[paste_deploy]
# ...
flavor = keystone

[glance_store]
# ...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```

cat /etc/glance/glance-api.conf | egrep -v "(^#.*|^$)"

### Edit /etc/glance/glance-registry.conf 
Sao lưu file config glance-registry
```
cp /etc/glance/glance-registry.conf /etc/glance/glance-registry.conf.bak
```

Chỉnh sửa file config glance-registry: /etc/glance/glance-registry.conf
```
[database]
# ...
connection = mysql+pymysql://glance:Welcome123@172.16.4.200/glance

[keystone_authtoken]
# ...
auth_uri = http://172.16.4.200:5000
auth_url = http://172.16.4.200:5000
memcached_servers = 172.16.4.200:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = Welcome123

[paste_deploy]
# ...
flavor = keystone
```

Populate the Image service database:
```
su -s /bin/sh -c "glance-manage db_sync" glance
```

Finalize installation
```
systemctl enable openstack-glance-api.service openstack-glance-registry.service
systemctl start openstack-glance-api.service openstack-glance-registry.service
```

Test
```
wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img

openstack image create "cirros" --file cirros-0.3.4-x86_64-disk.img --disk-format qcow2 --container-format bare --public

openstack image create "cirros" --file cirros-0.4.0-x86_64-disk.img --disk-format qcow2 --container-format bare --public

openstack image list

openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 825c647d-cfd8-41c6-81c0-57cf31d6e056 | cirros | active |
+--------------------------------------+--------+--------+
```

# Compute service

## Install and configure controller node

Prerequisites

Before you install and configure the Compute service, you must create databases, service credentials, and API endpoints.

To create the databases, complete these steps:
```
mysql -u root -pWelcome123
Enter password: Welcome123

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

```

Source the admin credentials to gain access to admin-only CLI commands:

```
. admin-openrc
```

Create the Compute service credentials:
- Create the nova user:
```
openstack user create --domain default --password-prompt nova
User Password: Welcome123
Repeat User Password: Welcome123
```

- Add the admin role to the nova user:
```
openstack role add --project service --user nova admin
```

- Create the nova service entity:
```
openstack service create --name nova --description "OpenStack Compute" compute
```

Create the Compute API service endpoints:
```
openstack endpoint create --region RegionOne compute public http://172.16.4.200:8774/v2.1

openstack endpoint create --region RegionOne compute internal http://172.16.4.200:8774/v2.1

openstack endpoint create --region RegionOne compute admin http://172.16.4.200:8774/v2.1
```

Create a Placement service user using your chosen PLACEMENT_PASS:
```
openstack user create --domain default --password-prompt placement
User Password: Welcome123
Repeat User Password: Welcome123
```

Add the Placement user to the service project with the admin role:
```
openstack role add --project service --user placement admin
```

Create the Placement API entry in the service catalog:
```
openstack service create --name placement --description "Placement API" placement
```

Create the Placement API service endpoints:
```
openstack endpoint create --region RegionOne placement public http://172.16.4.200:8778
openstack endpoint create --region RegionOne placement internal http://172.16.4.200:8778
openstack endpoint create --region RegionOne placement admin http://172.16.4.200:8778
```

## Install and configure components
Install the packages:
```
yum install openstack-nova-api openstack-nova-conductor \
  openstack-nova-console openstack-nova-novncproxy \
  openstack-nova-scheduler openstack-nova-placement-api -y
```

```
cp /etc/nova/nova.conf /etc/nova/nova.conf.bak

cat /etc/nova/nova.conf | egrep -v "(^#.*|^$)"
```

Edit the /etc/nova/nova.conf file and complete the following actions:
```
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:Welcome123@172.16.4.200
my_ip = 172.16.4.200
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api_database]
connection = mysql+pymysql://nova:Welcome123@172.16.4.200/nova_api

[database]
connection = mysql+pymysql://nova:Welcome123@172.16.4.200/nova

[api]
auth_strategy = keystone

[keystone_authtoken]
auth_url = http://172.16.4.200:5000/v3
memcached_servers = 172.16.4.200:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = Welcome123

[vnc]
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip

[glance]
api_servers = http://172.16.4.200:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[placement]
os_region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://172.16.4.200:5000/v3
username = placement
password = Welcome123
```

add config to /etc/httpd/conf.d/00-nova-placement-api.conf
```
....
<Directory /usr/bin>
   <IfVersion >= 2.4>
      Require all granted
   </IfVersion>
   <IfVersion < 2.4>
      Order allow,deny
      Allow from all
   </IfVersion>
</Directory>
```

Restart the httpd service:
```
systemctl restart httpd
```


Populate the nova-api database:
```
su -s /bin/sh -c "nova-manage api_db sync" nova
```

BUG:
```
[root@controller1 ~]# su -s /bin/sh -c "nova-manage api_db sync" nova
/usr/lib/python2.7/site-packages/oslo_db/sqlalchemy/enginefacade.py:332: NotSupportedWarning: Configuration option(s) ['use_tpool'] not supported
  exception.NotSupportedWarning

```
> Bỏ qua lỗi warning


Register the cell0 database:
```
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
```

Create the cell1 cell:
```
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova

#KQ
109e1d4b-536a-40d0-83c6-5f121b82b650
```

Populate the nova database:
```
su -s /bin/sh -c "nova-manage db sync" nova
```

Verify nova cell0 and cell1 are registered correctly:
```
nova-manage cell_v2 list_cells

KQ:
+-------+--------------------------------------+
| Name  | UUID                                 |
+-------+--------------------------------------+
| cell1 | 109e1d4b-536a-40d0-83c6-5f121b82b650 |
| cell0 | 00000000-0000-0000-0000-000000000000 |
+-------+--------------------------------------+
```

su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova

### Finalize installation
```
systemctl enable openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service

systemctl start openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service   openstack-nova-conductor.service openstack-nova-novncproxy.service
```

## Install and configure a compute node
### Install and configure components

Install the packages:
```
yum install openstack-nova-compute -y
```

```
cp /etc/nova/nova.conf /etc/nova/nova.conf.bak
```


Edit the `/etc/nova/nova.conf`
> my_ip = 172.16.4.201 # IP HOST COMPUTE management network interface 

```
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:Welcome123@172.16.4.200
my_ip = 172.16.4.201 
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api]
auth_strategy = keystone

[keystone_authtoken]
auth_url = http://172.16.4.200:5000/v3
memcached_servers = 172.16.4.200:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = Welcome123

[vnc]
enabled = True
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://172.16.4.200:6080/vnc_auto.html

[glance]
api_servers = http://172.16.4.200:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[placement]
os_region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://172.16.4.200:5000/v3
username = placement
password = Welcome123
```

## Finalize installation
Determine whether your compute node supports hardware acceleration for virtual machines:
```
egrep -c '(vmx|svm)' /proc/cpuinfo
```
> If this command returns a value of zero, your compute node does not support hardware acceleration and you must configure libvirt to use QEMU instead of KVM.

- Edit the [libvirt] section in the /etc/nova/nova.conf file as follows:
```
[libvirt]
# ...
virt_type = qemu
```

Start the Compute service including its dependencies and configure them to start automatically when the system boots:
```
systemctl enable libvirtd.service openstack-nova-compute.service
systemctl start libvirtd.service openstack-nova-compute.service
```

> If the nova-compute service fails to start, check `/var/log/nova/nova-compute.log`. The error message AMQP server on `controller:5672` is unreachable likely indicates that the firewall on the controller node is preventing access to port `5672`. Configure the firewall to open port `5672` on the controller node and restart nova-compute service on the compute node.

Sau khi cấu hình xong quay trở lại node controller và kiểm tra các service đã lên hay chưa.

```
. admin-openrc
openstack compute service list

+----+------------------+-------------+----------+---------+-------+----------------------------+
| ID | Binary           | Host        | Zone     | Status  | State | Updated At                 |
+----+------------------+-------------+----------+---------+-------+----------------------------+
|  1 | nova-consoleauth | controller1 | internal | enabled | up    | 2018-07-09T07:47:37.000000 |
|  2 | nova-conductor   | controller1 | internal | enabled | up    | 2018-07-09T07:47:38.000000 |
|  3 | nova-scheduler   | controller1 | internal | enabled | up    | 2018-07-09T07:47:38.000000 |
|  6 | nova-compute     | compute1    | nova     | enabled | up    | 2018-07-09T07:47:35.000000 |
+----+------------------+-------------+----------+---------+-------+----------------------------+
```


## Add the compute node to the cell database
> Run the following commands on the controller node.

Source the admin credentials to enable admin-only CLI commands, then confirm there are compute hosts in the database:
```
. admin-openrc
openstack compute service list --service nova-compute

VD:
[root@controller1 ~]# openstack compute service list --service nova-compute
+----+--------------+----------+------+---------+-------+----------------------------+
| ID | Binary       | Host     | Zone | Status  | State | Updated At                 |
+----+--------------+----------+------+---------+-------+----------------------------+
|  6 | nova-compute | compute1 | nova | enabled | up    | 2018-07-09T07:48:45.000000 |
+----+--------------+----------+------+---------+-------+----------------------------+

```

Discover compute hosts:
```
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova

KQ:
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting computes from cell 'cell1': e0512d74-aff1-4734-94f5-0538305d5383
Checking host mapping for compute host 'compute1': dbd4d559-a783-4162-a932-aa2cfd74a083
Creating host mapping for compute host 'compute1': dbd4d559-a783-4162-a932-aa2cfd74a083
Found 1 unmapped computes in cell: e0512d74-aff1-4734-94f5-0538305d5383

```

## Verify operation
Source the admin credentials to gain access to admin-only CLI commands:
```
. admin-openrc
```

List service components to verify successful launch and registration of each process:
```
openstack compute service list
```

List API endpoints in the Identity service to verify connectivity with the Identity service:
```
openstack catalog list
```

List images in the Image service to verify connectivity with the Image service:
```
openstack image list
```

Check the cells and placement API are working successfully:
```
nova-status upgrade check
```

# Networking service neutron
## Kiểm tra lại cấu hình toàn bộ IP tại các node ops và file Host
```
cat /etc/hosts
```
## Kiểm tra mạng 
From the controller node, test access to the Internet:
```
ping -c 4 openstack.org
```

From the controller node, test access to the management interface on the compute node:
```
fping 172.16.4.200 172.16.4.201 172.16.4.202
fping 10.0.3.10 10.0.3.11 10.0.3.12
fping 172.16.9.10 172.16.9.11 172.16.9.12
fping controller1 compute1 compute2
```

From the compute node, test access to the Internet:
```
ping -c 4 openstack.org
```
## Install and configure controller node
> Thực hiện tại controller

To create the database, complete these steps:
```
mysql -u root -pWelcome123
Enter password: Welcome123
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'Welcome123';

exit
```

Source the admin credentials to gain access to admin-only CLI commands:
```
. admin-openrc
```

To create the service credentials, complete these steps:
- Create the neutron user:
```
openstack user create --domain default --password-prompt neutron

User Password: Welcome123
Repeat User Password: Welcome123
```

- Add the admin role to the neutron user:
```
openstack role add --project service --user neutron admin
```

- Create the neutron service entity:
```
openstack service create --name neutron --description "OpenStack Networking" network

KQ:
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | ad524c972e53413fb457a50797e794ca |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+

```

- Create the Networking service API endpoints:
```
openstack endpoint create --region RegionOne network public http://172.16.4.200:9696
openstack endpoint create --region RegionOne network internal http://172.16.4.200:9696
openstack endpoint create --region RegionOne network admin http://172.16.4.200:9696
```

### Networking Option 1: Provider networks
> Lưu ý: Để có thể cài được self-service network, trước tiên phải cài provider network

Install the components
```
yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables -y
```

Sao lưu file cấu hình
```
cp /etc/neutron/neutron.conf /etc/neutron/neutron.conf.bak
```


Configure the server component

Edit the /etc/neutron/neutron.conf file and complete the following actions:
```
[DEFAULT]
core_plugin = ml2
service_plugins =
transport_url = rabbit://openstack:Welcome123@172.16.4.200
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

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
```

Configure the Modular Layer 2 (ML2) plug-in
> The ML2 plug-in uses the Linux bridge mechanism to build layer-2 (bridging and switching) virtual networking infrastructure for instances.

Sao lưu file cấu hình Modular Layer 2
```
cp /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/ml2_conf.ini.bak
```

Edit the /etc/neutron/plugins/ml2/ml2_conf.ini file and complete the following actions:
```
[ml2]
type_drivers = flat,vlan
tenant_network_types = 
mechanism_drivers = linuxbridge
extension_drivers = port_security

[ml2_type_flat]
flat_networks = provider

[securitygroup]
enable_ipset = true
```

Configure the Linux bridge agent
> The Linux bridge agent builds layer-2 (bridging and switching) virtual networking infrastructure for instances and handles security groups.

Sao lưu file cấu hình Linux bridge agent
```
cp /etc/neutron/plugins/ml2/linuxbridge_agent.ini /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak
```

Edit /etc/neutron/plugins/ml2/linuxbridge_agent.ini
> physical_interface_mappings = provider:ens224 # Interface provider

```
[linux_bridge]
physical_interface_mappings = provider:ens224

[vxlan]
enable_vxlan = false

[securitygroup]
# ...
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

Ensure your Linux operating system kernel supports network bridge filters by verifying all the following sysctl values are set to 1:
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
```

Configure the DHCP agent
> The DHCP agent provides DHCP services for virtual networks.

Sao lưu file cấu hình DHCP agent
```
cp /etc/neutron/dhcp_agent.ini /etc/neutron/dhcp_agent.ini.bak
```

Edit the `/etc/neutron/dhcp_agent.ini`
```
[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
```

Configure the metadata agent
> The metadata agent provides configuration information such as credentials to instances.



Edit the /etc/neutron/neutron.conf
```
[DEFAULT]
# ...
nova_metadata_host = 172.16.4.200
metadata_proxy_shared_secret = Welcome123
```

## Configure the Compute service to use the Networking service

Edit the `/etc/nova/nova.conf`
```
[neutron]
# ...
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
```

## Finalize installation
The Networking service initialization scripts expect a symbolic link `/etc/neutron/plugin.ini` pointing to the ML2 plug-in configuration file, `/etc/neutron/plugins/ml2/ml2_conf.ini`. If this symbolic link does not exist, create it using the following command:
```
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```

Populate the database:
```
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

Restart the Compute API service:
```
systemctl restart openstack-nova-api.service
```

Start the Networking services and configure them to start when the system boots.
```
systemctl enable neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service

systemctl start neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
```

## Install and configure compute node

Install the components
```
yum install openstack-neutron-linuxbridge ebtables ipset -y
```

### Configure the common component
> The Networking common component configuration includes the authentication mechanism, message queue, and plug-in.

Sao lưu file cấu hình
```
cp /etc/neutron/neutron.conf /etc/neutron/neutron.conf.bak
```

Edit the `/etc/neutron/neutron.conf`
> In the [database] section, comment out any connection options because compute nodes do not directly access the database.

```
[DEFAULT]
transport_url = rabbit://openstack:Welcome123@172.16.4.200
auth_strategy = keystone

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

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
```

### Networking Option 1: Provider networks
Configure the Linux bridge agent
> The Linux bridge agent builds layer-2 (bridging and switching) virtual networking infrastructure for instances and handles security groups.

Sao lưu file cấu hình
```
cp /etc/neutron/plugins/ml2/linuxbridge_agent.ini /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak
```

Edit `/etc/neutron/plugins/ml2/linuxbridge_agent.ini`
```
[linux_bridge]
physical_interface_mappings = provider:ens224

[vxlan]
enable_vxlan = false

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

Ensure your Linux operating system kernel supports network bridge filters by verifying all the following sysctl values are set to 1:
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
```

### Configure the Compute service to use the Networking service

Edit the `/etc/nova/nova.conf` file and complete the following actions:
```
[neutron]
...
url = http://172.16.4.200:9696
auth_url = http://172.16.4.200:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = Welcome123
```

### Finalize installation

Restart the Compute service:
```
systemctl restart openstack-nova-compute.service
```

Start the Linux bridge agent and configure it to start when the system boots:
```
systemctl enable neutron-linuxbridge-agent.service
systemctl start neutron-linuxbridge-agent.service
```



# Dashboard – horizon installation for Queens
> install and configure the dashboard on the controller node.

## Install and configure components
Install the packages:
```
yum install openstack-dashboard -y
```

Sao lưu file cấu hình
```
cp /etc/openstack-dashboard/local_settings /etc/openstack-dashboard/local_settings.bak
```

Edit the `/etc/openstack-dashboard/local_settings`
```
OPENSTACK_HOST = "172.16.4.200"

ALLOWED_HOSTS = ['*',] 

SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': '172.16.4.200:11211',
    }
}

OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST

OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True

OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 2,
}

OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"

OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

OPENSTACK_NEUTRON_NETWORK = {
    ...
    'enable_router': False,
    'enable_quotas': False,
    'enable_ipv6': False,
    'enable_distributed_router': False,
    'enable_ha_router': False,
    'enable_lb': False,
    'enable_firewall': False,
    'enable_vpn': False,
    'enable_fip_topology_check': False,
}

```

Add the following line to /etc/httpd/conf.d/openstack-dashboard.conf if not included.
```
echo "WSGIApplicationGroup %{GLOBAL}" >> /etc/httpd/conf.d/openstack-dashboard.conf
```

systemctl restart httpd.service memcached.service



Verify operation of the dashboard.

Access the dashboard using a web browser at http://controller/dashboard.

Authenticate using `admin` or `demo` user and `default` domain credentials.


# Install Cinder

## Install and configure a storage node
> Cấu hình trên controller (yêu cầu 2 ổ, 1 cho cinder volume)

Install the supporting utility packages:
```
yum install lvm2 device-mapper-persistent-data -y
```

Start the LVM metadata service and configure it to start when the system boots:
```
systemctl enable lvm2-lvmetad.service
systemctl start lvm2-lvmetad.service
```

Create the LVM volume /dev/sdb:
```
pvcreate /dev/sdb
vgcreate cinder-volumes /dev/sdb
```

Edit the `/etc/lvm/lvm.conf` file and complete the following actions:
```
devices {
...
filter = [ "a/sdb/", "r/.*/"]
```

Install and configure components
```
yum install openstack-cinder targetcli python-keystone -y
```

Edit: /etc/cinder/cinder.conf
```
[database]
connection = mysql+pymysql://cinder:Welcome123@172.16.4.200/cinder

[DEFAULT]
transport_url = rabbit://openstack:Welcome123@172.16.4.200
auth_strategy = keystone
enabled_backends = lvm
glance_api_servers = http://172.16.4.200:9292
my_ip = 172.16.4.200


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


# Nếu ko có thì tạo mới
[lvm] 
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
iscsi_protocol = iscsi
iscsi_helper = lioadm

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
```

Finalize installation
```
systemctl enable openstack-cinder-volume.service target.service
systemctl start openstack-cinder-volume.service target.service
```

## Install and configure controller node
Prerequisites

To create the database, complete these steps:
```
mysql -u root -pWelcome123
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' \
  IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' \
  IDENTIFIED BY 'Welcome123';

exit
```

Source the admin credentials to gain access to admin-only CLI commands:
```
. admin-openrc
```

To create the service credentials, complete these steps:
Create a cinder user:
```
openstack user create --domain default --password-prompt cinder
```

Add the admin role to the cinder user:
```
openstack role add --project service --user cinder admin
```


Create the cinderv2 and cinderv3 service entities:
```
openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2

openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
```

Create the Block Storage service API endpoints:
```
openstack endpoint create --region RegionOne volumev2 public http://172.16.4.200:8776/v2/%\(project_id\)s

openstack endpoint create --region RegionOne volumev2 internal http://172.16.4.200:8776/v2/%\(project_id\)s

openstack endpoint create --region RegionOne volumev2 admin http://172.16.4.200:8776/v2/%\(project_id\)s

openstack endpoint create --region RegionOne volumev3 public http://172.16.4.200:8776/v3/%\(project_id\)s

openstack endpoint create --region RegionOne \
  volumev3 internal http://172.16.4.200:8776/v3/%\(project_id\)s

openstack endpoint create --region RegionOne \
  volumev3 admin http://172.16.4.200:8776/v3/%\(project_id\)s
```

Install and configure components

Install the packages:
```
yum install openstack-cinder -y
```

Edit the /etc/cinder/cinder.conf file and complete the following actions:
```
[database]
connection = mysql+pymysql://cinder:Welcome123@172.16.4.200/cinder

[DEFAULT]
transport_url = rabbit://openstack:Welcome123@172.16.4.200
auth_strategy = keystone
my_ip = 172.16.4.200

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



[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
```

Populate the Block Storage database:
```
su -s /bin/sh -c "cinder-manage db sync" cinder
```
> Bỏ qua thông báo  Option "logdir" from group "DEFAULT" is deprecated. Use option "log-dir" from group "DEFAULT".

Finalize installation
Restart the Compute API service:
```
systemctl restart openstack-nova-api.service
```

Start the Block Storage services and configure them to start when the system boots:
```
systemctl enable openstack-cinder-api.service openstack-cinder-scheduler.service
systemctl start openstack-cinder-api.service openstack-cinder-scheduler.service
```