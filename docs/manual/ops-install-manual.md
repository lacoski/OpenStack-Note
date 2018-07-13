# Cài đặt OpenStack manual (Queen)
---
## Sơ đồ, quy hoạch tài nguyên
### Sơ đồ
![](images/ops-manual-1.PNG)
### Quy hoạch tài nguyên
![](images/ops-manual-2.PNG)

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
hostnamectl set-hostname controller1  
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

### Bước 2: Cấu hình Firewalld, SELinux, Kernel
> Tại tất cả các host

Disable firewalld, selinux
```
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
systemctl stop firewalld
systemctl disable firewalld
```

Cấu hình kernel
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

Đồng bộ thời gian
```
chronyc sources
``` 
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

### Khởi tạo
Tạo database cua Glance:
```
mysql -u root -pWelcome123 
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'Welcome123';

exit
```

Khởi tạo biến môi trường Admin
```
. admin-openrc
```

Tạo thông tin xác thực:
- Tạo Glance User:
  ```
  openstack user create --domain default --password-prompt glance
  User Password: Welcome123
  Repeat User Password: Welcome123
  ```

- Thêm Admin Role tới Glance user và service project:
  ```
  openstack role add --project service --user glance admin
  ```

- Tới đối tượng Glance Service:
  ```
  openstack service create --name glance --description "OpenStack Image" image
  ```

- Tạo Image service API endpoint:
  ```
  openstack endpoint create --region RegionOne image public http://172.16.4.200:9292
  openstack endpoint create --region RegionOne image internal http://172.16.4.200:9292
  openstack endpoint create --region RegionOne image admin http://172.16.4.200:9292
  ```

### Cài đặt các thành phần
Cài đặt gói
```
yum install openstack-glance -y
```

Sao lưu cấu hình file config glance-api
```
cp /etc/glance/glance-api.conf /etc/glance/glance-api.conf.bak
```

Chỉnh sửa file config `/etc/glance/glance-api.conf`:
```
vi /etc/glance/glance-api.conf
```
- Nội dung
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

> cat /etc/glance/glance-api.conf | egrep -v "(^#.*|^$)"

Sao lưu file config glance-registry
```
cp /etc/glance/glance-registry.conf /etc/glance/glance-registry.conf.bak
```

Chỉnh sửa file `/etc/glance/glance-registry.conf` 
```
vi /etc/glance/glance-registry.conf
```
- Nội dung
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

Đồng bộ Image service database:
```
su -s /bin/sh -c "glance-manage db_sync" glance
```

### Khởi tạo dịch vụ
```
systemctl enable openstack-glance-api.service openstack-glance-registry.service
systemctl start openstack-glance-api.service openstack-glance-registry.service
```

### Tạo Image demo
Tải gói và cài gói
```
wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img

openstack image create "cirros" --file cirros-0.3.4-x86_64-disk.img --disk-format qcow2 --container-format bare --public
```

Kết quả
```
openstack image list

openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 825c647d-cfd8-41c6-81c0-57cf31d6e056 | cirros | active |
+--------------------------------------+--------+--------+
```

## Phần 3: Cài đặt Compute service - Nova

### Thiết lập tại Controller node

#### Khởi tạo
> Trước khi cài đặt, cấu hình Nova, ta cần tạo DB, chứng thực, API endpoint.

Tạo Database:
```
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
```

Khởi tạo biến môi trường Admin CLI:
```
. admin-openrc
```

Tạo chứng thực Compute service:
- Tạo nova user:
  ```
  openstack user create --domain default --password-prompt nova
  User Password: Welcome123
  Repeat User Password: Welcome123
  ```
- Thêm admin role vào nova user:
  ```
  openstack role add --project service --user nova admin
  ```
- Tạo đối tượng Nova service:
  ```
  openstack service create --name nova --description "OpenStack Compute" compute
  ```

Tạo Compute API service endpoints:
```
openstack endpoint create --region RegionOne compute public http://172.16.4.200:8774/v2.1
openstack endpoint create --region RegionOne compute internal http://172.16.4.200:8774/v2.1
openstack endpoint create --region RegionOne compute admin http://172.16.4.200:8774/v2.1
```

