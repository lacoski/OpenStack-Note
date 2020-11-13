# Tích hợp Openstack HA với Ceph

## Phần 1: Chuẩn bị

### Cài đặt gói bổ sung trên các CTL, COM

> Thực hiện trên CTL và COM

yum install epel-release -y 
yum install python-rbd -y
yum install ceph-common -y

### Tạo pool trên Ceph

> Thực hiện trên CEPH 

ceph osd pool create volumes 64 64
ceph osd pool create vms 16 16
ceph osd pool create images 8 8
ceph osd pool create backups 32 32

rbd pool init volumes
rbd pool init vms
rbd pool init images
rbd pool init backups

### Chuyển key ceph tới 3 CTL, 1 COM

#### Copy sang CTL 1 2 3 (HA)

ssh 10.10.11.87 sudo tee /etc/ceph/ceph.conf < /etc/ceph/ceph.conf
ssh 10.10.11.88 sudo tee /etc/ceph/ceph.conf < /etc/ceph/ceph.conf
ssh 10.10.11.89 sudo tee /etc/ceph/ceph.conf < /etc/ceph/ceph.conf

#### Copy sang COM

ssh 10.10.11.94 sudo tee /etc/ceph/ceph.conf < /etc/ceph/ceph.conf

## Phần 2: Cấu hình CEPH làm backend cho Glance-Images

### Thực hiện trên Node Ceph

#### Tạo key glance

cd /ceph-deploy
ceph auth get-or-create client.glance mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=images' > ceph.client.glance.keyring

#### Chuyển key glance sang node glance (3 CTL)

ceph auth get-or-create client.glance | ssh 10.10.11.87 sudo tee /etc/ceph/ceph.client.glance.keyring
ceph auth get-or-create client.glance | ssh 10.10.11.88 sudo tee /etc/ceph/ceph.client.glance.keyring
ceph auth get-or-create client.glance | ssh 10.10.11.89 sudo tee /etc/ceph/ceph.client.glance.keyring

### Thực hiện trên 3 Node Controller

#### Dừng dịch vụ Glance trên cả 3 CTL

```
systemctl stop openstack-glance-api.service \
  openstack-glance-registry.service
```

#### Tại CTL 1

Phần quyền
```
sudo chown glance:glance /etc/ceph/ceph.client.glance.keyring
sudo chmod 0640 /etc/ceph/ceph.client.glance.keyring
```

Thêm cấu hinh `/etc/glance/glance-api.conf` lên CTL 1
```
[DEFAULT]
show_image_direct_url = True
...

[glance_store]
#stores = file,http
#default_store = file
#filesystem_store_datadir = /var/lib/glance/images/
default_store = rbd
stores = file,http,rbd
rbd_store_pool = images
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
rbd_store_chunk_size = 8
```

Restart lại dịch vụ glance
```
systemctl start openstack-glance-api.service \
  openstack-glance-registry.service
```

#### Tại CTL 2

Phần quyền
```
sudo chown glance:glance /etc/ceph/ceph.client.glance.keyring
sudo chmod 0640 /etc/ceph/ceph.client.glance.keyring
```

Thêm cấu hinh `/etc/glance/glance-api.conf` lên CTL 2
```
[DEFAULT]
show_image_direct_url = True
...

[glance_store]
#stores = file,http
#default_store = file
#filesystem_store_datadir = /var/lib/glance/images/
default_store = rbd
stores = file,http,rbd
rbd_store_pool = images
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
rbd_store_chunk_size = 8
```

Restart lại dịch vụ glance
```
systemctl start openstack-glance-api.service \
  openstack-glance-registry.service
```

#### Tại CTL 3

Phần quyền
```
sudo chown glance:glance /etc/ceph/ceph.client.glance.keyring
sudo chmod 0640 /etc/ceph/ceph.client.glance.keyring
```

