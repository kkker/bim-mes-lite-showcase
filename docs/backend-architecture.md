# 後端架構與資料表

本文給接手後端的開發者，用來確認細節、追蹤問題與除錯。內容包括：後端的分層與請求管線、資料表與欄位，以及每支 API 讀寫哪些資料表。

文中的路徑都在私有的原始碼 repo（`backend/`），不在本 repo。內容對應 2026-11-21 的程式（Migration 到 `AddUsersUnlockPermission`）。

## 分層

後端是 ASP.NET Core Web API（.NET 8），採 MVC 中的 Controller 部分，沒有 View。

```text
HTTP 請求
  │
  ▼
中介軟體管線（Program.cs）  安全標頭、轉發標頭、例外處理、驗證、頻率限制、授權、/uploads 靜態檔
  │
  ▼
Controllers/               路由、輸入檢查、權限標註（[Authorize(Policy = ...)]）、組回應 DTO
  │       │
  │       ▼
  │   Services/            IFC 解析、檔案儲存、JWT 簽發與驗證、權限載入、角色權重規則
  │       │
  ▼       ▼
AppDbContext（EF Core）     專案、構件與 Identity 資料表
  │
  ▼
SQLite（單一檔案）＋ 上傳目錄（原始 .ifc 檔）
```

- **沒有 Repository 層**：Controller 直接注入 `AppDbContext` 查詢與寫入。帳號相關的寫入改用 Identity 的 `UserManager`／`RoleManager`，它們底層也是同一個 `AppDbContext`。
- **DTO 與資料表分開**：`Dtos/` 是 API 的請求與回應模型，資料表對應的是 `Models/` 與 Identity 的型別，兩者不共用。
- **設定**：`Options/` 是強型別設定，數值來自 `appsettings.json` 或環境變數。必要的值缺少或不合理時，後端在啟動時失敗（`ValidateOnStart`）。

### 主要目錄

| 目錄 | 內容 |
|---|---|
| `Controllers/` | `AuthController`、`ProjectsController`、`ElementsController`、`SettingsController`、`UsersController`、`UserUnlockController`、`RolesController`。`WeatherForecastController` 是範本留下來的，只供測試「未標註的 API 預設要登入」，已從 Swagger 隱藏 |
| `Models/` | `BimProject`、`BimElement`、`ProductionStatus`（列舉） |
| `Data/` | `AppDbContext`，繼承 `IdentityDbContext<IdentityUser, AppRole, string>` |
| `Auth/` | `AppRole`（角色加權重）、`Roles`（三種角色的種子資料）、`Permissions`（權限常數）、`IdentityBootstrapper`（啟動時建立角色與首位管理者） |
| `Services/Ifc/` | `IIfcParsingService` → `XbimIfcParsingService`：解析 `.ifc`，取出牆、梁、柱、樓板 |
| `Services/Storage/` | `IFileStorageService` → `LocalFileStorageService`：把原始檔存到上傳目錄 |
| `Services/Auth/` | `JwtTokenService`（簽發）、`JwtTokenValidation`（每次請求驗證）、`UserAccessService`（載入權限）、`RoleWeightRules`（權重規則）、`UserWeightLookup`（查使用者、角色與權重）、`JwtSigningKey` |
| `Validation/` | `IfcGlobalId`（22 字元格式）、`StepFileSignature`（檔頭 `ISO-10303-21;`） |
| `Middleware/` | `SecurityHeadersMiddleware` |
| `RateLimiting/` | 上傳、登入、改密碼三種頻率限制的分區鍵 |
| `Migrations/` | EF Core Migration |

## 請求管線

順序寫在 `Program.cs`，由上到下執行：

