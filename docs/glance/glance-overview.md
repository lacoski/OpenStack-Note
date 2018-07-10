# Tổng quan Image Service - Glance
---
## Giới thiệu
- OpenStack Image service là dịch vụ trung tâm trong kiến trúc Infrastructure-as-a-Service (IaaS). Nó chấp nhận các API requests cho disk hoặc server images và metadata từ phía người dùng hoặc từ Compute service. Nó cũng hỗ trợ lưu trữ disk hoặc server images trên rất nhiều loại repository, bao gồm cả OpenStack Object Storage - Swift.

Các Image trong glance lưu trữ giống như các template. Chúng sử dụng để vận hành máy ảo mới.

Glance là giải pháp để quản lý các image trên cloud. Nó cũng có thể lấy bản snapshots từ các máy ảo đang chạy để thực hiện dự phòng cho các VM và trạng thái các máy ảo đó.

- Openstack glance là một dịch vụ image, cung cấp các chức năng: discovering, registering, retrieving for disk and server images.
- Là Trung tâm quản lý các image
- VM image được tạo sẵn, thông qua glance có thể được lưu trữ trong nhiều vị trí khác nhau, từ các hệ thống tệp tin đơn giản đến các hệ thống lưu trữ đối tượng như là OpenStack Swift project.
