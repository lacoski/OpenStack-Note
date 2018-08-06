# Quyền trong OpenStack
---
## Tổng quan
Mỗi project trong OpenStack (Identity, Compute, Networking, ..) có các policy riêng, chúng có thể gọi là các role. Các policy sẽ xác định user có quyền truy cập vào các tính năng trong project, các policy này được định nghĩa trong file `policy.json`.
> Đường dẫn của file `policy.json` trong `/etc/<project>/policy.json`. (Tương ứng với từng project cụ thể)

Khi API OpenStack Service project được gọi, policy engine của service sẽ đọc các định nghĩa policy để xác định request có hợp lệ. Các thay đổi trong file `policy.json` có hiệu lực ngay lập tức.

Các đặc điểm file `policy.json`: 
- Dạng JSON. 
- Mỗi policy định nghĩa bởi một dòng "<target>" : "<rule>".
- Các `target` (hay `action`) ứng với một hành động trên service.


## Cấu trúc file policy.json
- Cấu trúc File
  ```
  {
        "alias 1" : "definition 1",
        "alias 2" : "definition 2",
        "target 1" : "rule 1",
        "target 2" : "rule 2",
  }
  ```

Lưu ý:
- `alias (rule)` : đại diện cho một tập role.
- `target (action)` : đại diện cho 1 hành động của service openstacl (Tạo VM, lấy danh sách image, ..)
- `rule`: chỉ ra rule được gán

Có 2 cách gán role cho 1 action:
- Cách 1:
  ```
  "rule_1" : "role: role_1",
  "identity:create_user" : "rule: rule_1",
  ```
- Cách 2:
  ```
  "identity:create_user" : "role: role_1"
  ```

Policy hỗ trợ các toán tự `and or not`
```
"stacks:create": "not role:heat_stack_user",
"stacks:create": "role:admin or role:monitor",
```

Các rule có thể:
- Luôn đúng: khi có giá trị `"", [], @`
  ```
  "target 1" : "",
  "target 2" : [],
  "target 3" : "@",
  ```
- Luôn sai: "!"
- Kết hợp 2 biểu thức
- Biếu thức cơ bản 
  ```
  "target 1" : "not role:abc and role:bbb",  
  ```

Các giá trị có thể:
- String, number, boolean (true, false)
- Tham số API
- Tham số object 
- flag: `is_admin`

> Tham số API có thể: project_id, user_id hoặc domain_id.

## Ví dụ 
> Mục tiêu: Cấp quyền lấy danh sách image của project `glance` cho user có role monitor

> Xem thêm docs sử dụng openstack cli 

### Bước 1: Tạo project, user, role, gán role user
Thao tác:
```bash
# Tạo project
openstack project create --description 'demo role' demo1 --domain default

# Tạo user
openstack user create --project demo1 --password 123 thanhnb

# Tạo role 
openstack role create monitor 

# Gán user cho role 
openstack role add --user thanhnb --project demo1 monitor

# Liệt kê role trên user
openstack role list --user thanhnb --project demo1
```

Tạo biến môi trường cho user
```
# Tạo biến môi trường
[root@controller1 ~]# echo '
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=demo1
export OS_USERNAME=thanhnb
export OS_PASSWORD=123
export OS_AUTH_URL=http://172.16.4.200:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2 ' > thanhnb-openrc

# Khởi tạo biến môi trường
[root@controller1 ~]# source thanhnb-openrc

# Lấy token
[root@controller1 ~]# openstack token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2018-08-06T10:01:41+0000                                                                                                                                                                |
| id         | gAAAAABbaA51fG9JqlNOJ3lH-io1E6v9WHpgHtRyGX6_wZsKKcy0fhcwRrTyGzyUaH6Ymrmwt1uK6IPYs13ckdpkZKYQGj-U20TCLy9Ty0gFRp03ugEyh53RQU48A8U2SeP-FlgvZt39z9TrXZTn8gDJiorD296c1ZHNtvdDJAGB9v8T-mi0z80 |
| project_id | f6026a96fb0247fdbc93e6b12284a29b                                                                                                                                                        |
| user_id    | 9573c19cd8b24ddc8030bb39be32701e                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

# Lấy danh sách image (Role chưa có quyền liệt kê danh sách image)
[root@controller1 ~]# glance image-list
403 Forbidden: You are not authorized to complete get_images action. (HTTP 403)

```

### Bước 2: Định nghĩa file policy.json glance

Tạo bản backup cho file `policy.json`:
```
cd /etc/glance/
cp policy.json policy.json.bak
```

Chỉnh nội dung file thành:
``` 
vi /etc/glance/policy.json
```
- Nội dung
  ```
  {
    "context_is_admin":  "role:admin",
    "default": "",  
    "add_image": "role:admin",
    "delete_image": "role:admin",
    "get_image": "role:admin",
    "get_images": "role:admin or role:monitor", # Cho phép role monitor có quyền get list image (Xem thêm định dạng policy.json của glance)
    "modify_image": "role:admin",
    "publicize_image": "role:admin",
    "communitize_image": "role:admin",
    "copy_from": "role:admin",

    "download_image": "role:admin",
    "upload_image": "role:admin",

    "delete_image_location": "role:admin",
    "get_image_location": "role:admin",
    "set_image_location": "role:admin",

    "add_member": "role:admin",
    "delete_member": "role:admin",
    "get_member": "role:admin",
    "get_members": "role:admin",
    "modify_member": "role:admin",

    "manage_image_cache": "role:admin",

    "get_task": "role:admin",
    "get_tasks": "role:admin",
    "add_task": "role:admin",
    "modify_task": "role:admin",
    "tasks_api_access": "role:admin",

    "deactivate": "role:admin",
    "reactivate": "role:admin",

    "get_metadef_namespace": "role:admin",
    "get_metadef_namespaces":"role:admin",
    "modify_metadef_namespace":"role:admin",
    "add_metadef_namespace":"role:admin",

    "get_metadef_object":"role:admin",
    "get_metadef_objects":"role:admin",
    "modify_metadef_object":"role:admin",
    "add_metadef_object":"role:admin",

    "list_metadef_resource_types":"role:admin",
    "get_metadef_resource_type":"role:admin",
    "add_metadef_resource_type_association":"role:admin",

    "get_metadef_property":"role:admin",
    "get_metadef_properties":"role:admin",
    "modify_metadef_property":"role:admin",
    "add_metadef_property":"role:admin",

    "get_metadef_tag":"role:admin",
    "get_metadef_tags":"role:admin",
    "modify_metadef_tag":"role:admin",
    "add_metadef_tag":"role:admin",
    "add_metadef_tags":"role:admin"

  }
  ```

Kiểm tra lại
```
[root@controller1 ~]# glance image-list
+--------------------------------------+------------+
| ID                                   | Name       |
+--------------------------------------+------------+
| 825c647d-cfd8-41c6-81c0-57cf31d6e056 | cirros     |
| 7bafcdfb-e866-41de-bcf8-28bb58eb1047 | cirros04   |
| 24673355-bb23-4857-bc8a-d7dc52ff8e03 | cirrostest |
+--------------------------------------+------------+
```

> Hiện tại Role `monitor` đã có quyền liệt kê image 

# Nguồn 

https://github.com/hocchudong/thuctap012017/blob/master/DucPX/OpenStack/Keystone/docs/Define_role.md

https://github.com/hocchudong/thuctap012017/blob/master/XuanSon/OpenStack/Keystone/docs/Define%20Role.md

https://docs.openstack.org/ocata/config-reference/policy-json-file.html