| 順序 | 中介軟體 | 作用 | 何時啟用 |
|---|---|---|---|
| 1 | `SecurityHeadersMiddleware` | 所有回應加上 `nosniff`、`X-Frame-Options: DENY`、`Referrer-Policy: no-referrer`（包含 401、404、413、429、500） | 一律 |
| 2 | `UseForwardedHeaders` | 從 `X-Forwarded-For` 還原用戶端 IP，只信任一層 | `ReverseProxy:TrustForwardedFor=true`（只有正式版 compose 開啟） |
| 3 | `UseExceptionHandler` | 未處理的例外改回 ProblemDetails，不含堆疊追蹤 | 非 Development |
| 4 | `UseSwagger`、`UseSwaggerUI` | API 文件，可按 Authorize 帶 JWT | 只有 Development |
| 5 | `UseHttpsRedirection` | 框架預設。後端只開 HTTP 埠，實際上不會轉址；HTTPS 由前方的反向代理負責 | 一律 |
| 6 | `UseAuthentication` | 驗證 JWT，並在 `OnTokenValidated` 查資料庫（見下方「驗證與權限的資料流」） | 一律 |
| 7 | `UseRateLimiter` | 上傳、登入、改密碼三個 policy，超過回 429 ProblemDetails 並帶 `Retry-After` | 只作用在標了 `[EnableRateLimiting]` 的 API |
| 8 | `UseAuthorization` | `FallbackPolicy` 要求登入；每個權限各有一個同名 policy | 一律 |
| 9 | `/uploads` 權限檢查（`UseWhen` 分支） | 靜態檔不是 endpoint，另外檢查：未登入 401、沒有 `projects.view` 403 | 路徑以 `/uploads` 開頭時 |
| 10 | `UseStaticFiles`（`/uploads`） | 從上傳目錄提供檔案，只認 `.ifc` | 路徑以 `/uploads` 開頭時 |
| 11 | `MapControllers` | 交給 Controller | 一律 |

驗證排在頻率限制之前，所以上傳與改密碼的頻率限制能依登入者分區。上傳與改密碼依使用者計次（`user:<使用者 Id>`），無法辨識登入者時依 IP（`ip:<位址>`）；登入一律依 IP。

上傳 API 另有一個 resource filter（`UploadSizeLimitFilter`），把超過大小上限的例外轉成 413 ProblemDetails。

### 錯誤回應從哪裡來

| 狀態碼 | 產生位置 | `type` |
|---|---|---|
| 400 | Controller 的輸入檢查，或 `[ApiController]` 的模型驗證 | 依情況，例如 `urn:bim-mes:not-step-file`、`urn:bim-mes:weak-password` |
| 401 | `JwtBearerEvents.OnChallenge`（`Program.cs`）；登入失敗由 `AuthController` 回 | `urn:bim-mes:unauthorized`；登入失敗為 `urn:bim-mes:invalid-credentials` |
| 403 | `JwtBearerEvents.OnForbidden`；權重規則不符由 Controller 回 | `urn:bim-mes:forbidden`；`urn:bim-mes:role-weight-forbidden` |
| 409 | Controller（GlobalId 已存在、帳號名稱重複、對已停用帳號解除鎖定） | 依情況，例如 `urn:bim-mes:username-taken` |
| 413 | `UploadSizeLimitFilter` | `urn:bim-mes:file-too-large` |
| 429 | `AddRateLimiter` 的 `OnRejected` | `urn:bim-mes:rate-limited` |
| 500 | `UseExceptionHandler`（非 Development） | — |

## 啟動流程

1. 註冊服務，並驗證所有強型別設定（`ValidateOnStart`）。
2. `Database.Migrate()`：套用尚未執行的 Migration。資料庫檔不存在時會建立。
3. `IdentityBootstrapper.RunAsync()`：
   - 三種角色不存在就建立，並寫入預設權限。**已存在的角色不會被改回預設值**，保留日後的調整。
   - `AspNetUsers` 沒有任何資料、且環境變數有首位管理者的帳密時，建立該使用者，指派 `Admin` 角色，並給他使用者層級的 `users.manage-peers` 權限。log 會出現 `Bootstrap admin '<名稱>' created`。
   - 已經有使用者時，忽略這兩個環境變數。