Thêm cấu hinh `/etc/glance/glance-api.conf` lên CTL 2
```
[DEFAULT]
show_image_direct_url = True
...

[glance_store]
#stores = file,http
#default_store = file
#filesystem_store_datadir = /var/lib/glance/images/
default_store = rbd
stores = file,http,rbd
rbd_store_pool = images
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
rbd_store_chunk_size = 8
```

Restart lại dịch vụ glance
```
systemctl start openstack-glance-api.service \
  openstack-glance-registry.service
```

#### Kiểm tra

Test theo test case sau:
- Bật Glance tại CTL 1, tắt Glance trên CTL 2 3, test up image
- Bật Glance tại CTL 2, tắt Glance trên CTL 1 3, test up image
- Bật Glance tại CTL 3, tắt Glance trên CTL 1 2, test up image

Tắt dịch vụ glance
```
systemctl stop openstack-glance-api.service \
  openstack-glance-registry.service
```

Up thử Image (thực hiện tại CTL1), Up Image với dịch vụ Glance CTL 1 up, CTL 2 3 down
```
source admin-openrc
wget http://download.cirros-cloud.net/0.3.5/cirros-0.3.5-x86_64-disk.img

openstack image create "cirros-ceph-ctl1" \
  --file cirros-0.3.5-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --public
```

Quay lại kiểm tra trên node Ceph
```
rbd -p images ls
```

Up thử Image (thực hiện tại CTL1), Up Image với dịch vụ Glance CTL 2 up, CTL 1 3 down
```
source admin-openrc
wget http://download.cirros-cloud.net/0.3.5/cirros-0.3.5-x86_64-disk.img

openstack image create "cirros-ceph-ctl2" \
  --file cirros-0.3.5-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --public
```


Up thử Image (thực hiện tại CTL1), Up Image với dịch vụ Glance CTL 3 up, CTL 1 2 down
```
source admin-openrc
wget http://download.cirros-cloud.net/0.3.5/cirros-0.3.5-x86_64-disk.img

openstack image create "cirros-ceph-ctl3" \
  --file cirros-0.3.5-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --public
```

Quay lại kiểm tra trên node Ceph
```
rbd -p images ls
```

## Phần 3: Cấu hình CEPH làm backend cho Cinder-volume, Cinder-backup

### Thực hiện trên Node Ceph

#### Tạo key cinder và cinder-backup

cd /ceph-deploy
ceph auth get-or-create client.cinder mon 'allow r, allow command "osd blacklist", allow command "blacklistop"' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=images' > ceph.client.cinder.keyring
ceph auth get-or-create client.cinder-backup mon 'profile rbd' osd 'profile rbd pool=backups' > ceph.client.cinder-backup.keyring

#### Chuyển key cinder tới các node CTL

ceph auth get-or-create client.cinder | ssh 10.10.11.87 sudo tee /etc/ceph/ceph.client.cinder.keyring
ceph auth get-or-create client.cinder-backup | ssh 10.10.11.87 sudo tee /etc/ceph/ceph.client.cinder-backup.keyring

ceph auth get-or-create client.cinder | ssh 10.10.11.88 sudo tee /etc/ceph/ceph.client.cinder.keyring
ceph auth get-or-create client.cinder-backup | ssh 10.10.11.88 sudo tee /etc/ceph/ceph.client.cinder-backup.keyring

ceph auth get-or-create client.cinder | ssh 10.10.11.89 sudo tee /etc/ceph/ceph.client.cinder.keyring
ceph auth get-or-create client.cinder-backup | ssh 10.10.11.89 sudo tee /etc/ceph/ceph.client.cinder-backup.keyring

#### Chuyển key cinder tới các node COM

ceph auth get-or-create client.cinder | ssh 10.10.11.94 sudo tee /etc/ceph/ceph.client.cinder.keyring
ceph auth get-key client.cinder | ssh 10.10.11.94 tee /root/client.cinder

### Thực hiện trên 3 Node Controller

#### Dừng dịch vụ Cinder trên cả 3 CTL

```
systemctl stop openstack-cinder-api.service openstack-cinder-volume.service openstack-cinder-scheduler.service
```

