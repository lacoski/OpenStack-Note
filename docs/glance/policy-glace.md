# Policy.json Glance
---
## Quyền 
```
get_images - List available image entities
    GET /v1/images
    GET /v1/images/detail
    GET /v2/images

get_image - Retrieve a specific image entity
    HEAD /v1/images/<IMAGE_ID>
    GET /v1/images/<IMAGE_ID>
    GET /v2/images/<IMAGE_ID>

download_image - Download binary image data
    GET /v1/images/<IMAGE_ID>
    GET /v2/images/<IMAGE_ID>/file

upload_image - Upload binary image data
    POST /v1/images
    PUT /v1/images/<IMAGE_ID>
    PUT /v2/images/<IMAGE_ID>/file

copy_from - Copy binary image data from URL
    POST /v1/images
    PUT /v1/images/<IMAGE_ID>

add_image - Create an image entity
    POST /v1/images
    POST /v2/images

modify_image - Update an image entity
    PUT /v1/images/<IMAGE_ID>
    PUT /v2/images/<IMAGE_ID>

publicize_image - Create or update public images
    POST /v1/images with attribute is_public = true
    PUT /v1/images/<IMAGE_ID> with attribute is_public = true
    POST /v2/images with attribute visibility = public
    PUT /v2/images/<IMAGE_ID> with attribute visibility = public

communitize_image - Create or update community images
    POST /v2/images with attribute visibility = community
    PUT /v2/images/<IMAGE_ID> with attribute visibility = community
    delete_image - Delete an image entity and associated binary data
    DELETE /v1/images/<IMAGE_ID>
    DELETE /v2/images/<IMAGE_ID>

add_member - Add a membership to the member repo of an image
    POST /v2/images/<IMAGE_ID>/members
    get_members - List the members of an image
    GET /v1/images/<IMAGE_ID>/members
    GET /v2/images/<IMAGE_ID>/members
    delete_member - Delete a membership of an image
    DELETE /v1/images/<IMAGE_ID>/members/<MEMBER_ID>
    DELETE /v2/images/<IMAGE_ID>/members/<MEMBER_ID>
    modify_member - Create or update the membership of an image
    PUT /v1/images/<IMAGE_ID>/members/<MEMBER_ID>
    PUT /v1/images/<IMAGE_ID>/members
    POST /v2/images/<IMAGE_ID>/members
    PUT /v2/images/<IMAGE_ID>/members/<MEMBER_ID>

manage_image_cache - Allowed to use the image cache management API
```

# Nguồn 

https://docs.openstack.org/glance/latest/admin/policies.html

