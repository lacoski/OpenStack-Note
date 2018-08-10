# Tổng quan Image Service - Glance
---
## Giới thiệu
- OpenStack Image service là dịch vụ trung tâm trong kiến trúc Infrastructure-as-a-Service (IaaS). Hiểu đơn giản, nó là trung tâm quản lý các image
- Openstack glance là một dịch vụ image, cung cấp các chức năng: discovering, registering, retrieving for disk and server images.
- Hỗ trợ các API request cho các disk hoặc server images và metadata từ phía client hoặc từ Compute service. 
- Các image tương tự như các template sử dụng để tạo VM
- VM image được tạo, lưu trữ tại Glance. Thông qua glance, các image có thể được lưu trữ tại nhiều vị trí khác nhau, từ các hệ thống tệp tin đơn giản đến các hệ thống lưu trữ đối tượng như là OpenStack Swift project.
- Glance được thiết kế để trở thành dịch vụ độc lập, đáp ứng các vấn đề quản lý số lượng virtual disk images lớn, đáp ứng như cầu dịch vụ cloud.
- Hỗ trợ snapshots các máy ảo đang chạy để sao lưu trạng thái của VM.

> Glance là giải pháp để quản lý các image trên cloud. 

## Các thành phần của Glance

Glance có các thành phần:
- glance-api : tiếp nhận lời gọi API để tìm kiếm, thu thập và lưu trữ image
- glance-registry: thực hiện tác vụ lưu trữ, xử lý và thu thập metadata của images. - Metadata bao gồm các thông tin như kích thước và loại image.
- database: cơ sở dữ liệu lưu trữ metadata của image
- storage repository : Tích hợp các thành phần OpenStack: 
  - File system: lưu trữ các image của các máy ảo trong hệ thống tệp tin thông thường theo mặc định, hỗ trợ đọc ghi các các image file dễ dàng vào hệ thống tệp tin. (Mặc định)
  - Object Storage: Hệ thống lưu trữ do OpenStack Swift cung cấp lưu trữ các image dưới dạng các object.
  - BlockStorage: Hệ thống lưu trữ do OpenStack Cinder cung cấp, lưu trữ các image dưới dạng khối.
  - RADOS Block Device(RBD): Lưu trữ các image trong Ceph, sử dụng giải pháp RBD của Ceph.
  - HTTP: OpenStack Image Service có thể đọc các vitual machine service có sẵn trên internet sử dụng giao thức HTTP. Lưu trữ này chỉ có thể đọc.
- Metadata definition service:
  - API cho nhà cung cấp, admin, service, user định nghĩa custom metadata. 

## Kiến trúc của Glance
- Glance có kiến trúc client-server, cung cấp REST API cho user tương tác.
- Glance Domain Controller quản lí các hoạt động bên trong. Các hoạt động được chia ra thành các tầng khác nhau. Mỗi tầng thực hiện một chức năng riêng biệt.
- Glane store là lớp giao tiếp giữa glane và storage back end ở ngoài glane hoặc local filesystem và nó cung cấp giao diện thống nhất để truy cập. Glance sử dụng SQL central Database để truy cập cho tất cả các thành phần trong hệ thống.
- Glance bao gồm một số thành phần sau:
  - Client: Bất kỳ ứng dụng nào sử dụng Glance server đều được gọi là client.
  - REST API: dùng để gọi đến các chức năng của Glance thông qua REST.
  - Database Abstraction Layer (DAL): một API để thống nhất giao tiếp giữa Glance và database.
  - Glance Domain Controller: là middleware thực hiện các chức năng chính của Glance là: authorization, notifications, policies, database connections.
  - Glance Store: Giao diện tích hợp giữa Glance và các data store.
  - Registry Layer: Layer không bắt buộc để tổ chức giao tiếp mang tính bảo mật giữa domain và DAL nhờ việc sử dụng một dịch vụ riêng biệt.

![](images/glance-overview-1.png)

## Các định dạng image của Glance
- Khi upload một image lên glance, ta phải chỉ rõ định dạng của các Virtual machine image.
- Glane hỗ trợ nhiều kiểu định dạng như Disk format và Container format.
- Virtual disk tương tự như server boot driver vật lý, chỉ tập trung vào trong một tệp tin. Điều khác là Virtualation hỗ trợ nhiều định dạng disk khác nhau.

| Disk format | Notes |
|-------------|-------|
| Raw | Định dạng đĩa phi cấu trúc |
| VHD | Định dạng chung hỗ trợ bởi nhiều công nghệ ảo hóa trong OpenStack, ngoại trừ KVM |
| VMDK | Định dạng hỗ trợ bởi VMWare |
| qcow2 | Định dạng đĩa QEMU, định dạng mặc định hỗ trợ bởi KVM vfa QEMU, hỗ trợ các chức năng nâng cao |
| VDI | Định dạng ảnh đĩa ảo hỗ trợ bởi VirtualBox |
| ISO | 	Định dạng lưu trữ cho đĩa quang |
| AMI, ARI, AKI | Định dạng ảnh Amazon machine, ramdisk, kernel |

## Container Format

| Container format | Notes |
|------------------|-------|
| bare | Định dạng xác định không có container hoặc metadata đóng gói cho image |
| ovf | Định dạng container OVF |
| aki | Xác định lưu trữ trong Glance là Amazon kernel image |
| ari | Xác định lưu trữ trong Glance là Amazon ramdisk image |
| ami | Xác định lưu trữ trong Glance là Amazon machine image |
| ova | Xác định lưu trữ trong Glance là file lưu trữ OVA |
| docker | Xác định lưu trữ trong Glance và file lưu trữ Docker |

## Glance Status Flow

Glance status flow cho biết trạng thái của image trong quá trình tải lên. 

