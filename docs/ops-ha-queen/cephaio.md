# Cài đặt Ceph AIO

https://github.com/lacoski/tutorial-ceph/blob/master/docs/setup/ceph-luminous-aio.md

## Phần 1: Chuẩn bị

### Phân hoạch

vlan CephCOM: eth0: 10.10.11.96
vlan CephREP: eth1: 10.10.12.96

### Setup node

hostnamectl set-hostname cephaio1196

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

snapshot