# Migrate máy ảo trong OpenStack
---
## Tổng quát 

Migration là quá trình di chuyển máy ảo từ host vật lí này sang một host vật lí khác. Migration được sinh ra để làm nhiệm vụ bảo trì nâng cấp hệ thống. Ngày nay tính năng này đã được phát triển để thực hiện nhiều tác vụ hơn:
- Cân bằng tải: Di chuyển VMs tới các host khác kh phát hiện host đang chạy có dấu hiệu quá tải.
- Bảo trì, nâng cấp hệ thống: Di chuyển các VMs ra khỏi host trước khi tắt nó đi.
- Khôi phục lại máy ảo khi host gặp lỗi: Restart máy ảo trên một host khác.

Trong OpenStack, việc migrate được thực hiện giữa các node compute với nhau hoặc giữa các project trên cùng 1 node compute.

Openstack cung cấp 1 số phương pháp khác nhau đề Migrate VM. Các phương pháp này có các hạn chế khác nhau:
- Không thể live-migrate với shared storage.
- Không thể live-migrate nếu đã bật enabled.
- Không thể select target host nếu sử dụng nova migrate.

## Hạn chế migrate
Đối với `nova migrate`, shut down instance, chuyển sang hypervisor khác.
- Không thể chỉ định VM
- Không yêu cầu shared storage, migration mất nhiều thời gian
- Openstack lựa chọn host dựa trên tài nguyên và khả năng đáp ứng.
- Hoạt động với nhiều loại instance.

Đối với `nova live-migration`, không có down time.
- Instance sẽ chuyển trạng thái sang suppend, không tắt.
- Live-migration cho phép chỉ định host, tuy nhiên có 1 số giới hạn
- Yêu cầu shared storage
- Quá trình migration không thế thực hiện nếu host đích ko đủ tài nguyên.

Trường hợp sử dụng,
- Nếu quá trình chuyển VM có lựa chọn host đích, downtime thấp, sử dụng `nova live-migration` cmd.
- Nếu không cần chọn host, có configdrive, sử dụng `nova migrate`

## Các kiểu migrate trong OpenStack
OpenStack hỗ trợ 2 kiểu migration:
- Cold migration : Non-live migration
- Live migration :
  - True live migration (shared storage or volume-based)
  - Block live migration

### Workflow
Workflow khi thực hiện cold migrate:
- Tắt máy ảo (giống với virsh destroy) và ngắt kết nối với volume
- Di chuyển thư mục hiện tại của máy ảo (instance_dir -> instance_dir_resize)
- Nếu sử dụng QCOW2 với backing files (chế độ mặc định) thì image sẽ được convert thành dạng flat
- Với shared storage, di chuyển thư mục chứa máy ảo. Nếu không, copy toàn bộ thông qua SCP.

Workflow khi thực hiện live migrate:
- Kiểm tra lại xem storage backend có phù hợp với loại migrate sử dụng không
  - Thực hiện check shared storage với chế độ migrate thông thường
  - Không check khi sử dụng block migrations
  - Việc kiểm tra thực hiện trên cả 2 node gửi và nhận, chúng được điều phối bởi RPC call từ scheduler.
- Đối với nơi nhận
  - Tạo các kết nối càn thiết với volume.
  - Nếu dùng block migration, tạo thêm thư mục chứa máy ảo, truyền vào đó những backing files còn thiếu từ Glance và tạo disk trống.
- Tại nơi gửi, bắt đầu quá trình migration (qua url)
- Khi hoàn thành, generate lại file XML và define lại nó ở nơi chứa máy ảo mới.

### Ưu nhược điểm
Cold migrate:
- Ưu điểm:
  - Đơn giản, dễ thực hiện
  - Thực hiện được với mọi loại máy ảo
- Hạn chế:
  - Thời gian downtime lớn
  - Không thể chọn được host muốn migrate tới.
  - Quá trình migrate có thể mất một khoảng thời gian dài

Live migrate
- Ưu điểm:
  - Có thể chọn host muốn migrate
  - Tiết kiệm chi phí, tăng sự linh hoạt trong khâu quản lí và vận hành.
  - Giảm thời gian downtime và gia tăng thêm khả năng "cứu hộ" khi gặp sự cố
