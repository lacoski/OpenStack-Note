## Cài đặt Compute service - Nova

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

### Thiết lập tại Compute node
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
