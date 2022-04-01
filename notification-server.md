# Notification Server

## Giới thiệu

`Notification Server` là 1 service module thuộc kiến trúc hệ thống DPS, có khả năng gửi thông báo bất đồng bộ:

 - Host socket: `wss://socket.jee.vn/notification`
 - Host API: `https://notification.jee.vn`

Có 2 kiểu gửi thông báo. Gửi message qua websocket và gửi email.

## Cài đặt chung

Trong file `Startup.cs` phần `ConfigureServices` thêm vào service notification. Lưu ý: Notification cần kafka để chạy, phải thêm vào sau khi đã thêm service kafka.

``` cs
services.addNotificationService();
```
Cũng trong file `Startup.cs` phần `ConfigureServices` thêm vào đoạn code này để lấy internal_secret từ vault.
```cs
Secret<SecretData> jwtSecret = vaultClient.V1.Secrets.KeyValue.V2.ReadSecretAsync(path: "jwt", mountPoint: "kv").Result;
IDictionary<string, object> jwtData = jwtSecret.Data.Data;
Configuration["Jwt:internal_secret"] = jwtData["internal_secret"].ToString();
```
Ở controller cần sử dụng, chỉ cần inject service vào rồi gọi method gửi thông báo tương ứng.

## Websocket

### Kết nối đầu nhận

