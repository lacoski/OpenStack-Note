# Cấu hình cơ bản keystone
---
## Tổng quan
Cấu hình cơ bản keystone sẽ bao gồm:
- 1. Cấu hình db mysql cho keystone
  - Khởi tạo database, user
- 2. Cài đặt gói
- 3. Cấu hình file khởi tạo:
  - Loại token sẽ sử dụng (UUID hoặc Fernet))
  - Kết nối db
- 4. Đồng bộ db sau khi đã kết nối
- 5. Khởi tạo fernet key (trường hợp dùng fernet token)  
- 6. Khởi tạo Bootstrap identity service (khởi tạo thông số cơ bản keystone)
  
> Xem thêm các bài viết (đã có giải thích quá trình)


## Cài đặt thủ công
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
> Cấu hình UUID: [xem thêm](config-uuid-keystone.md)

### Đồng bộ Identity service database:
```
su -s /bin/sh -c "keystone-manage db_sync" keystone
```
> Cấu trúc db keystone [xem thêm](cautruc-db-keystone.md)

> keystone-manage: [xem thêm](keystone-manager.md)

### Khởi tạo Fernet key repositories:
```
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```
> Quá trình khởi tạo fernet token [xem thêm](fernet-token.md)

### Khởi tạo Bootstrap Identity service:
```
keystone-manage bootstrap --bootstrap-password Welcome123 \
  --bootstrap-admin-url http://172.16.4.200:5000/v3/ \
  --bootstrap-internal-url http://172.16.4.200:5000/v3/ \
  --bootstrap-public-url http://172.16.4.200:5000/v3/ \
  --bootstrap-region-id RegionOne
```
> Cấu trúc db keystone [xem thêm](cautruc-db-keystone.md)

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