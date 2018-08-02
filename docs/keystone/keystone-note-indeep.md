# Bổ sung khái niệm KeyStone
---

## Keystone API gồm
- Policy Backend: Quản lý interface và rule xác thực trên đó
- Token backend: Chứa các token
- Catalog backend: Chứa các endpoint registry
- Identity backend: Chứa User và group (DB riêng hoặc phụ thuộc dịch vụ xác thực khác LDAP)
- Assignment backend: Chứa domain, project, role, role gán 
- Credentials: Chứa ..

## Khái niệm endpoint
- Endpoint đơn gián là URL sử dụng truy cập service bên trong OPS. Là các đầu mối các dịch vụ. 
- Có 3 loại: 
 - admin url: sử dụng cho admin user,
 - Internal url: Sử dụng cho các service giao tiếp với nhau
 - public url: Các user khác truy cập sử dụng dịch vụ

Các thuật ngữ
- Service: Các service OPS như Nova, Swift, Keystone
- role: quyển truy cập các user
- token: chuỗi string chứa quyền truy cập các service OPS
- user: các user truy cập service, 
- tenants: group các user, giờ chuyển project

Keystone PKI - Token validate
- Bước 1: User gửi user/pass tới keystone 
- Bước 2: Keystone xác thực user pass trả lại token
- Bước 3: User sử dụng token đó để giao tiếp API OPS.


Note:

Catalog:
- Nơi đăng ký dịch vụ mới, sử dụng cho việc liệt kê, tìm kiếm


## Phương pháp tạo token: 
- UUID
- PKI
- PKIZ
- Fernet


### UUID (universally unique identifier):
- Là tiêu chuẩn định dạnh được sử dụng trong xây dựng phần mềm. Mục đích của UUIDs là cho phép các hệ thống phân phối để nhận diện thông tin mà không cần điều phối trung tâm.
- A UUID is a 16-octet (128-bit) number.
- UUID được đại diện bởi 32 chữ số thập lục phân,hiển thị trong năm nhóm, phân cách bằng dấu gạch nối, với dạng 8-4-4-4-12. Có tổng cộng 36 ký tự, trong đó 32 ký tự chữ với 4 dấu gạch ngang.
- UUID có tổng cộng 5 phiên bản, trong đó keystone sử dụng UUIDv4.

Nguồn: 

https://en.wikipedia.org/wiki/Universally_unique_identifier#Definition

https://tools.ietf.org/html/rfc4122.html

https://docs.python.org/3/library/uuid.html

#### Đặc điểm UUID trong keystone
- Có độ dài 32 byte, nhỏ, dễ sử dụng, không nén.
- Không mang theo đủ thông tin, do đó luôn phải gửi lại keystone để xác thực hoạt động ủy quyền => thắt nút cổ chai.
- Được lưu vào database.
- Sử dụng thời gian dài làm giảm hiệu suất hoạt động, CPU tăng và thời gian đáp ứng lâu.
- Sử dụng câu lệnh keystone-manager token flush để làm tăng hiệu suất hoạt động.
Ví dụ 1 đoạn token

#### UUID Token Generation Workflow 
- Bước 1: Xác nhận user, lấy UserID.
- Bước 2: Xác nhận project, lấy project id và domain id.
- Bước 3: Lấy roles cho user trên project hoặc domain đó. Trả lại kết quả Failure nếu user không có roles đó.
- Bước 4: Lấy các services và endpoitns
- Bước 5: Gộp các thông tin Identity, Resource, Assignment, Catalog vào token payload. Tạo token id bằng hàm uuid.uuid4().hex.

> Lưu giữ các thông tin Token ID, Expiration, Valid, User ID, Extra vào backend.

#### UUID Token Validation Workflow 
- Bước 1: Xác nhận token bằng cách gửi một phương thức GET đến Token KVS.
- Bước 2: Token KVS sẽ kiểm tra trong backend. Kết quả trả về nếu không là Token not found, nếu có, chuyển sang bước 3.
- Bước 3: Phân tích token và lấy các metadata: UserID, Project ID, Audit ID, Token Expiry.
- Bước 4: Kiểm tra thời gian hiện tại với thời gian hết hạn của token. Nếu token hết hạn, trả về Token not found. Nếu còn hạn, chuyển sang bước 5
- Bước 5: Kiểm tra token có bị thu hồi không, nếu no, trả về cho người dùng thông điệp HTTP/1.1 200OK (token sử dụng được).

