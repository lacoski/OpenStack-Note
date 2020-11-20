# Kiến trúc Network trong OpenStack (Open vSwitch)

## 1. Provider Network

![](/docs/ovs-network/images/2-openvswitch-ops/pic1.png)

Traffic sẽ đi:
- 1) Gói tin đi ra từ eth0 VM qua tap tới Port `tap` Linux Bridge (`qbr`).
- Cần thành phần Linux Bridge vì OpenStack sử dụng các rules của iptables trên tap devices để thực hiện security groups. OVS tại phiên bản cũ chưa hỗ trợ IPTable nên cần sinh ra thành phần Linux Bridge. Hiện nay OVS đã hỗ trợ Linux Bridge nên sẽ có option để Instance kết nối trực tiếp với OVS Bridge.
- 2) Tại Linux Bridge (`qbr`) từ Port tap, gói tin sẽ đi qua chuỗi iptable (bảo gồm port security và các security rule) tới Port `qvb`. Sau đó gói tin được forward từ Port `qvb` tới `qvo` tại OVS Intergration bridge (`br-int`)
- 3) Tại OVS Integration Bridge (br-init), gói tin đi từ port `qvo` tới patch `int-br-provider`, sau đó được forward tới patch `phy-br-provider` tại OVS Provider Bridge (`br-provider`)
- 4) Tại OVS Provider Bridge (br-provider), gói tin từ patch `phy-br-provider` tới Interface vật lý sau đó ra ngoài internet

### Các lưu ý:
- Linux bridge: Chứa các rules dùng cho security group. Nó có 1 đầu là tap interface có địa chỉ MAC trùng với địa chỉ MAC của card mạng trên máy ảo và một đầu là qvb... được nối với qvo... trên integration bridge.
- Integration Bridge: Bridge này thường có tên là br-int thường được dùng để "tag" và "untag" VLAN cho traffic vào và ra VM
- External bridge: Bridge này sẽ được gán với interface external để đi ra ngoài internet (có thể dùng bond hoặc không)

### Vlan Provider Network

![](/docs/ovs-network/images/2-openvswitch-ops/pic1-1.png)

## 2. Self-service Network

![](/docs/ovs-network/images/2-openvswitch-ops/pic2.png)

Traffic sẽ đi:
- 1) Gói tin đi ra từ eth0 VM qua tap tới Port `tap` Linux Bridge (`qbr`).
- 2) Tại OVS Integration Bridge (br- 2) Tại Linux Bridge (`qbr`) từ Port `tap`, gói tin sẽ đi qua chuỗi iptable (bảo gồm port security và các security rule) tới Port `qvb`. Sau đó gói tin được forward từ Port `qvb` tới `qvo` tại OVS Intergration bridge (`br-int`)
-init), gói tin đi từ port `qvo` tới patch `tun`, sau đó được forward tới patch `int` tại OVS Tunnel Bridge (`br-tun`)
- 3) Tại OVS Tunnel Bridge (br-tun), gói tin từ patch `int` tới Interface vật lý sau đó được overlays tới Network Node
- 4) Gói tin sau khi tới Network Node, thông qua Interface vật lý (interface 3) tới OVS Tunnel Bridge (`br-tun`)
- 5) Tiếp theo gói tin đi qua Patch `int` tới Patch `tun` tại OVS Integration Bridge (`br-int`) và tại đó được định tuyền bởi Router Namespace (`qrouter`).
- 6) Sau khi được định tuyến, gói tin tới Patch `int-br-provider`, sau đó được forward tới patch `phy-br-provider` và tới Interface vật lý (Interface 2) sau đó sẽ ra được internet

### Tunnel Bridge

Traffic tới từ node compute sẽ được chuyển tới node controller thông qua GRE/VXLAN tunnel trên bridge tunnel (br-tun). Nếu sử dụng VLAN thì nó sẽ chuyển đổi VLAN-tagged traffic từ intergration bridge sang GRE/VXLAN tunnels. Việc chuyển đỏi qua lại giữa VLAN IDs và tunnel IDs được thực hiện bởi OpenFlow rules trên br-tun.

Tunnel trong trường hợp trên nối từ local interface trên node compute (192.168.11.12) tới remote ip trên node controller (192.168.11.10)

Tunnel này sử dụng bảng định tuyến trên host để trao đổi gói tin vì thế ko cần bất cứ yêu cầu nào đối với hai endpoints hai bên, khác với khi sử dụng VLAN. Tất cả các interface trên br-tun được coi là internal đối với Open vSwitch. Vì thế nó sẽ không thể nhìn thấy từ bên ngoài, để giám sát traffic, bạn phải tạo ra mirror port cho patch-tun trên br-int bridge.

Tunnel bridge trên node controller về cơ bản không có gì khác nhiều so với tunnel trên node compute

### DHCP Router on Controller

DHCP server thường được chạy trên node controller hoặc compute. Nó là một instance của dnsmasq chạy trong một network namespace. Network namespace là một Linux kernel facility cho phép thực hiện một loạt các tiến trình tạo ra network stack (interfaces, routing tables, iptables rules).

### Router on Controller

Router cũng là một network namespace với một loạt các routing rules và iptables rules để thực hiện việc định tuyến giữa các subnets.

Chúng ta cũng có thể xem cấu hình của router với câu lệnh ip netns exec. Mỗi router sẽ có 2 interface, một cổng sẽ kết nối tới gateway (được tạo bởi câu lệnh router-gateway-set), một cổng sẽ nối tới integration bridge.

## Network traffic

![](/docs/ovs-network/images/2-openvswitch-ops/pic3.png)

![](/docs/ovs-network/images/2-openvswitch-ops/pic4.png)

![](/docs/ovs-network/images/2-openvswitch-ops/pic5.png)

![](/docs/ovs-network/images/2-openvswitch-ops/pic6.png)

![](/docs/ovs-network/images/2-openvswitch-ops/pic7.png)

![](/docs/ovs-network/images/2-openvswitch-ops/pic8.png)

![](/docs/ovs-network/images/2-openvswitch-ops/pic9.png)

## Kiểm chứng



### Tham khảo thêm

https://docs.openstack.org/neutron/pike/admin/deploy-ovs-provider.html

https://docs.openstack.org/neutron/pike/admin/deploy-ovs-selfservice.html

https://github.com/meditechopen/meditech-ghichep-openstack/blob/master/docs/04.Neutron/neutron-openvswitch.md

https://blog.oddbit.com/post/2013-11-14-quantum-in-too-much-detail/