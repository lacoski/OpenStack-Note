# Federated Identity
---
## Khái niệm

A federated identity in information technology is the means of linking a person's electronic identity and attributes, stored across multiple distinct identity management systems.

### Identity Provider (IdP)
A trusted provider of identity information. This may be backed by LDAP, SQL, AD, or Mongo. The identity provider is able to construct user attributes into Assertions or Claims.

### Service Provider (SP)

A service that consumes identity information. In our case, Keystone will be the service provider. Many web applications that utilize single sign-on are examples of service providers.

### SAML
An XML-based standard approach to federated identity. The resultant document that contains user attributes is called an Assertion.

### OpenID Connect
A newer standard approach to federated identity. Leverages OAuth 2.0, and ditches XML in favor of JSON. The resultant information about the user is called a Claim.

### Assertions and Claims
A standard method of representing information and attributes about a user. An identity provider may issue Assertions or Claims, based on the standard they use, for service providers to consume.

## Keystone-Specific Federation Concepts

### Identity Provider

Keystone now has an Identity Provider resource, which is served through the follow‐
ing API: /OS-FEDERATION/identity_providers .
This endpoint allows a cloud administrator to create, delete, update, retrieve, and list protocols per identity provider. A protocol has only a name and reference to the mapping that will be used (more on that later). The reasoning behind setting the protocol per identity provider is that an external identity provider may support more than one type of federated authentication, so we must allow an identity provider to have multiple protocols as well. The protocols are meant to mirror the federated identity standard that is being used. Common values for protocol names are saml2 (for SAML) and oidc (for OpenID Connect).

### Mapping

Mappings are the lynchpin to federated identity in Keystone. A mapping resource has now been created and served at the following API: /OS-FEDERATION/mappings . This endpoint allows a cloud administrator to create, delete, update, retrieve, and list map‐ pings. A mapping is specified when creating a protocol that an identity provider sup‐ ports.

## The Mapping Engine



## Authentication Flow: What’s It Look Like?

Federated authentication flow:


# OpenStack Keystone Workflow & Token Scoping

## Step 1: Obtain an unscoped token from Keystone
Để xác định quyền project được truy cập, ta cần sử dụng Keystone unscoped token. unscoped token nghĩa là token chưng, không thuộc bất kỳ project nào, nhưng token này ko có quyền gì cả.

unscoped token sử dụng để phát hiện các service được sử dụng trong project. Không thể sử dụng với các project khác ngoài keystone.

Nếu chứng thực thành công, ta sẽ nhận được token id, lưu ý, token id sẽ gán với token `X-Auth-Token` nằm tại header. 

# Step 2: Discover tenants you have access to
Bước tiếp theo sau khi có unscoped token, là sẽ sử dụng token này để xác định các project được quyền truy cập. 

Quyền truy cập dựa theo quyền được gán với user, tức mức truy cập sử dụng các service endpoint, giới hạn trên các endpoint.  

Tất cả các hoạt động thực hiện trên service endpoint yêu cầu `scoped token`. 

Để lấy đc scoped token, ta sẽ sử dụng sử dụng unscoped token lấy list các project. Từ các project, ta sẽ lựa chọn và sinh scoped token từ đây.


# Step 3: Obtain a scoped token
Sau khi xác định project sẽ sử dụng, ta sẽ tạo scoped token từ nó. scoped token sẽ chứa thông tin về project, cung cấp các metadata về quyền sẽ được làm việc trên đó.

Có thể tại scoped token từ unscoped token hoặc user, passwd user.

Khi sinh scoped token, ta sẽ nhận được thêm cả các service endpoint có thể sự dùng trên project. (Quyền sẽ dựa vào role user).

Một số ý chính của keystone:
- Keystone quản lý các service được định nghĩa trong service catalog. 
- Keystone quản lý số lượng các endpoint, bao gồm loại dịch vù và URL
- Mỗi endpint liên kết với 1 service.

# Step 4: Invoke the target endpoint service API
Sau khi có được scoped token, ta sẽ đem token đó để làm việc với các endpoint service hiện tại.

> Xem thêm tài liệu Openstack api 

# Nguồn
http://bodenr.blogspot.com/2014/03/openstack-keystone-workflow-token.html
