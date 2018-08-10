# Overcommitting CPU, RAM
---
## Tổng quan
OPS cho phép overcommit CPU và Ram trên compute node. Điều này cho phép tăng số lượng instance chạy trên code, giảm giá thành và tối ưu hiệu năng của instances.

Tỷ lệ mặc định:
```
CPU allocation ratio: 16:1
RAM allocation ratio: 1.5:1
DISK ratio: 1:1
```

Tỷ lệ cấp phát 16:1 tức sheduler phân bố 16 virual core trên 1 physical core. VD: Flavor có 4 virtual cores / instance, ratio sẽ cung cấp được 48 instance trên mỗi physical node.

Công thức tính số VM trên Compute node:
```
(OR*PC)/VC
```
Thuật ngữ:
- OR: CPU overcommit ratio (virtual cores tương ứng với mỗi physical core)
- PC: Số physical cores.
- VC: Số virtual cores mỗi instance.


## Cấu hình
- Có thể chỉnh lại chỉ số ratio cho CPU, RAM tại node controller
- Cấu hình file `/etc/nova/nova.conf`
  ```
  [DEFAULT]
  cpu_allocation_ratio = 0.0
  ram_allocation_ratio = 0.0
  disk_allocation_ratio = 0.0
  ```
- Khởi động lại dịch vụ
  ```
  systemctl restart openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service   openstack-nova-conductor.service openstack-nova-novncproxy.service
  ```

Có thể dùng horizon để hiện thị số tài nguyên sử dụng
pic 1


# Nguồn 

https://docs.openstack.org/juno/config-reference/content/list-of-compute-config-options.html

https://github.com/hocchudong/thuctap012017/blob/master/XuanSon/OpenStack/Nova/docs/Overcommit.md

https://docs.openstack.org/arch-design/design-compute/design-compute-overcommit.html