Tạo Placement service user:
```
openstack user create --domain default --password-prompt placement
User Password: Welcome123
Repeat User Password: Welcome123
```

Thêm placement user tới service project với admin role:
```
openstack role add --project service --user placement admin
```

Tạo đối tượng placement API trong service catalog:
```
openstack service create --name placement --description "Placement API" placement
```

Tạo Placement API service endpoints:
```
openstack endpoint create --region RegionOne placement public http://172.16.4.200:8778
openstack endpoint create --region RegionOne placement internal http://172.16.4.200:8778
openstack endpoint create --region RegionOne placement admin http://172.16.4.200:8778
```

#### Cài đặt, cấu hình các thành phần
Cài đặt gói:
```
yum install openstack-nova-api openstack-nova-conductor \
  openstack-nova-console openstack-nova-novncproxy \
  openstack-nova-scheduler openstack-nova-placement-api -y
```

Tạo file backup cấu hình:
```
cp /etc/nova/nova.conf /etc/nova/nova.conf.bak
```
> cat /etc/nova/nova.conf | egrep -v "(^#.*|^$)"


Chỉnh sửa file `/etc/nova/nova.conf`:
```
vi /etc/nova/nova.conf
```
- Nội dung
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

Thêm cấu hình /etc/httpd/conf.d/00-nova-placement-api.conf
```
echo '
<Directory /usr/bin>
   <IfVersion >= 2.4>
      Require all granted
   </IfVersion>
   <IfVersion < 2.4>
      Order allow,deny
      Allow from all
   </IfVersion>
</Directory>' >> /etc/httpd/conf.d/00-nova-placement-api.conf
```

Khởi động lại httpd service:
```
systemctl restart httpd
```

Đồng bộ nova-api database:
```
su -s /bin/sh -c "nova-manage api_db sync" nova
```

Lưu ý:
- BUG
  ```
  [root@controller1 ~]# su -s /bin/sh -c "nova-manage api_db sync" nova
  /usr/lib/python2.7/site-packages/oslo_db/sqlalchemy/enginefacade.py:332: NotSupportedWarning: Configuration option(s) ['use_tpool'] not supported
    exception.NotSupportedWarning
  ```
  > Bỏ qua lỗi warning


Đăng ký cell0 database:
```
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
```

Tạo cell1 cell:
```
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova

#KQ
109e1d4b-536a-40d0-83c6-5f121b82b650
```

Đồng bộ nova database:
```
su -s /bin/sh -c "nova-manage db sync" nova
```

Xác thực nova cell0 và cell1 đã được đăng ký chính xác:
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

Discover cell
```
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
```
#### Khởi tạo dịch vụ
```
systemctl enable openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service

systemctl start openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service   openstack-nova-conductor.service openstack-nova-novncproxy.service
```

### Cấu hình tại Compute node
#### Cài đặt, cấu hình các thành phân

Cài đặt các gói:
```
yum install openstack-nova-compute -y
```

Sao lưu file cấu hình
```
cp /etc/nova/nova.conf /etc/nova/nova.conf.bak
```


Chỉnh sửa `/etc/nova/nova.conf`
> my_ip = 172.16.4.201 # IP HOST COMPUTE management network interface 
```
vi /etc/nova/nova.conf
```
- Nội dung
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

#### Khởi tạo dịch vụ
Kiểm tra loại ảo hóa của phần cứng
```
egrep -c '(vmx|svm)' /proc/cpuinfo
```
> Nếu giá trị trả là là '0', Ta cần cấu hình libvirt để sử dụng QEMU, KVM.

- Chỉnh sửa [libvirt] section trong file `/etc/nova/nova.conf`:
  ```
  [libvirt]
  # ...
  virt_type = qemu
  ```

Chạy dịch vụ Compute:
```
systemctl enable libvirtd.service openstack-nova-compute.service
systemctl start libvirtd.service openstack-nova-compute.service
```

> Nếu dịch vụ nova-compute không chạy, kiểm tra log tại `/var/log/nova/nova-compute.log`. 

> Lỗi cơ bản thường gặp AMQP tại controller bị chặn bởi firewall theo port `5672`. (Khi có cấu hình firewall)


### Sau khi cấu hình nova compute, quay trở lại node controller và kiểm tra các service đã lên hay chưa.

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


