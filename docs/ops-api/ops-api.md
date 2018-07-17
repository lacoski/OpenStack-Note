# Cách sử dụng OpenStack API
---
## Tổng quan 
### API là gì
Một giao diện lập trình ứng dụng (tiếng Anh Application Programming Interface, viết tắt API) là một giao diện mà một hệ thống máy tính hay ứng dụng cung cấp để cho phép các yêu cầu dịch vụ có thể được tạo ra từ các chương trình máy tính khác, và/hoặc cho phép dữ liệu có thể được trao đổi qua lại giữa chúng. 

Chẳng hạn, một chương trình máy tính có thể (và thường là phải) dùng các hàm API của hệ điều hành để xin cấp phát bộ nhớ và truy xuất tập tin. Nhiều loại hệ thống và ứng dụng hiện thực API, như các hệ thống đồ họa, cơ sở dữ liệu, mạng, dịch vụ web, và ngay cả một số trò chơi máy tính. Đây là phần mềm hệ thống cung cấp đầy đủ các chức năng và các tài nguyên mà các lập trình viên có thể rút ra từ đó để tạo nên các tính năng giao tiếp người- máy như: các trình đơn kéo xuống, tên lệnh, hộp hội thoại, lệnh bàn phím và các cửa sổ. Một trình ứng dụng có thể sử dụng nó để yêu cầu và thi hành các dịch vụ cấp thấp do hệ điều hành của máy tính thực hiện. Hệ giao tiếp lập trình ứng dụng giúp ích rất nhiều cho người sử dụng vì nó cho phép tiết kiệm được nhiều thời gian tìm hiểu các chương trình mới, do đó khích lệ mọi người dùng nhiều ứng dụng hơn.

### RESTful API Là Gì
RESTful API là một tiêu chuẩn dùng trong việc thiết kế các thiết kế API cho các ứng dụng web để quản lý các resource. RESTful là một trong những kiểu thiết kế API được sử dụng phổ biến nhất ngày nay.

Trọng tâm của REST quy định cách sử dụng các HTTP method (như GET, POST, PUT, DELETE...) và cách định dạng các URL cho ứng dụng web để quản các resource. Ví dụ với một trang blog để quản lý các bài viết chúng ta có các URL đi với HTTP method như sau:
- URL tạo bài viết: `http://my-blog.xyz/posts`. Tương ứng với HTTP method là POST
- URL đọc bài viết với ID là 123: `http://my-blog.xyz/posts/123`. Tương ứng với HTTP method là GET
- URL cập nhật bài viết với ID là 123: `http://my-blog.xyz/posts/123`. Tương ứng với HTTP method là PUT
- URL xoá bài viết với ID là 123: `http://my-blog.xyz/posts/123`. Tương ứng với HTTP method là DELETE

Với các ứng dụng web được thiết kế sử dụng RESTful, lập trình viên có thể dễ dàng biết được URL và HTTP method để quản lý một resource. Bạn cũng cần lưu ý bản thân RESTful không quy định logic code ứng dụng và RESTful cũng không giới hạn bởi ngôn ngữ lập trình ứng dụng. Bất kỳ ngôn ngữ lập trình (hoặc framework) nào cũng có thể áp dụng RESTful trong việc thiết kế API cho ứng dụng web.


## API trong Openstack
### Cơ bản
Sử dụng OpenStack API cho phép người dùng tạo server, image, gán metadata cho instance, tạo storage container, object, v.v. Nói chung sẽ cho phép người dùng thao tác với OpenStack cloud.

### Để thao tác với API OpenStack, các phương thực cơ bản
- CURL:
 - Công cụ dòng lệnh cho phép tương tác với các giao thức FTP, FTPS, HTTP, HTTPS, IMAP, SMTP, v.v.
 - Không có giao diện, chạy trên dòng lệnh, nhanh gọn.
- Openstack command-line client:
 - Mỗi Project trong OpenStack cho phép người dùng tương tác với chính nó thông qua cli. Bản chất các CLI sử dụng API OPS.
- REST client:
 - Giao diện độ họa, cho phép người sử dụng làm việc nhanh chóng với API.
- Openstack Python Software Development Kit (SDK):
 - Viết trên Python scripts, cho phép user làm việc với tài nguyên OpenStack.
 - SDK sử dụng python để làm việc với các API OpenStack.
 - OpenStack CLI được xây dựng trên Python SDK.


## Cách sử dụng API
### Chuẩn bị môi trường
Có thể sử dụng Extension App của Chrome như:
- Restlet Client
 Link: https://chrome.google.com/webstore/detail/restlet-client-rest-api-t/aejoelaoggembcahagimdiliamlcdmfm

Download app trên Windown
- Postman
 Link: https://www.getpostman.com/ 

> Trong bài viết sẽ sử dụng Postman để làm việc với OPS API 

### Lưu ý trước khi lab:

Lưu ý: Trong OpenStack có khái niệm `scope` hay còn gọi là phạm vi ủy quyền
- Token sẽ thể hiện quyền khác nhau với các scope khác nhau. Ví dụ như tập quyền trên các project, domain, chứng thực trong 1 thời điểm khác nhau. 
- Bản chất mỗi token được cấp sẽ có quyền khác nhau khi làm việc với các project khác nhau

Unscoped token:
- Token chứa có quyền sử dụng bất kỳ catalog, role, project, hoặc domain. Sử dụng token này đơn giản để chứng thực danh tính tại KeyStone tại 1 số thời điểm.

> Xem thêm vd docs bên dưới

Project-scoped token:
- Cho phép thực hiện hành động trên 1 số môi trường cụ thể,
- Nó bào gồm service log, tập role, chi tiết về project có quyền tương tác

> Xem thêm vd lab bên dưới

Domain-scoped token:
- Token cho quyền thực hiện hành động trên domain chỉ định. 
- Mỗi domain thường bao gồm nhiều project và user

### Bước 1: Tạo chứng thực
> Tìm hiểu thêm cơ chế làm việc với KeyStone theo docs 


- Làm việc với images
- Làm việc với network 
- Làm việc với flavor



# Nguồn

https://github.com/hocchudong/API-Openstack

https://www.codehub.vn/RESTful-API-Cho-Nguoi-Bat-Dau

