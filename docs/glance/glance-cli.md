# Glance CLI
---
## Tạo biến môi trường
```
```

## Có 2 phương pháp tương tác với Glance CLI
- Glance CLI
- Openstack client CLI

## Sử dụng Glance command line

### List image
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

### Tạo image
Cú pháp:
```
glance image-create [--architecture <ARCHITECTURE>]
                           [--protected [True|False]] [--name <NAME>]
                           [--instance-uuid <INSTANCE_UUID>]
                           [--min-disk <MIN_DISK>] [--visibility <VISIBILITY>]
                           [--kernel-id <KERNEL_ID>]
                           [--tags <TAGS> [<TAGS> ...]]
                           [--os-version <OS_VERSION>]
                           [--disk-format <DISK_FORMAT>]
                           [--os-distro <OS_DISTRO>] [--id <ID>]
                           [--owner <OWNER>] [--ramdisk-id <RAMDISK_ID>]
                           [--min-ram <MIN_RAM>]
                           [--container-format <CONTAINER_FORMAT>]
                           [--property <key=value>] [--file <FILE>]
                           [--progress]
```
VD:
```
[root@controller1 ~]# glance image-create --disk-format qcow2 --container-format bare --file cirros-0.4.0-x86_64-disk.img --name cirros-cli
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | 443b7623e27ecf03dc9e01ee93f67afe     |
| container_format | bare                                 |
| created_at       | 2018-08-10T02:49:47Z                 |
| disk_format      | qcow2                                |
| id               | 606d920d-b5fd-4620-a49f-b3ad0c57ef56 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | cirros-cli                           |
| owner            | 91e4db1098934a3e9cc7babf97edf007     |
| protected        | False                                |
| size             | 12716032                             |
| status           | active                               |
| tags             | []                                   |
| updated_at       | 2018-08-10T02:49:47Z                 |
| virtual_size     | None                                 |
| visibility       | shared                               |
+------------------+--------------------------------------+
```

### Show image
Cú pháp:
```
glance image-show [--human-readable] [--max-column-width <integer>] <IMAGE_ID>
```
VD:
```
[root@controller1 ~]# glance image-show 606d920d-b5fd-4620-a49f-b3ad0c57ef56
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | 443b7623e27ecf03dc9e01ee93f67afe     |
| container_format | bare                                 |
| created_at       | 2018-08-10T02:49:47Z                 |
| disk_format      | qcow2                                |
| id               | 606d920d-b5fd-4620-a49f-b3ad0c57ef56 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | cirros-cli                           |
| owner            | 91e4db1098934a3e9cc7babf97edf007     |
| protected        | False                                |
| size             | 12716032                             |
| status           | active                               |
| tags             | []                                   |
| updated_at       | 2018-08-10T02:49:47Z                 |
| virtual_size     | None                                 |
| visibility       | shared                               |
+------------------+--------------------------------------+
```

### Upload image
Cú pháp:
```
glance image-upload [--file <FILE>] [--size <IMAGE_SIZE>] [--progress]
                           <IMAGE_ID>
```

Lưu ý:
- Hoạt động update sẽ cần 1 image rỗng có sẵn từ trước

```
[root@controller1 ~]# glance image-create --name cirros-template --container-format bare --disk-format qcow2
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | None                                 |
| container_format | bare                                 |
| created_at       | 2018-08-10T02:55:33Z                 |
| disk_format      | qcow2                                |
| id               | a63878b7-69d9-4b38-8a76-a18a73369e38 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | cirros-template                      |
| owner            | 91e4db1098934a3e9cc7babf97edf007     |
| protected        | False                                |
| size             | None                                 |
| status           | queued                               |
| tags             | []                                   |
| updated_at       | 2018-08-10T02:55:33Z                 |
| virtual_size     | None                                 |
| visibility       | shared                               |
+------------------+--------------------------------------+
[root@controller1 ~]# glance image-upload --file cirros-0.4.0-x86_64-disk.img --progress a63878b7-69d9-4b38-8a76-a18a73369e38
[=============================>] 100%
```

### Thay đổi trạng thái image
Deactive image:
```
glance image-deactivate <IMAGE_ID>
```

Active image:
```
glance image-reactivate <IMAGE_ID>
```

Xóa image:
```
glance image-delete <IMAGE_ID> [<IMAGE_ID> ...]
```

## Sử dụng Openstack client cli

### List images (glance)
```
[root@controller1 ~]# openstack image list
+--------------------------------------+-----------------+--------+
| ID                                   | Name            | Status |
+--------------------------------------+-----------------+--------+
| 825c647d-cfd8-41c6-81c0-57cf31d6e056 | cirros          | active |
| 606d920d-b5fd-4620-a49f-b3ad0c57ef56 | cirros-cli      | active |
| a63878b7-69d9-4b38-8a76-a18a73369e38 | cirros-template | active |
| 7bafcdfb-e866-41de-bcf8-28bb58eb1047 | cirros04        | active |
| 24673355-bb23-4857-bc8a-d7dc52ff8e03 | cirrostest      | active |
+--------------------------------------+-----------------+--------+
```

### Show image
```
[root@controller1 ~]# openstack image show cirros
+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | 443b7623e27ecf03dc9e01ee93f67afe                     |
| container_format | bare                                                 |
| created_at       | 2018-07-09T04:45:37Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/825c647d-cfd8-41c6-81c0-57cf31d6e056/file |
| id               | 825c647d-cfd8-41c6-81c0-57cf31d6e056                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | cirros                                               |
| owner            | 91e4db1098934a3e9cc7babf97edf007                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 12716032                                             |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2018-07-09T04:45:37Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+
```

### Create image

VD:
```
[root@controller1 ~]# openstack image create --disk-format qcow2 --container-format bare --public --file cirros-0.4.0-x86_64-disk.img cirros-ops-client
+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | 443b7623e27ecf03dc9e01ee93f67afe                     |
| container_format | bare                                                 |
| created_at       | 2018-08-10T03:04:11Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/8b140594-39f7-47b3-af0c-d886d685732b/file |
| id               | 8b140594-39f7-47b3-af0c-d886d685732b                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | cirros-ops-client                                    |
| owner            | 91e4db1098934a3e9cc7babf97edf007                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 12716032                                             |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2018-08-10T03:04:11Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+
```

# Nguồn

https://docs.openstack.org/python-glanceclient/latest/cli/details.html

https://github.com/hocchudong/thuctap012017/blob/master/TamNT/Openstack/Glance/docs/3.Cac_thao_tac_su_dung_Glance.md

https://docs.openstack.org/newton/user-guide/common/cli-manage-images.html