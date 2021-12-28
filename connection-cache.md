
# Connection Cache
## Giới thiệu
Namespace: `DPSinfra.ConnectionCache`
Vì mỗi CustomerID sẽ có 1 DB khác nhau và hàm GetConnectionString sẽ trả về chuổi connection string theo CustomerID và AppCode của từng app
**để sử dụng cần cài phiên bản mới nhất**
## Cài đặt chung
Trong file `Startup.cs` phần `ConfigureServices` thêm vào đoạn code này để lấy access_secret từ vault.
```cs
Secret<SecretData> jwtSecret = vaultClient.V1.Secrets.KeyValue.V2.ReadSecretAsync(path: "jwt", mountPoint: "kv").Result;
IDictionary<string, object> jwtData = jwtSecret.Data.Data;
Configuration["Jwt:internal_secret"] = jwtData["internal_secret"].ToString();
```
Cũng trong file `Startup.cs` phần `ConfigureServices` thêm 2 service bên dưới.
```cs
services.AddMemoryCache(); 
services.addConnectionCacheService();
```
cài đặt trong appsettings.Development.json
```json
{
  "KafkaConfig": {
    "Brokers": "jee.vn:9093,jee.vn:9094,jee.vn:9095",
    "ProjectName": "ten-project" //Viết ở dạng kebab-case
  },
  "VaultConfig": {
    "Endpoint": "https://vault.jee.vn",
    "Token": "s.CnO0sRfOCIvMAYcSnoceE1A5"
  }, 
  "Host": {
    "JeeAccount_API": "https://jeeaccount-api.jee.vn"  // cần khai báo để gọi đến JeeAccount lấy dữ liệu của khách hàng
  },
 "AppConfig": {
    "AppCode": "REQ", // app code của từng app
    "DecryptKey": "REQ" // key giải mã của từng app
  }
}
```
## Cách sử dụng

Trong 1 controller bất kỳ, muốn sử dụng cần inject private IConnectionCache vào nội hàm:
```cs
public class apiController : ControllerBase
{
	private IConnectionCache _cache;
    public apiController(IConnectionCache connectionCache...)
    {
	       _cache = connectionCache;
    }
}
```
**Gọi hàm sử dụng để lấy connection string**

Gọi hàm sử dụng
```cs
_cache.GetConnectionString(CustomerID);
// CustomerID truyền customerID vào lấy từ data trong jwt
```

Chuổi connectionString nhận được:
```cs
// Ví dụ
Data Source=.;Initial Catalog=JeeRequest;User ID=sa;Password=123
```