4. 取得 JWT 簽章金鑰：非 Development 缺少或太短時，在這一步失敗；Development 缺少時產生隨機金鑰並記錄警告。

## 資料庫

- **SQLite**，連線字串 `ConnectionStrings:DefaultConnection`。
  - 開發：`backend/bim_mes.db`（不進 git）。
  - 正式版：容器內的 `/data/bim_mes.db`，放在具名 volume `bim_data`。
- **上傳目錄** `Storage:UploadsPath`：開發為 `backend/wwwroot/uploads/`，正式版為 `/data/uploads`（同一個 volume）。
- **Migration**（`backend/Migrations/`，依時間順序）：

| Migration | 內容 |
|---|---|
| `InitialCreate` | `BimProjects`、`BimElements` |
| `AddIdentity` | Identity 的 7 張資料表，`AspNetRoles` 多一個 `Weight` 欄位。不改前兩張資料表 |
| `AddUsersUnlockPermission` | 只寫資料：已有的 `Admin` 角色補上 `users.unlock` 權限。新資料庫在這一步還沒有角色，不寫入任何資料，由啟動時的 bootstrapper 建立 |

### 關聯

```text
BimProjects 1 ──< BimElements                    （ProjectId，刪除專案時串聯刪除）

AspNetUsers 1 ──< AspNetUserRoles >── 1 AspNetRoles
     │                                       │
     └──< AspNetUserClaims                   └──< AspNetRoleClaims
          （使用者層級權限）                        （角色的權限）

AspNetUsers 1 ──< AspNetUserLogins、AspNetUserTokens（未使用）
```

BIM 資料表與帳號資料表之間**沒有關聯**：構件沒有記錄是誰上傳或誰改了狀態。

### 型別的對應

SQLite 只有少數儲存型別，EF Core 的對應如下，查資料時要注意：

| C# 型別 | SQLite 儲存 | 說明 |
|---|---|---|
| `int`、`enum`、`bool` | `INTEGER` | `bool` 為 0／1 |
| `string` | `TEXT` | |
| `decimal` | `TEXT` | 例如 `50000.0`。資料庫端不能對它做數值的 SUM，專案總成本在程式中計算 |
| `DateTime` | `TEXT` | 例如 `2026-11-20 07:12:34.5678901`。程式一律寫入 UTC，但字串本身不帶時區 |
| `DateTimeOffset` | `TEXT` | 帶時區，例如 `2026-11-20 07:27:34.1234567+00:00` |

## 資料表與欄位

### BimProjects：專案

每次成功上傳一個模型就新增一列。

| 欄位 | 型別 | 必填 | 說明 |
|---|---|---|---|
| `Id` | INTEGER | 是 | 主鍵，自動遞增。前端網址的 `?projectId=` 就是它 |
| `ProjectName` | TEXT | 是 | 上傳時輸入的名稱，去除前後空白，長度上限為 `Upload:MaxProjectNameLength`（預設 100）。可以重複 |
| `StorageKey` | TEXT | 是 | 上傳目錄中的檔名，格式 `{32 位十六進位 GUID}.ifc`，由伺服器產生。下載網址為 `/uploads/{StorageKey}` |
| `UploadedAt` | TEXT（DateTime） | 是 | 上傳時間（UTC） |

- 沒有刪除專案的 API。
- 早期（T43 之前）上傳的檔案，`StorageKey` 是 `{GUID}_{原始檔名}`，為了相容仍可讀取。

### BimElements：構件

上傳時解析出的牆、柱、梁、樓板，一個構件一列。

