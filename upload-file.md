# UploadFile
## Giới thiệu
Namespace: `DPSinfra.UploadFile`
Dùng để upload file và image, để sử dụng cần cài phiên bản mới nhất
## Cài đặt chung
Trong file `Startup.cs` phần `ConfigureServices` thêm vào đoạn code này để lấy access_key và secret_key từ vault
```cs
Secret<SecretData> minioSecret = vaultClient.V1.Secrets.KeyValue.V2.ReadSecretAsync(path: "minio", mountPoint: "kv").Result;
IDictionary<string, object> minioData = minioSecret.Data.Data;
Configuration["MinioConfig:MinioAccessKey"] = minioData["access_key"].ToString();
Configuration["MinioConfig:MinioSecretKey"] = minioData["secret_key"].ToString();
```
cài đặt trong appsettings.Development.json
```json
MinioConfig{ 
    "MinioSSL": "true",
    "MinioServer": "cdn.jee.vn"
  },
```

## Cách sử dụng 
Trong 1 controller bất kỳ, muốn sử dụng cần inject private IConfiguration vào nội hàm:
```cs
public class apiController : ControllerBase
{
	private IConfiguration _config;
    public apiController(IConfiguration configuration...)
    {
	       _config = configuration;
    }
}
```

```cs
// class đã có trong DPSinfra.UploadFile không tạo mới
public class upLoadFileModel
{
	 public byte[] bs { get; set; }
     public string Linkfile { get; set; }
     public string FileName { get; set; }
}
```

Gọi hàm sử dụng
```cs
upLoadFileModel up = new upLoadFileModel()
{
      bs = Convert.FromBase64String(Base64), //Convert sang dạng byte
      FileName = FileName, // ví dụ test.docx
      Linkfile = "File" // Folder chứ File ví dụ File
};
UploadFile.UploadFileMinio(up, _config);
//_config khai báo ở hàm khởi tạo
```
**Sử dụng các phương thức static để uploadFile**
```cs 
UploadFile.UploadFileImageMinio(upLoadFileModel, _config);
//upload Image
```
```cs
UploadFile.UploadFileAllTypeMinio(upLoadFileModel, _config, Contenttype);
//upload tất cả File - Contenttype là kiểu dữ liệu của file (ví dụ: "image/jpg" or "image/" or "application/")
//có thể để null nếu ko xác định được Contenttype 
UploadFile.UploadFileAllTypeMinio(upLoadFileModel, _config);
```
```cs
UploadFile.UploadFileMinio(upLoadFileModel, _config);
//upload File
```
```cs
UploadFile.RemoveImageMinio(FileName, _config);
// xóa File: FileName là link trả về ví dụ: /File/test.docx
```
Model kết quả trả về
```cs
// class đã có trong DPSinfra.UploadFile không tạo mới
public class UploadResult
{
    public bool status { get; set; }
    public string link { get; set; }
} 
//kết quả trả về sẽ là  
{
  "status": true,
  "link": "project-name/File/test.docx"
}
```
**note** Để hiển thị được hình ảnh các app phải tự cộng chuổi:
MinioServer + link
Ví dụ: 
```cs
"https://cdn.jee.vn" + "project-name/File/test.docx"
// MinioServer lấy từ appsetting
```
**Để hiển thị được hình ảnh lên giao diện cần phải đc cấp quyền cho Bucket**

    

