Viewed README.md:18-46

在 .NET (ASP.NET Core) 的開發中，**EntityFramework Core (EF Core)** 是官方推薦的 ORM 框架，它能讓您用物件導向的 C# 語法來直接操作資料庫，而不需手寫 SQL 指令。

以下結合您專案的實際架構，為您詳細說明 EF Core 的**資料庫連線配置**以及**新增、修改、刪除、查詢**的實作方式：

---

## 1. 資料庫連線配置方式

在專案中，連線的配置主要是透過 **設定檔 (JSON)** ➔ **啟動引導 (Startup)** ➔ **資料庫上下文 (DbContext)** 三個步驟串接而成：

### 步驟 A：在 `appsettings.json` 設定連線字串 (ConnectionString)
在設定檔中，配置了 SQL Server 的連線參數（如主機 IP、資料庫名稱、帳號與密碼）：
```json
"ConnectionStrings": {
  "DefaultConnection": "Data Source={IP};Initial Catalog={DB};User ID={ID};Password={PWD};TrustServerCertificate=true;Encrypt=yes;"
}
```

### 步驟 B：在 `Program.cs` 註冊 DbContext 服務
程式啟動時，會從設定檔讀取 `DefaultConnection`，並將資料庫上下文服務注入到 ASP.NET Core 的相依性注入 (DI) 容器中：
```csharp
// 讀取連線字串
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");

// 註冊 DbContext，並指定使用 SQL Server 驅動
builder.Services.AddDbContext<testDbContext>(options =>
    options.UseSqlServer(connectionString));
```

### 步驟 C：定義 `DbContext.cs`
這是資料庫的模型映射中心，每個 `DbSet<T>` 都代表資料庫中的一張資料表：
```csharp
public class testDbContext : DbContext
{
    public testDbContext(DbContextOptions<testDbContext> options)
        : base(options)
    {
    }

    // 註冊資料表與實體模型的映射關係
    public DbSet<SystemUser> USERS { get; set; } = null!;
    public DbSet<Template> TEMPLETE { get; set; } = null!;
    public DbSet<CreateRecord> CREATE_RECORD { get; set; } = null!;
}
```

---

## 2. 新增、修改、刪除、查詢資料庫 (CRUD) 的操作語法

在 Controller 中，您可以透過建構函式注入 `testDbContext`（通常命名為 `_context`），並以 C# 語法進行操作：

### 🔍 查詢 (Retrieve / Read)
EF Core 提供了豐富的 LINQ 語法，支援非同步（Async）查詢，避免阻塞系統：

```csharp
// 1. 查詢全部使用者，並依帳號排序 (回傳 List)
var allUsers = await _context.USERS.OrderBy(u => u.USERNAME).ToListAsync();

// 2. 依據主鍵 (PrimaryKey) 快速查詢單一使用者
var user = await _context.USERS.FindAsync(username);

// 3. 使用條件篩選查詢特定使用者 (若找不到會回傳 null)
var activeAdmin = await _context.USERS
    .FirstOrDefaultAsync(u => u.ROLE == "Admin" && u.STATUS == 1);
```

---

### ➕ 新增 (Create)
新增資料的步驟是：**建立實體物件** ➔ **使用 `Add` 加入追蹤** ➔ **呼叫 `SaveChangesAsync()` 寫入資料庫**：

```csharp
public async Task<IActionResult> CreateUser(SystemUser newUser)
{
    // 1. 實例化模型資料 (若是由前端傳入則不需手動 new)
    newUser.CREATE_TIME = DateTime.Now;

    // 2. 將新物件加入 DB 上下文的追蹤區
    _context.USERS.Add(newUser);

    // 3. 將變更存回實體 SQL 資料庫 (此時才會真正執行 INSERT INTO SQL 指令)
    await _context.SaveChangesAsync();
    
    return Ok("新增成功");
}
```

---

### ✏️ 修改 / 更新 (Update)
EF Core 擁有強大的 **Change Tracker (變更追蹤機制)**。修改資料最推薦且安全的方式是：**先查出來** ➔ **修改物件屬性** ➔ **存檔**：