### Thêm compute node tới cell database
> Thực hiện tại node controller

Khởi tạo biến môi trường Admin CLI
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

Discover các compute host:
```
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova

KQ:
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting computes from cell 'cell1': e0512d74-aff1-4734-94f5-0538305d5383
Checking host mapping for compute host 'compute1': dbd4d559-a783-4162-a932-aa2cfd74a083
Creating host mapping for compute host 'compute1': dbd4d559-a783-4162-a932-aa2cfd74a083
Found 1 unmapped computes in cell: e0512d74-aff1-4734-94f5-0538305d5383
```

### Xác nhận dịch vụ
> Kiểm tra các dịch vụ đã khởi tạo

Khởi tạo biến môi trường Admin CLI
```
. admin-openrc
```

Liệt kê service compute đã cấu hình:
```
openstack compute service list
```

Liệt kê danh sách API endpoint trong Identity service, xác thực Identity service:
```
openstack catalog list
```

Liệt kê các Image service, xác thực trạng thái service:
```
openstack image list
```

Kiểm tra các cell và Placement API:
```
nova-status upgrade check
```

## Phần 4: Cài đặt Networking service - Neutron
###  Kiểm tra mạng 
Yêu cầu:
> Kiểm tra lại cấu hình toàn bộ IP tại các node OPS và file Host, kiểm tra các node đã ping thông tới mạng và các node theo mô hình

Từ các node, test truy cập Internet:
```
ping -c 4 openstack.org
```

Tại các node, kiểm tra ping:
```
fping 172.16.4.200 172.16.4.201 172.16.4.202
fping 10.0.3.10 10.0.3.11 10.0.3.12
fping 172.16.9.10 172.16.9.11 172.16.9.12
fping controller1 compute1 compute2
```

### Cài đặt tại Controller node
> Thực hiện tại controller

#### Khởi tạo cấu hình cơ bản
Tạo database:
```
mysql -u root -pWelcome123

CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'Welcome123';
exit
```

Khởi tạo biến môi trường admin, thực hiện CLI:
```
. admin-openrc
```

Tạo chứng thực service:
- Tạo neutron user:
  ```
  openstack user create --domain default --password-prompt neutron

  User Password: Welcome123
  Repeat User Password: Welcome123
  ```

- Tạo admin role tới neutron user:
  ```
  openstack role add --project service --user neutron admin
  ```

- Tạo đối tượng neutron service:
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

Tạo Networking service API endpoints:
```
openstack endpoint create --region RegionOne network public http://172.16.4.200:9696
openstack endpoint create --region RegionOne network internal http://172.16.4.200:9696
openstack endpoint create --region RegionOne network admin http://172.16.4.200:9696
```

#### Cấu hình: Provider network
> Thực hiện tại Controller

> Lưu ý: Để có thể cài được self-service network, trước tiên phải cài provider network

Cài đặt các thành phần
```
yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables -y
```

Sao lưu file cấu hình
```
cp /etc/neutron/neutron.conf /etc/neutron/neutron.conf.bak
```

Cấu hình Chỉnh sửa file `/etc/neutron/neutron.conf`:
```
vi /etc/neutron/neutron.conf
```
- Nội dung
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

#### Cấu hình Modular Layer 2 (ML2) plug-in
> ML2 plug-in sử dụng cho kỹ thuật Linux brigde, xây dựng virtual network layer-2 (bridging and switching) sử dụng cho instance


Sao lưu file cấu hình Modular Layer 2
```
cp /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/ml2_conf.ini.bak
```

Cấu hình file `/etc/neutron/plugins/ml2/ml2_conf.ini`:
```
vi /etc/neutron/plugins/ml2/ml2_conf.ini
```
- Nội dung
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

Cấu hình Linux bridge agent
> Linux bridge agent xây dựng layer-2 (bridging and switching) virtual networking infrastructure cho instances xử lý các security group.

Sao lưu file cấu hình Linux bridge agent
```
cp /etc/neutron/plugins/ml2/linuxbridge_agent.ini /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak
```

Chỉnh sửa `/etc/neutron/plugins/ml2/linuxbridge_agent.ini`
> physical_interface_mappings = provider:ens224 # Interface provider

