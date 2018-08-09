# Cài đặt Glance - Image service
---

## Khởi tạo
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

## Cài đặt các thành phần


### Cài đặt gói
```
yum install openstack-glance -y
```

Glance gồm 2 file cấu hình chính:
- glance-registry.conf
- glance-api.conf

### Cấu hình glance-api
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

Lưu ý:
- Cấu hình kết nối database (`[database]`)
- Cấu hình kết nối keystone (`[keystone_authtoken]`)


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

## Khởi tạo dịch vụ
```
systemctl enable openstack-glance-api.service openstack-glance-registry.service
systemctl start openstack-glance-api.service openstack-glance-registry.service
```

## Tạo Image demo
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
