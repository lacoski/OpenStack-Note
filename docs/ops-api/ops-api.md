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
- Token không có quyền sử dụng bất kỳ catalog, role, project, hoặc domain. Sử dụng token này đơn giản để chứng thực danh tính tại KeyStone tại 1 số thời điểm.

> Xem thêm vd docs bên dưới

Project-scoped token:
- Cho phép thực hiện hành động trên 1 số môi trường cụ thể,
- Nó bào gồm service log, tập role, chi tiết về project có quyền tương tác

> Xem thêm vd lab bên dưới

Domain-scoped token:
- Token cho quyền thực hiện hành động trên domain chỉ định. 
- Mỗi domain thường bao gồm nhiều project và user

# Sử dụng API OPENSTACK

Danh sách API OPS:

http://172.16.4.200/dashboard/project/api_access/

pic 2

## Phần 1: Làm việc với Identity API v3 service
> Tìm hiểu thêm cơ chế làm việc với KeyStone theo docs 

### Cơ bản về dịch vụ Identity


Dịch vụ Identity service sử dụng để sinh token. Token tượng trưng cho chứng thực định danh user, tổ chức, quyền hạn trên các project, domain và hệ thống


Có 2 phương thức chứng thực:
- Password
- Token

Trong các token sẽ chứa:
- Credential (Thông tin xác thực)
- Authorization scope (Phạm vi quyền hạn)

Token trả lại bao gồm:
- Token IDs và giá trị X-Subject-Token tại header response

Sau khi có token ta có thể:
- Tạo các REST API tới các dịch vụ OpenStack khác.
- Cần khai báo giá trị `X-Auth-Token` tại request header
- Validate token, liệt kê danh sách các domain, project, role, endpoint token cho phép truy cập
- Thu hồi token

pic 1

### Chứng thực password dạng unscoped authorization
> POST: /v3/auth/tokens

Lưu ý:
- Token dạng unscoped, tức Token không có quyền sử dụng bất kỳ catalog, role, project, hoặc domain. Sử dụng token này đơn giản để chứng thực danh tính tại KeyStone tại 1 số thời điểm.
- Phương thức chứng thực dạng password, user cần khai báo id or name, password

Body request raw dạng json
```
{
    "auth": {
        "identity": {
            "methods": [
                "password"
            ],
            "password": {
                "user": {
                    "name": "admin",
                    "domain": {
                        "name": "Default"
                    },
                    "password": "devstacker"
                }
            }
        }
    }
}
```

VD:

pic 3

KQ:

pic 4

pic 5


### Chứng thực password dạng scoped authorization
> POST: /v3/auth/tokens

Lưu ý:
- Phương thức chứng thực cho phép truy cập các project, domain, system
- Request body cần bao gồm pasword, thêm các thông tin về project, domain, system

Các loại chứng thực cơ bản:
- Chứng thực system scoped
 ```
 {
    "auth": {
        "identity": {
            "methods": [
                "password"
            ],
            "password": {
                "user": {
                    "id": "ee4dfb6e5540447cb3741905149d9b6e",
                    "password": "devstacker"
                }
            }
        },
        "scope": {
            "system": {
                "all": true
            }
        }
    }
 }
 ```
- Chứng thực Domain-Scoped
 ```
 {
    "auth": {
        "identity": {
            "methods": [
                "password"
            ],
            "password": {
                "user": {
                    "id": "ee4dfb6e5540447cb3741905149d9b6e",
                    "password": "devstacker"
                }
            }
        },
        "scope": {
            "domain": {
                "id": "default"
            }
        }
    }
 }
 ```
- Chứng thực Project-Scoped
 ```
 {
    "auth": {
        "identity": {
            "methods": [
                "password"
            ],
            "password": {
                "user": {
                    "id": "ee4dfb6e5540447cb3741905149d9b6e",
                    "password": "devstacker"
                }
            }
        },
        "scope": {
            "project": {
                "id": "a6944d763bf64ee6a275f1263fae0352"
            }
        }
    }
 }
 ```
VD:
pic 6 7 8  

### Chứng thực token dạng unscoped authorization
> POST: /v3/auth/tokens

Lưu ý:
- Token dạng unscoped, tức Token không có quyền sử dụng bất kỳ catalog, role, project, hoặc domain. Sử dụng token này đơn giản để chứng thực danh tính tại KeyStone tại 1 số thời điểm.
- Phương thức chứng thực dạng password, user cần khai báo id or name, password

