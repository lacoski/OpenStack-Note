# Tích hợp Openstack với Ceph
---
## Yêu cầu
- Cấu hình
  ```
  172.16.4.200 Controller
  172.16.4.201 Compute1
  172.16.4.202 Compute2
  ```
- Cài các gói sau trên tất cả node client (Ơ đây là Openstack)
  ```
  sudo yum install python-rbd -y
  sudo yum install ceph-common -y
  ```

## 1. Tích hợp Glance
### Tại Ceph
- Chuyến SSH key tới các node CTL, COM1, COM2
 ```
 ssh-copy-id root@172.16.4.200
 ssh-copy-id root@172.16.4.201
 ssh-copy-id root@172.16.4.202
 ```
- Tạo pool trên ceph cho glance va cinder tren node ceph
 ```
 ceph osd pool create volumes 128 128
 ceph osd pool create images 12 12
 ceph osd pool create vms 12 12
 ```
- Khởi tạo pool đối với Ceph luminous
 ```
 rbd pool init volumes
 rbd pool init images
 rbd pool init vms
 ```
- Enable pools
 ```
 ceph osd pool application enable volumes rbd
 ceph osd pool application enable images rbd
 ceph osd pool application enable vms rbd
 ```
- Chuyển cấu hình tới các node
 ```
 ssh 172.16.4.200 sudo tee /etc/ceph/ceph.conf < /etc/ceph/ceph.conf
 ssh 172.16.4.201 sudo tee /etc/ceph/ceph.conf < /etc/ceph/ceph.conf
 ssh 172.16.4.202 sudo tee /etc/ceph/ceph.conf < /etc/ceph/ceph.conf
 ```
- Cấu hình ceph làm backend cho glance
  - Tạo user glance trên node ceph
    ```
    ceph auth get-or-create client.glance mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=images'
    ```
- Chuyển key glance sang node ctl
  ```
  ceph auth get-or-create client.glance | ssh 172.16.4.200 sudo tee /etc/ceph/ceph.client.glance.keyring
  ssh 172.16.4.200 sudo chown glance:glance /etc/ceph/ceph.client.glance.keyring
  ```

### Tại Controller
- Sửa cấu hình tại Controller `/etc/glance/glance-api.conf`
  ```
  [glance_store]
  show_image_direct_url = True
  default_store = rbd
  stores = file,http,rbd
  rbd_store_pool = images
  rbd_store_user = glance
  rbd_store_ceph_conf = /etc/ceph/ceph.conf
  rbd_store_chunk_size = 8
  ```
  > Lưu ý, nếu cấu hình Glance local đã có từ trước, xóa sạch thẻ [glance_storage, điền lại]
- Khởi động lại dịch vụ
  ```
  systemctl restart openstack-glance-*
  ```
- Tạo images
  ```
  wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img 
  openstack image create "cirros-ceph" --file cirros-0.3.4-x86_64-disk.img --disk-format qcow2 --container-format bare --public
  ```
- Nếu thành công, quay lại node Ceph
  ```
  rbd -p images ls
  KQ: <image id>
  ```

## 2. Cấu hình tích hợp Cinder
### Tại Ceph
- Tạo chứng thực cho Cinder client
  ```
  ceph auth get-or-create client.cinder mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=images'
  ```
- Chuyến chứng thực các các node client cinder (Cinder sử dụng tại đâu, chuyên key tới đó), ở đây là cinder được cài trên controller và sử dụng bởi các node compute
  ```
  # Tại Controller
  ceph auth get-or-create client.cinder | ssh 172.16.4.200 sudo tee /etc/ceph/ceph.client.cinder.keyring
  ssh 172.16.4.200 sudo chown cinder:cinder /etc/ceph/ceph.client.cinder.keyring

  # Tại compute
  ceph auth get-or-create client.cinder | ssh 172.16.4.201 sudo tee /etc/ceph/ceph.client.cinder.keyring
  ceph auth get-key client.cinder | ssh 172.16.4.201 tee /root/client.cinder.key

  ceph auth get-or-create client.cinder | ssh 172.16.4.202 sudo tee /etc/ceph/ceph.client.cinder.keyring
  ceph auth get-key client.cinder | ssh 172.16.4.202 tee /root/client.cinder.key
  ```