- Nhược điểm:
  - Dù chọn được host nhưng vẫn có những giới hạn nhất định
  - Quá trình migrate có thể fails nếu host bạn chọn không có đủ tài nguyên.
  - Bạn không được can thiệp vào bất cứ tiến trình nào trong quá trình live migrate.
  - Khó migrate với những máy ảo có dung lượng bộ nhớ lớn và trường hợp hai host khác CPU.

Bảng so sánh
pic 1

## Hướng dẫn cấu hình cold migrate trong OpenStack
> Sử dụng SSH Tunneling để migrate hoặc resize VM

Các bước cấu hình SSH Tunneling giữa các Nodes compute

### Tại node nguồn
- Tạo usermode
  ```
  usermod -s /bin/bash nova
  ```
- Thực hiện tạo key pair trên node compute nguồn cho user nova
  ```
  su nova
  ssh-keygen
  echo 'StrictHostKeyChecking no' >> /var/lib/nova/.ssh/config
  exit
  ```
- Thực hiện với quyền root, scp key pair tới compute node. Nhập mật khẩu khi được yêu cầu.
  ```
  scp /var/lib/nova/.ssh/id_rsa.pub root@compute2:/root/
  ```
### Tại node đích
- Thay đổi quyền của key pair cho user nova và add key pair đó vào SSH
  ```
  mkdir -p /var/lib/nova/.ssh
  cat /root/id_rsa.pub >> /var/lib/nova/.ssh/authorized_keys
  echo 'StrictHostKeyChecking no' >> /var/lib/nova/.ssh/config
  chown -R nova:nova /var/lib/nova/.ssh
  ```

Kiểm tra lại tại host nguồn
```
su nova
ssh compute2
exit
```

Lưu ý:
- Đối với trường hơp nhiều node compute, có thể sử dụng chung 1 cặp key, copy căp key đó tới tất cả các node, để tắt cả các node có thể xác thực lẫn nhau.

### Cold Migrate VM
- Truy cập controller
  ```
  source admin-openrc
  openstack token issue
  ```