#### UUID Token Revocation Workflow 
- Bước 1: Gửi một yêu cầu DELETE token. Trước khi revoke token thì phải xác nhận lại token (Token validation workflow)
- Bước 2: Kiểm tra Audit ID. Nếu không có audit ID, chuyển sang bước 3. Nếu có audit ID, chuyển sang bước 6.
- Bước 3: Token được thu hồi khi hết hạn, chuyển sang bước 4.
- Bước 4: Tạo một event revoke với các thông tin: User ID, Project ID, Revoke At, Issued Before, Token expiry.
- Bước 5: Chuyển sang bước 9.
- Bước 6: Token được thu hồi bởi audit id.
- Bước 7: Tạo event revoke với các thông tin: audit id và thời điểm revoke trước khi hết hạn.
- Bước 8: Lọc các event revoke đang tồn tại dựa trên Revoke At.
- Bước 9: Set giá trị false vào token avs của token.

#### Ưu nhược điểm
Ưu điểm:

Định dạng token đơn giản và nhỏ.
Đề nghị được sử dụng trong các môi trường OpenStack đơn giản.
Nhược điểm

Định dạng token cố định.
Xác nhận token chỉ được hoàn thành bởi dịch vụ Identity.
Không khả thi cho môi trường OpenStack multiple.

### PKI - PKIZ:
- Mã hóa bằng Private Key, kết hợp Public key để giải mã, lấy thông tin.
- Token chứa nhiều thông tin như Userid, project id, domain, role, service catalog, create time, exp time,...
- Xác thực ngay tại user, không cần phải gửi yêu cầu xác thực đến Keystone.
- Có bộ nhớ cache, sử dụng cho đến khi hết hạn hoặc bị thu hồi => truy vấn đến keystone ít hơn.
- Kích thước lớn, chuyển token qua HTTP, sử dụng base64.
- Kích thước lớn chủ yếu do chứa thông tin service catalog.
- Tuy nhiên, Header của HTTP chỉ giới hạn 8kb. Web server không thể xử lý nếu không cấu hình lại, khó khăn hơn UUID
- Để khắc phục lỗi trên thì phải tăng kích thước header HTTP của web server, tuy nhiên đây không phải là giải pháp cuối cùng hoặc swift có thể thiết lập không cần catalog service.
- Lưu token vào database.

#### PKIZ
- Tương tự PKI.
- Khắc phục nhược điểm của PKI, token sẽ được nén lại để có thể truyền qua HTTP. Tuy nhiên, token dạng này vẫn có kích thước lớn.

- Bước 1: User request token với các thông tin là: username, password, project name.
- Bước 2: Keystone sẽ xác nhận định danh, resource và assignmetn.
- Bước 3: Tạo một JSON, chứa token payload.
- Bước 4: Sign JSON này với các Signing Key và Signing Certificate. Sau đó, với dạng PKI chuyển sang bước 5. Nếu là dạng PKIZ chuyển sang bước 11.
- Bước 5: Convert JSON trên sang dạng UTF-8.
- Bước 6: Convert CMS Signed Token in PEM format to custom URL Safe format:
“/” replaced with “-”
Deleted: “\n”, “----BEGIN CMS----”,“----END CMS-
- Bước 7: Sử dụng zlib để nén JSON.
- Bước 8: Mã hóa Base64 URL Safe.
- Bước 9: Convert JSON sang dạng UTF-8
- Bước 10: PKIZAppend Prefix
- Bước 11: Lưu trữ token vào SQL/KVS

#### Token PKI/PKIZ Validation Workflow 

Cũng tương tự UUID, chỉ khác ở chỗ là:

Trước khi gửi yêu cầu GET đến Token KVS thì pki token sẽ được hash với thuật toán đã cấu hình trước.

## Mục đích các endpoint URL KeyStone:
### Định danh sử dụng API - Cho user (Public URL)
- Lấy Token: 
 - Token sử dụng cho truy cập Resource và Services
 - Chứa quyền User trên project
- Lấy service catalog
 - URL endpoint tới các servuce khác
- Lấy token = /v3/auth/tokens

### Định danh sử dụng API - Cho admin (Admin URL)
- Định nghĩa
 - User, group
 - Project
 - Roles
 - Roles user trên project
 - Service, endpoint service

### Đinh danh sử dụng API - Cho service (Internal URL)
- xác thực token
- discover các service endpoint khác


