# Gọi đến service khác
Khi viết chương trình theo dạng micro-service, các service khác nhau sẽ có các công việc cụ thể, riêng biệt và thường xuyên phải giao tiếp với nhau để trao đổi dữ liệu.
Có 2 dạng gọi service trong kiến trúc DPS:

 - Gọi đồng bộ (Dùng khi có lệnh gọi cần trả kết quả ngay lập tức - Gọi bằng HTTP request - ví dụ: gọi update user `customdata` trong `IdentityServer`)
 - Gọi bất đồng bộ (Dùng khi không cần trả kết quả ngay, khi chuyển bước trong quy trình, khi dữ liệu lớn và liên tục cần gọi dịch xử lý - Gọi bằng `Kafka broker` - ví dụ: gọi ghi log, gọi lệnh gửi email)

## Quy ước

Các thông tin cần để thực hiện kết nối đến service khác (URL, kafka topic, ...) cần được đặt trong file config `appsettings.Development.json` của service và gọi ra sử dụng ở dạng config. Không được hardcode. 

## Gọi đồng bộ

Gọi bằng `HttpClient` của `.NET framework`. Ví dụ nằm trong code mẫu, api `testcall50` thuộc `TestCore3.1` và api `testcall31` thuộc `TestCore5.0`.

## Gọi bất đồng bộ

Gọi thông qua kafka, sử dụng thư viện `DPSinfra`. Đối với kafka broker sẽ có các khái niệm trong gửi nhận dữ liệu là:

 - Producer (Đầu gửi dữ liệu, thực hiện gửi message vào broker).
 - Consumer (Đầu nhận dữ liệu, nhận dữ liệu từ broker theo `Consumer Group`.
 - Topic (Là key xác định chủ đề gửi-nhận dữ liệu giữa các `Producer-Consumer`, giống như là roomchat)

### Lưu ý

 - `Producer` có thể gửi tự do, chỉ cần tên `Topic` là gửi được message. Toàn bộ dữ liệu gửi qua `Producer` phải ở dạng `String`.
 - `Consumer` muốn nhận message cần phải quan tâm `Topic` và `Consumer Group`. 
	 - Nếu có nhiều `Consumer` nằm trong cùng 1 `Consumer Group` kết nối đến 1 `Topic` thì chỉ 1 `Consumer` tại 1 thời điểm được phép nhận message từ `Topic` đó (Cơ chế `Load Balance`).
	 - Nếu nhiều `Consumer Group` cùng kết nối đến 1 `Topic` thì tại 1 thời điểm tất cả các `Consumer Group` đều nhận được message từ `Topic` đó (Cơ chế `Broadcast`).

### Thêm service Kafka vào chương trình

Việc thêm Kafka vào code sẽ thực hiện ở file `Startup.cs` của Project. Trước khi thêm, cần phải có thông tin username và password được lấy từ `Vault`.
``` cs
public void ConfigureServices(IServiceCollection services)
{
...
    // vaultClient là Client của Vault đã được thêm vào trước
    Secret<SecretData> kafkaSecret = vaultClient.V1.Secrets.KeyValue.V2.ReadSecretAsync(path: "kafka", mountPoint: "kv").Result;
    IDictionary<string, object> kafkaData = kafkaSecret.Data.Data;
    string KafkaUser = kafkaData["username"].ToString();
    string KafkaPassword = kafkaData["password"].ToString();

    Configuration["KafkaConfig:username"] = KafkaUser;
    Configuration["KafkaConfig:password"] = KafkaPassword;

    // Thêm Kafka vào
    services.addKafkaService();
...
}
```

### Tạo Producer

Sau khi thêm Kafka vào chương trình, `Kafka client` đã được thư viện `DPSinfra` lưu thành `Singleton Service`. Trong controller chỉ cần inject producer vào là sử dụng được. `Producer` chỉ quan tâm đến Topic cần gửi và message. Cần lưu ý giá trị `Topic` khi tạo `Producer` (nên đặt trong settings, không hardcode)
``` cs
...
using DPSinfra.Kafka;

namespace TestCore5._0.Controllers
{
    [ApiController]
    [Route("[controller]")]
    public class apiController : ControllerBase
    {
	    ...
        private IProducer _producer;

        public apiController(..., IProducer producer)
        {
	        ...
            _producer = producer;
        }

        [HttpGet]
        [Route("testkafkafrom50")]
        public void testkafkafrom50()
        {
            _producer.PublishAsync("Topic ở đây", "Message ở đây");
        }
    }
}
```

### Tạo Consumer

`Consumer` là đầu nhận dữ liệu nên cần chạy luồng riêng bất đồng bộ với chương trình và luôn luôn chạy ngầm. Nên tách `consumer` ra thư mục riêng. `Consumer` cần implement interface `IHostedService` để chạy bất đồng bộ.
Cần lưu ý giá trị `Topic` và `Consumer Group` khi tạo `consumer` (nên đặt trong settings, không hardcode)

``` cs
using DPSinfra.Kafka;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Hosting;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;

namespace TestCore5._0.ConsumerServices
{
    public class testkafka : IHostedService
    {
        private IConfiguration _config;
        private Consumer testkafkaConsumer;
        public testkafka(IConfiguration config)
        {
            _config = config;
            testkafkaConsumer = new Consumer(_config, "Consumer Group ở đây");
        }
        public Task StartAsync(CancellationToken cancellationToken)
        {
            _ = Task.Run(() =>
            {
                testkafkaConsumer.SubscribeAsync("Topic ở đây", Console.WriteLine);
            }, cancellationToken);
            return Task.CompletedTask;
        }
        public async Task StopAsync(CancellationToken cancellationToken)
        {
            await testkafkaConsumer.closeAsync();
        }
    }
}
```
Trong đoạn code trên, phương thức `SubscribeAsync` có tham số thứ 2 `Console.WriteLine` chính là 1 `Delegate function` trong c#. Nếu có 1 message được `Consumer` này nhận thì `Delegate` này sẽ được gọi.
Có thể tạo nhiều `Consumer` để lấy dữ liệu từ nhiều `Topic` khác nhau.
<br />
Tiếp theo cần thêm `Consumer` vừa tạo ra vào chương trình khi khởi động. Việc này thực hiện ở file `Startup.cs`. Chỉ cần thêm 1 `Singleton service` như sau
``` cs
services.AddSingleton<IHostedService, testkafka>();
```
Trong đó `testkafka` chính là `Consumer` class đã tạo ở trên.