Body request
```
{
"auth": {
    "identity": {
        "methods": [
            "token"
        ],
        "token": {
            "id": "'$OS_TOKEN'"
        }
    }
}
}
```
VD:

pic 9 10 11

### Chứng thực token dạng scoped authorization
> POST: /v3/auth/tokens

Lưu ý:
- Phương thức chứng thực cho phép truy cập các project, domain, system
- Request body cần bao gồm pasword, thêm các thông tin về project, domain, system

- Chứng thực dạng System-Scoped
 ```
 {
    "auth": {
        "identity": {
            "methods": [
                "token"
            ],
            "token": {
                "id": "'$OS_TOKEN'"
            }
        },
        "scope": {
            "system": {
                "all": true
            }
        }
    }
 }
 ```
- Chứng thực dạng Domain-Scoped 
 ```
 {
    "auth": {
        "identity": {
            "methods": [
                "token"
            ],
            "token": {
                "id": "'$OS_TOKEN'"
            }
        },
        "scope": {
            "domain": {
                "id": "default"
            }
        }
    }
 }
 ```
- Chứng thực dạng Project-Scoped
 ```
 {
    "auth": {
        "identity": {
            "methods": [
                "token"
            ],
            "token": {
                "id": "'$OS_TOKEN'"
            }
        },
        "scope": {
            "project": {
                "domain": {
                    "id": "default"
                },
                "name": "admin"
            }
        }
    }
 }
 ```
VD:

pic 12 13 14

### Chứng thực token và show thông tin token
> GET: /v3/auth/tokens

Trả lại thông tin các token

Lưu ý: Cần 2 tham số
- Cần tham số X-Auth-Token: Token hiện tại
- Cần tham số X-Subject-Token: Token cần chứng thực

pic 15 16 17

### Kiểm tra token 
> HEAD /v3/auth/tokens

Kiểm tra token giống chứng thực nhưng không có kết quả trả về.

Yêu cầu 2 tham số:
- Cần tham số X-Auth-Token: Token hiện tại
- Cần tham số X-Subject-Token: Token cần kiểm tra

pic 18 19

### Thu hồi token
> DELETE: /v3/auth/tokens

Giống chứng thực, nhưng mục đích là thu hồi token

pic 20 21

### Lấy catalog service được sử dụng
> Lưu ý sử dụng token dạng scoped như project scoped

VD:
pic 22 23


### Lấy project có thể sử dụng
> Lưu ý sử dụng token dạng scoped như project scoped

pic 24, 25


### Lưu ý
```
Sau khi có token (X-Auth-Token) trong header request, user có thể tương tác với các project, service khác trong ops theo scoped user
```

### Liệt các các service có thể sử dụng

pic 26, 27

## Phần 2: Làm với image service
> Làm việc với project glance

> Yêu cầu đã chứng thực scoped token với quyền trên system, hoặc project. X-Auth-Token trên header

### Liệt kê danh sách các image
> GET: /v2/images

> Yêu cầu X-Auth-Token tại header

pic 28 29

### Xem thông tin chi tiết image
> GET: /v2/images/{image_id}

pic 30 31

### Delete Image

pic 32 33

### Cách tạo Image bằng API OPS
Cần 2 bước để tạo Image:
- Tạo Image trống chưa có file data (Sau khi tạo status dạng queue)
- Upload file data (Sau khi upload image status chuyển sang dạng active)

Tạo khung image:
- Cần X-Auth-Token tại header
- Trong body request cần tham số
 - "container_format"
 - "disk_format"
 - "name"

pic 34 35

Upload image:

pic 36 37


## Phần 3: Làm với network api
> Làm việc với project neutron

> Yêu cầu đã chứng thực scoped token với quyền trên system, hoặc project. X-Auth-Token trên header


### Lấy danh sách network

pic 38 39

### Xem chi tiết network

pic 40 41

### Xóa network

pic 42 43

### Tạo network

pic 44 45

### Lấy danh sách subnet

pic 46 47

### Xem chi tiết subnet

pic 48 49

### Tạo subnet mới

pic 50 51

### Xóa subnet

pic 52 53

----
- Làm việc với flavor, tạo vm
# Nguồn

https://github.com/hocchudong/API-Openstack

https://www.codehub.vn/RESTful-API-Cho-Nguoi-Bat-Dau