- Liệt kê danh sách VM
  ```
  nova list
  ```
  +--------------------------------------+-----------+--------+------------+-------------+------------------------+
  | ID                                   | Name      | Status | Task State | Power State | Networks               |
  +--------------------------------------+-----------+--------+------------+-------------+------------------------+
  | dec4144a-499d-465d-bd97-fdd5c4930faf | cinder-1  | ACTIVE | -          | Running     | private=192.168.20.110 |
  | 1f1e09e9-8ce6-478a-8c44-3c796bee4a80 | nova-test | ACTIVE | -          | Running     | private=192.168.20.102 |
  +--------------------------------------+-----------+--------+------------+-------------+------------------------+
  ```
- Cold migrate VM
  ```
  nova migrate cinder-1
  ```
- Xác thực resize
  ```
  nova list
  +--------------------------------------+-----------+---------------+------------+-------------+------------------------+
  | ID                                   | Name      | Status        | Task State | Power State | Networks               |
  +--------------------------------------+-----------+---------------+------------+-------------+------------------------+
  | dec4144a-499d-465d-bd97-fdd5c4930faf | cinder-1  | VERIFY_RESIZE | -          | Running     | private=192.168.20.110 |
  | 1f1e09e9-8ce6-478a-8c44-3c796bee4a80 | nova-test | ACTIVE        | -          | Running     | private=192.168.20.102 |
  +--------------------------------------+-----------+---------------+------------+-------------+------------------------+

  nova resize-confirm cinder-1
  ```
- Kết quả
  ```
  [root@controller ~]# nova list
  +--------------------------------------+-----------+--------+------------+-------------+------------------------+
  | ID                                   | Name      | Status | Task State | Power State | Networks               |
  +--------------------------------------+-----------+--------+------------+-------------+------------------------+
  | dec4144a-499d-465d-bd97-fdd5c4930faf | cinder-1  | ACTIVE | -          | Running     | private=192.168.20.110 |
  | 1f1e09e9-8ce6-478a-8c44-3c796bee4a80 | nova-test | ACTIVE | -          | Running     | private=192.168.20.102 |
  +--------------------------------------+-----------+--------+------------+-------------+------------------------+
  ```

Lưu ý:
- Câu lệnh liệt kê VM trên Compute xác định
  ```
  [root@controller ~]# nova list --host compute1
  +--------------------------------------+-----------+--------+------------+-------------+------------------------+
  | ID                                   | Name      | Status | Task State | Power State | Networks               |
  +--------------------------------------+-----------+--------+------------+-------------+------------------------+
  | 1f1e09e9-8ce6-478a-8c44-3c796bee4a80 | nova-test | ACTIVE | -          | Running     | private=192.168.20.102 |
  +--------------------------------------+-----------+--------+------------+-------------+------------------------+
  ```
- Liệt kê hypervisior 
  ```
  nova hypervisor-list
  +--------------------------------------+---------------------+-------+---------+
  | ID                                   | Hypervisor hostname | State | Status  |
  +--------------------------------------+---------------------+-------+---------+
  | 0840adc8-7e50-48a4-9c7d-392f863b9b63 | compute1            | up    | enabled |
  | 04f35188-f239-4b70-a571-43bb3ec10b76 | compute2            | up    | enabled |
  +--------------------------------------+---------------------+-------+---------+
  ```

## Cấu hình live migrate trong Openstack
### OpenStack hỗ trợ 2 loại live migrate:
- True live migration (shared storage or volume-based) : Trong trường hợp này, máy ảo sẽ được di chuyển sửa dụng storage mà cả hai máy computes đều có thể truy cập tới. Nó yêu cầu máy ảo sử dụng block storage hoặc shared storage.
- Block live migration : Mất một khoảng thời gian lâu hơn để hoàn tất quá trình migrate bởi máy ảo được chuyển từ host này sang host khác. Tuy nhiên nó lại không yêu cầu máy ảo sử dụng hệ thống lưu trữ tập trung.

Các yêu cầu chung:
- Cả hai node nguồn và đích đều phải được đặt trên cùng subnet và có cùng loại CPU.
- Cả controller và compute đều phải phân giải được tên miền của nhau.
- Compute node buộc phải sử dụng KVM với libvirt.

Lưu ý, `live-migration` làm việc với 1 số loại VM và storage
- Shared storage: Cả 2 hypervisor có quyền truy cập shared storage chung.
- Block storage: VM sử dụng root disk, không tương thích với các loại read-only devices.
- Volume storage: VM sử dụng iSCSI volumes.

### Cấu hình live migration
- Cấu hình tại các node Compute
  ```
  sed -i 's/#listen_tls = 0/listen_tls = 0/g' /etc/libvirt/libvirtd.conf
  sed -i 's/#listen_tcp = 1/listen_tcp = 1/g' /etc/libvirt/libvirtd.conf
  sed -i 's/#auth_tcp = "sasl"/auth_tcp = "none"/g' /etc/libvirt/libvirtd.conf
  sed -i 's/#LIBVIRTD_ARGS="--listen"/LIBVIRTD_ARGS="--listen"/g' /etc/sysconfig/libvirtd
  ```
- Khởi động lại dịch vụ
  ```
  systemctl restart libvirtd
  systemctl restart openstack-nova-compute.service
  ```
- Nếu sử dụng dạng block device:
  ```
  [libvirt]
  block_migration_flag=VIR_MIGRATE_UNDEFINE_SOURCE, VIR_MIGRATE_PEER2PEER, VIR_MIGRATE_LIVE, VIR_MIGRATE_NON_SHARED_INC
  ```
- Khởi động lại dịch vụ
  ```
  systemctl restart openstack-nova-compute.service
  ```

### Live migrate VM:
- Cấu trúc:
  ```
  nova live-migration <Tên VM> <Node chuyển VM tới> [--block-migrate]
  ```

Lưu ý:
- Nếu sử dụng block live migrate, thêm tham sô `--block-migrate`
- Không thể sử dụng block migrate cho VM sử dụng share storage

- Hiện thị thông tin trạng thái VM
  ```
  nova show <Tên VM>
  ```

VD:
```
# Dạng true Migrate
nova live-migration vm04 compute1

