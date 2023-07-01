+++
title = 'Azure Static Web Apps Api'
date = '2023-04-23T20:37:20+08:00'
draft = false
summary = 'Azure Static Web App 是一個可以快速建立靜態網站的服務，但是它也可以用來建立後端 API。'
tags = ['Azure', 'Static Web Apps', 'Azure Function']
+++
> Azure Static Web Apps 是一個提供建置和部屬網頁應用程式的服務，可以方便的與 Github Action 整合，以及使用 Azure Functions 作為後端 API。這邊記錄一下如何使用 Azure Function 作為後端 API 讓前端靜態網頁呼叫。
### 簡易說明
Azure Static Web Apps 服務的免費方案可以提供建置 API 隨著前端一起部屬，如果是標準方案的話，可以使用既有的 Azure Function 作為後端 API。

除了 API 來源之外，免費方案的 Azure Function 也有一些限制：[文件](https://learn.microsoft.com/en-us/azure/static-web-apps/apis-overview#constraints)

### 預先準備
- VS Code
- VS Code Static Web App Extension
- 已經建立好的 Static Web App 專案

### 前端
這邊簡單用一個 html 檔案，內容是一個簡單的登入表單，以及顯示結果的地方：
```html
<form class="d-flex flex-column align-items-center justify-content-center" style="height: 50vh">
    <label for="name">Name</label>
    <input type="text" id="name" />
    <label for="password">Password</label>
    <input type="password" id="password" />
    <button type="submit" class="m-2 btn btn-primary" onclick="sampleCall()">
        Submit
    </button>
</form>

<div class="d-flex flex-column align-items-center justify-content-center">
    <h1 id="result"></h1>
</div>
```
表單的 submit 事件會呼叫 `sampleCall()` 這個 function，這個 function 會呼叫後端的 API，並將結果顯示在畫面上：
```javascript
const sampleCall = async () => {
    console.log("sample call");
    // cancel the default behavior of the form
    event.preventDefault();

    const name = document.getElementById("name").value;
    const password = document.getElementById("password").value;
    const result = document.getElementById("result");
    const response = await fetch("<api_url>", {
    method: "POST",
    headers: {
        "Content-Type": "application/json",
    },
    body: JSON.stringify({ name, password }),
    });

    const data = await response.json();
    console.log(data);

    // display the result
    result.innerHTML = data.response;
};
```

### 建立 Azure Function 作為後端 API
在 VS Code 中，點選 `F1` 打開命令列，輸入 `Azure Static Web Apps: Create HTTP Function...`，依據提示輸入 Function 名稱、語言、以及要放置的資料夾等選項。這邊用 JavaScript 當 Azure Function 的語言，並放在 `api` 資料夾下，Function 名稱為 SampleFunction，資料夾結構如下：

```
|── .github/workflows
│   └── azure-static-web-apps-<project-name>.yml
├── api
│   └── SampleFunction
│   │   ├── function.json
│   │   └── index.js
│   ├── .funcignore
|   |── package.json
|   |── package-lock.json
|   |── local.settings.json
│   └── host.json
└── src
    └── index.html
```

修改 `index.js` 的內容，判斷使用者輸入的帳號密碼是否正確，正確的話回傳歡迎訊息，不正確的話回傳錯誤訊息：
```javascript
module.exports = function (context, req) {
  context.log("JavaScript HTTP trigger function processed a request.");

  const name = req.body.name;
  const password = req.body.password;

  // get the correct password from the environment variable
  const correctPassword = process.env["CORRECT_PASSWORD"];

  if (name && name === "Abin" && password && password === correctPassword) {
    context.res.json({
      response: "Welcome Abin!",
    });
  }
  else {
    context.res.status(400).json({
      response: "Incorrect user name or password",
    });
  }
};
```

> 這邊的 `CORRECT_PASSWORD` 是一個環境變數，可以從 Azure Portal 的 Static Web Apps 頁面中設定。
> 
> 如果是本地端開發時也可以在 `local.settings.json` 檔案中設定。

### 修改前端程式碼
整合進 Static Web Apps 的 Azure Function 在免費方案中 URL 是固定以 `/api` 開頭，所以要修改前端程式碼來呼叫 Azure Function：

```javascript
...
await fetch("/api/SampleFunction", 
...
```

### 本地測試
因為是使用 Azure Function 作為後端 API，所以如果要測試 Function 可以直接透過 [Azure Function Core Tools](https://learn.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=v4%2Cwindows%2Ccsharp%2Cportal%2Cbash) 來測試：
```bash
$ cd api
$ func start
```
也可以透過 Static Web App 的 CLI 來測試，更符合 Static Web App 的真實運作狀況。這邊要先安裝 CLI：
```bash
npm install -g @azure/static-web-apps-cli
```
接著 `init` 來初始化專案，會產生一個 `swa.config.json` 檔案，需要按照提示輸入專案的資訊：

```json
{
  "$schema": "https://aka.ms/azure/static-web-apps-cli/schema",
  "configurations": {
    "static-web-apps-sample": {
      "appLocation": "src",
      "apiLocation": "api",
      "outputLocation": ".",
      "apiBuildCommand": "npm run build --if-present"
    }
  }
}
```

接著 `start` 來啟動專案，會同時啟動前端和 API，預設會使用 4280 port，就可以開啟瀏覽器來測試了：

```bash
$ swa start
```

> Static Web Apps CLI 有詳細的文件可以參考：[連結](https://azure.github.io/static-web-apps-cli/)
> 保哥也有寫過詳細的說明：[連結](https://blog.miniasp.com/post/2022/10/22/Deploy-Website-using-Azure-Static-Web-Apps-CLI)


> 在使用 Static Web Apps CLI 的時候有遇到執行 `swa start` 時 CLI 無法正確連結到 Azure Function 的問題，發生的原因不明，有 [issue](https://github.com/Azure/static-web-apps-cli/issues/335) 是類似的情況，改 node 版本後就可以正常執行。使用 devcontainers 也可以解決這個問題。

### 部屬

部屬前要確認 `.github/workflows/azure-static-web-apps-<project-name>.yml` 檔案，將 api 位置指定到 `api` 資料夾下：

```yml
api_location: "api" # Api source code path - optional
```

如果先前建立專案的步驟正確的話，就可以 push 到 GitHub 上確認部屬是否成功了。


> 專案：[連結](https://github.com/AaaBin/static-web-apps-sample)

### 參考資料
- [What is Azure Static Web Apps?](https://learn.microsoft.com/en-us/azure/static-web-apps/overview)
- [Static Web Apps CLI](https://azure.github.io/static-web-apps-cli/)
- [如何使用 Azure Static Web Apps CLI 手動部署靜態網站應用程式](https://blog.miniasp.com/post/2022/10/22/Deploy-Website-using-Azure-Static-Web-Apps-CLI)
- [Azure Function Core Tools](https://learn.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=v4%2Cwindows%2Ccsharp%2Cportal%2Cbash)