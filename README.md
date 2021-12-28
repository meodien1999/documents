# Configs ứng dụng
## Phiên bản hỗ trợ

Các hướng dẫn trong phần này chỉ sử dụng được trên `.NET core` framework.
Các hướng dẫn đã được test trên .NET core `3.1` và `5.0`.

<!-- [Download Code mẫu](https://www.dropbox.com/s/txfpyfd8z03x5zc/DPSinfra.zip?dl=0) -->

## Thư viện hỗ trợ

Để sử dụng các công cụ và tích hợp service vào kiến trúc DPS cần cài đặt các thư viện sau:

 - DPSinfra  (Lệnh cài đặt: Install-Package DPSinfra -Version 1.6.0)

## Develop

File `appsettings.Development.json` sẽ quản lý các config app trên môi trường dev.
Các config bắt buộc cần có:

``` json
{
  "Logging": {
    "LogLevel": {
      "Default": "Trace",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "KafkaConfig": {
    "Brokers": "jee.vn:9093,jee.vn:9094,jee.vn:9095",
    "ProjectName": "ten-project" //Viết ở dạng kebab-case
  },
  "VaultConfig": {
    "Endpoint": "https://vault.jee.vn",
    "Token": "s.F2Ds4Bv6q1KxIO6uk58k9Tfu"
  }
}
```

## Production


File `appsettings.json` sẽ quản lý các config app trên môi trường production.
Các config bắt buộc cần có:

``` json
{
  "Logging": {
    "LogLevel": {
      "Default": "Trace",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "AllowedHosts": "*"
}
```
