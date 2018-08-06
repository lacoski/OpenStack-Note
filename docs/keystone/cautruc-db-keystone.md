# Cấu trúc DB KeyStone
---
## Tổng quan
DB keystone gồm 5 thành phần chính
- Identity:
  - user
  - group
  - user-group-membership
- Assignment
  - assignment
  - role
- Resource
  - project
  - domain
- Credential
  - credential
- Trust
  - trust
  - trust_role

Các thành phần bổ sung:
- Service endpoint
- Token
## Thành phần Identity
- Idendity backend của keystone lưu trữ user, group, user-group
- Có 3 bảng để thể hiện mục đích này
  - user
  - group
  - user-group-membership

> Thành phần này sẽ được chia sẻ tới các db của service khác (vd: nova) để khoanh vùng tài nguyên sử dụng của Openstack

### user table
```
id (Primary Key)    | domain_id (Unique Key) | name (Unique Key) | enabled | password | extra | default_project_id
```

### group table
```
id (Primary Key) | domain_id (Unique Key) | name (Unique Key) | extra | description
```

### user_group_membership table
```
user_id | group_id

Primary Key: (user_id, group_id), Key: (group_id)
```

## Thành phần Assignment
- Assignment backend nắm giữ data về role assignment. Nó có vai trò phụ trợ cho "role backend".
- Có 2 loại bảng trong MySQL DB
  - assignment
  - role

### assignment table
```
type | actor_id | target_id | role_id | inherited

Primary Key: (type, actor_id, target_id, role_id), Key: (actor_id), Key: (role_id)
```

### role table
```
id | name | extra

Primary Key: (id), Unique Key: (name)
```

## Thành phần Resource
- resource backend lưu data về project và domain. 
- Có 2 loại table trong backend
  - project
  - domain

> Thành phần này sẽ được chia sẻ tới các db của service khác (vd: nova) để khoanh vùng tài nguyên sử dụng của Openstack

### project Table
```
id | domain_id | name | enabled | description | extra | parent_id

Primary Key:(id), Unique Key: (domain_id, name), Key: (parent_id)
```

### domain Table
```
id | name | extra

Primary Key: (id), Unique Key: (name)
```

## Thành phân Credential
- Credential backend giữ data liên quan tới EC2 credentials
- Có 1 bảng:
  - credential

### credential Table
```
id | user_id | project_id | blob | type | extra

Primary Key: (id)
```

## Thành phần Trust
- Trust backend lưu dữ liệu liên quan tới các quyền ủy thác cho người dùng
- có 2 bảng:
  - trust
  - trust_role

### trust table
```
id | trustor_user_id | trustee_user_id | project_id | impersonation | deleted_at | expires_at	 | remaining_uses | extra

Primary Key: (id)
```

### trust_role
```
trust_id | role_id

Primary Key: (trust_id, role_id)
```

## Thành phần service endpoint
- Quản lý endpoint service trong openstack
- gồm các bảng:
  - endpoint
  - service
  - region

### endpoint table
```
id | interface | service_id | url | enable | region_id 

Primary Key: (id)
Unique Key: (service_id, region_id)
```

### service table 
```
id | type | enable | extra 

primary Key: (id)
```

### region table 
```
id | description | extra

Primary Key: (id)
```

## Token
- Chứa token được sinh ra (UUID))
- gồm 1 bảng
  - Token

### token table
```
id | exprires | extra | valid | trust_id | user_id 

Primary Key: (id)
Unique Key: (trust_id, user_id)
```
# Nguồn

https://wiki.openstack.org/wiki/Keystone-schema-in-cassandra


