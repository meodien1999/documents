# Sử dụng Vault
## Giới thiệu

`Vault` là 1 service module trong kiến trúc DPS chịu trách nhiệm lưu trữ toàn bộ các thông tin bí mật của hệ thống và các service dưới dạng mã hóa. Các dữ liệu trong Vault hiện tại được lưu ở định dạng `key-value` nằm trong 1 `path` cụ thể.
`path` và `key` sẽ được các manager tạo ra. Khi gọi `Vault`, cần có các thông tin này để lấy dữ liệu chính xác.

## Thêm Vault vào service

Việc thêm `Vault` vào service thực hiện ở file `Startup.cs`. Đầu thêm thêm 1 phương thức như sau:
``` cs
private VaultClient ConfigVault(IServiceCollection services)
        {
            var serviceConfig = Configuration.GetVaultConfig();
            return services.addVaultService(serviceConfig);
        }
```
Tiếp theo gọi hàm ở phần khai báo services:
``` cs
var vaultClient = ConfigVault(services);
```
`vaultClient` là Client đã được tạo ra. Ngoài ra thư viện `DPSinfra` cũng tự động tạo 1 `Singleton service` cho client này. Ở các controller nếu cần sử dụng `Vault`, chỉ cần inject `VaultClient` vào là được.

## Gọi Vault

Khi đã có Client, có thể gọi phương thức như sau để truy xuất `secret path` và lấy `value` từ các `key`
``` cs
Secret<SecretData> kafkaSecret = vaultClient.V1.Secrets.KeyValue.V2.ReadSecretAsync(path: "kafka", mountPoint: "kv").Result;
IDictionary<string, object> kafkaData = kafkaSecret.Data.Data;
string KafkaUser = kafkaData["username"].ToString();
string KafkaPassword = kafkaData["password"].ToString();
```
Ở phương thức `ReadSecretAsync`, tham số `path` chính là path lưu trữ secret. `username` và `password` trong `Dictionary` là các `key` trong `path`. Truy xuất đúng sẽ lấy ra được `value` tương ứng.