- Sinh UUID
  ```
  uuidgen
  75fcfc41-ee46-458c-880a-adbc91b385a3
  ```
### Tạo Controller 
- Sửa cấu hình tại Cinder `/etc/cinder/cinder.conf`
  ```
  [DEFAULT]
  enabled_backends = ceph
  glance_api_version = 2

  [ceph]
  volume_driver = cinder.volume.drivers.rbd.RBDDriver
  volume_backend_name = ceph
  rbd_pool = volumes
  rbd_ceph_conf = /etc/ceph/ceph.conf
  rbd_flatten_volume_from_snapshot = false
  rbd_max_clone_depth = 5
  rbd_store_chunk_size = 4
  rados_connect_timeout = -1
  rbd_user = cinder
  rbd_secret_uuid = 75fcfc41-ee46-458c-880a-adbc91b385a3
  report_discard_supported = true
  ```
  > Lưu ý, nếu cấu hình Cinder LVM đã có từ trước, có thể xóa sạch hoặc để lại (OPS hỗ trợ multi storage)
- Khởi động lại dịch vụ (trên controller)
  ```
  systemctl restart openstack-cinder-api.service openstack-cinder-volume.service openstack-cinder-scheduler.service
  ```
### Tại Compute
- Cấu hình trên Compute node
  ```
  cat > secret.xml <<EOF
  <secret ephemeral='no' private='no'>
  <uuid>75fcfc41-ee46-458c-880a-adbc91b385a3</uuid>
  <usage type='ceph'>
      <name>client.cinder secret</name>
  </usage>
  </secret>
  EOF

  sudo virsh secret-define --file secret.xml
  virsh secret-set-value --secret 75fcfc41-ee46-458c-880a-adbc91b385a3 --base64 $(cat client.cinder.key)
  ```
- Khởi động lại dịch vụ các node compute
  ```
  systemctl restart openstack-nova-compute
  ```
  - Nếu thành công, tạo máy ảo mới, kiểm tra lại tại Ceph
  ```
  # Tại Ceph
  rbd -p volumes ls
  <Id Volume>
  ```

## Cấu hình tích hợp Nova với Ceph
### Cấu hình tại Ceph
- Tạo chứng thực cho Nova
  ```
  ceph auth get-or-create client.nova mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=vms, allow rx pool=images' -o /etc/ceph/ceph.client.nova.keyring
  ```
- Chuyển key tới các node Compute (Các node lưu trữ, chạy VM)
  ```
  scp /etc/ceph/ceph.client.nova.keyring root@172.16.4.201:/etc/ceph/
  scp /etc/ceph/ceph.client.nova.keyring root@172.16.4.202:/etc/ceph/
  ```
- Tạo key file tạo Nova client
  ```
  ceph auth get-key client.nova |ssh 172.16.4.201 tee /etc/ceph/client.nova.key
  ceph auth get-key client.nova |ssh 172.16.4.202 tee /etc/ceph/client.nova.key
  ```
  > Nova compute sẽ cần `client.nova.key` cho bước xử lý sau
### Tại các node compute
- Cấu hình quyền
  ```
  chgrp nova /etc/ceph/ceph.client.nova.keyring
  chmod 0640 /etc/ceph/ceph.client.nova.keyring
  ```
- Sinh UUID
  ```
  uuidgen
  805b9716-7fe8-45dd-8e1e-5dfdeff8b9be
  ```
- Chỉnh sửa `/etc/nova/nova.conf`
  ```
  [libvirt]
  images_rbd_pool=vms
  images_type=rbd
  rbd_secret_uuid=805b9716-7fe8-45dd-8e1e-5dfdeff8b9be
  rbd_user=nova
  images_rbd_ceph_conf = /etc/ceph/ceph.conf
  ```
- Khởi động lại dịch vụ
  ```
  systemctl restart openstack-nova-compute
  ```
