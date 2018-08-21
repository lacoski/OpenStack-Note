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

# Nguồn

http://docs.ceph.com/docs/mimic/rbd/rbd-openstack/

