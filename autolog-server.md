# AsyncLog Server

## Giới thiệu

`AsyncLog Server` là 1 service module thuộc kiến trúc hệ thống DPS, có khả năng ghi log bất đồng bộ:
<br />
Index pattern: `asynclogger.generallog-*`, `asynclogger.activitylog-*`
<br />
Query dashboard: [https://kibana.jee.vn](https://kibana.jee.vn)

## Thiết lập cần thiết


Để sử dụng cần phải có thư viện DPSinfra v1.2.0 trở lên

<br />

Để sử dụng cần phải có Kafka Producer có thể Produce được message

<br />

Đảm bảo file `settings` đang sử dụng có phần `logging` được cài đặt như sau:
``` json
{
	"Logging": {
		"LogLevel": {
			"Default": "Trace",
			"Microsoft": "Warning",
			"Microsoft.Hosting.Lifetime": "Information"
		}
	},
	...
}
```

Phần `ConfigureServices` trong file `Startup` thêm vào `provider` mới cho logger như sau:
``` cs
services.AddLogging(builder => {
                builder.addAsyncLogger<AsyncLoggerProvider>(p => new AsyncLoggerProvider(p.GetService<IProducer>()));
            });
```

## Cách sử dụng

Trong 1 controller bất kỳ, muốn sử dụng cần inject logger vào nội hàm:
``` cs
public class apiController : ControllerBase
    {
        private readonly ILogger<apiController> _logger;
        ...

        public apiController(ILogger<apiController> logger,...)
        {
            _logger = logger;
            ...
        }
...
}
```

Tại method cần log, sử dụng service log đã inject để gọi đúng method có `LogLevel` cần sử dụng log:
``` cs
public void testautolog()
        {
            var d1 = new GeneralLog()
            {
                name = "name",
                message = "message",
                data = "data"
            };
            _logger.LogTrace(JsonConvert.SerializeObject(d1));
            _logger.LogDebug(JsonConvert.SerializeObject(d1));
            _logger.LogInformation(JsonConvert.SerializeObject(d1));
            _logger.LogWarning(JsonConvert.SerializeObject(d1));
            _logger.LogError(JsonConvert.SerializeObject(d1));
            _logger.LogCritical(JsonConvert.SerializeObject(d1));
            var d2 = new ActivityLog()
            {
                username = "username",
                category = "category",
                action = "action",
                data = "data"
            };
            _logger.LogTrace(JsonConvert.SerializeObject(d2));
            _logger.LogDebug(JsonConvert.SerializeObject(d2));
            _logger.LogInformation(JsonConvert.SerializeObject(d2));
            _logger.LogWarning(JsonConvert.SerializeObject(d2));
            _logger.LogError(JsonConvert.SerializeObject(d2));
            _logger.LogCritical(JsonConvert.SerializeObject(d2));
        }
```
`LogLevel` theo định nghĩa của `Microsoft` có thể tham khảo [tại đây](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.logging.loglevel?view=dotnet-plat-ext-5.0)
<br />
`GeneralLog` và `ActivityLog` là 2 class được hỗ trợ bởi AsyncLog Server, chỉ có dữ liệu là instance của 2 class này mới được lưu trữ vào server.
<br />
Tùy theo mục đích log, cần lựa chọn loại class log và `LogLevel` phù hợp để sử dụng.
