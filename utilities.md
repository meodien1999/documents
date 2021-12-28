# Utilities

Các phương thức hỗ trợ của thư viện <br/>
Namespace: `DPSinfra.Utils`

## JsonWebToken

Các phương thức liên quan đến JWT

### Tạo token

Sử dụng phương thức static `JsonWebToken.issueToken` để tạo token. Ví dụ:

``` cs
public string issuetoken()
        {
            var secret = _config.GetValue<string>("Jwt:internal_secret");
            var projectName = _config.GetValue<string>("KafkaConfig:ProjectName");
            var token = JsonWebToken.issueToken(new TokenClaims { projectName = projectName }, secret);
            return token;
        }
```

Giá trị `internal_secret` được lấy từ Vault, dành riêng cho token nội bộ.

``` cs
Secret<SecretData> jwtSecret = vaultClient.V1.Secrets.KeyValue.V2.ReadSecretAsync(path: "jwt", mountPoint: "kv").Result;
IDictionary<string, object> jwtData = jwtSecret.Data.Data;
string internal_secret = jwtData["internal_secret"].ToString();
Configuration["Jwt:internal_secret"] = internal_secret;
```