| 欄位 | 型別 | 必填 | 說明 |
|---|---|---|---|
| `Id` | INTEGER | 是 | 主鍵，自動遞增。API 不使用它，一律用 `GlobalId` |
| `ProjectId` | INTEGER | 是 | 外鍵 → `BimProjects.Id`，串聯刪除；有索引 |
| `GlobalId` | TEXT | 是 | IFC 的 GlobalId（22 字元）。**整個資料庫唯一**（唯一索引），所以同一個模型無論專案名稱為何都只能上傳一次，重複上傳回 409。可能含 `$` 等字元，放進網址時要編碼 |
| `ElementName` | TEXT | 是 | IFC 的 `Name`，沒有時為空字串 |
| `Category` | TEXT | 是 | `IfcWall`、`IfcColumn`、`IfcBeam`、`IfcSlab` 其中之一。子型別（例如 `IfcWallStandardCase`）歸入父類別 |
| `Material` | TEXT | 否 | 材質名稱；有多個材質或材質層時以 `, ` 串接。沒有材質時為 NULL |
| `EstimatedCost` | TEXT（decimal） | 是 | 依分類給的固定模擬金額：牆 50000、梁 30000、柱 40000、樓板 80000 |
| `ProductionStatus` | INTEGER | 是 | 生產狀態：`0` 未開始、`1` 工廠製造中、`2` 已出貨、`3` 工地已安裝。上傳時為 0 |
| `UpdatedAt` | TEXT（DateTime） | 是 | 最後修改狀態的時間（UTC）。上傳時等於專案的 `UploadedAt` |

- 範例模型 `AC20-Institute-Var-2.ifc` 上傳後是 149 列（牆 121、柱 2、樓板 26、梁 0），總成本 8,210,000。
- 只記錄目前的狀態，**沒有狀態變更的歷史**。

### AspNetRoles：角色

| 欄位 | 型別 | 說明 |
|---|---|---|
| `Id` | TEXT | 主鍵（GUID 字串） |
| `Name` | TEXT | 角色名稱：`Admin`、`Operator`、`Viewer` |
| `NormalizedName` | TEXT | 大寫名稱，唯一索引 `RoleNameIndex`。依名稱查詢時用它 |
| `ConcurrencyStamp` | TEXT | Identity 的樂觀並行控制 |
| `Weight` | INTEGER | **本專案新增**。權重越大層級越高：Admin 100、Operator 50、Viewer 10。管理帳號時只能管理、指派權重比自己低的使用者與角色 |

### AspNetRoleClaims：角色的權限

| 欄位 | 型別 | 說明 |
|---|---|---|
| `Id` | INTEGER | 主鍵 |
| `RoleId` | TEXT | 外鍵 → `AspNetRoles.Id`，串聯刪除 |
| `ClaimType` | TEXT | 權限一律為 `bim-mes:permission` |
| `ClaimValue` | TEXT | 權限名稱，例如 `projects.view` |

預設內容（新資料庫共 8 列）：

| 角色 | 權限 |
|---|---|
| Admin | `projects.view`、`projects.upload`、`elements.update-status`、`users.manage`、`users.unlock` |
| Operator | `projects.view`、`elements.update-status` |
| Viewer | `projects.view` |

程式只檢查權限、不看角色名稱，所以**直接修改這張表就能調整角色的權限**，而且下一個請求就生效（不需要重新登入）。目前沒有調整權限的 API 或畫面。

### AspNetUsers：使用者

Identity 的標準欄位。本專案實際用到的如下：

| 欄位 | 型別 | 說明 |
|---|---|---|
| `Id` | TEXT | 主鍵（GUID 字串），也是 JWT 的 `sub` |
| `UserName` | TEXT | 帳號名稱，1～64 字元（建立時檢查） |
| `NormalizedUserName` | TEXT | 大寫帳號名稱，唯一索引 `UserNameIndex`。登入與重複檢查用它，所以帳號名稱不分大小寫 |
| `PasswordHash` | TEXT | Identity 預設的雜湊（PBKDF2），不存明文 |
| `SecurityStamp` | TEXT | 每個 JWT 都帶著它；停用、重設密碼、改角色、自行改密碼時更新，舊 token 因此立即失效 |
| `ConcurrencyStamp` | TEXT | Identity 的樂觀並行控制 |
| `LockoutEnabled` | INTEGER | 是否啟用鎖定機制。建立時為 1 |
| `LockoutEnd` | TEXT（DateTimeOffset） | 鎖定到何時（UTC）。見下方「帳號狀態的判斷」 |
| `AccessFailedCount` | INTEGER | 連續登入失敗次數。達到 `Auth:Lockout:MaxFailedAttempts`（預設 5）時鎖定並歸零 |

