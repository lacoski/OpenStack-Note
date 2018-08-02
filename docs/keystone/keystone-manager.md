# Tìm hiểu câu lệnh keystone-manage
---
## Giới thiệu

keystone-manage là công cụ dòng lệnh tương tác với Keystone để thiết lập và cập nhật dữ liệu trong việc quản lý các dịch vụ của keystone. 

> keystone-manage chỉ được sử dụng để hoạt động mà không thể thực hiện thông qua các API của HTTP - như là import/export dữ liệu và di chuyển database.

## Cấu trúc câu lệnh
```
keystone-manage [options] <action> [additional args]
```

Lưu ý:
- `--help` : display verbose help output.

Một số action:
- `bootstrap`: Perform the basic bootstrap process.
- `credential_migrate`: Encrypt credentials using a new primary key.
- `credential_rotate`: Rotate Fernet keys for credential encryption.
- `credential_setup`: Setup a Fernet key repository for credential encryption.
- `db_sync`: Sync the database.
- `db_version`: Print the current migration version of the database.
- `doctor`: Diagnose common problems with keystone deployments.
- `domain_config_upload`: Upload domain configuration file.
- `fernet_rotate`: Rotate keys in the Fernet key repository.
- `fernet_setup`: Setup a Fernet key repository for token encryption.
- `mapping_populate`: Prepare domain-specific LDAP backend.
- `mapping_purge`: Purge the identity mapping table.
- `mapping_engine`: Test your federation mapping rules.
- `saml_idp_metadata`: Generate identity provider metadata.
- `token_flush`: Purge expired tokens.


# Nguồn 

https://docs.openstack.org/keystone/pike/cli/index.html#keystone-manage

https://github.com/hocchudong/thuctap012017/blob/master/TamNT/Openstack/Keystone/docs/6.Tim_hieu_them_ve_Keystone.md

https://docs.openstack.org/keystone/queens/cli/index.html