```
vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini
```
- Nội dung
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

#### Cấu hình DHCP agent
> DHCP agent cung cấp dịch vụ DHCP cho virtual network.

Sao lưu file cấu hình DHCP agent
```
cp /etc/neutron/dhcp_agent.ini /etc/neutron/dhcp_agent.ini.bak
```

Chỉnh sửa `/etc/neutron/dhcp_agent.ini`
```
vi /etc/neutron/dhcp_agent.ini
```
- Nội dung
  ```
  [DEFAULT]
  interface_driver = linuxbridge
  dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
  enable_isolated_metadata = true
  ```

#### Cấu hình metadata agent
> Metadata agent cung cấp thông tin cấu hình cho instance (như chứng thực instance).

Chỉnh sửa `/etc/neutron/neutron.conf`
```
vi /etc/neutron/neutron.conf
```
- Nội dung
  ```
  [DEFAULT]
  # ...
  nova_metadata_host = 172.16.4.200
  metadata_proxy_shared_secret = Welcome123
  ```

#### Cấu hình Compute service sử dụng the Networking service
> Tại Controller 

Chỉnh sửa `/etc/nova/nova.conf`
```
vi /etc/nova/nova.conf
```
- Nội dung
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

### Khởi tạo dịch vụ
Các Networking service initialization script yêu cầu symbolic link `/etc/neutron/plugin.ini` tới ML2 plug-in config file `/etc/neutron/plugins/ml2/ml2_conf.ini`

Nếu Symbolic link không tồn tại, tạo cmd:
```
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```

Đồng bộ database:
```
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

Khởi động lại Compute API service:
```
systemctl restart openstack-nova-api.service
```

Chạy Networking services.
```
systemctl enable neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service

