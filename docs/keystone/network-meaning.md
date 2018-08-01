public_url: This parameter is the URL that end users would connect on. In a public cloud environment, this would be a public URL that resolves to a public IP address.

admin_url: This parameter is a restricted address for conducting administration. In a public deployment, you would keep this separate from the public_url by presenting the service you are configuring on a different, restricted URL. Some services have a different URI for the admin service, so this is configured using this attribute.

internal_url: This parameter would be the IP or URL that existed only within the private local area network. The reason for this is that you can connect to services from your cloud environment internally without connecting over a public IP address space, which could incur data charges for traversing the Internet. It is also potentially more secure and less complex to do so.

# Bootstrapping Identity
Sau khi cấu hình, triến khai KeyStone, nó cần khởi tạo 1 số thông tin đề khởi tạo sau này. 

Xứ lý này gọi là bootstrapping. Nó sẽ tạo thông tin liên quan về system first user, project, domain, service, endpoint, ...

Đây là cách khuyến nghị khi cài đặt mới.

Thứ 2, phương pháp này sẽ khởi tạo các thông tin bí mật khi tạo middleware bên trong identity service. Thông tin này được biết đến là "ADMIN_TOKEN". 

Nếu bất kỳ request tương tác với identity API với `ADMIN_TOKEN` được quyền sử dụng tất cả API còn lại.

# 

1) Authentication is totally pluggable. You can write our own custom auth method.  Beause of this extensible auth method, now keystone supports oauth1, federation ( federation is not fully done)
2)  Authorization : V2 is either "admin" or none. In v3 you can control who can call each method. ( Provided you diefine your own policy file )
3) Separate drivers for assignments and identity
4) Rich set of APIs. There are lot more API available than v2.0. Also there are no vendor specic extension. If you check  v2.0,  most of the role  apis are Rackspace extensions

Your questions is mostly towards difference between v2 and v3. If you are planning to move to v3, you need to know few things.

  1) None of the services support v3.
  2) Keystone command line client is only v2. It is not going to support v3. Suggestion is to use openstack client.  I don't think openstack client supports v3. So you have to use REST API to create domains/groups/users/ etc in v3
   3) If you use keystone client to create users, then most likely it won't work with v3 api. ( you can use default domain to make it work)
   4)  I'm not sure about this, but horizon only supports v3 auth and not v3 user/domain creation
   5)  Username/Project names are unqiue only ...