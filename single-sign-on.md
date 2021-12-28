# Single Sign On

## Giới thiệu

`Single Sign On` là 1 chức năng phụ thuộc identity server, host bởi portal, có tính năng xử lý luồng đăng nhập và redirect cho tất cả các ứng dụng trong hệ thống.

Host: [https://portal.jee.vn](https://portal.jee.vn)

## Sử dụng

### Đối với Portal

Đối với Portal, có thể truy cập vào site [https://portal.jee.vn](https://portal.jee.vn) để sử dụng Portal, ứng dụng sẽ tự check auth và chuyển hướng đăng nhập.
<br  />
Trang chức năng của Portal đang chờ maintain...

### Đối với các service khác

Đối với các Service khác có UI. Sử dụng 1 Middleware/Guard để chạy các tác vụ sau (Lưu ý cần phải chạy ngay khi người dùng vào trang app, sau đó chạy lại mỗi lần người dùng thao tác đến các controller trên trang hoặc set timer chạy 1 phút 1 lần hoặc chạy cả hai trương hợp trên):

1. Kiểm tra giá trị `sso_token` trên cookie domain có tồn tại hay không.

	1.1. Trường hợp không tồn tại thì tìm giá trị `sso_token` trên URL.

	1.2. Nếu vẫn không có thì chuyển hướng người dùng đến `https://portal.jee.vn/?redirectUrl=URL-chuyển-hướng-của-app` để lấy token.

	1.3. Khi người dùng đăng nhập xong, chuyển hướng về trang. Trên URL sẽ có `sso_token` trên query string, sử dụng giá trị này lưu vào cookie `sso_token`.

3. Lấy giá trị cookie `sso_token` gọi đến API `https://identityserver.jee.vn/user/me` có try catch.

	2.1. Trường hợp gọi API hoàn tất sẽ trả về 3 giá trị:
	
		- user: Chứa thông tin người dùng, có thể lưu lại vào app context để sử dụng.
		
		- access_token: Dùng token này lưu vào cookie `sso_token` thay thế token cũ. Đây là token có chứa data user bên trên.
		
		- refresh_token: Lưu token này vào cookie `sso_token_refresh`, đè lên giá trị cũ nếu có.

	2.2. Trường hợp API trên trả lỗi (catch lỗi) nghĩa là token đã hết hạn hoặc sai. Thực hiện gọi API `https://identityserver.jee.vn/user/refresh` với data là token lưu trong `sso_token_refresh`.
	
		- Trường hợp gọi API hoàn tất sẽ trả về tương tự 2.1, thực hiện tương tự 2.1
			
		- Trường hợp gọi API lỗi nghĩa là refresh_token hết hạn hoặc sai. Thực hiện xóa hết tất cả các cookie liên quan đến authentication rồi chuyển hướng người dùng đến `https://portal.jee.vn/?redirectUrl=URL-chuyển-hướng-của-app`.