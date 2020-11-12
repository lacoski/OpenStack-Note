# Cài đặt Ceph AIO (Luminous)

https://github.com/lacoski/tutorial-ceph/blob/master/docs/setup/ceph-luminous-aio.md

## Phần 1: Chuẩn bị

### Phân hoạch

vlan CephCOM: eth0: 10.10.11.96
vlan CephREP: eth1: 10.10.12.96

4 Disk:
- vda: sử dụng để cài OS
- vdb,vdc,vdd: sử dụng làm OSD (nơi chứa dữ liệu)


### Setup node

hostnamectl set-hostname cephaio

echo "Setup IP eth0"
nmcli c modify eth0 ipv4.addresses 10.10.11.96/24
nmcli c modify eth0 ipv4.gateway 10.10.11.1
nmcli c modify eth0 ipv4.dns 8.8.8.8
nmcli c modify eth0 ipv4.method manual
nmcli con mod eth0 connection.autoconnect yes

echo "Setup IP eth1"
nmcli c modify eth1 ipv4.addresses 10.10.12.96/24
nmcli c modify eth1 ipv4.method manual
nmcli con mod eth1 connection.autoconnect yes

sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
systemctl stop firewalld
systemctl disable firewalld

curl -Lso- https://raw.githubusercontent.com/nhanhoadocs/scripts/master/Utilities/cmdlog.sh | bash

timedatectl set-timezone Asia/Ho_Chi_Minh
yum -y install chrony
systemctl enable chronyd.service
systemctl restart chronyd.service
chronyc sources

init 6

cat << EOF >> /etc/hosts
10.10.11.96 cephaio
EOF

Lưu ý snapshot begin

### Bổ sung user cephuser

sudo useradd -d /home/cephuser -m cephuser
sudo passwd cephuser

echo "cephuser ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/cephuser
sudo chmod 0440 /etc/sudoers.d/cephuser

### Bổ sung repo cài đặt Ceph

cat <<EOF> /etc/yum.repos.d/ceph.repo
[ceph]
name=Ceph packages for $basearch
baseurl=https://download.ceph.com/rpm-luminous/el7/x86_64/
enabled=1
priority=2
gpgcheck=1
gpgkey=https://download.ceph.com/keys/release.asc

[ceph-noarch]
name=Ceph noarch packages
baseurl=https://download.ceph.com/rpm-luminous/el7/noarch
enabled=1
priority=2
gpgcheck=1
gpgkey=https://download.ceph.com/keys/release.asc

[ceph-source]
name=Ceph source packages
baseurl=https://download.ceph.com/rpm-luminous/el7/SRPMS
enabled=0
priority=2
gpgcheck=1
gpgkey=https://download.ceph.com/keys/release.asc
EOF

yum update -y

### Cài đặt python-setuptools và Ceph deploy

yum install python-setuptools -y
yum install ceph-deploy -y

ceph-deploy --version

### Cấu hình ssh key

ssh-keygen

cat <<EOF> /root/.ssh/config
Host cephaio
    Hostname cephaio
    User cephuser
EOF

ssh-copy-id cephaio

### Cấu hình Ceph

mkdir /ceph-deploy && cd /ceph-deploy

ceph-deploy new cephaio

cat << EOF >> ceph.conf
osd pool default size = 2
osd pool default min size = 1
osd pool default pg num = 128
osd pool default pgp num = 128

osd crush chooseleaf type = 0

public network = 10.10.11.96/24
cluster network = 10.10.12.96/24
EOF

### Cài đặt Ceph qua Ceph deploy

ceph-deploy install --release luminous cephaio

sudo ceph -v 

ceph-deploy mon create-initial
ceph-deploy admin cephaio

ceph-deploy mgr create cephaio

sudo ceph mgr module enable dashboard
sudo ceph mgr services

### Bổ sung OSD
ceph-deploy disk zap cephaio /dev/vdb
ceph-deploy disk zap cephaio /dev/vdc
ceph-deploy disk zap cephaio /dev/vdd

ceph-deploy osd create --data /dev/vdb cephaio
ceph-deploy osd create --data /dev/vdc cephaio
ceph-deploy osd create --data /dev/vdd cephaio

### Thay đổi cấu hình Crush map

cd /ceph-deploy
sudo ceph osd getcrushmap -o crushmap
sudo crushtool -d crushmap -o crushmap.decom
sudo sed -i 's|step choose firstn 0 type osd|step chooseleaf firstn 0 type osd|g' crushmap.decom
sudo crushtool -c crushmap.decom -o crushmap.new
sudo ceph osd setcrushmap -i crushmap.new

Snapshot cephaiook