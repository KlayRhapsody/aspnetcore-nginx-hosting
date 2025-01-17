# aspnetcore-nginx-hosting

Host ASP.NET Core on Linux with Nginx

## 前期準備

### **使用 GCP 的 Compute Engine 服務，建立一台 VM**

* 選擇 `Ubuntu 22.04 LTS` 作業系統
* 機器類型 `e2-micro`
    - `vCPU`: `0.25 - 2 vCPU` (1 個共用核心)
    - `Memory`: `1 GB`
* 防火牆設定開啟 `HTTP` 和 `HTTPS` 流量

### **安裝 .NET Core Runtime**

```bash
wget https://dot.net/v1/dotnet-install.sh -O dotnet-install.sh

chmod +x ./dotnet-install.sh

./dotnet-install.sh --version latest --runtime aspnetcore
```

設定環境變數

* `DOTNET_ROOT`：指定 .NET 安裝目錄
* `PATH`：將 .NET CLI 加入執行路徑

```bash
echo 'export DOTNET_ROOT=$HOME/.dotnet' >> ~/.bashrc
echo 'export PATH=$PATH:$HOME/.dotnet:$HOME/.dotnet/tools' >> ~/.bashrc
source ~/.bashrc
```

驗證安裝

```bash
dotnet --list-runtimes
# Microsoft.AspNetCore.App 9.0.0 [/home/klayhung/.dotnet/shared/Microsoft.AspNetCore.App]
# Microsoft.NETCore.App 9.0.0 [/home/klayhung/.dotnet/shared/Microsoft.NETCore.App]

dotnet --info
```

### **安裝 Nginx**
    
更新 APT 套件清單

```bash
sudo apt-get update
sudo apt-get upgrade -y
```
    
安裝 HTTPS 傳輸協定
    
```bash
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
```

新增 Nginx 官方套件來源

```bash
curl -fsSL https://nginx.org/keys/nginx_signing.key | sudo apt-key add -
```

新增 Nginx 官方來源到 APT 清單

```bash
# Ubuntu 22.04 (Jammy)
echo "deb http://nginx.org/packages/ubuntu/ jammy nginx" | sudo tee /etc/apt/sources.list.d/nginx.list
```

更新 APT 套件清單

```bash
sudo apt-get update
```

安裝 Nginx

```bash
sudo apt-get install -y nginx
```

驗證 Nginx 版本

```bash
nginx -v
```

啟動 Nginx

```bash
sudo systemctl start nginx
```

驗證 Nginx 是否正在運行

```bash
sudo systemctl status nginx
```

在瀏覽器輸入伺服器的 IP 地址 或 網域名稱

IP 為 GCP VM 的外部 IP

```
http://YOUR_SERVER_IP
```

### **常用 Nginx 指令**

重新啟動 Nginx

```bash
sudo systemctl restart nginx
```

重新載入設定（不中斷服務）：

```bash
sudo nginx -t    # 測試 Nginx 設定是否正確
sudo systemctl reload nginx
```

停止 Nginx

```bash
sudo systemctl stop nginx
```

### **調整 Web Api 應用程式**

如果應用程式在開發環境中本地運行且伺服器未將其配置為建立安全 HTTPS 連接

```csharp
// 選擇關閉 HTTPS 重導向
if (!app.Environment.IsDevelopment())
{
    app.UseHttpsRedirection();
}
```

從 `applicationUrl` 屬性中刪除 `https://localhost:5001` （如果存在）


### **發布並運行應用程式**

```bash
dotnet publish --configuration Release -o ./publish
```

使用整合到組織工作流程中的工具（例如SCP 、 SFTP ）將 ASP.NET Core 應用程式複製到伺服器

在這裡使用 gcloud 從本地上傳檔案到 GCP VM

安裝 gcloud SDK

```bash
# Ubuntu / Debian 安裝
sudo apt-get install google-cloud-sdk
# 登入 GCP
gcloud auth login
# 設定預設專案
gcloud config set project YOUR_PROJECT_ID
# 查詢 GCP VM 的資訊
gcloud compute instances list
```

上傳檔案

```bash
# 範例 gcloud compute scp --recurse ./publish my-user@my-vm-name:~/app --zone=asia-east1-a

gcloud compute scp --recurse ./publish USERNAME@VM_NAME:~/app --zone=ZONE
```