- Tạo file ceph.xml
  ```
    cat << EOF > ceph.xml
    <secret ephemeral="no" private="no">
    <uuid>805b9716-7fe8-45dd-8e1e-5dfdeff8b9be</uuid>
    <usage type="ceph">
    <name>client.nova secret</name>
    </usage>
    </secret>
    EOF
  ```
- Tạo file secret cho libvirt
  ```
  virsh secret-define --file ceph.xml
  virsh secret-set-value --secret 805b9716-7fe8-45dd-8e1e-5dfdeff8b9be --base64 $(cat /etc/ceph/client.nova.key)
  ```
- Nếu thành công, tạo máy ảo mới, kiểm tra lại tại Ceph
  ```
  # Tại Ceph
  rbd -p vms ls
  <Id VM>
  ```

# File cấu hình đầy đủ

- Glance
```
[root@controller1 ~]# cat /etc/glance/glance-api.conf | egrep -v "(^#.*|^$)"
[DEFAULT]
enable_v1_api=False
bind_host=0.0.0.0
bind_port=9292
workers=4
image_cache_dir=/var/lib/glance/image-cache
registry_host=0.0.0.0
debug=False
log_file=/var/log/glance/api.log
log_dir=/var/log/glance
transport_url=rabbit://guest:guest@172.16.4.200:5672/
[cors]
[database]
connection=mysql+pymysql://glance:Welcome123@172.16.4.200/glance
[glance_store]
show_image_direct_url = True
default_store = rbd
stores = file,http,rbd
rbd_store_pool = images
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
rbd_store_chunk_size = 8
[image_format]
[keystone_authtoken]
auth_uri=http://172.16.4.200:5000/v3
auth_type=password
auth_url=http://172.16.4.200:35357
username=glance
password=Welcome123
user_domain_name=Default
project_name=services
project_domain_name=Default
[matchmaker_redis]
[oslo_concurrency]
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
driver=messagingv2
topics=notifications
[oslo_messaging_rabbit]
ssl=False
default_notification_exchange=glance
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
policy_file=/etc/glance/policy.json
[paste_deploy]
flavor=keystone
[profiler]
[store_type_location_strategy]
[task]
[taskflow_executor]

```
- Cinder
```
[DEFAULT]
enable_v3_api=True
storage_availability_zone=nova
default_availability_zone=nova
default_volume_type=iscsi
auth_strategy=keystone
osapi_volume_listen=0.0.0.0
osapi_volume_workers=4
debug=False
log_dir=/var/log/cinder
transport_url=rabbit://guest:guest@172.16.4.200:5672/
control_exchange=openstack
api_paste_config=/etc/cinder/api-paste.ini
glance_host=172.16.4.200
nova_catalog_info=compute:nova:publicURL
nova_catalog_admin_info=compute:nova:adminURL
enabled_backends = ceph
glance_api_version = 2
[backend]
[backend_defaults]
[barbican]
[brcd_fabric_example]
[cisco_fabric_example]
[coordination]
[cors]
[database]
connection=mysql+pymysql://cinder:Welcome123@172.16.4.200/cinder
[fc-zone-manager]
[healthcheck]
[key_manager]
backend=cinder.keymgr.conf_key_mgr.ConfKeyManager
[keystone_authtoken]
auth_uri=http://172.16.4.200:5000/
auth_type=password
auth_url=http://172.16.4.200:35357
username=cinder
password=Welcome123
user_domain_name=Default
project_name=services
project_domain_name=Default
[matchmaker_redis]
[nova]
[oslo_concurrency]
lock_path=/var/lib/cinder/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
driver=messagingv2
[oslo_messaging_rabbit]
ssl=False
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
policy_file=/etc/cinder/policy.json
[oslo_reports]
[oslo_versionedobjects]
[profiler]
[service_user]
[ssl]
[vault]
[ceph]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
volume_backend_name = ceph
rbd_pool = volumes
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = -1
rbd_user = cinder
rbd_secret_uuid = 75fcfc41-ee46-458c-880a-adbc91b385a3
report_discard_supported = true
```