systemctl start neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
```

### Cài đặt tại Compute node

#### Yêu cầu khởi tạo
Cài đặt các thành phần
```
yum install openstack-neutron-linuxbridge ebtables ipset -y
```

#### Cấu hình các thành phần
> Các thành phần cần cấu hình bao gồm authentication mechanism, message queue, plug-in.

Sao lưu file cấu hình
```
cp /etc/neutron/neutron.conf /etc/neutron/neutron.conf.bak
```

Cấu hình `/etc/neutron/neutron.conf`
> In the [database] section, comment out any connection options because compute nodes do not directly access the database.

```
vi /etc/neutron/neutron.conf
```
- Nội dung
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

#### Cấu hình Provider network
Cấu hình Linux bridge agent
> Linux bridge agent xây dựng layer-2 (bridging và switching) virtual networking infrastructure cho instances và xử lý các security group.

Sao lưu file cấu hình
```
cp /etc/neutron/plugins/ml2/linuxbridge_agent.ini /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak
```

Chỉnh sửa `/etc/neutron/plugins/ml2/linuxbridge_agent.ini`
```
vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini
```
- Nội dung
  ```
  [linux_bridge]
  physical_interface_mappings = provider:ens224

  [vxlan]
  enable_vxlan = false

  [securitygroup]
  enable_security_group = true
  firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
  ```

#### Cấu hình Compute service sử dụng Networking service

Cấu hình file `/etc/nova/nova.conf`:
```
vi /etc/nova/nova.conf
```
- Nội dung
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

#### Khởi tạo dịch vụ

Khởi động lại Compute service:
```
systemctl restart openstack-nova-compute.service
```

Chạy Linux bridge agent:
```
systemctl enable neutron-linuxbridge-agent.service
systemctl start neutron-linuxbridge-agent.service
```

## Phần 5: Cài đặt Cinder service
Lưu ý:
- Với bài, Storage node và Controller là 1, nên cấu hình 2 thành phần sẽ chồng lên nhau

### Cài đặt storage node
> Cấu hình trên controller (yêu cầu 2 ổ, 1 cho cinder volume) 

> Môi hình ghép, cơ bản có thể tách riêng Storage node ra node riêng

#### Thiết lập gói hỗ trợ
Cài đặt gói
```
yum install lvm2 device-mapper-persistent-data -y
```

Chạy LVM metadata service:
```
systemctl enable lvm2-lvmetad.service
systemctl start lvm2-lvmetad.service
```

Tạo LVM volume /dev/sdb:
```
pvcreate /dev/sdb
vgcreate cinder-volumes /dev/sdb
```

Chỉnh sửa file `/etc/lvm/lvm.conf`:
```
vi /etc/lvm/lvm.conf
```
- Nội dung
  ```
  devices {
  ...
  filter = [ "a/sdb/", "r/.*/"]
  ```

#### Cài đặt các thành phần
Cài đặt gói
```
yum install openstack-cinder targetcli python-keystone -y
```

Chỉnh sửa: `/etc/cinder/cinder.conf`
```
vi /etc/cinder/cinder.conf
```
- Nội dung
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

#### Khởi tạo dịch vụ
```
systemctl enable openstack-cinder-volume.service target.service
systemctl start openstack-cinder-volume.service target.service
```

### Cài đặt controller node
#### Yêu cầu thiết lập

Tạo database:
```
mysql -u root -pWelcome123
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' \
  IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' \
  IDENTIFIED BY 'Welcome123';
exit
```

Khởi tạo biến môi trường admin:
```
. admin-openrc
```

Tạo chứng chỉ user:
- Tạo cinder user:
  ```
  openstack user create --domain default --password-prompt cinder
  User Password:
  Repeat User Password:
  ```
- Tạo admin role cho cinder user:
  ```
  openstack role add --project service --user cinder admin
  ```
- Tạo đối tượng cinderv2 và cinderv3 service:
  ```
  openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2
  openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
  ```
- Tạo Block Storage service API endpoints:
  ```
  openstack endpoint create --region RegionOne volumev2 public http://172.16.4.200:8776/v2/%\(project_id\)s
  openstack endpoint create --region RegionOne volumev2 internal http://172.16.4.200:8776/v2/%\(project_id\)s
  openstack endpoint create --region RegionOne volumev2 admin http://172.16.4.200:8776/v2/%\(project_id\)s

  openstack endpoint create --region RegionOne volumev3 public http://172.16.4.200:8776/v3/%\(project_id\)s
  openstack endpoint create --region RegionOne volumev3 internal http://172.16.4.200:8776/v3/%\(project_id\)s
  openstack endpoint create --region RegionOne volumev3 admin http://172.16.4.200:8776/v3/%\(project_id\)s
  ```

#### Cài đặt các thành phần

Cài đặt gói:
```
yum install openstack-cinder -y
```

Chỉnh sửa file cấu hình `/etc/cinder/cinder.conf`:
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

Đồng bộ Block Storage database:
```
su -s /bin/sh -c "cinder-manage db sync" cinder
```
> Bỏ qua thông báo  Option "logdir" from group "DEFAULT" is deprecated. Use option "log-dir" from group "DEFAULT".

#### Khởi tạo dịch vụ
Khởi động lại Compute API service:
```
systemctl restart openstack-nova-api.service
```

Chạy Block Storage services:
```
systemctl enable openstack-cinder-api.service openstack-cinder-scheduler.service
systemctl start openstack-cinder-api.service openstack-cinder-scheduler.service
```

## Phần 6: Cài đặt Dashboard OpenStack– Horizon 
> install and configure the dashboard on the controller node.

### Cài đặt và cấu hình các thành phần
Cài đặt gói
```
yum install openstack-dashboard -y
```

Sao lưu file cấu hình
```
cp /etc/openstack-dashboard/local_settings /etc/openstack-dashboard/local_settings.bak
```

Chỉnh sửa `/etc/openstack-dashboard/local_settings`
```
vi /etc/openstack-dashboard/local_settings
```
- Nội dung
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

Tạo và thêm /etc/httpd/conf.d/openstack-dashboard.conf nếu không có.
```
echo "WSGIApplicationGroup %{GLOBAL}" >> /etc/httpd/conf.d/openstack-dashboard.conf
```

Khởi động lại dịch vụ
```
systemctl restart httpd.service memcached.service
```

#### Truy cập Dashboard.

Truy cập thông qua `http://172.16.4.200/dashboard`.

Mật khẩu mặc đinh `admin` hoặc `demo` user, domain `default`.

# Nguồn

https://docs.openstack.org/install-guide/index.html