未使用的欄位：`Email`、`NormalizedEmail`、`EmailConfirmed`、`PhoneNumber`、`PhoneNumberConfirmed`、`TwoFactorEnabled`。

**帳號狀態的判斷**（依 `LockoutEnd`）：

| `LockoutEnd` | 狀態 | 如何產生 | 如何恢復 |
|---|---|---|---|
| NULL 或早於現在 | 正常 | — | — |
| 晚於現在的一般時間 | 暫時鎖定 | 連續登入失敗，鎖 `Auth:Lockout:Minutes`（預設 15）分鐘 | 時間到、`unlock` API、`enable` API |
| `DateTimeOffset.MaxValue`（9999-12-31 的最後一刻） | 停用 | `disable` API | 只有 `enable` API |

鎖定與停用時，登入一律回同樣的 401，持有的舊 token 也會被拒絕。

### AspNetUserRoles：使用者的角色

| 欄位 | 型別 | 說明 |
|---|---|---|
| `UserId` | TEXT | 複合主鍵；外鍵 → `AspNetUsers.Id`，串聯刪除 |
| `RoleId` | TEXT | 複合主鍵；外鍵 → `AspNetRoles.Id`，串聯刪除；有索引 |

規則上每個使用者**恰好一個角色**（API 改角色時會先移除舊的）。資料表本身不限制，資料異常有多個角色時，程式取權重最高的那個。

### AspNetUserClaims：使用者層級的權限

| 欄位 | 型別 | 說明 |
|---|---|---|
| `Id` | INTEGER | 主鍵 |
| `UserId` | TEXT | 外鍵 → `AspNetUsers.Id`，串聯刪除 |
| `ClaimType` | TEXT | `bim-mes:permission` |
| `ClaimValue` | TEXT | 權限名稱 |

目前只有一筆：首位管理者的 `users.manage-peers`（能管理同權重的使用者）。使用者的有效權限 = 角色的權限 ∪ 這張表的權限。沒有寫入這張表的 API。

### AspNetUserLogins、AspNetUserTokens

Identity 的標準資料表（外部登入、token provider），本專案**未使用**，應該是空的。

## API 與資料表的對應

「讀」「寫」只列該 API 本身的存取。另外，每個需要登入的請求在驗證階段都會讀 `AspNetUsers`、`AspNetUserRoles`、`AspNetRoles`、`AspNetRoleClaims`、`AspNetUserClaims`（見下一節）。

