# Quotas
---
## Tổng quan
- Để tránh việc sử dụng hết tài nguyên, ta có thể thiết lập `quotas` để giới hạn tài nguyên có thể được sử dụng trên project.

## Quotas của Nova bao gồm
| Quota name	 | 	Description |
|---|---|
| cores  |  Number of instance cores (VCPUs) allowed per project. |
| fixed-ips |	Number of fixed IP addresses allowed per project. This number must be equal to or greater than the number of allowed instances.|
| floating-ips |	Number of floating IP addresses allowed per project. |
| injected-file-content-bytes | Number of content bytes allowed per injected file. |
| injected-file-path-bytes | Length of injected file path. |
| injected-files | Number of injected files allowed per project. |
| instances | Number of instances allowed per project. |
| key-pairs | Number of key pairs allowed per user. |
| metadata-items | Number of metadata items allowed per instance. |
|ram | Megabytes of instance ram allowed per project. | 
| security-groups | Number of security groups per project. |
| security-group-rules | Number of security group rules per project. |
| server-groups | Number of server groups per project. |
| server-group-members | Number of servers per server group. |

## Thây đổi quotas bằng CLI

### View and update Compute quotas for a project

List all default quotas for all projects:

```
[root@controller1 ~]# openstack quota show --default
+----------------------+-------+
| Field                | Value |
+----------------------+-------+
| cores                | 20    |
| fixed-ips            | -1    |
| floating_ips         | None  |
| health_monitors      | None  |
| injected-file-size   | 10240 |
| injected-files       | 5     |
| injected-path-size   | 255   |
| instances            | 10    |
| key-pairs            | 100   |
| l7_policies          | None  |
| listeners            | None  |
| load_balancers       | None  |
| location             | None  |
| name                 | None  |
| networks             | 100   |
| pools                | None  |
| ports                | 500   |
| project              | None  |
| project_name         | admin |
| properties           | 128   |
| ram                  | 51200 |
| rbac_policies        | 10    |
| routers              | None  |
| secgroup-rules       | 100   |
| secgroups            | 10    |
| server-group-members | 10    |
| server-groups        | 10    |
| subnet_pools         | -1    |
| subnets              | 100   |
+----------------------+-------+
```

Update a default value for a new project, for example:
```
openstack quota set --instances 15 default
```

### To view quota values for an existing project

List the currently set quota values for a project:
```
[root@controller1 ~]# openstack quota show admin
+----------------------+----------------------------------+
| Field                | Value                            |
+----------------------+----------------------------------+
| cores                | 20                               |
| fixed-ips            | -1                               |
| floating_ips         | None                             |
| health_monitors      | None                             |
| injected-file-size   | 10240                            |
| injected-files       | 5                                |
| injected-path-size   | 255                              |
| instances            | 10                               |
| key-pairs            | 100                              |
| l7_policies          | None                             |
| listeners            | None                             |
| load_balancers       | None                             |
| location             | None                             |
| name                 | None                             |
| networks             | 100                              |
| pools                | None                             |
| ports                | 500                              |
| project              | 91e4db1098934a3e9cc7babf97edf007 |
| project_name         | admin                            |
| properties           | 128                              |
| ram                  | 51200                            |
| rbac_policies        | 10                               |
| routers              | None                             |
| secgroup-rules       | 100                              |
| secgroups            | 10                               |
| server-group-members | 10                               |
| server-groups        | 10                               |
| subnet_pools         | -1                               |
| subnets              | 100                              |
+----------------------+----------------------------------+

```

To update quota values for an existing project
```
[root@controller1 ~]# openstack quota set --instances 20 admin

[root@controller1 ~]# openstack quota show admin
+----------------------+----------------------------------+
| Field                | Value                            |
+----------------------+----------------------------------+
| cores                | 20                               |
| fixed-ips            | -1                               |
| floating_ips         | None                             |
| health_monitors      | None                             |
| injected-file-size   | 10240                            |
| injected-files       | 5                                |
| injected-path-size   | 255                              |
| instances            | 20                               |
| key-pairs            | 100                              |
| l7_policies          | None                             |
| listeners            | None                             |
| load_balancers       | None                             |
| location             | None                             |
| name                 | None                             |
| networks             | 100                              |
| pools                | None                             |
| ports                | 500                              |
| project              | 91e4db1098934a3e9cc7babf97edf007 |
| project_name         | admin                            |
| properties           | 128                              |
| ram                  | 51200                            |
| rbac_policies        | 10                               |
| routers              | None                             |
| secgroup-rules       | 100                              |
| secgroups            | 10                               |
| server-group-members | 10                               |
| server-groups        | 10                               |
| subnet_pools         | -1                               |
| subnets              | 100                              |
+----------------------+----------------------------------+
```

# Nguồn

https://docs.openstack.org/nova/pike/admin/quotas.html#view-and-update-compute-quotas-for-a-project

