# Docker 筆記
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

用dockerfile建立image
```sh
sudo docker build -t [image name] -f [Dockerfile name] .
```

### docker nginx用的ssl key
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

### net core用的ssl key
```sh
# -ep 路徑/檔名
# -p 密碼
dotnet dev-certs https -ep ${HOME}/.aspnet/https/aspnetapp.pfx -p bpm7749
```

### nginx.conf
設定docker nginx
```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /var/run/nginx.pid;
events {
    worker_connections 1024;
}
http {
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

    upstream mini-upstream {
        server mini:80;
    }

    server {
        listen 80;
        listen 443 ssl;
        ssl_certificate /etc/ssl/certs/self.crt;
        ssl_certificate_key /etc/ssl/private/self.key;
        server_name lab34.starlux-airlines.com;
        # ~* 不區分大小寫
        # (.*)$ 取得URL其餘部分
        location ~* /internalapi/(.*)$ {
            # internalapi-upstream對應上方upstream internalapi-upstream
            # 使用 $1 變量將 URL 的捕獲部分傳遞給上游服務器 。 $is_args$args 部分用於保留原始請求中的任何查詢參數。
            proxy_pass https://internalapi-upstream/$1$is_args$args;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection keep-alive;
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location ~* /externalapi/(.*)$ {
            proxy_pass https://externalapi-upstream/$1$is_args$args;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection keep-alive;
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location ~* /backend/(.*)$ {
            proxy_pass https://backend-upstream/$1$is_args$args;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection keep-alive;
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /mr {
            proxy_pass http://mini-upstream/;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection keep-alive;
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}

```

### docker compose
- docker compose的設定文件，docker-compose.yml。
- docker compose可以理解為簡易的k8s，啟動一組docker服務的方法，通常用於本地測試，正式部署會用k8s。
```yaml
version: '1.0'

# 定義Docker服務
services:
  # 測試用的docker
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