Khi tạo một image: 
- Bước 1: queuing, image được đưa vào hàng đợi và được nhận diện trong một khoảng thời gian ngắn, được bảo vệ và sẵn sàng để tải lên. 
- Bước 2: Chuyển sang trạng thái Saving nghĩa là quá trình tải lên chưa hoàn thành. 
- Bước 3: Khi image được tải lên hoàn toàn, trạng thái image chuyển sang Active. Khi quá trình tải lên thất bại nó sẽ chuyển sang trạng thái bị hủy hoặc bị xóa. 

Ta có thể deactive và reactive các image đã upload thành công bằng cách sử dụng command. Glance status flow được mô tả theo hình sau:

![](images/glance-overview-2.jpg)

Các trạng thái của image:
- queued : Định danh của image được bảo vệ trong Glance registry. Không có dữ liệu nào của image được tải lên Glance và kích thước của image không được thiết lập về zero khi khởi tạo.
- saving : Biểu thị rằng dữ liệu của image đang được upload lên glance. Khi một image đăng ký với một call đến POST /image và có một x-image-meta-location header, image đó sẽ không bao giờ được trong tình trạng saving (dữ liệu Image đã có sẵn ở vị trí khác).
- active : Biểu thị một image đó là hoàn toàn có sẵn trong Glane. Điều này xảy ra khi các dữ liệu image được tải lên.
- deactivated : Trạng thái biểu thị việc không được phép truy cập vào dữ liệu của image với tài khoản không phải admin. Khi image ở trạng thái này, ta không thể tải xuống cũng như export hay clone image.
- killed : Trạng thái biểu thị rằng có vấn đề xảy ra trong quá trình tải dữ liệu của image lên và image đó không thể đọc được
- deleted : Trạng thái này biểu thị việc Glance vẫn giữ thông tin về image nhưng nó không còn sẵn sàng để sử dụng nữa. Image ở trạng thái này sẽ tự động bị gỡ bỏ vào ngày hôm sau.

## Các file cấu hình của glance
- `glance-api.conf` : File cấu hình cho API của image service.
- `glance-registry.conf` : File cấu hình cho glance image registry - nơi lưu trữ metadata về các image.
- `glance-scrubber.conf` : Được dùng để dọn dẹp các image đã được xóa
- `policy.json` : Bổ sung truy cập kiểm soát áp dụng cho các image service. Trong này, chúng ta có thể xác định vai trò, chính sách, làm tăng tính bảo mật trong Glane OpenStack.


## Image and Instance
Phân biệt:
- Disk image được lưu trữ giống như các template. Image service kiểm soát việc lưu trữ và quản lý của các image. 
- Instance là một máy ảo riêng biệt chạy trên compute node, compute node quản lý các instances. 

User có thể vận hành bao nhiêu máy ảo tùy ý với cùng một image. Mỗi máy ảo đã được vận hành được tạo nên bởi một bản sao của image gốc, bởi vậy bất kỳ chỉnh sửa nào trên instance cũng không ảnh hưởng tới image gốc. 

Ta có thể tạo bản snapshot của các máy ảo đang chạy nhằm mục đích dự phòng hoặc vận hành một máy ảo khác.

Khi ta vận hành một máy ảo, ta cần phải chỉ ra flavor của máy ảo đó. Flavor đại diện cho tài nguyên ảo hóa cung cấp cho máy ảo, định nghĩa số lượng CPU ảo, tổng dung lượng RAM cấp cho máy ảo và kích thước ổ đĩa không bền vững cấp cho máy ảo. OpenStack cung cấp một số flavors đã định nghĩa sẵn, ta có thể tạo và chỉnh sửa các flavors theo ý mình. 

Trước khi vận hành máy ảo, các thành phần ban đầu:
- Image store chỉ số lượng các images đã được định nghĩa trước
- Compute node chứa các vcpu có sẵn, tài nguyên bộ nhớ và tài nguyên đĩa cục bộ
- Cinder-volume chứa số lượng volumes đã định nghĩa trước đó.

![](images/glance-overview-3.jpg)
> Sơ đồ dưới đây chỉ ra trạng thái của hệ thống trước khi vận hành máy ảo


Trước khi vận hành 1 máy ảo, ta phải chọn:
- Image
- Flavor: Lựa chọn flavor nào cung cấp root volume, 
- Các thuộc tính tùy chọn. 

Ở đây, 
- `vda`: Các image được sao chép vào các local disk. VDA là disk đầu tiên mà các instance được truy cập.
- `vdb`: Ổ tạm thời (Dạng ephemeral - không bền vững), tạo ra cùng với instance sẽ bị xóa khi kết thúc instance.
- `vdc`: Kết nối với cinder-volume sử dụng iSCSI. Sau khi compute node quy định vCPU và tài nguyên bộ nhớ. Các instance boots up từ root volume VDA. Instance chạy và thay đổi dữ liệu trên disk (map từ cinder-volumen).

> Khi máy ảo bị xóa, ephemeral storage (khối lưu trữ không bền vững) bị xóa; tài nguyên vCPU và bộ nhớ được giải phóng. Image không bị thay đổi sau tiến trình này.

> Nếu volume store nằm trên một mạng riêng biệt , tùy chọn `my_block_storage_ip` trong tập tin cấu hình storage node sẽ chỉ đạo giao tiếp với compute node.

![](images/glance-overview-4.jpg)
# Nguồn

https://github.com/thaonguyenvan/meditech-thuctap/blob/master/ThaoNV/Tim%20hieu%20OpenStack/docs/glance/glance-overview.md

https://github.com/hocchudong/thuctap012017/blob/master/DucPX/OpenStack/glance/docs/overviewglance.md















