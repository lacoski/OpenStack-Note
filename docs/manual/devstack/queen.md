Cài đặt DevStack Queen
---

Cài đặt git
```
yum install git -y
```


You can quickly create a separate stack user to run DevStack with

```
sudo useradd -s /bin/bash -d /opt/stack -m stack
```

Since this user will be making many changes to your system, it should have sudo privileges:
```
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack

sudo su - stack
```

```
git clone https://git.openstack.org/openstack-dev/devstack

cd devstack
```

Create a local.conf
```
vi local.conf

[[local|localrc]]
# ADMIN_PASSWORD = passwd cho openstack
ADMIN_PASSWORD=secret 
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD

# FLOATING_RANGE=IP LOCAL NETWORK CAP
FLOATING_RANGE=192.168.2.224/27 

# FIXED_RANGE = Dai cap noi bo
FIXED_RANGE=10.11.12.0/24
FIXED_NETWORK_SIZE=256

# FLAT_INTERFACE = ip gan voi local
FLAT_INTERFACE=ens33

```


Start the install
```
git checkout stable/queens
./stack.sh
```

disable iptables sau khi cai xong
```
systemctl status iptables
systemctl stop iptables
systemctl disable iptables
```