+++
title = '[筆記]ASP.NET Core 中的 Response Cache 和 Output Cache'
date = '2023-08-27T16:26:00+08:00'
draft = false
summary = '關於 ASP .NET Core response cache 以及 output cache 的筆記'
tags = ['ASP.NET Core']
+++
## Response Cache
ASP.NET Core 中提供的 response cache 為兩個部分，**ResponseCache Attribute** 和 **Response Caching Middleware**：
- ResponseCache Attribute 的作用是設定回應的標頭，用來讓 client 端或 proxy 依據相對應的[規範](https://www.rfc-editor.org/rfc/rfc9111)來採取對應的行為。 
- Response Caching Middleware 則是提供了 server side 的 cache，作用的依據一樣是標頭，所以會被 ResponseCache Attribute 影響。

### 相關 header
- [Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control): 在請求或回應的標頭中，以指令控制 cache 機制，例如：
    - response 中設定 `public` 表示任何對象（client 或 proxy）都可以將回應 cache 住。
    - response 或 request 中設定 `max-age` 表示這個 response 應該被 cache 多少秒。
- Vary: 代表應該要以哪一個 header 欄位，判斷下次應該使用 cache 住的內容或重新向 server 發起請求，例如設定 `X-User` 代表未來要請求同樣的資源時，會判斷 `X-User` header 值是否相同，相同就可以使用 cache 過的 response。
- Age: 這個 header 代表的是 response 被 cache 的時間（秒），會在使用 cache 內容時被加入，例如第一次請求後過了15秒，第二次請求如果使用了 cache，response 就會有 `Age: 15` 的 header。

### 實際範例

1. ResponseCache Attribute

```csharp
[HttpGet]
[ResponseCache(Duration = 30, VaryByHeader = "User-Agent")]
public IEnumerable<WeatherForecast> Get()
{
	// 省略
}
```
呼叫第一次後，在 response header 會看到 `Cache-control: public,max-age=30` 和 `Vary: User-Agent`，且 30 秒內的重複請求會直接從瀏覽器的 cache 中取得。

⚠️ Client side caching 行為沒辦法用瀏覽器輸入網址並重新整理的方式進行觀察，因為瀏覽器重新整理會預設帶入 cache-control:max-age=0，用來避免 cache。

___


2. Response Caching Middleware

```csharp

builder.Services.AddResponseCaching(opt =>
{
    opt.MaximumBodySize = 1024 * 1024;
});

builder.Services.AddCors(opt =>
{
    // 省略
});

var app = builder.Build();

// Configure the HTTP request pipeline.

app.UseCors("AllowAll");

app.UseAuthorization();

app.UseResponseCaching();

app.Use(async (context, next) =>
{
    context.Response.GetTypedHeaders().CacheControl = new CacheControlHeaderValue
    {
        Public = true,
        MaxAge = TimeSpan.FromSeconds(30),
    };

    context.Response.Headers[HeaderNames.Vary] = new string[] { "Accept-Encoding" };

    await next();
});

```
response caching 提供的是 server side caching，但控制的依據一樣是透過 `Cache-Control` [相關 header](#相關-header)，只設置 `AddResponseCaching` 和 `UseResponseCaching` 並不會有任何的效果，需要配合 ResponseCache Attribute 或自定義 middleware 等方式來設定 header，而自訂義 middleware 中的相關 header 設置會被 ResponseCache Attribute 覆蓋掉。

因為有 `Cache-Control` 設定為 public，所以會有跟設置 `ResponseCache Attribute` 一致的行為，但用 postman 這類工具呼叫會發現第二次請求開始，response header 會增加一個 `Age` ，`Age` 的值代表這一 response 產生後已經過了幾秒，同時觀察 server 端的 log 會發現除了第一次 request，後續 30 秒內的 request 都只會到 middleware 就回傳。

⚠️ 需要注意 Response Caching Middleware 的位置，如果有使用 cors middleware 的話 cors 需要在前面。

___

## Output Caching

Output Caching 是 .NET 7 開始提供的新 middleware，類似原本的 Response Caching Middleware，提供 server side 的 cache 機制，但有一些調整，例如：

- 原先的 Response Caching 行為是依循 Http Header 定義的，代表 Client 端可以影響 caching 行為，例如 request header 帶入 `Cache-Control: max-age=0` 時會停用 cache，即使 server 端有可用的 cache。Output Cache 不依循 Http Header，該如何提供快取完全由 server 端進行控制。
- 提供從 server 端讓 cache entry 失效的方式。
- 有 Cache revalidation 機制，server 可以回應 `304 Not Modified` 來省去回應完整 response body 的流量。
- 有 resource locking 的機制來避免 [cache stampede](https://en.wikipedia.org/wiki/Cache_stampede)、[thundering herd problem](https://en.wikipedia.org/wiki/Thundering_herd_problem)。

### 基本設定
也是從 `AddOutputCache` 和 `UseOutputCache` 開始，跟 Response Caching middleware 一樣需要注意順序，必須要在 UseCors 之後。
```csharp
builder.Services.AddOutputCache(opt => {
    opt.AddBasePolicy(builder => builder.Cache());
});

var app = builder.Build();

// 省略

app.UseOutputCache();
```
Output cache 的設定方式是依據 policy，`AddCoutputCache` 提供選項設定 BasePolicy，`builder.Cache()` 表示使用預設的 cache policy，規則包括 response status code 200 才會被 cache，以及 GET、HEAD 方法才會被 cache。

Base policy 的規則會套用至所有的 endpoints，也可以透過 `AddPolicy` 方法增加額外 policy，並使用 `[OutputCache]` attribute 或 `CacheOutput` 方法來使用。
```csharp

// in Program.cs
builder.Services.AddOutputCache(opt =>
{
    opt.AddPolicy("100seconds", builder => builder.Expire(TimeSpan.FromSeconds(100)));
});

// 省略

app.MapGet("/", () => $"Hello World! at {DateTime.Now}").CacheOutput("100seconds");
// 這樣也可以
// app.MapGet("/", [OutputCache(PolicyName = "100seconds")] () => $"Hello World! at {DateTime.Now}");

// or in controller
[HttpGet]
[OutputCache(PolicyName ="100seconds")]
public IEnumerable<WeatherForecast> Get()
{
    // 省略
}
```

### 自行實作 Policy
可以透過實作 `IOutputCachePolicy` 來自定義，這個介面有三個方法：

- `CacheRequestAsync` 會在 output caching middleware 被調用前呼叫，可針對 request context 決定是否啟用 cache。
- `ServeFromCacheAsync` 會在使用 cached response 之前呼叫。
- `ServeResponseAsync` 會在 response 被提供且將被 cache 前呼叫，可依據 response context 決定是否將 response cache 。

### Cache key

預設的 cache entry key 值包含了 URL 的全部部分（scheme, host, port, path, query string），如果需要只依照特定部分，可以透過 `SetVaryByXXX` 

```csharp
builder.Services.AddOutputCache(options =>
{
    options.AddPolicy("Query", builder => builder.SetVaryByQuery("min"));
});
```
Output Cache policy 設定為 Query 的 endpoint 只要有相同的 query string `min` 就會被提供相同的 cache 內容。

### [Cache revalidation](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/output?view=aspnetcore-7.0#cache-revalidation)

Cache revalidation 是預設就被啟用的，不需要額外的 policy 設定，是根據 http header 來選擇採用的行為：

要讓 server 回應 `304 Not Modified` 有兩種方式，一是 request header 使用 [If-Modified-Since](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/If-Modified-Since) ，帶入 GMT 時間，如果在這個給定的時間之後，請求的 resource 有修改則走正常流程，否則回應 `304 Not Modified`。

第二種方式需要在 response header之中加入 [ETag](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/ETag)，表示這個 resource 的版本（要有雙引號 `ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"`），有了 ETag header ，後續的 request 只要有帶上 [If-None-Match](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/If-None-Match) header 並且值是符合既有的 response ETag 值，OutputCaching middleware 就會回應 `304 Not Modified`。

### Evict

可以透過加 tag 來將 cache 分類，並且以 IOutputCacheStore 提供的 `EvictByTagAsync` 方法將 cache 清除。

```csharp
options.AddBasePolicy(builder => builder.Cache().Tag("all"));

// 省略

app.MapPost("/evict-all", (IOutputCacheStore store) => store.EvictByTagAsync("all", default));
```

### Locking

當同時有多個請求針對同一資源時，Output Caching middleware 會有 locking 機制來確保只有第一個請求被執行且寫入 cache 後，後續的請求就會使用 cache 中的內容，確保不會因太多請求同時執行、寫 cache 導致系統過載。

詳細說明：[cache stampede](https://en.wikipedia.org/wiki/Cache_stampede)、[thundering herd problem](https://en.wikipedia.org/wiki/Thundering_herd_problem)。

___


## 參考資料

[Output caching middleware in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/output?view=aspnetcore-7.0#override-the-default-policy)

[有關 Cache 的一些筆記 - Kakashi's Blog](https://kkc.github.io/2020/03/27/cache-note/#Cache-Stampede-Thundering-Herd-problem)

[Exploring the new output caching middleware](https://timdeschryver.dev/blog/exploring-the-new-output-caching-middleware#default-or-and-quot-base-and-quot-policies)