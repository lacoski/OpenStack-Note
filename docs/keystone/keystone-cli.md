# Sử dụng KeyStone tham số dòng lệnh
---
## Khởi tạo biến môi trường:

Tạo file khởi tạo biến môi trường 

```
vi admin-env
```
- Nội dung
  ```
  export OS_PROJECT_DOMAIN_NAME=Default
  export OS_USER_DOMAIN_NAME=Default
  export OS_PROJECT_NAME=admin
  export OS_USERNAME=admin
  export OS_PASSWORD=Welcome123
  export OS_AUTH_URL=http://172.16.4.200:5000/v3
  export OS_IDENTITY_API_VERSION=3
  export OS_IMAGE_API_VERSION=2
  ```

Cấp quyền thực thi
```
chmod +x admin-env 
```

Kiểm tra biên môi trường
```
env | grep OS
```

## Check token
> Sử dụng `OpenStack client` hay Openstack cli

> Có thể sử dụng Openstack restful API (Xem thêm docs)

```
openstack token issue

+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2018-07-30T08:24:46+0000                                                                                                                                                                |
| id         | gAAAAABbXr0-Wu5ODtPpwe22DOGWKkIdCb1Jwb6UaU0Yl-8QIeIWJs4bpw-HLFL7UrFGTop0hfcOODPoqTY6gk3WUAV0Yx-zKoOCxXV3nlPtBvHgJQLKtG-IENAQvHNMEab-tijfIjO-_GJf3AE_SmNukqCuIZSWlY-9JMBfCEjJyG-zL16zG48 |
| project_id | 91e4db1098934a3e9cc7babf97edf007                                                                                                                                                        |
| user_id    | 659edb24617f4ca785f35dcb9d926f2b                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

```

## Liệt kê danh sách User

```
[root@controller1 ~]# openstack user list
+----------------------------------+-----------+
| ID                               | Name      |
+----------------------------------+-----------+
| 3b7e6eff22084565b6812680309e1459 | placement |
| 659edb24617f4ca785f35dcb9d926f2b | admin     |
| 94b41b8080014b758a5c9774c34e5073 | glance    |
| b7cc988bd22342bf9240fe3b2643bfa2 | nova      |
| d8b652cfef014428ad87f0955d6e0e9e | neutron   |
| e875df01361547688e43bc76acdee6bf | demo      |
+----------------------------------+-----------+

```

## Liệt kê các project


```
[root@controller1 ~]# openstack project list
+----------------------------------+---------+
| ID                               | Name    |
+----------------------------------+---------+
| 093000fcc44f4f8baa3826457e475161 | demo    |
| 91e4db1098934a3e9cc7babf97edf007 | admin   |
| a0b9b65b33a94e1b9c9dd703a152a3c3 | service |
+----------------------------------+---------+

```


## Liệt kê các group
```
[root@controller1 ~]# openstack project list
+----------------------------------+---------+
| ID                               | Name    |
+----------------------------------+---------+
| 093000fcc44f4f8baa3826457e475161 | demo    |
| 91e4db1098934a3e9cc7babf97edf007 | admin   |
| a0b9b65b33a94e1b9c9dd703a152a3c3 | service |
+----------------------------------+---------+

```

## Liệt kê các role
```
[root@controller1 ~]# openstack role list
+----------------------------------+-------+
| ID                               | Name  |
+----------------------------------+-------+
| 6394ac334bd3426da0d1c7bec4ae3274 | user  |
| c16444aa886d4aa18cc8bbe5c44d2a17 | admin |
+----------------------------------+-------+

```

## Liệt kê các domain
```
[root@controller1 ~]# openstack domain list
+----------------------------------+---------+---------+--------------------+
| ID                               | Name    | Enabled | Description        |
+----------------------------------+---------+---------+--------------------+
| 426ff2cd6d1348db8decfc0843b24d5b | example | True    | An Example Domain  |
| default                          | Default | True    | The default domain |
+----------------------------------+---------+---------+--------------------+
```

## Tạo domain mới
```
[root@controller1 ~]# openstack domain create acme
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description |                                  |
| enabled     | True                             |
| id          | ba7195cda21d46038f1becd884446d12 |
| name        | acme                             |
| tags        | []                               |
+-------------+----------------------------------+
```

## Tạo project trong domain
```
[root@controller1 ~]# openstack project create tims_project --domain acme --description "tims dev project"
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | tims dev project                 |
| domain_id   | ba7195cda21d46038f1becd884446d12 |
| enabled     | True                             |
| id          | 49c5aad914ac4081b0452453049b3a26 |
| is_domain   | False                            |
| name        | tims_project                     |
| parent_id   | ba7195cda21d46038f1becd884446d12 |
| tags        | []                               |
+-------------+----------------------------------+
```

## Tạo người dùng trong domain

```
[root@controller1 ~]# openstack user create tim --email tim@tim.ca \
> --domain acme --description "tims openstack user account" \
> --password s3cr3t
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| description         | tims openstack user account      |
| domain_id           | ba7195cda21d46038f1becd884446d12 |
| email               | tim@tim.ca                       |
| enabled             | True                             |
| id                  | ac061e13f989411d8496b4b3fcce032f |
| name                | tim                              |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

```

## Nguồn tham khảo

https://docs.openstack.org/python-openstackclient/latest/cli/command-list.html

https://github.com/thaonguyenvan/meditech-thuctap/blob/master/ThaoNV/Tim%20hieu%20OpenStack/docs/keystone/using-keystone.md#command