```csharp
public async Task<IActionResult> UpdateUser(SystemUser userUpdatePayload)
{
    // 1. 先從資料庫查出目前已存在的舊資料
    var existingUser = await _context.USERS.FindAsync(userUpdatePayload.USERNAME);
    if (existingUser == null)
    {
        return NotFound("找不到該使用者");
    }

    // 2. 直接修改該物件的屬性欄位 (EF Core 會自動偵測到哪些欄位被變更了)
    existingUser.ROLE = userUpdatePayload.ROLE;
    existingUser.STATUS = userUpdatePayload.STATUS;

    // 3. 儲存變更 (EF Core 會自動產生對應欄位的 UPDATE SQL 指令)
    await _context.SaveChangesAsync();
    
    return Ok("更新成功");
}
```
> **💡 小撇步：** 只要查出來的物件在同一個 `_context` 生命週期內，您對該物件所做的任何 C# 屬性修改，在最後呼叫 `SaveChangesAsync()` 時都會自動更新到資料庫，不需額外呼叫 `_context.USERS.Update(existingUser)`。

---

### ❌ 刪除 (Delete)
刪除資料的步驟是：**查出該筆資料** ➔ **使用 `Remove` 標記為刪除** ➔ **呼叫 `SaveChangesAsync()` 寫入資料庫**：

```csharp
public async Task<IActionResult> DeleteUser(string username)
{
    // 1. 找出要刪除的資料
    var user = await _context.USERS.FindAsync(username);
    if (user == null)
    {
        return NotFound("找不到該使用者，無法刪除");
    }

    // 2. 從追蹤中移除該物件
    _context.USERS.Remove(user);

    // 3. 儲存變更 (執行 DELETE FROM SQL 指令)
    await _context.SaveChangesAsync();
    
    return Ok("刪除成功");
}
```

---

## 3. EF Core 運作的核心關鍵：`SaveChangesAsync()`

在 EF Core 中，當您執行 `Add`、`Remove` 或修改欄位時，這些變更都只存在於**主機的記憶體快取**中。
只有在最後呼叫 **`await _context.SaveChangesAsync();`** 時，EF Core 才會啟動資料庫交易 (Transaction)，並把所有累積的變更一次打包，生成對應的 SQL 陳述式發送給 SQL Server。這具有以下好處：
1. **交易保護**：如果有多個表格一起新增或修改，若其中一個寫入失敗，整批操作會自動 Rollback (復原)，保證資料一致性。
2. **效能優化**：減少了與資料庫來回進行網路連線通信的次數。


在 EF Core 中，當 LINQ 語法難以表達某些過於複雜的查詢（例如：多層子查詢、複雜的 `JOIN`、Window Functions，或需要呼叫**預存程序 Stored Procedure**）時，您可以使用 **Raw SQL (原生 SQL)**。

EF Core 提供了以下三種主要方式來執行複雜的 SQL，您可以根據「**回傳的資料結構**」來選擇：

---

### 方法一：`FromSql` 查詢（適用於：結果能對應到現有的資料表 Model）

如果您的複雜 SQL 最終回傳的欄位**可以完整映射回專案中的某個實體（如 `Template` 或 `SystemUser`）**，可以使用 `FromSql` 系列方法。

> **⚠️ 重要安全守則：** 絕對不要用字串拼接 SQL 以防 SQL 注入 (SQL Injection) 攻擊。請一律使用 **`FromSqlInterpolated`**（它會自動將 C# 變數轉為資料庫參數）。

#### 範例：使用字串插值執行複雜 SELECT
```csharp
int minStatus = 1;
string searchKeyword = "%卡片%";

// EF Core 會自動將 {minStatus} 與 {searchKeyword} 參數化為 @p0 與 @p1
var templates = await _context.tbmTEMPLETE
    .FromSqlInterpolated(@$"
        SELECT t.* 
        FROM TEMPLETE t
        INNER JOIN CATEGORY c ON t.TYPE_NO = c.TYPE_NO
        WHERE t.STATUS >= {minStatus} 
          AND (t.TITLE LIKE {searchKeyword} OR t.CONTENT LIKE {searchKeyword})
    ")
    .AsNoTracking() // 若只讀不改，加上此行可大幅優化效能
    .ToListAsync();
```