# Dạng block Migrate
nova live-migration --block-migrate vm04 compute1
```

### Workflow Live migrate

pic 2 3


## Lưu ý:

### Với cold migrate, không chuyển key, hoặc cấu hình key sai giữa các node Compute.
```
Stdout: u''
Stderr: u'Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password).\r\n'
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server Traceback (most recent call last):
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/site-packages/oslo_messaging/rpc/server.py", line 163, in _process_incoming
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server     res = self.dispatcher.dispatch(message)
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/site-packages/oslo_messaging/rpc/dispatcher.py", line 220, in dispatch
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server     return self._do_dispatch(endpoint, method, ctxt, args)
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/site-packages/oslo_messaging/rpc/dispatcher.py", line 190, in _do_dispatch
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server     result = func(ctxt, **new_args)
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/site-packages/nova/exception_wrapper.py", line 76, in wrapped
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server     function_name, call_dict, binary)
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/site-packages/oslo_utils/excutils.py", line 220, in __exit__
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server     self.force_reraise()
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/site-packages/oslo_utils/excutils.py", line 196, in force_reraise
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server     six.reraise(self.type_, self.value, self.tb)
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/site-packages/nova/exception_wrapper.py", line 67, in wrapped
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server     return f(self, context, *args, **kw)
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/site-packages/nova/compute/manager.py", line 186, in decorated_function
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server     "Error: %s", e, instance=instance)
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/site-packages/oslo_utils/excutils.py", line 220, in __exit__
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server     self.force_reraise()
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/site-packages/oslo_utils/excutils.py", line 196, in force_reraise
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server     six.reraise(self.type_, self.value, self.tb)
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/site-packages/nova/compute/manager.py", line 156, in decorated_function
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server     return function(self, context, *args, **kwargs)
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/site-packages/nova/compute/utils.py", line 976, in decorated_function
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server     return function(self, context, *args, **kwargs)
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/site-packages/nova/compute/manager.py", line 214, in decorated_function
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server     kwargs['instance'], e, sys.exc_info())
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/site-packages/oslo_utils/excutils.py", line 220, in __exit__
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server     self.force_reraise()
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/site-packages/oslo_utils/excutils.py", line 196, in force_reraise
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server     six.reraise(self.type_, self.value, self.tb)
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/site-packages/nova/compute/manager.py", line 202, in decorated_function
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server     return function(self, context, *args, **kwargs)
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/site-packages/nova/compute/manager.py", line 4245, in resize_instance
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server     self._revert_allocation(context, instance, migration)
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/site-packages/oslo_utils/excutils.py", line 220, in __exit__
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server     self.force_reraise()
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/site-packages/oslo_utils/excutils.py", line 196, in force_reraise
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server     six.reraise(self.type_, self.value, self.tb)
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/site-packages/nova/compute/manager.py", line 4242, in resize_instance
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server     instance_type, clean_shutdown)
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/site-packages/nova/compute/manager.py", line 4300, in _resize_instance
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server     migration, image, disk_info, migration.dest_compute)
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server   File "/usr/lib64/python2.7/contextlib.py", line 35, in __exit__
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server     self.gen.throw(type, value, traceback)
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/site-packages/nova/compute/manager.py", line 7468, in _error_out_instance_on_exception
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server     raise error.inner_exception
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server ResizeError: Resize error: not able to execute ssh command: Unexpected error while running command.
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server Command: ssh -o BatchMode=yes 172.16.4.201 mkdir -p /var/lib/nova/instances/86e076f0-c9b9-4ee3-aa61-5684db2cafbe
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server Exit code: 255
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server Stdout: u''
2018-08-23 23:25:58.156 1889 ERROR oslo_messaging.rpc.server Stderr: u'Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password).\r\n'
```



# Nguồn
https://github.com/thaonguyenvan/meditech-thuctap/blob/master/ThaoNV/Tim%20hieu%20OpenStack/docs/advance/migration.md

https://raymii.org/s/articles/Openstack_-_(Manually)_migrating_(KVM)_Nova_Compute_Virtual_Machines.html

https://01.org/sites/default/files/dive_into_vm_live_migration_2.pdf

