# Điều khiển truy cập trên cơ sở vai trò
---
## Tổng quan
Điều khiển truy cập trên cơ sở vai trò (Role-Based Access Control - viết tắt là RBAC) là phương pháp điều khiển và đảm bảo quyền sử dụng cho người dùng. 

Đây là phương pháp thay thế Điều khiển truy cập tùy quyền (discretionary access control - DAC) và Điều khiển truy cập bắt buộc (mandatory access control - MAC).

Các vai trò (roles) được tạo để đảm nhận các chức năng công việc khác nhau. Mỗi role được gắn liền với một số quyền hạn cho phép nó thao tác một số hoạt động cụ thể ('permissions'). 

Các User (hoặc những người dùng trong hệ thống) được phân phối một vai trò riêng, và thông qua việc phân phối vai trò này mà họ tiếp thu được một số những quyền hạn cho phép họ thi hành những chức năng cụ thể trong hệ thống.

Vì người dùng không được cấp phép một cách trực tiếp, User thu được những quyền hạn thông qua vai trò (role) của user (hoặc các vai trò (role)), việc quản lý quyền hạn của người dùng trở thành một việc đơn giản, và người quản trị chỉ cần chỉ định những vai trò thích hợp cho User. 

Việc chỉ định vai trò đơn giản hóa những công việc thông thường như việc cho thêm một người dùng vào trong hệ thống, hay tập quyền của người dùng.

RBAC khác với các danh sách điểu khiển truy cập (access control list - ACL) được dùng trong hệ thống điều khiển truy cập tùy quyền, ở chỗ, nó chỉ định các quyền hạn tới từng hoạt động cụ thể với ý nghĩa trong cơ quan tổ chức, thay vì tới các đối tượng dữ liệu hạ tầng.

# Nguồn 

https://vi.wikipedia.org/wiki/%C4%90i%E1%BB%81u_khi%E1%BB%83n_truy_c%E1%BA%ADp_tr%C3%AAn_c%C6%A1_s%E1%BB%9F_vai_tr%C3%B2