連線到 GCP VM

```bash
gcloud compute ssh USERNAME@VM_NAME --zone=ZONE
```

運行應用程式

```bash
dotnet WepApi.dll
```


### **應用程式新增 Forwarded Headers Middleware**

ASP.NET Core 應用程式中，新增中間件，使用 `X-Forwarded-Proto` 標頭更新 `Request.Scheme` 以便重定向 URI 和其他安全策略正常運作

```csharp
using Microsoft.AspNetCore.HttpOverrides;
using System.Net;

var builder = WebApplication.CreateBuilder(args);

// Configure forwarded headers
builder.Services.Configure<ForwardedHeadersOptions>(options =>
{
    options.KnownProxies.Add(IPAddress.Parse("10.0.0.100"));
});

builder.Services.AddAuthentication();

var app = builder.Build();

app.UseForwardedHeaders(new ForwardedHeadersOptions
{
    ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto
});

app.UseAuthentication();

app.MapGet("/", () => "10.0.0.100");

app.Run();
```


### **設定 Nginx 反向代理**

使用單一 Nginx 實例並與 HTTP 伺服器一起運行在同一台伺服器上

設定 Nginx 反向代理，將請求轉發到 ASP.NET Core 應用程式

```nginx
map $http_connection $connection_upgrade {
  "~*Upgrade" $http_connection;
  default keep-alive;
}

server {
  listen        80;
  server_name   example.com *.example.com; # 替換成你的網域名稱
  location / {
      proxy_pass         http://127.0.0.1:5000/;
      proxy_http_version 1.1;
      proxy_set_header   Upgrade $http_upgrade;
      proxy_set_header   Connection $connection_upgrade;
      proxy_set_header   Host $host;
      proxy_cache_bypass $http_upgrade;
      proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header   X-Forwarded-Proto $scheme;
  }
}
```

建立符號連結，啟用 Nginx 的網站設定，讓 Nginx 正式使用你的設定檔

```bash
sudo mkdir /etc/nginx/sites-enabled

# 1️⃣ /etc/nginx/sites-available/
# - 存放所有網站設定檔（無論是否啟用）
# 2️⃣ /etc/nginx/sites-enabled/
# - 存放已啟用的網站設定檔（Nginx 會讀取這裡的設定）
sudo ln -s /etc/nginx/sites-available/default /etc/nginx/sites-enabled/default
# 多個設定檔可以建立多個符號連結
# default -> /etc/nginx/sites-available/default
# xxxweb -> /etc/nginx/sites-available/xxxweb

ls -l /etc/nginx/sites-enabled/
# default -> /etc/nginx/sites-available/default
```

在 `nginx.conf` 中加入 `include` 指令，讓 Nginx 讀取 `sites-enabled` 目錄下的設定檔

```nginx
# ⭐ 這行很重要！載入 sites-enabled 目錄下所有啟用的網站設定
# .NET 官網沒有提到這步驟
include /etc/nginx/sites-enabled/*;
```

測試 Nginx 設定

```bash
sudo nginx -t
# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
# nginx: configuration file /etc/nginx/nginx.conf test is successful

# 確認自定義設定是否被正確載入
sudo nginx -T
```

重新啟動 Nginx

```bash
sudo systemctl reload nginx
```


### **Debug Nginx**

查看 Access Log

```bash
sudo tail -f /var/log/nginx/access.log
```

查看 Error Log

```bash
sudo tail -f /var/log/nginx/error.log
```

### **在外部機器執行 API 查詢**

```bash
curl -X GET http://${外部IP or Domain Name}/weatherforecast
```


## Nginx 各項功能設定

### **增加 keepalive_requests**

Nginx 在 Keep-Alive 連線中，最多允許每個 TCP 連線處理 N 次 HTTP 請求

```nginx
server {
  keepalive_requests 10; # 每個連線最多允許處理 10 次請求，第 11 次會關閉連線
}
```

搭配 nginx_status 模組，可以查看 Nginx 連線狀態

```nginx
server {
    location /nginx_status {
        stub_status;           # 啟用 Nginx 狀態頁面
        allow 127.0.0.1;       # 允許本機存取
        deny all;              # 禁止其他 IP 存取
    }
}
```

範例結果

```bash
Active connections: 2 
server accepts handled requests
 10 10 20 
Reading: 0 Writing: 1 Waiting: 1
```