---

### 方法二：`ExecuteSql` 執行（適用於：INSERT, UPDATE, DELETE 或執行預存程序）

如果您不需要回傳查詢結果，而是要執行**非查詢指令（Non-Query）**，例如大批修改資料、呼叫沒有回傳資料表的預存程序，可透過 `_context.Database` 來執行：

#### 範例 1：大批更新或刪除資料
```csharp
int targetStatus = 0;
DateTime cutoffDate = DateTime.Now.AddDays(-180);

// 回傳受影響的資料筆數 (Affected Rows)
int affectedRows = await _context.Database.ExecuteSqlInterpolatedAsync(@$"
    UPDATE USERS 
    SET STATUS = {targetStatus} 
    WHERE LAST_LOGIN < {cutoffDate} OR LAST_LOGIN IS NULL
");
```

#### 範例 2：呼叫帶有參數的預存程序
```csharp
string newRole = "Editor";
string userToPromote = "DesignerA";

await _context.Database.ExecuteSqlInterpolatedAsync(@$"
    EXEC sp_PromoteUser @UserName={userToPromote}, @Role={newRole}
");
```

---

### 方法三：`Database.GetDbConnection()` + Dapper（適用於：回傳任意自訂欄位、暫時性報表資料）

如果您的 SQL 極度複雜（例如做了很多 `GROUP BY` 與 `SUM`，只打算回傳如 `名稱, 總次數` 這種臨時的統計資料結構），因為它**無法對應至現有的 DbSet 實體模型**，強行用 EF Core 會非常麻煩。

此時最推薦的業界做法是：**直接使用 Dapper (微型 ORM)**，它可以完美地與 EF Core 共用同一個資料庫連線：

#### 步驟 1：安裝 NuGet 套件
在專案中加入 [Dapper](https://www.nuget.org/packages/Dapper) 套件。

#### 步驟 2：利用 EF 連線進行 Dapper 查詢
```csharp
using Dapper;
using Microsoft.EntityFrameworkCore;

public async Task<List<UserStatsDto>> GetUserGenerationStatsAsync()
{
    // 1. 取得 EF Core 當前正在使用的底層資料庫連線
    var dbConnection = _context.Database.GetDbConnection();

    // 2. 撰寫複雜的 SQL，並宣告一個對應欄位名稱的 C# DTO 類別 (UserStatsDto)
    string sql = @"
        SELECT 
            u.USERNAME AS UserName,
            COUNT(r.RECORD_NO) AS TotalGenerated,
            MAX(r.CREATE_TIME) AS LastGeneratedTime
        FROM USERS u
        LEFT JOIN CREATE_RECORD r ON u.USERNAME = r.CREATE_NAME
        GROUP BY u.USERNAME
        ORDER BY TotalGenerated DESC";

    // 3. 直接使用 Dapper 的 QueryAsync 進行映射
    var stats = await dbConnection.QueryAsync<UserStatsDto>(sql);

    return stats.ToList();
}

// 用於接收 SQL 結果的暫時性資料傳輸物件 (DTO)
public class UserStatsDto
{
    public string UserName { get; set; } = "";
    public int TotalGenerated { get; set; }
    public DateTime? LastGeneratedTime { get; set; }
}
```

### 🎯 總結建議
- **一般 CRUD / 簡單篩選**：一律使用 **LINQ**（如 `_context.tbmUSERS.Where(...)`）。
- **複雜查詢，但格式仍為單一資料表**：使用 **`FromSqlInterpolated`**。
- **批次更新、呼叫預存程序**：使用 **`ExecuteSqlInterpolatedAsync`**。
- **報表、多表 JOIN 後只取少數欄位、臨時統計結果**：使用 **Dapper** (搭配 EF Core 的 `GetDbConnection()`)。