- Nova
```
[DEFAULT]
rootwrap_config=/etc/nova/rootwrap.conf
allow_resize_to_same_host=False
vif_plugging_is_fatal=True
vif_plugging_timeout=300
cpu_allocation_ratio=16.0
ram_allocation_ratio=1.5
force_snat_range=0.0.0.0/0
metadata_host=172.16.4.200
dhcp_domain=novalocal
firewall_driver=nova.virt.firewall.NoopFirewallDriver
state_path=/var/lib/nova
report_interval=10
service_down_time=60
enabled_apis=osapi_compute,metadata
osapi_compute_listen=0.0.0.0
osapi_compute_listen_port=8774
osapi_compute_workers=4
metadata_listen=0.0.0.0
metadata_listen_port=8775
metadata_workers=4
debug=False
log_dir=/var/log/nova
transport_url=rabbit://guest:guest@172.16.4.200:5672/
image_service=nova.image.glance.GlanceImageService
osapi_volume_listen=0.0.0.0
[api]
auth_strategy=keystone
use_forwarded_for=False
fping_path=/usr/sbin/fping
[api_database]
connection=mysql+pymysql://nova_api:Welcome123@172.16.4.200/nova_api
[barbican]
[cache]
[cells]
[cinder]
[compute]
[conductor]
workers=4
[console]
[consoleauth]
[cors]
[crypto]
[database]
connection=mysql+pymysql://nova:Welcome123@172.16.4.200/nova
[devices]
[ephemeral_storage_encryption]
[filter_scheduler]
host_subset_size=1
max_io_ops_per_host=8
max_instances_per_host=50
available_filters=nova.scheduler.filters.all_filters
enabled_filters=RetryFilter,AvailabilityZoneFilter,RamFilter,DiskFilter,ComputeFilter,ComputeCapabilitiesFilter,ImagePropertiesFilter,ServerGroupAntiAffinityFilter,ServerGroupAffinityFilter,CoreFilter
use_baremetal_filters=False
weight_classes=nova.scheduler.weights.all_weighers
[glance]
api_servers=http://172.16.4.200:9292
[guestfs]
[healthcheck]
[hyperv]
[ironic]
[key_manager]
[keystone]
[keystone_authtoken]
auth_uri=http://172.16.4.200:5000/
auth_type=password
auth_url=http://172.16.4.200:35357
username=nova
password=Welcome123
user_domain_name=Default
project_name=services
project_domain_name=Default
[libvirt]
vif_driver=nova.virt.libvirt.vif.LibvirtGenericVIFDriver
[matchmaker_redis]
[metrics]
[mks]
[neutron]
url=http://172.16.4.200:9696
ovs_bridge=br-int
default_floating_pool=public
extension_sync_interval=600
service_metadata_proxy=True
metadata_proxy_shared_secret=Welcome123
timeout=60
auth_type=v3password
auth_url=http://172.16.4.200:35357/v3
project_name=services
project_domain_name=Default
username=neutron
user_domain_name=Default
password=Welcome123
region_name=RegionOne
[notifications]
notify_on_state_change=vm_and_task_state
notify_api_faults=False
[osapi_v21]
[oslo_concurrency]
lock_path=/var/lib/nova/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
driver=messagingv2
[oslo_messaging_rabbit]
ssl=False
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
policy_file=/etc/nova/policy.json
[pci]
[placement]
os_region_name=RegionOne
auth_type=password
auth_url=http://172.16.4.200:5000/
project_name=services
project_domain_name=Default
username=placement
user_domain_name=Default
password=Welcome123
[quota]
[rdp]
[remote_debug]
[scheduler]
host_manager=host_manager
driver=filter_scheduler
max_attempts=3
[serial_console]
[service_user]
[spice]
[upgrade_levels]
[vault]
[vendordata_dynamic_auth]
project_domain_name=Default
user_domain_name=Default
[vmware]
[vnc]
novncproxy_host=0.0.0.0
novncproxy_port=6080
auth_schemes=none
[workarounds]
[wsgi]
api_paste_config=api-paste.ini
[xenserver]
[xvp]
[placement_database]
connection=mysql+pymysql://nova_placement:Welcome123@172.16.4.200/nova_placement
```

# Nguồn

http://docs.ceph.com/docs/mimic/rbd/rbd-openstack/

