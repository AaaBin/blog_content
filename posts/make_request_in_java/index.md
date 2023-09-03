+++
title = '［筆記］Make Request In Java'
date = '2023-09-03T22:00:00+08:00'
draft = false
summary = '關於在 Java 中幾種用來發送 HTTP 請求的方法筆記'
tags = ['Java', 'Spring']
isCJKLanguage = true
+++

> 這篇筆記是剛開始學習 Java 的時候做的，隨著 Java 以及 Spring 版本的更新，應該有些 API 或觀念不適用 （又或者是我當初就弄錯的）  
> 
> 更新： Spring 的 `WebClient` 似乎比較像是一個 wrapper，其實需要 [HTTP Client Library](https://docs.spring.io/spring-framework/reference/web/webflux-webclient/client-builder.html#webflux-client-builder-jdk-httpclient)才能使用，這部分之前沒有注意到。


[一、Java 原生 API](#一java-原生-api)  
[二、Spring 提供](#二spring-提供)  
[三、非同步請求](#三非同步請求)  
[四、比較及結語](#四比較及結語)  


## 一、Java 原生 API

1. `HttpURLConnection`/`HttpsURLConnection`（package: `java.net`）
    
    `HttpURLConnection` 是從 JDK1.1 就已經有的 HTTP Client API，提供簡單的功能，僅支援同步。
    
    使用上首先從 `URL` 開啟連線，接著設定 request method 以及 header，最後使用 `getInputStream()` 來發送請求並取得回應：

    - Get
        ```java
        URL url = new URL(baseUrl + "/User");
        HttpURLConnection con = (HttpURLConnection) url.openConnection();
        con.setRequestMethod("GET");
    
        con.setRequestProperty("Content-Type", "application/json; charset=UTF-8");
    
            InputStream inputStream = con.getInputStream();
        String responseContent = readStreamToString(inputStream);
    
        con.disconnect();
        ```
        
    - 若不需要特別設定 request method 或 header，可以直接從 `URL` 的 `openStream()` 來取得回應：
        
        ```java
        URL url = new URL(baseUrl + "/User/5");
    
        String responseContent = readStreamToString(url.openStream());
        ```
        
    - Post
        
        傳送 request body 需先設定 `DoOutput` 為 `true`，並將 request body 寫入 outputStream 中：
        
        ```java
        URL url = new URL(baseUrl + "/User");
        HttpURLConnection con = (HttpURLConnection) url.openConnection();
        con.setRequestMethod("POST");
        con.setDoOutput(true);
        con.setRequestProperty("Content-Type", "application/json; charset=UTF-8");
    
        User newUser = new User("foo");
        String output = mapper.writeValueAsString(newUser);
    
        try (OutputStream outputStream = con.getOutputStream()) {
            byte[] outputBytes = output.getBytes(StandardCharsets.UTF_8);
            outputStream.write(outputBytes);
        }
    
        String responseContent = readStreamToString(con.getInputStream());
        ```
        
2. `HttpClient`（package: `java.net.http`）
    
    Java 11 推出的新API，提供同步／非同步的請求呼叫方式以及 HTTP/2 支援等新功能。
    
    核心為 `HttpRequest`, `HttpClient` 以及 `HttpResponse` 三個類別。
    
    - Get  
        `HttpClient` 可以設定要使用 HTTP/1 或是 HTTP/2。
        
        發送請求時還需以 `BodyHandler` 來指定 response body 型別。
        ```java
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(baseUrl + "/User"))
                .GET()
                .header("Content-Type", "application/json; charset=UTF-8")
                .build();
        
        HttpResponse<String> httpResponse = HttpClient.newBuilder()
                .version(HttpClient.Version.HTTP_2)
                .build()
                .send(request, HttpResponse.BodyHandlers.ofString(StandardCharsets.UTF_8));
        ```
        
    - Post  
        設定 request body 是由 `BodyPublisher`
        
        ```java
        User newUser = new User("foo");
        String outputData = mapper.writeValueAsString(newUser);
        
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(baseUrl + "/User"))
                .header("Content-Type", "application/json; charset=UTF-8")
                .method("POST", HttpRequest.BodyPublishers.ofString(outputData, StandardCharsets.UTF_8))
                .build();
        
        HttpResponse<byte[]> httpResponse = HttpClient.newHttpClient()
                .send(request, HttpResponse.BodyHandlers.ofByteArray());
        
        String responseContent = new String(httpResponse.body(), StandardCharsets.UTF_8);
        ```
        

## 二、Spring 提供

- `RestTemplate`（package: `org.springframework.web.client`）
    
    Spring框架提供的發送請求 API（僅限同步），針對 REST API 有提供許多預定義好的方法，也提供 `exchange()` method 來對應非 REST 模式的使用情境。另外可以指定要接收的response body 型別來自動轉換。

    HTTP 方法加上 `...ForObject()` 用於發送請求並回傳物件，`...ForEntity()` 則是回傳 `ResponseEntity`。
    - Get
        ```java
        RestTemplate restTemplate = new RestTemplate();
        
        User[] users = restTemplate.getForObject(URI.create(baseUrl + "/User"), User[].class);
        ```
        
    - Post
        ```java
        User newUser = new User("foo");
        
        ResponseEntity<User> response = restTemplate.postForEntity(URI.create(baseUrl + "/User"), newUser, User.class);
        ```
        
    - Exchange
        
        ```java
        int targetId = 5;
        ResponseEntity<User> response = restTemplate
                .exchange(baseUrl + "/User/" + targetId, HttpMethod.DELETE, null, User.class);
        ```
        
    
- `WebClient` （package: `org.springframework.web.reactive.function.client`）
    
    作為 Spring Web Reactive 的一部分，WebClient 提供非阻塞式、響應式的 HTTP 客戶端 API。回傳 `Flux` 或 `Mono` 物件。

    - Get
        
        `WebClient` 提供 builder，可以設定 header, cookie。
        
        ```java
        Flux<User> response = WebClient.builder()
              .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
              .build()
              .get()
              .uri(URI.create(baseUrl + "/User"))
              .retrieve()
              .bodyToFlux(User.class);
        ```
        
    - Post
        
        ```java
        User newUser = new User("foo");
        String requestContent = mapper.writeValueAsString(newUser);
        
        Mono<User> userMono = WebClient.create()
                .post()
                .uri(baseUrl + "/User")
                .contentType(MediaType.APPLICATION_JSON)
                .body(BodyInserters.fromValue(newUser))
                .retrieve()
                .bodyToMono(User.class)
                .onErrorMap(e -> new Exception("Error", e));
        
        System.out.println(userMono.block());
        ```
        

---

## 三、非同步請求

上述四種發送請求方式中，`HttpClient` 以及 `WebClient` 提供非同步的呼叫方式，可以用於同時發送多個請求以減少等待時間，或是直接將非同步物件回傳呼叫端，釋放出執行緒以接受更大量的請求。

-  `HttpClient` - `CompletableFuture`

使用 `sendAsync()` 以非同步方式發送請求，回傳 `CompletableFuture<HttpResponse<T>>` 物件。

```java
public abstract <T> CompletableFuture<HttpResponse<T>>
    sendAsync(HttpRequest request,
              BodyHandler<T> responseBodyHandler);
```

`CompletableFuture` 是代表非同步運算的回傳結果，加上 Listenable、Composible、Combinable 的特性。

   - Listenable
       
       提供註冊callback，當非同步運算執行結束後呼叫。
       
       ```java
       asyncTask.whenCompleteAsync((r, throwable) ->
       {
           if (throwable != null) {
               System.out.println(throwable.getMessage());
           } else {
               System.out.println("Second Task...");
           }
       });
       ```
       
   - Composible
       
       按照順序執行
       
       ```java
       CompletableFuture<Boolean> tasks = CompletableFuture
                   .supplyAsync(demo::firstTask)
                   .thenApplyAsync(demo::secondTask)
                   .thenApply(demo::thirdTask);
       ```
       
   - Combinable
       
       同時執行
       
       ```java
       CompletableFuture<Object> tasks = CompletableFuture.anyOf(task1, task2, task3);
       ```

- `WebClient` - `Mono`, `Flux`

Spring WebClient 作為 Spring Web Reactive 框架的一部分，`WebClient` 發送請求後會回應 `Mono` 或 `Flux` 類別。

`Mono`/`Flux` 是 Spring Reactive 架構中的核心，代表非同步的資料流，`Mono` 是有0-1個元素的序列，而 `Flux` 是0-N個。

在 Reactive 架構中，分為 Publisher 和 Subscriber，代表資料的發送及接收，且 Reactive 的特性是由 Subscribe 的事件進行驅動，代表沒有 Subscriber 就不會有資料被處理。

`Mono`/`Flux` 是 Publisher，代表沒有Subscriber就不會進行資料處理。 `subscribe()` 的基本寫法類似 `foreach()`，可以指定每一筆資料要進行什麼處理。

```java
Flux<Integer> ints = Flux.range(0, 5);
ints.subscribe(System.out::println);
// output:
// 0
// 1
// 2
// 3
// 4
```

但概念上傳統的作法是將序列的元素一個個 pull 出來進行操作，但在 Reactive 框架中，是由 Publisher 來 push 元素到 Subscriber 中進行操作，`Publisher.subscribe()`方法就是在定義每個元素被推送時要進行的操作。

```java
public interface Subscriber<T> {
    void onSubscribe(Subscription var1);

    void onNext(T var1);

    void onError(Throwable var1);

    void onComplete();
}
```

`Mono`/`Flux` 也有類似stream的操作例如 `filter()` 或是 `map()`，提供處理資料流的方式。

```java
flux.map(Post::getTitle)
    .sort(Comparator.comparing(String::length))
    .takeLast(1)
```

## 四、比較及結語

上述四種發送請求的方式除了是否支援非同步的差異外，在其他功能上也不盡相同，例如 HTTP/2 或是 GZip 的支援。但這邊聚焦在用法、同步/非同步上：

- `HttpURLConnection`:
    
    作為最早提供的API，`HttpURLConnection` 的優點在於不會有無法支援的版本。但支援的功能最少這個缺點也很明顯，寫法上先開起連線再設定method或header也比較不符合邏輯，算是目前各種方法中比較不被推薦使用的。
    
- `HttpClient`:
    
    `HttpClient` 提供了更多的功能支援，並有了非同步的方法。以 Request, Client, Response 為核心的使用邏輯也明確許多。若 Java 版本有支援到 11，又不希望使用第三方套件， `HttpClient` 就會是好選擇。
    
- `RestTemplate`:
    
    可惜 Spring framework 提供的 `RestTemplate` 僅支援同步，否則 `RestTemplate` 是本次介紹的四種發送請求方式中寫法最精簡的一種，尤其是目標API符合REST架構時可以發揮最強大功效。
    
    `RestTemplate` 目前是僅維護而不會有重大更新的狀態，未來應該會被 `WebClient` 完全取代。
    
- `WebClient`:
    
    `WebClient`不僅僅是非同步版的 `RestTemplate`，最大的特點在於響應式的設計模式。雖然有者最高的學習成本，但非阻塞的特性也是其他三種方式所沒有的。
    

總結來說，若使用 Java 11 以上，`HttpClient` 會是容易上手且功能豐富的請求發送方式。