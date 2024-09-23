### 使用EF Core加入資料庫的模型

#### 安裝nuget
- Microsoft.EntityFrameworkCore.SqlServer
- Microsoft.EntityFrameworkCore.Design
- Microsoft.EntityFrameworkCore.Tools


#### 指令
```
Scaffold-DbContext "data Source=192.168.2.210;Initial Catalog=Management_test;TrustServerCertificate=true;persist security info=True;User ID=yuhui;password=wd@@20240906;MultipleActiveResultSets=True;" Microsoft.EntityFrameworkCore.SqlServer -OutputDir "輸出資料夾" -Tables "table名稱"
```