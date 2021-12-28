# Identity Server

## Giới thiệu

`Identity Server` là 1 service module thuộc kiến trúc hệ thống DPS, có các tính năng liên quan đến user và xác thực user:

 - Đăng nhập
 - Lấy thông tin user đang đăng nhập
 - Refresh token
 - Đăng xuất (Revoke token)
 - Đổi mật khẩu
 - Thêm mới user (User action create_new_user)
 - Update Customdata (User action update_custom_data)
 - Tạm khóa / Kích hoạt user  (User action change_user_state)

Lưu ý :  `Identity Server` chỉ xác thực người dùng (`authentication process`), KHÔNG ủy quyền người dùng (`authorization process`). Việc ủy quyền sẽ tùy thuộc từng project riêng lẻ dựa trên bộ `customdata` của user.
  

Host: [https://identityserver.jee.vn](https://identityserver.jee.vn)

## Cấu trúc User tổng quát

Về tổng thể 1 user được lưu trữ ở `IdentityServer` sẽ có bộ dữ liệu bao gồm:

 - username: `String`
 - password: `String`
 - refreshToken: `String`
 - customData: `Anything`

`password` được lưu dưới dạng đã hash, không thể truy xuất ngược.
<br />
`refreshToken` dùng để lấy lại `accessToken` khi hết hạn
<br />
`customData` là bộ dữ liệu riêng của user, phi cấu trúc. Tùy thuộc vào từng dự án, bộ dữ liệu này sẽ được thêm vào user và có thể thay đổi bằng api (ví dụ: dữ liệu phân quyền roles, ...)

## Các API

### Đăng nhập
Path: `/user/login` <br />
Method: `POST` <br />

Body: application/json
``` json
{
	"username":  "string",
	"password":  "string"
}
```

### Lấy thông tin user đang đăng nhập
Path: `/user/me` <br />
Method: `GET` <br />
Header:

 - Authorization: `access-token`

### Refresh token
Path: `/user/refresh` <br />
Method: `POST` <br />
Header:

 - Authorization: `refresh-token`

### Đăng xuất
Path: `/user/logout` <br />
Method: `POST` <br />
Header:

 - Authorization: `access-token`

### Đổi mật khẩu
Path: `/user/changePassword` <br />
Method: `POST` <br />
Header:

 - Authorization: `access-token`

Body: application/json
``` json
{
	"password_old":  "string",
	"password_new":  "string"
}
```

### Thêm mới user
Path: `/user/addnew` <br />
Method: `POST` <br />
Header:

 - Authorization: `access-token`

Body: application/json
``` json
{
	"username":  "string",
	"password":  "string",
	"customData":  "anything"
}
```
Mặc định, khi tạo mới user mà không có customData hoặc customData không có field `personalInfo` thì hệ thống sẽ tự tạo ra field này với giá trị mặc định là:
``` json
{
	"Avatar":  '',
	"Firstname":  '',
	"Lastname":  '',
	"Jobtitle":  '',
	"Department":  '',
	"Birthday":  '',
	"Phonenumber":  '',
}
```

### Thêm mới user internal
Path: `/user/addnew/internal` <br />
Method: `POST` <br />
Header:

 - Authorization: `access-token-internal`

Body: application/json
``` json
{
	"username":  "string",
	"password":  "string",
	"customData":  "anything"
}
```
API này tương tự Addnew user bên trên nhưng sử dụng token nội bộ. Dùng để gọi ngầm trong cụm server tạo user mặc định

### Cập nhật Customdata
Path: `/user/updateCustomData` <br />
Method: `POST` <br />
Header:

 - Authorization: `access-token`

Body: application/json
``` json
{
	"userId":  "string", // 1 trong 2
	"username": "string", // 1 trong 2
	"updateField":  "string",
	"fieldValue":  {
		//anything
	}
}
```  
API sẽ cập nhật giá trị `fieldValue` vào field có tên = `updateField` nằm trong `customData` của user. Nếu chưa có `updateField` sẽ được tạo mới, đã có thì sẽ ghi đè.


### Cập nhật Customdata internal

Path: `/user/updateCustomData/internal`  <br  />

Method: `POST`  <br  />
Header:
- Authorization: `access-token-internal`

Body: application/json

``` json
{
	"userId":  "string", // 1 trong 2
	"username": "string", // 1 trong 2
	"updateField":  "string",
	"fieldValue":  {
		//anything
	}
}
```
API sẽ cập nhật giá trị `fieldValue` vào field có tên = `updateField` nằm trong `customData` của user. Nếu chưa có `updateField` sẽ được tạo mới, đã có thì sẽ ghi đè.

### Khóa / Kích hoạt user
Path: `/user/changeUserState` <br />
Method: `POST` <br />
Header:

 - Authorization: `access-token`

Body: application/json
``` json
{
	"userId":  "string", // 1 trong 2
	"username": "string", // 1 trong 2
	"disabled":  boolean
}
```
### Đổi mật khẩu internal
Path: `/user/changePassword/internal` <br />
Method: `POST` <br />
Header:

 - Authorization: `access-token-internal`

Body: application/json
``` json
{
	"userId":  "string", // 1 trong 2
	"username": "string", // 1 trong 2
	"password_old":  "string",
	"password_new":  "string"
}
```
### Khóa / Kích hoạt user internal
Path: `/user/changeUserState/internal` <br />
Method: `POST` <br />
Header:

 - Authorization: `access-token-internal`

Body: application/json
``` json
{
	"userId":  "string", // 1 trong 2
	"username": "string", // 1 trong 2
	"disabled":  boolean
}
```