Đầu nhận message từ server này cần sử dụng thư viện từ [socket.io](https://socket.io/). ***LƯU Ý: Chỉ sử dụng phiên bản client có khả năng connect vào socket.io server phiên bản 2.x
<br />
Các bước kết nối (Thực hiện sau khi đăng nhập và có được JWT token):

 1. Thực hiện kết nối từ Socket IO Client đến server qua host phía trên, trong bước kết nối đầu tiên quá trình handshake cần được gửi kèm 1 `extraHeaders` có key là `x-auth-token`, với value là JWT token sau khi hoàn tất các bước đăng nhập nhận được.
 2. Dùng instance đã connect đến server listen 1 event tên là `notification`. Các data được emit đến event này sẽ là data notification được server gửi đến.

Để có thể nhận thêm thông báo push từ OneSignal cần thực hiện thêm kiểm tra đăng ký. Sau khi thực hiện xác thực người dùng và đã lấy được `username`, thực hiện thêm đoạn script sau vào trang layout của ứng dụng:

``` js
    const username = // sau khi xác thực có username
    const host = {
      portal: 'https://portal.jee.vn',
    }

    // Thiết lập iframe đến trang đăng ký 
    const iframeSource = `${host.portal}/?getstatus=true`
    const iframe = document.createElement('iframe')
    iframe.setAttribute('src', iframeSource)
    iframe.style.display = 'none'
    document.body.appendChild(iframe)

    // Thiết lập Event Listener để xác nhận người dùng đăng ký chưa
    window.addEventListener(
      'message',
      (event) => {
        if (event.origin !== host.portal) return // Quan trọng, bảo mật, nếu không phải message từ portal thì ko làm gì cả, tránh XSS attack
        // event.data = false là user chưa đăng ký nhận thông báo, nếu đăng ký rồi thì là true
        if (event.data === false) {
            // Đoạn setTimeout này chỉ là 1 ví dụ -> Nếu người dùng vào trang mà chưa đăng ký thì 2s sau sẽ hiện popup cho người dùng đăng ký
            // Có thể tùy chỉnh đoạn này, thêm vào cookie, popup, button,... để tự chủ động trong việc đăng ký
          setTimeout(() => {
              // Lệnh window.open này chính là lệnh gọi mở popup đến trang đăng ký
              // Trang này vừa có thể đăng ký, vừa có thể hủy đăng ký
              // Có thể sử dụng lệnh này gán vào 1 nút nào đó trên trang cho người dùng chủ động trong việc đăng ký hoặc hủy đăng ký
            window.open(
              `${host.portal}/notificationsubscribe?username=${username}`, // username điền vào đây
              'childWin',
              'width=400,height=400'
            )
          }, 2000)
        }
      },
      false
    )
```

Lưu ý: đọc kỹ các comment trong đoạn code bên trên và sử dụng phù hợp với mục đích của trang

### Kết nối đầu gửi

Tại controller muốn gửi thông báo, inject service `notifier` với interface là `INotifier` lấy được từ DPSinfra. Trong ví dụ dưới, service đã được inject vào biến `_notifier`. Gọi method `sendSocket` với data thuộc class `socketMessage` để gửi thông báo qua websocket

``` cs
public void testnotification()
{
    var json = new
    {
        abc = "123"
    };
    socketMessage asyncnotice = new socketMessage()
    {
        sender = "Username người gửi",
        receivers = new string[] { "superuser" },
        message_text = "test abcde",
        message_html = "<h1>abc</h1>",
        message_json = JsonConvert.SerializeObject(json),

        // Các field dưới đây là của OneSignal
        osTitle = "Test title",
        osMessage = "Test message",
        osWebURL = "https://google.vn",
        osAppURL = "https://google.vn",
        osIcon = "https://api.jeehr.com/images/logokhachhang/25.jpg"
    };
    _notifier.sendSocket(asyncnotice);
}
```

Lưu ý: Push notify socket sẽ hoạt động nếu 1 trong 3 field `message_text`, `message_html`, `message_json` có dữ liệu. Push notify của OneSignal sẽ hoạt động nếu 1 trong 2 field `osTitle` hoặc `osMessage` có dữ liệu. User nhận notify sẽ dùng chung field `receivers`

## Email

Tại controller muốn gửi thông báo, inject service `notifier` với interface là `INotifier` lấy được từ DPSinfra. Trong ví dụ dưới, service đã được inject vào biến `_notifier`. Gọi method `sendEmail` với data thuộc class `emailMessage` để gửi thông báo qua websocket
**Gửi email sẽ được chia làm 2 trường hợp dưới đây**
- Trường hợp dùng access_token
``` cs
public async void testnotificationemail()
{
    emailMessage asyncnotice = new emailMessage()
    {
	    access_token = "abc...",
        from = "ABCDE",
        to = "hoaauquoc@gmail.com",
        subject = "subject",
        html = "<h1>Hello World</h1>"
    };
    await _notifier.sendEmail(asyncnotice);
}
```
- Trường hợp dùng CustomerID, nếu vừa bỏ CustomerID và access_token vào model thì lib sẽ sử dụng CustomerID
 ``` cs
public async void testnotificationemail()
{
    emailMessage asyncnotice = new emailMessage()
    {
	    CustomerID = 1,
        from = "ABCDE",
        to = "hoaauquoc@gmail.com",
        subject = "subject",
        html = "<h1>Hello World</h1>"
    };
    await _notifier.sendEmail(asyncnotice);
}
```

Method `sendEmail` là async, nếu dùng await có thể dùng thêm try-catch để bắt lỗi, nếu có.
## Desktop

Tại controller muốn gửi thông báo đến desktop, inject service `notifier` với interface là `INotifier` lấy được từ DPSinfra. Trong ví dụ dưới, service đã được inject vào biến `_notifier`. Gọi method `sendDesktop` với data thuộc class `desktopMessage` để gửi thông báo qua Desktop
``` cs
public async void testnotificationdesktop()
{
    desktopMessage asyncnotice = new desktopMessage()
    {
	      sender = "Username người gửi",
        receivers = new string[] { "superuser" },
        message_text = "test abcde",
        message_html = "<h1>abc</h1>",
        message_json = JsonConvert.SerializeObject(json),

        // Các field dưới đây là của Desktop
        dtTitle = "Test title",
        dtMessage = "Test message",
        dtWebURL = "https://google.vn",
        dtAppURL = "https://google.vn",
        dtIcon = "https://api.jeehr.com/images/logokhachhang/25.jpg"
    };
    await _notifier.sendDesktop(asyncnotice);
}
```

Method `sendDesktop` là async, nếu dùng await có thể dùng thêm try-catch để bắt lỗi, nếu có.
## API Truy xuất dữ liệu

### Lấy thông báo của người dùng hiện tại
Path: `/notification/pull` <br />
Method: `GET` <br />
Header:

 - Authorization: `access-token`

Query param:

 - status: enum["read", "unread"]

Query param `status` là optional, truyền vào `read` sẽ lấy các thông báo đã đọc, `unread` là các thông báo chưa đọc. Nếu không truyền vào sẽ lấy tất cả.

### Đọc thông báo
Path: `/notification/read` <br />
Method: `POST` <br />
Header:

 - Authorization: `access-token`

Body: application/json
``` json
{
	"id":  "string"
}
```

Query param `id` là bắt buộc, khi người dùng có thao tác liên quan đến thông báo thì gọi API này để ghi lại thông tin, thực hiện với cả thông báo chưa được đọc và đã từng đọc.


### Đọc tất cả thông báo
Path: `/notification/readall` <br />
Method: `POST` <br />
Header:

 - Authorization: `access-token`

### Lấy thông báo mới của người dùng hiện tại (Bao gồm thông báo chuông, mainMenu, subMenu, Item)
Path: `/notification/pull` <br />
Method: `GET` <br />
Header:

 - Authorization: `access-token`

+ Lấy thông danh sách tất cả các thông báo mới:
Query param `status` là optional, truyền vào `new` sẽ lấy các thông báo mới, `unnew` là các thông báo không còn mới. Nếu không truyền vào sẽ lấy tất cả (Count hoặc Length để đếm số lượng thông báo mới).
+ Lấy thông danh sách các thông báo mới ở MainMenu:
Query param `status` và `id` là optional, với param `status` truyền vào `mainmenu` và với param `id` truyền vào `ID` của mainMenu đó (Count hoặc Length để đếm số lượng thông báo mới).
+ Lấy thông danh sách các thông báo mới ở SubMenu:
Query param `status` và `id` là optional, với param `status` truyền vào `submenu` và với param `id` truyền vào `ID` của subMenu đó (Count hoặc Length để đếm số lượng thông báo mới).
+ Lấy thông danh sách các thông báo mới ở Item:
Query param `status` và `id` là optional, với param `status` truyền vào `item` và với param `id` truyền vào `ID` của item đó (Count hoặc Length để đếm số lượng thông báo mới).

+ các id của menu vào item và cũng như là Appcode sẽ được truyền bằng thư viện DPSinfra Version 1.7.3 trở lên, sẽ bao gồm các thông tin cần truyền như sau:
socketMessage asyncnotice = new socketMessage()
``` cs
public void testnotification()
{
    var json = new
    {
        abc = "123"
    };
    socketMessage asyncnotice = new socketMessage()
    {
        ...
        idObject = 1, // tương ứng với itemid
        AppCode = "CODE",
        menuID = 1,
        subMenuID = 1,
    };
    _notifier.sendSocket(asyncnotice);
}
```

### Đánh dấu đã xem tất cả thông báo mới
Path: `/notification/readallnew` <br />
Method: `POST` <br />
Header:

 - Authorization: `access-token`

### Đánh dấu đã xem thông báo ở từng mục (mainmenu, submenu, item)
Path: `/notification/readnew` <br />
Method: `POST` <br />
Header:

 - Authorization: `access-token`

Body: application/json
``` json
{
	"appCode":  "string",
	"mainMenuID":  "number",
	"subMenuID":  "number",
	"itemID":  "number" (idObject gửi từ thư viện)
}
```

Query param `AppCode` là bắt buộc các thông tin khác có thể để trống

### Đánh dấu đã xem và đã đọc thông báo từ trang chi tiết
- Thay vì click vào cái chuông xong click vào thông báo thì mới tính là đã đọc đó,thay vào đó người dùng vào chi tiết (gõ tay hoặc click onesignal để vào chi tiết) sẽ gọi api này để đánh dấu đã đọc cho thông báo đó
Path: `/notification/readDetail` <br />
Method: `POST` <br />
Header:

 - Authorization: `access-token`

Body: application/json
``` json
{
	"appCode":  "string",
	"itemID":  "number" (idObject gửi từ thư viện)
}
```
