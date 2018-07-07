# Tìm hiểu tổng quan về keystone
---
## Giới thiệu tổng quan về chức năng và lợi ích khi sử dụng keystone

Môi trường cloud theo mô hình dịch vụ Infrastructure-as-a-Service layer (IaaS) cung cấp cho người dùng khả năng truy cập vào các tài nguyên quan trọng ví dụ như máy ảo, kho lưu trữ và kết nối mạng. Bảo mật vẫn luôn là vấn đề quan trọng đối với mọi môi trường cloud. Trong OpenStack, keystone có trách nhiệm đảm nhận việc bảo mật, kiểm soát truy cập tới tất cả các tài nguyên của cloud. Nó sẽ chịu trách nhiệm cung cấp các kết nối mang bảo mật cao tới các tài nguyên OPS.

### Ba chức năng chính của keystone
Identity:
- Nhận diện những người đang cố truy cập vào các tài nguyên cloud
- Trong keystone, identity thường được hiểu là định danh User

> Tại những mô hình OpenStack nhỏ, identity của user thường được lưu trữ trong database của keystone. Đối với những mô hình lớn cho doanh nghiệp thì 1 external Identity Provider thường được sử dụng.

Authentication
- Là quá trình xác thực, nhận dạng user (user's identity)
- Keystone có tính pluggable tức là nó có thể liên kết với những dịch vụ xác thực người dùng khác như LDAP hoặc Active Directory.
- Thường thì keystone sử dụng Password cho việc xác thực người dùng. Đối với những phần còn lại, keystone sử dụng tokens.
- OpenStack dựa rất nhiều vào tokens để xác thực và keystone chính là dịch vụ duy nhất có thể tạo ra tokens
- Token có giới hạn về thời gian được phép sử dụng. Khi token hết hạn thì user sẽ được cấp một token mới. Cơ chế này làm giảm nguy cơ user bị đánh cắp token.
- Hiện tại, keystone đang sử dụng cơ chế bearer token. Có nghĩa là bất cứ ai có token thì sẽ có khả năng truy cập vào tài nguyên của cloud. Vì vậy việc giữ bí mật token rất quan trọng.

Access Management (Authorization)
- Access Management hay còn được gọi là Authorization là quá trình xác định những tài nguyên mà user được phép truy cập tới.
- Trong OpenStack, keystone kết nối users với những Projects hoặc Domains bằng cách gán role cho user vào những project hoặc domain ấy.
- Các project trong OpenStack như Nova, Cinder...sẽ kiểm tra mối quan hệ giữa role và các user's project và xác định giá trị của những thông tin này theo cơ chế các quy định (policy engine). Policy engine sẽ tự động kiểm tra các thông tin (thường là role) và xác định xem user được phép thực hiện những gì.

### Lợi ích của KeyStone
- Cung cấp giao diện xác thực và quản lí truy cập cho các services của OpenStack. Nó cũng đồng thời chịu trách nhiệm cho toàn bộ công việc giao tiếp và làm việc với các hệ thống xác thực bên ngoài.

- Cung cấp danh sách đăng kí các container ("Projects") mà nhờ vậy các services khác của OpenStack có thể dùng nó để "tách" tài nguyên (servers, images,...)

Cung cấp danh sách đăng kí các Domains được dùng để định nghĩa các khu vực riêng biệt cho users, groups, và projects khiến các khách hàng trở nên "tách biệt" với nhau.

Danh sách đăng kí các Roles được keystone dùng để ủy quyền cho các services của OpenStack thông qua file policy.

Assignment store cho phép users và groups được gán các roles trong projects và domains.

Catalog lưu trưc các services của OpenStack, endpoints và regions, cho phép người dùng tìm kiếm các endpoints của services mà họ cần.