| API | Controller | 經過的服務 | 讀 | 寫 |
|---|---|---|---|---|
| `POST /api/auth/login` | `AuthController` | `UserManager`、`SignInManager`、`UserAccessService`、`JwtTokenService` | `AspNetUsers`、`AspNetUserRoles`、`AspNetRoles`、`AspNetRoleClaims`、`AspNetUserClaims` | `AspNetUsers`（`AccessFailedCount`、`LockoutEnd`） |
| `GET /api/auth/me` | `AuthController` | — | （權限已在驗證階段載入） | — |
| `POST /api/auth/change-password` | `AuthController` | `UserManager` | `AspNetUsers` | `AspNetUsers`（`PasswordHash`、`SecurityStamp`） |
| `POST /api/projects/upload` | `ProjectsController` | `IIfcParsingService`、`IFileStorageService` | `BimElements`（GlobalId 是否已存在） | 上傳目錄（.ifc 檔）、`BimProjects`、`BimElements`（一次交易寫入） |
| `GET /api/projects/{projectId}` | `ProjectsController` | `IFileStorageService`（組下載網址） | `BimProjects`、`BimElements`（依狀態、依金額分組計數） | — |
| `GET /api/projects/{projectId}/elements` | `ProjectsController` | — | `BimProjects`、`BimElements` | — |
| `GET /api/elements/{globalId}` | `ElementsController` | — | `BimElements` | — |
| `PUT /api/elements/{globalId}/status` | `ElementsController` | — | `BimElements` | `BimElements`（`ProductionStatus`、`UpdatedAt`） |
| `GET /api/settings/upload` | `SettingsController` | — | （只讀設定值） | — |
| `GET /api/users` | `UsersController` | `UserWeightLookup`、`RoleWeightRules` | `AspNetUsers`、`AspNetUserRoles`、`AspNetRoles`、`AspNetRoleClaims`、`AspNetUserClaims` | — |
| `POST /api/users` | `UsersController` | `UserManager`、`RoleManager`、`RoleWeightRules` | `AspNetUsers`、`AspNetRoles` | `AspNetUsers`、`AspNetUserRoles` |
| `PUT /api/users/{id}/role` | `UsersController` | 同上 | 同上 | `AspNetUserRoles`、`AspNetUsers`（`SecurityStamp`） |
| `POST /api/users/{id}/disable` | `UsersController` | `UserManager`、`RoleWeightRules` | 同上 | `AspNetUsers`（`LockoutEnd` 設為最大值、`SecurityStamp`） |
| `POST /api/users/{id}/enable` | `UsersController` | 同上 | 同上 | `AspNetUsers`（`LockoutEnd` 清除、`AccessFailedCount` 歸零、`SecurityStamp`） |
| `POST /api/users/{id}/reset-password` | `UsersController` | 同上 | 同上 | `AspNetUsers`（`PasswordHash`、`SecurityStamp`） |
| `POST /api/users/{id}/unlock` | `UserUnlockController` | `UserWeightLookup`、`RoleWeightRules`、`UserManager` | 同上 | `AspNetUsers`（`LockoutEnd` 清除、`AccessFailedCount` 歸零；**不**更新 `SecurityStamp`） |
| `GET /api/roles` | `RolesController` | `UserWeightLookup`、`RoleWeightRules` | `AspNetRoles` 與呼叫者的角色、權限 | — |
| `GET /uploads/{StorageKey}` | （靜態檔） | — | 上傳目錄 | — |

帳號管理的寫入都包在一個資料庫交易中；任何一步失敗就整個復原。

### 驗證與權限的資料流

```text
登入（POST /api/auth/login）
  ├─ 依 NormalizedUserName 找使用者；不存在時仍做一次密碼雜湊，讓回應時間接近
  ├─ 檢查密碼與鎖定（失敗時 AccessFailedCount +1，達上限設定 LockoutEnd）
  ├─ UserAccessService：角色 → AspNetRoleClaims，加上 AspNetUserClaims → 權限清單
  └─ JwtTokenService：簽發 JWT
       claim：sub（使用者 Id）、name、role、bim-mes:stamp（SecurityStamp）
       不含權限

之後的每個請求（JwtBearerEvents.OnTokenValidated）
  ├─ 驗簽章、簽發者、對象、有效期
  ├─ 依 sub 查 AspNetUsers：不存在、鎖定或停用、SecurityStamp 不同 → 401
  ├─ 移除 token 中自帶的權限 claim（防止偽造）
  └─ 重新從資料庫載入權限，加到這次請求的身分上
       → [Authorize(Policy = "projects.upload")] 等標註依此判斷
```

因為權限每次從資料庫載入，改角色或改 `AspNetRoleClaims` 後，同一個 token 的下一個請求就適用新權限；代價是每個請求都會查幾次資料庫。

## 除錯時常用的查詢

在 repo 根目錄，用暫時的容器查開發環境的資料庫（本機沒有安裝 sqlite3）。先停掉後端，避免同時寫入：

```sh
docker compose stop backend
docker run --rm -v "$PWD/backend:/data" alpine sh -c \
  "apk add -q sqlite && sqlite3 -header -column /data/bim_mes.db '<SQL>'"
```