#### Tại CTL 1

Phần quyền
```
sudo chown cinder:cinder /etc/ceph/ceph.client.cinder*
sudo chmod 0640 /etc/ceph/ceph.client.cinder*
```

Bổ sung thêm cấu hinh /etc/cinder/cinder.conf tren cac node controller

```
[DEFAULT]
....
## Thêm các giá trị này
notification_driver = messagingv2
enabled_backends = ceph
glance_api_version = 2
backup_driver = cinder.backup.drivers.ceph
backup_ceph_conf = /etc/ceph/ceph.conf
backup_ceph_user = cinder-backup
backup_ceph_chunk_size = 134217728
backup_ceph_pool = backups
backup_ceph_stripe_unit = 0
backup_ceph_stripe_count = 0
restore_discard_excess_bytes = true
host=ceph

## Bô sung section ceph
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
rbd_secret_uuid = 414ba151-4068-40c6-9d7b-84998ce6a5a6
report_discard_supported = true
```

Restart lại dịch vụ trên Node Controller
```
systemctl restart openstack-cinder-api.service openstack-cinder-volume.service
systemctl restart openstack-cinder-scheduler.service
```

Kiểm tra
```
[root@ctl01 ~]# openstack volume service list
+------------------+-----------+------+---------+-------+----------------------------+
| Binary           | Host      | Zone | Status  | State | Updated At                 |
+------------------+-----------+------+---------+-------+----------------------------+
| cinder-scheduler | ctl01     | nova | enabled | down  | 2020-11-13T03:43:52.000000 |
| cinder-scheduler | ctl02     | nova | enabled | down  | 2020-11-13T03:43:49.000000 |
| cinder-scheduler | ctl03     | nova | enabled | down  | 2020-11-13T03:43:57.000000 |
| cinder-scheduler | ceph      | nova | enabled | up    | 2020-11-13T03:56:03.000000 |
| cinder-volume    | ceph@ceph | nova | enabled | up    | 2020-11-13T03:56:06.000000 |
+------------------+-----------+------+---------+-------+----------------------------+
```

Lưu ý:
- Chỉ quan tâm `cinder-scheduler: ceph` và `cinder-volume: ceph@ceph` phải up

cinder type-create ceph
cinder type-key ceph set volume_backend_name=ceph

#### Tại CTL 2

Phần quyền
```
sudo chown cinder:cinder /etc/ceph/ceph.client.cinder*
sudo chmod 0640 /etc/ceph/ceph.client.cinder*
```

Bổ sung thêm cấu hinh `/etc/cinder/cinder.conf` tren cac node controller

```
[DEFAULT]
....
## Thêm các giá trị này
notification_driver = messagingv2
enabled_backends = ceph
glance_api_version = 2
backup_driver = cinder.backup.drivers.ceph
backup_ceph_conf = /etc/ceph/ceph.conf
backup_ceph_user = cinder-backup
backup_ceph_chunk_size = 134217728
backup_ceph_pool = backups
backup_ceph_stripe_unit = 0
backup_ceph_stripe_count = 0
restore_discard_excess_bytes = true
host=ceph

## Bô sung section ceph
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
rbd_secret_uuid = 414ba151-4068-40c6-9d7b-84998ce6a5a6
report_discard_supported = true
```

Restart lại dịch vụ trên Node Controller
```
systemctl restart openstack-cinder-api.service openstack-cinder-volume.service
systemctl restart openstack-cinder-scheduler.service
```

Kiểm tra
```
[root@ctl01 ~]# openstack volume service list
+------------------+-----------+------+---------+-------+----------------------------+
| Binary           | Host      | Zone | Status  | State | Updated At                 |
+------------------+-----------+------+---------+-------+----------------------------+
| cinder-scheduler | ctl01     | nova | enabled | down  | 2020-11-13T03:43:52.000000 |
| cinder-scheduler | ctl02     | nova | enabled | down  | 2020-11-13T03:43:49.000000 |
| cinder-scheduler | ctl03     | nova | enabled | down  | 2020-11-13T03:43:57.000000 |
| cinder-scheduler | ceph      | nova | enabled | up    | 2020-11-13T03:56:03.000000 |
| cinder-volume    | ceph@ceph | nova | enabled | up    | 2020-11-13T03:56:06.000000 |
+------------------+-----------+------+---------+-------+----------------------------+
```

