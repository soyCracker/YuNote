# linux docker 入門步驟

## 1. Windows工具

### Windows Terminal/Powersell
- 安裝
https://github.com/microsoft/terminal/releases
- 遠端連線指令
```sh
ssh username@[ip] -p [port]

#預設目錄
# / 是根目錄，你可以想成是C:/
# ~ 就是你的家，可以想成是我的文件夾
# 比如~ = /home/username/

#切換目錄
cd 目的地

#輸出當前目錄
ll
ls -a

#系統管理者權限
sudo 任何指令

#複製
cp 來源 目標

#移動
mv 來源 目標

#刪除
rm xxx

#輸出
cat xxxxx

#中止指令
ctrl + C
```
- windows遠端登入linux用ssh key不用密碼的參考資料
https://blog.gtwang.org/linux/linux-ssh-public-key-authentication/
https://learn.microsoft.com/zh-tw/windows-server/administration/openssh/openssh_keymanagement
1. windows用powershell產生ssh key
2. C:\Users\username\.ssh\會出現id_rsa.pub
3. id_rsa.pub 貼到linux的~/.ssh/authorized_keys
4. 如果還是不行，authorized_keys權限改為> 600 使用者/使用者

### Visual Studio Code
#### 擴充套件
- Remote - SSH
- .NET Core Extension Pack
- Docker
- NGINX Configuration Language Support


## 2. Linux環境
### 安裝Git
https://git-scm.com/book/zh-tw/v2/%E9%96%8B%E5%A7%8B-Git-%E5%AE%89%E8%A3%9D%E6%95%99%E5%AD%B8
https://utho.com/docs/tutorial/how-to-install-git-on-centos-7/
```sh
#安裝git
sudo yum install git-all
```

設定gitlab ssh-key
https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent
https://docs.gitlab.com/ee/user/ssh.html
```sh
# 產生ssh key
ssh-keygen -t rsa -b 2048 -C "your_email@example.com"
# ssh key會在 ~/.ssh/
# id_rsa
# id_rsa.pub 這個貼到gitlab 
# https://gitlab.starlux-airlines.com/-/profile/keys
```

### 安裝 .NET SDK
https://learn.microsoft.com/zh-tw/dotnet/core/install/linux-centos

### 安裝Docker
https://docs.docker.com/engine/install/centos/

## 3. .NET 操作
```sh
#產生專案資料夾
mkdir mini
cd mini
#Minimal API
dotnet new web
#運行
dotnet run
```

- Minimal API
https://learn.microsoft.com/zh-tw/dotnet/core/tools/dotnet-new
https://blog.darkthread.net/blog/minimal-api/

## 4. Docker 操作

- 微軟net core dockerfile範例
https://learn.microsoft.com/zh-tw/dotnet/core/docker/build-container?tabs=windows
```dockerfile=
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build-env
WORKDIR /App

# Copy everything
COPY . ./
# Restore as distinct layers
RUN dotnet restore
# Build and publish a release
RUN dotnet publish -c Release -o out

# Build runtime image
FROM mcr.microsoft.com/dotnet/aspnet:6.0
WORKDIR /App
COPY --from=build-env /App/out .
ENTRYPOINT ["dotnet", "mini.dll"]
```

- 用Dockerfile產生image
```sh
docker build -t [名稱] -f Dockerfile .
``` 

- 列出docker image
```sh
sudo docker image ls
```

- 運行docker image
```sh
sudo docker run -p 7777:80 --name [image name] [image id]
```

- 列出docker container
```sh
sudo docker ps -a
```

- 暫停docker container
```sh
sudo docker stop [CONTAINER ID]
```

- 刪除docker container
```sh
sudo docker stop [CONTAINER ID]
```

https://colobu.com/2018/05/15/Stop-and-remove-all-docker-containers-and-images/