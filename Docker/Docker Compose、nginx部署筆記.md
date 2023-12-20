# Docker Compose、nginx部署筆記

## 目標
- 學習如何透過dockerfile建立net core docker
- 學習如何設定nginx代理伺服器docker
- 學習如何透過docker compose啟動多個docker協作，docker compose通常只在本機或開發環境使用，正式環境需要另外學習k8s

## 建置環境
### Dockerfile
以InternalAPI為例，internalAPI_Dockerfile
```dockerfile
# 指定建置環境的基礎
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build-env
# 設定工作目錄為 /app
WORKDIR /app
# 公開port
EXPOSE 80
EXPOSE 443

# 將目前目錄下InternalAPI/、APILibrary/的所有檔案複製到 /app/InternalAPI 和 /app/APILibrary
COPY ./InternalAPI ./InternalAPI
COPY ./APILibrary ./APILibrary
# Restore as distinct layers
RUN dotnet restore InternalAPI
# 建置並發佈到 /app/out
RUN dotnet publish InternalAPI -c Release -o out

# 指定執行環境image
FROM mcr.microsoft.com/dotnet/aspnet:6.0
WORKDIR /app
# 設定時區為 Asia/Taipei
ENV TZ="Asia/Taipei"
# 從建置環境複製已發行的輸出到執行環境的 /app/out 目錄下
COPY --from=build-env /app/out .
# 指定容器啟動時要執行的命令
ENTRYPOINT ["dotnet", "InternalAPI.dll"]
```

用dockerfile建立image指令，本範例image name為bpm-internalapi，不一定要按照這個檔案名稱，但要相應的更改之後的內容
```sh
sudo docker build -t [image name] -f [Dockerfile name] .
```

## 部署環境

資料夾結構，不一定要按照這個結構或檔案名稱，但要相應的更改之後的內容
```sh
#資料夾 web_publish/
docker-compose.yml
SettingConfig/
ssl-key/
nginx/

#資料夾 SettingConfig/
AppSettingsSecrets.json

#資料夾 ssl-key/
aspnetapp.pfx
self.crt
self.key

#資料夾 nginx/
sites-available/bpm.conf
nginx.conf
```

### 產生docker nginx用的ssl key(只應該在開發環境使用)
```sh
#PARENT是目標DNS url
PARENT="lab34.starlux-airlines.com"
openssl req \
-x509 \
-newkey rsa:4096 \
-sha256 \
-days 365 \
-nodes \
-keyout self.key \
-out self.crt \
-subj "/CN=${PARENT}" \
-extensions v3_ca \
-extensions v3_req \
-config <( \
  echo '[req]'; \
  echo 'default_bits= 4096'; \
  echo 'distinguished_name=req'; \
  echo 'x509_extension = v3_ca'; \
  echo 'req_extensions = v3_req'; \
  echo '[v3_req]'; \
  echo 'basicConstraints = CA:FALSE'; \
  echo 'keyUsage = nonRepudiation, digitalSignature, keyEncipherment'; \
  echo 'subjectAltName = @alt_names'; \
  echo '[ alt_names ]'; \
  echo "DNS.1 = www.${PARENT}"; \
  echo "DNS.2 = ${PARENT}"; \
  echo '[ v3_ca ]'; \
  echo 'subjectKeyIdentifier=hash'; \
  echo 'authorityKeyIdentifier=keyid:always,issuer'; \
  echo 'basicConstraints = critical, CA:TRUE, pathlen:0'; \
  echo 'keyUsage = critical, cRLSign, keyCertSign'; \
  echo 'extendedKeyUsage = serverAuth, clientAuth')

openssl x509 -noout -text -in self.crt
```

### 產生net core用的ssl key(只應該在開發環境使用)
```sh
# -ep 路徑/檔名
# -p 密碼，可自訂，但需與docker-compose.yml內容一致
dotnet dev-certs https -ep ${HOME}/.aspnet/https/aspnetapp.pfx -p bpm7749
```