**使用者、角色與帳號狀態**

```sql
SELECT u.UserName, r.Name AS Role, r.Weight, u.LockoutEnd, u.AccessFailedCount
FROM AspNetUsers u
LEFT JOIN AspNetUserRoles ur ON ur.UserId = u.Id
LEFT JOIN AspNetRoles r ON r.Id = ur.RoleId
ORDER BY r.Weight DESC, u.UserName;
```

**某位使用者的有效權限**（角色的權限加上使用者層級的權限）

```sql
SELECT rc.ClaimValue AS Permission, 'role' AS Source
FROM AspNetUsers u
JOIN AspNetUserRoles ur ON ur.UserId = u.Id
JOIN AspNetRoleClaims rc ON rc.RoleId = ur.RoleId AND rc.ClaimType = 'bim-mes:permission'
WHERE u.NormalizedUserName = 'SUPADMIN'
UNION
SELECT uc.ClaimValue, 'user'
FROM AspNetUsers u
JOIN AspNetUserClaims uc ON uc.UserId = u.Id AND uc.ClaimType = 'bim-mes:permission'
WHERE u.NormalizedUserName = 'SUPADMIN';
```

**各專案的構件數與狀態分布**

```sql
SELECT p.Id, p.ProjectName, p.StorageKey,
       COUNT(e.Id) AS Elements,
       SUM(e.ProductionStatus = 0) AS NotStarted,
       SUM(e.ProductionStatus = 1) AS InProduction,
       SUM(e.ProductionStatus = 2) AS Shipped,
       SUM(e.ProductionStatus = 3) AS Installed
FROM BimProjects p
LEFT JOIN BimElements e ON e.ProjectId = p.Id
GROUP BY p.Id;
```

**最近修改過狀態的構件**

```sql
SELECT GlobalId, ElementName, Category, ProductionStatus, UpdatedAt
FROM BimElements
WHERE UpdatedAt <> (SELECT UploadedAt FROM BimProjects WHERE Id = BimElements.ProjectId)
ORDER BY UpdatedAt DESC
LIMIT 20;
```

**找沒有對應資料庫記錄的上傳檔**：比對上傳目錄的檔名與 `SELECT StorageKey FROM BimProjects;`。正常流程不會產生這種檔案（先解析再存檔，寫入資料庫失敗時刪除剛存的檔案）。

正式版的資料在 volume `bim_data` 中，查詢時把 `-v "$PWD/backend:/data"` 換成 `-v bim-mes-prod_bim_data:/data`，並先停掉正式版的後端。

## 接手時要注意的地方

- **改資料表**：在 `Models/` 或 `AppDbContext` 修改後，在容器中用 `dotnet ef migrations add <名稱>` 產生 Migration；後端啟動時會自動套用。Migration 要檢查只動到預期的資料表。
- **新增權限**：在 `Auth/Permissions.cs` 加常數並加進 `Permissions.All`（`Program.cs` 依它註冊 policy，有測試檢查兩者一致）。要給**已存在的角色**時，另寫一個只寫資料的 Migration（參考 `AddUsersUnlockPermission`），因為 bootstrapper 不會改已存在的角色。
- **首位管理者忘記密碼**：沒有任何 API 能處理他（他擁有 `users.manage-peers`，誰都不能管理他）。只能直接改資料庫，或清空帳號資料表後依 `.env` 重新建立（會刪掉所有帳號）。
- **GlobalId 全域唯一**：同一個模型不能上傳到兩個專案。要重新上傳，只能先刪掉資料庫中的舊專案與構件（目前沒有 API）。
- **沒有稽核紀錄**：誰上傳、誰改了狀態、誰管理了帳號，資料庫都沒有記錄；只能看應用程式的 log。
- **時間**：`DateTime` 欄位存的是 UTC 但字串不帶時區，與當地時間比較時要自行換算（台灣為 UTC+8）。