Lưu ý:
- Chỉ quan tâm `cinder-scheduler: ceph` và `cinder-volume: ceph@ceph` phải up

#### Tại CTL 3

Phần quyền
```
sudo chown cinder:cinder /etc/ceph/ceph.client.cinder*
sudo chmod 0640 /etc/ceph/ceph.client.cinder*
```

Bổ sung thêm cấu hinh `/etc/cinder/cinder.conf` tren cac node controller

```
[DEFAULT]
....
## Thêm các giá trị này
notification_driver = messagingv2
enabled_backends = ceph
glance_api_version = 2
backup_driver = cinder.backup.drivers.ceph
backup_ceph_conf = /etc/ceph/ceph.conf
backup_ceph_user = cinder-backup
backup_ceph_chunk_size = 134217728
backup_ceph_pool = backups
backup_ceph_stripe_unit = 0
backup_ceph_stripe_count = 0
restore_discard_excess_bytes = true
host=ceph

## Bô sung section ceph
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
rbd_secret_uuid = 414ba151-4068-40c6-9d7b-84998ce6a5a6
report_discard_supported = true
```

Restart lại dịch vụ trên Node Controller
```
systemctl restart openstack-cinder-api.service openstack-cinder-volume.service
systemctl restart openstack-cinder-scheduler.service
```

Kiểm tra
```
[root@ctl01 ~]# openstack volume service list
+------------------+-----------+------+---------+-------+----------------------------+
| Binary           | Host      | Zone | Status  | State | Updated At                 |
+------------------+-----------+------+---------+-------+----------------------------+
| cinder-scheduler | ctl01     | nova | enabled | down  | 2020-11-13T03:43:52.000000 |
| cinder-scheduler | ctl02     | nova | enabled | down  | 2020-11-13T03:43:49.000000 |
| cinder-scheduler | ctl03     | nova | enabled | down  | 2020-11-13T03:43:57.000000 |
| cinder-scheduler | ceph      | nova | enabled | up    | 2020-11-13T03:56:03.000000 |
| cinder-volume    | ceph@ceph | nova | enabled | up    | 2020-11-13T03:56:06.000000 |
+------------------+-----------+------+---------+-------+----------------------------+
```

Lưu ý:
- Chỉ quan tâm `cinder-scheduler: ceph` và `cinder-volume: ceph@ceph` phải up

#### Thực hiện trên COM

cat > ceph-secret.xml <<EOF
<secret ephemeral='no' private='no'>
<uuid>414ba151-4068-40c6-9d7b-84998ce6a5a6</uuid>
<usage type='ceph'>
	<name>client.cinder secret</name>
</usage>
</secret>
EOF

sudo virsh secret-define --file ceph-secret.xml

virsh secret-set-value --secret 414ba151-4068-40c6-9d7b-84998ce6a5a6 --base64 $(cat /root/client.cinder)

systemctl restart openstack-nova-compute

#### Kiểm tra

Test theo test case sau:
- Bật Cinder tại CTL 1, tắt Cinder trên CTL 2 3, test up tạo mới Volume, boot VM với Cinder Volume trên horizon
- Bật Cinder tại CTL 2, tắt Cinder trên CTL 1 3, test up tạo mới Volume, boot VM với Cinder Volume trên horizon
- Bật Cinder tại CTL 3, tắt Cinder trên CTL 1 2, test up tạo mới Volume, boot VM với Cinder Volume trên horizon

```
systemctl stop openstack-cinder-api.service openstack-cinder-volume.service openstack-cinder-scheduler.service
openstack volume service list
```

Kiểm tra tại Ceph
```
rbd -p volumes ls
```