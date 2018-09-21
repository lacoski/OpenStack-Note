# Resize VM
---
## Thay đổi cấu hình VM
- Xem thông tin VM
  ```
  [root@controller ~]# openstack server show cinder-1
  +-------------------------------------+----------------------------------------------------------+
  | Field                               | Value                                                    |
  +-------------------------------------+----------------------------------------------------------+
  | OS-DCF:diskConfig                   | AUTO                                                     |
  | OS-EXT-AZ:availability_zone         | nova                                                     |
  | OS-EXT-SRV-ATTR:host                | compute2                                                 |
  | OS-EXT-SRV-ATTR:hypervisor_hostname | compute2                                                 |
  | OS-EXT-SRV-ATTR:instance_name       | instance-00000005                                        |
  | OS-EXT-STS:power_state              | Running                                                  |
  | OS-EXT-STS:task_state               | None                                                     |
  | OS-EXT-STS:vm_state                 | active                                                   |
  | OS-SRV-USG:launched_at              | 2018-08-27T02:33:50.000000                               |
  | OS-SRV-USG:terminated_at            | None                                                     |
  | accessIPv4                          |                                                          |
  | accessIPv6                          |                                                          |
  | addresses                           | private=192.168.20.110                                   |
  | config_drive                        |                                                          |
  | created                             | 2018-08-26T10:42:47Z                                     |
  | flavor                              | md.test (322ac2ff-25e8-41aa-82cc-a0c01ec472cf)           |
  | hostId                              | 7ced8cc4243339fa5ce19bae586e46dac4d113d95c0721d8acea70a7 |
  | id                                  | dec4144a-499d-465d-bd97-fdd5c4930faf                     |
  | image                               |                                                          |
  | key_name                            | None                                                     |
  | name                                | cinder-1                                                 |
  | progress                            | 0                                                        |
  | project_id                          | 74d88864c07a48b19ce513b901ad34d5                         |
  | properties                          |                                                          |
  | security_groups                     | name='default'                                           |
  | status                              | ACTIVE                                                   |
  | updated                             | 2018-08-27T02:34:43Z                                     |
  | user_id                             | efede9417c044d1baefddef4def5fb32                         |
  | volumes_attached                    | id='8d65753d-49e4-4cb3-9519-d08ec12d3761'                |
  +-------------------------------------+----------------------------------------------------------+
  ```
  > VM đang sử dụng flavor: md.test
- List Flavor
  ```
  [root@controller ~]# openstack flavor list
  +--------------------------------------+-----------+------+------+-----------+-------+-----------+
  | ID                                   | Name      |  RAM | Disk | Ephemeral | VCPUs | Is Public |
  +--------------------------------------+-----------+------+------+-----------+-------+-----------+
  | 322ac2ff-25e8-41aa-82cc-a0c01ec472cf | md.test   | 1024 |    5 |         0 |     1 | True      |
  | 36b989a4-b259-41f9-8d47-0e7e7b8f5c55 | md.resize | 2048 |    2 |         0 |     1 | True      |
  +--------------------------------------+-----------+------+------+-----------+-------+-----------+
  ```
- Resize VM
  ```
  nova resize cinder-1 36b989a4-b259-41f9-8d47-0e7e7b8f5c55 --poll
  Server resizing... 100% complete
  ```
- Check list VM status
  ```
  [root@controller ~]# openstack server list
  +--------------------------------------+-----------+---------------+------------------------+-------------+-----------+
  | ID                                   | Name      | Status        | Networks               | Image       | Flavor    |
  +--------------------------------------+-----------+---------------+------------------------+-------------+-----------+
  | 1f1e09e9-8ce6-478a-8c44-3c796bee4a80 | nova-test | ACTIVE        | private=192.168.20.102 | cirros-ceph | md.test   |
  | dec4144a-499d-465d-bd97-fdd5c4930faf | cinder-1  | VERIFY_RESIZE | private=192.168.20.110 |             | md.resize |
  +--------------------------------------+-----------+---------------+------------------------+-------------+-----------+  
  ```
- Chấp nhận thay đổi
  ```
  openstack server resize --confirm dec4144a-499d-465d-bd97-fdd5c4930faf
  ```
- Nếu muốn hủy bỏ thay đổi
  ```
  openstack server resize --revert dec4144a-499d-465d-bd97-fdd5c4930faf
  ```

# Nguồn
https://docs.openstack.org/newton/user-guide/cli-change-the-size-of-your-server.html

