+++
title = '陽春題目產生器 QuizMaker 紀錄'
date = '2024-02-06T23:11:00+08:00'
draft = false
summary = '久違的 .NET 小專案紀錄'
tags = ['.NET', 'Cloud Run']
isCJKLanguage = true
+++

## 陽春題目產生器 QuizMaker 紀錄

一直有讀英文的需求（？），前陣子看到[這個影片](https://youtu.be/MwgWnuSlybY)，其中提到一點是關於用測驗來增加記憶以及理解是有幫助的，想了想發現很久沒有寫到英文題目了，決定試著用ＡＩ幫助產生題目給我練習。做完後把過程中的新嘗試和困難記錄下來。

## 架構

使用 .NET Console Application，部署環境是 GCP Cloud Run，非常簡易的設定 Scheduler 定期執行。

## 功能

自動抓 [6 Minute English](https://www.bbc.co.uk/learningenglish/english/features/6-minute-english_2024) 最新的主題（以穩定每週四發文為前提），用 OpenAI gpt-3.5-turbo 產生練習題，分為單字練習以及綜合練習。會產生兩個題目以及一個單字整理共三個 pdf 檔案，寄送到信箱。

## 新嘗試／小問題

### .NET Publish Container

要發部到 Cloud Run 需要將程式容器化，寫 dockerfile 不困難，但因為用的電腦是 M1 晶片的 Mac 所以 build image 還需要用到 docker buildx 來編譯，因此這次試著用 `dotnet publish` 的功能來發布。

```bash
dotnet publish --os linux --arch x64 /t:PublishContainer -c Release
```

用一個簡單的指令就可以打包並發布 image，預設是發布到 local 的 registry，要 push 到 docker hub 可以在 csproj 設定

```xml
<EnableSdkContainerSupport>true</EnableSdkContainerSupport>
<ContainerRegistry>docker.io</ContainerRegistry>
<ContainerRepository>username/repository</ContainerRepository>
<ContainerImageTags>v0.1;latest</ContainerImageTags>
```

### Mailjet

希望寄送的 Email 有比較精美的排版，找到 Mailjet 這個服務，有免費方案，支援模板，使用上也容易上手，但過程中有遇到小問題，在 mail 上附加檔案的時候會出現不明錯誤，Mailjet service 回應的錯誤訊息也很模糊，後來發現mailjet-apiv3-dotnet 雖然顧名思義用的是 v3 的 api，但其實也已經實作 v3.1 的呼叫方式了，換成呼叫 `SendTransactionalEmailAsync` 就是 v3.1 API，成功解決問題。

### Cloud Run

原先的構想其實是製作一個瀏覽器擴充套件，並透過 REST API 的方式觸發 Cloud Run Job，同時將想要製作出題目的英文單字或文章作為參數傳送過去，但在身份認證的這個部分卡關，雖然可以透過 OAuth 取的足夠執行 Cloud Run 的權限，但因為不像是 IAM 一樣可以詳細控制 scope，能夠使用的只有 `https://www.googleapis.com/auth/cloud-platform` 這個超泛用 scope，考量到安全性還是選擇放棄。

但還是有學到東西，要在觸發 Cloud Run 的時候帶入參數，使用的方式是觸發時複寫 Cloud Run 的設定（需要 `run.jobs.runWithOverrides` 權限），如果是 REST API 的話，就是在 `POST https://{endpoint}/apis/run.googleapis.com/v1/{name}:run` 加上 request body，例如：

```json
{
    "overrides": {
        "containerOverrides": [
            {
                "name": "",
                "args": [
                    "1",
                    "2"
                ],
                "env": [
                    {
                        "name": "TEST",
                        "value": "HELLOWORLD"
                    }
                ],
                "clearArgs": false
            }
        ],
        "taskCount": 1,
        "timeoutSeconds": 30
    }
}
```

## 參考資料

* [.NET Publish Container](https://learn.microsoft.com/en-us/dotnet/core/docker/publish-as-container?source=recommendations&pivots=dotnet-8-0#publish-net-app)
* [mailjet-apiv3-dotnet](https://github.com/mailjet/mailjet-apiv3-dotnet)
* [Override job configuration for a specific execution](https://cloud.google.com/run/docs/execute/jobs#override-job-configuration)
