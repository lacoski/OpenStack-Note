# Tổng quan về OpenStack
---
## Về Cloud Computing
Định nghĩa lược dịch từ NIST:
```
Cloud Computing  là mô hình cho phép truy cập qua mạng để lựa chọn và sử dụng tài nguyên  có thể được tính toán (ví dụ: mạng, máy chủ, lưu trữ, ứng dụng và dịch vụ) theo nhu cầu một cách thuận tiện và nhanh chóng; đồng thời cho phép kết thúc sử dụng dịch vụ, giải phóng tài nguyên dễ dàng, giảm thiểu các giao tiếp với nhà cung cấp”
```

Cần nắm dược 5 – 4 – 3 trong Cloud Computing: đó là 5 đặc tính, 4 mô hình dịch vụ và 3 mô hình triển khai.

5 đặc điểm:
- Khả năng thu hồi và cấp phát tài nguyên (Rapid elasticity)
- Truy nhập qua các chuẩn mạng (Broad network access)
- Dịch vụ sử dụng đo đếm được (Measured service,) hay là chi trả theo mức độ sử dụng pay as you go.
- Khả năng tự phục vụ (On-demand self-service).
- Chia sẻ tài nguyên (Resource pooling).

4 mô hình dịch vụ (mô hình sản phẩm): 
- Public Cloud: Đám mây công cộng (là các dịch vụ trên nền tảng Cloud Computing để cho các cá nhân và tổ chức thuê, họ dùng chung tài nguyên).
- Private Cloud: Đám mây riêng (dùng trong một doanh nghiệp và không chia sẻ với người dùng ngoài doanh nghiệp đó)
- Community Cloud: Đám mây cộng đồng (là các dịch vụ trên nền tảng Cloud computing do các công ty cùng hợp tác xây dựng và cung cấp các dịch vụ cho cộng đồng). 
- Hybrid Cloud : Là mô hình kết hợp (lai) giữa các mô hình Public Cloud và Private Cloud 

3 mô hình triển khai: tức là triển khai Cloud Computing để cung cấp:
- Hạ tầng như một dịch vụ (Infrastructure as a Service)
- Nền tảng như một dịch vụ (Platform as a Service)
- Phần mềm như một dịch vụ (Software as a Service)


## Về OpenStack
OpenStack là một phần mềm mã nguồn mở, dùng để triển khai Cloud Computing, bao gồm private cloud và public cloud (nhiều tài liệu giới thiệu là Cloud Operating System). `Open source software for building private and public clouds`


> Tên các phiên bản được bắt đầu theo thứ tự A, B, C, D …trong bảng chữ cái.

### Thành phần cơ bản
- OpenStack compute (code-name Nova): là module quản lý và cung cấp máy ảo. Tên phát triển của nó Nova. Nó hỗ trợ nhiều hypervisors gồm KVM, QEMU, LXC, XenServer... Compute là một công cụ mạnh mẽ mà có thể điều khiển toàn bộ các công việc: networking, CPU, storage, memory, tạo, điều khiển và xóa bỏ máy ảo, security, access control. Bạn có thể điều khiển tất cả bằng lệnh hoặc từ giao diện dashboard trên web.

- OpenStack Glance (code-name Glance): là OpenStack Image Service, quản lý các disk image ảo. Glance hỗ trợ các ảnh Raw, Hyper-V (VHD), VirtualBox (VDI), Qemu (qcow2) và VMWare (VMDK, OVF). Bạn có thể thực hiện: cập nhật thêm các virtual disk images, cấu hình các public và private image và điều khiển việc truy cập vào chúng, và tất nhiên là có thể tạo và xóa chúng.

- OpenStack Object Storage (code-name Swift): dùng để quản lý lưu trữ. Nó là một hệ thống lưu trữ phân tán cho quản lý tất cả các dạng của lưu trữ như: archives, user data, virtual machine image … Có nhiều lớp redundancy và sự nhân bản được thực hiện tự động, do đó khi có node bị lỗi thì cũng không làm mất dữ liệu, và việc phục hồi được thực hiện tự động.

- Identity Server(code-name Keystone): quản lý xác thực cho user và projects.