### nginx.conf
設定docker nginx
```sh
user nginx;
worker_processes auto; # 請設定與 CPU 核心數相等的數字，進程切換代價最小，auto 為系統自動偵測 CPU 核心數，預設為 1

error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;


events {
    worker_connections 1024;
    multi_accept on; # 儘可能接受最多的連接數
    use epoll;
}

http {
    server_tokens off; # 隱藏 Nginx 版本資訊
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    client_max_body_size 10M; # 上傳檔案大小限制(10MB)

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
    ssl_ecdh_curve secp384r1;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
    '$status $body_bytes_sent "$http_referer" '
    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;

    keepalive_timeout 30s;
    client_header_timeout 30s;
    client_body_timeout 30s;
    reset_timedout_connection on;
    send_timeout 30s;

    # Gzip 設置
    gzip on;
    gzip_proxied any;
    gzip_buffers 16 8k;
    gzip_disable "MSIE [1-6]\.(?!.*SV1)";
    gzip_http_version 1.1;
    gzip_vary on; # 增加回應標頭 "Vary: Accept-Encoding"
    gzip_comp_level 6; # 壓縮比1~9, 數字越高壓縮程度越大但越耗 CPU 效能
    gzip_min_length 1024; # 超過 1024k 才壓縮, 預設是 0 全壓縮
    gzip_types text/plain text/css application/javascript application/x-javascript text/javascript application/json application/xml application/xml+rss text/xml;

    # 根據 $http_connection 來決定變數 $connection_upgrade 的值(SignalR 會用到)
    map $http_connection $connection_upgrade {
        "~*Upgrade" $http_connection;
        default keep-alive;
    }
    proxy_buffering off;
    proxy_set_header Host $host;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Server $host;
    proxy_set_header X-Forwarded-Host $host:$server_port;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_cache_bypass $http_upgrade;
    proxy_http_version 1.1;

    upstream internalapi-upstream {
        # 這裡的internalapi-service對應docker-compose.yml中internalapi-service服務的名稱
        server internalapi-service:443;
    }

    upstream externalapi-upstream {
        server externalapi-service:443;
    }

    upstream backend-upstream {
        server backend-service:443;
    }

    include /etc/nginx/conf.d/*.conf;
    # 加入這行包含sites-available底下所有conf
    include /etc/nginx/sites-available/*.conf; 
}
```

### bpm.conf
放在nginx/sites-available/bpm.conf，會對應到docker nginx的/etc/nginx/sites-available/*.conf
```sh
server {
    listen 80;
    listen 443 ssl;
    ssl_certificate /etc/ssl/certs/self.crt;
    ssl_certificate_key /etc/ssl/private/self.key;
    server_name lab34.starlux-airlines.com;

    # ~* 不區分大小寫
    location ~* /internalapi {
        # internalapi-upstream對應上方upstream internalapi-upstream
        proxy_pass https://internalapi-upstream;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location ~* /externalapi {
        proxy_pass https://externalapi-upstream;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    #web mvc須配合app.UsePathBase("/backend");
    location ~* /backend {
        proxy_pass https://backend-upstream;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### docker compose
- docker compose的設定文件，docker-compose.yml，該範例會啟動nginx作為proxy server，以及InternalAPI、ExternalAPI、BackEnd。
- docker compose可以理解為簡易的k8s，啟動一組docker服務的方法，通常用於本地測試，正式部署會用k8s。
```yaml=
version: '1.0'

# 定義Docker服務
services:
  # mini測試用的docker，可略過
  mini:
    image: netcore-mini-docker
    ports:
      - "8700:80"

  # 定義名為"internalapi-service"的服務
  internalapi-service:
    # 使用名為"bpm-internalapi"的Docker映像
    image: bpm-internalapi
    # 將主機端口8701和8702映射到容器端口443和80
    ports:
      - "8701:443"
      - "8702:80"
    # 定義環境變數
    environment:
      # 設定net core的ssl
      - ASPNETCORE_URLS=https://+:443;http://+:80
      - ASPNETCORE_Kestrel__Certificates__Default__Password=bpm7749
      - ASPNETCORE_Kestrel__Certificates__Default__Path=/https/aspnetapp.pfx
    # 將本地檔案映射到容器中
    volumes:
      - ./SettingConfig/AppSettingsSecrets.json:/SettingConfig/AppSettingsSecrets.json
      - ./ssl-key/aspnetapp.pfx:/https/aspnetapp.pfx

  externalapi-service:
    image: bpm-externalapi
    ports:
      - "8703:443"
      - "8704:80"
    environment:
      - ASPNETCORE_URLS=https://+:443;http://+:80
      - ASPNETCORE_Kestrel__Certificates__Default__Password=bpm7749
      - ASPNETCORE_Kestrel__Certificates__Default__Path=/https/aspnetapp.pfx
    volumes:
      - ./SettingConfig/AppSettingsSecrets.json:/SettingConfig/AppSettingsSecrets.json
      - ./ssl-key/aspnetapp.pfx:/https/aspnetapp.pfx
  
  backend-service:
    image: bpm-backend
    ports:
      - "8705:443"
      - "8706:80"
    environment:
      - ASPNETCORE_URLS=https://+:443;http://+:80
      - ASPNETCORE_Kestrel__Certificates__Default__Password=bpm7749
      - ASPNETCORE_Kestrel__Certificates__Default__Path=/https/aspnetapp.pfx
    volumes:
      - ./SettingConfig/AppSettingsSecrets.json:/SettingConfig/AppSettingsSecrets.json
      - ./ssl-key/aspnetapp.pfx:/https/aspnetapp.pfx

  nginx:
    image: nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl-key/self.crt:/etc/ssl/certs/self.crt
      - ./ssl-key/self.key:/etc/ssl/private/self.key
    # 定義依賴關係，確保在啟動nginx之前先啟動其他服務
    depends_on:
      - internalapi-service
      - externalapi-service
      - backend-service
      - mini
```
啟動docker compose
```sh
sudo docker compose up
```