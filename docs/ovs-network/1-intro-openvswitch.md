# Tổng quan về Open vSwitch

## Tổng quan

- Cũng giống như Linux Bridge, OpenVSwitch là phần mềm cung cấp virtual switch cho các giải pháp ảo hóa network.
- Là phần mềm mã nguồn mở, sử dụng cho ảo hóa vswitch trong môi trường ảo hóa của server.
- Open Vswitch có thể forwards traffic giữa các máy VM trên cùng một máy chủ vật lý và forwards traffic giữa các máy VM và máy vật lý.
- OpenVSwitch được thiết kế tương thích với các switch hiện đại.
- Open vSwitch phù hợp làm việc như là một switch ảo trong môi trường máy ảo VM. Ngoài việc kiểm soát và có khả năng hiển thị giao diện chuẩn cho các lớp mạng ảo, nó được thiết kế để hỗ trợ phân phối trên nhiều máy chủ vật lý. Open vSwitch hỗ trợ nhiều công nghệ ảo hóa Linux-based như là Xen/Xen server, KVM và Virtual Box.
- OpenVSwitch có thể chạy trên các nền tảng Linux, FreeBSD, Windows, non-POSIX embedded Systems,...


## Các tính năng

Phiên bản hiện tại của Open vSwitch hỗ trợ những tính năng sau:
- Hỗ trợ tính năng VLAN chuẩn 802.1Q với các cổng trunk và access như một switch layer thông thường.
- Hỗ trợ giao diện NIC bonding có hoặc không có LACP trên cổng uplink switch.
- Hỗ trợ NetFlow, sFlow(R), và mirroring để tăng khả năng hiển thị.
- Hỗ trợ cấu hình QoS (Quality of Service) và các chính sách thêm vào khác.
- Hỗ trợ tạo tunnel GRE, VXLAN, STT và LISP.
- Hố trợ tính năng quản lý các kết nối 802.1aq
- Hỗ trợ OpenFlow các phiên bản từ 1.0 trở lên.
- Cấu hình cơ sở dữ liệu với C và Python.
- Hoạt động forwarding với hiệu suất cao sử dụng module trong nhân Linux.


## Các thành phần

- ovs-vswitchd: đóng vai trò daemon switch thực hiện các chức năng chuyển mạch kết hợp với module trong kernel Linux cho flow-based swtiching.
- ovsdb-server: Database server mà ovs-vswitchd truy vấn tới để lấy cấu hình.
- ovs-dpctl: Công cụ cấu hình module chuyển mạch trong kernel.
- ovs-vsctl: Công cụ thực hiện truy vấn và cập nhật các cấu hình của ovs-vswitchd.
- ovs-appctl: Công cụ gửi các lệnh tới Open Vswitch deamon. Open vSwitch cũng cung cấp một số công cụ sau:
- ovs-ofctl: Công cụ truy vấn và điều khiển chuyển mạch Open Flow và controller.
- ovs-pki: Công cụ cho phép tạo và quản lý các public-key cho các Open Flow switch.
- ovs-testcontroller: một OpenFlow controller đơn giản có thể quản lý một số switch ảo thông qua giao thức Open Flow, khiến chúng hoạt động như các switch lớp 2 hoặc như hub. Phù hợp để kiểm tra các mạng Open Flow ban đầu.

![](/docs/ovs-network/images/1-intro-openvswitch/pic1.png)

![](/docs/ovs-network/images/1-intro-openvswitch/pic2.png)

Kiến trúc OpenvSwitch Architecture

![](/docs/ovs-network/images/1-intro-openvswitch/pic3.jpg)

## Tham khảo

https://docs.openvswitch.org/en/latest/intro/install/general/
https://github.com/hocchudong/thuctap012017/blob/master/TamNT/Virtualization/docs/Virtual_Switch/2.Tim_hieu_Open_Vswitch.md