- OpenStack Netwok (code-name Neutron): là thành phần quản lý network cho các máy ảo. Cung cấp chức năng network as a service. Đây là hệ thống có các tính chất pluggable, scalable và API-driven.

- OpenStack dashboard (code-name Horizon): cung cấp cho người quản trị cũng như người dùng giao diện đồ họa để truy cập, cung cấp và tự động tài nguyên cloud. Việc thiết kế có thể mở rộng giúp dễ dàng thêm vào các sản phẩm cũng như dịch vụ ngoài như billing, monitoring và các công cụ giám sát khác.

```

Project core của OpenStack:
- Cinder, nova, neutron, cinder, keystone, glance
```

### Ý nghĩa Project:

Dashboard (Horizon):
- Ứng dụng web chạy nên apache
- Cung cấp giao diên cho Administrator quản trị dịch vụ OpenStack
- Tương thích với EC2 API của amazon

Compute (Nova)
- Thành phần quản lý máy ảo (Virtual Compute Instance)
- Tương thích EC2 Amazone
- Gọi bằng OpenStack API hoặc EC2 API
- Hỗ trợ nhiều công nghệ ảo hóa: Xen, KVM, QEMU, vSphere, Hyper-V

Image service (Glance)
- Dịch vụ lưu trữ trên ổ đĩa ảo (Virtual Disk Image - VDI)
- Hỗ trợ nhiều định dạng
- 3 tính năng chính:
 - Quản trị template có sẵn, cho phép tạo image nhanh chóng
 - Tạo VM từ VDI có sẵn
 - Lưu VM nhanh = tính năng snapshots

Networking (Neutron)
- Cung cấp dịch vụ mạng (Network as a service) cho các thành phần OpenStack
- Sử dụng kiến trúc "plug-in": Các plug-in được thực thi trên nhiều kiến trúc khác nhau (NVP, Open vSwitch, Linux bridge, Cisco)
- Cho phép tùy biến, mở rộng
- Cho phép tạo private network
- Có các tính năng tạo vSwitch, Firewall, DHCP, VPN, Load balancing

Storage (Swift)
- Cung cấp dịch vụ lưu trữ giống S3 Amazon
- Có khả năng mở rộng, lưu trữ phân tán, có backup
- Tương thích S3 API

Storage (Cinder)
- Cung cấp Thiết bị lưu trữ ảo cho VM OpenStack
- Tương tự dịch vụ EBS Amazon
- Có khả năng mở rộng phân tán

Phân biệt Swift, Image, Cinder
- Cinder cung cấp block storage, từ đó có thể mount các volume tới các thực thi (VM)
- Glance cung cấp dịch vụ cho lưu trữ, tìm lại các OS image
- Swift cung cấp object storage (lưu trữ dạng đối tượng). 

Nguồn tìm hiểu thêm

https://docs.openstack.org/project-deploy-guide/openstack-ansible/newton/overview-storage-arch.html

Identity service (Keystone)
- Dịch vụ xác thực người dùng 
- Hỗ trợ nhiều kiểu xác thực
- Phần quyền dựa trên tính năng role-base access control (RBAC)

Telemetry service (Ceilometer)
- Dịch vụ giám sát thông kê
- Thu thập thông tin về quá trình sử dụng để tính toán hóa đơn, xác định mức độ sử dụng hệ thống

Orchestration Service (Heat)
- Cung cấp các template cho ứng dụng phổ biến
- Template cung cấp các thành phần compute, storage, networking đáp ứng các ứng dụng
- Kết hợp với Ceilometer để tính toán tài nguyên sử dụng hợp lý
- Tương thích AWS CloudFormation APIs.

## Nguồn

https://vietstack.wordpress.com/2014/02/15/openstack-la-gi-va-de-lam-gi/

https://viblo.asia/p/tim-hieu-ve-dien-toan-dam-may-voi-openstack-ZabG9zZ5vzY6

https://www.slideshare.net/lanhuonga3/tm-hiu-v-openstack

https://github.com/thaonguyenvan/meditech-thuctap/blob/master/ThaoNV/Tim%20hieu%20OpenStack/docs/general/tim-hieu-chung-OpenStack.md

