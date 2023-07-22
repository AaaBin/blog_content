+++
title = "Postgresql Benchmark"
date = '2023-04-23T20:37:43+08:00'
draft = false
summary = '要針對 PostgreSQL Server 做基準測試的話，可以選擇內建的 `pgbench` 工具，大略筆記一下使用 `pgbench` 的方式、以及實際使用情境，測試使用 Load Balancing 將附載分配到兩台 PostgreSQL Server，比較與單一 PostgreSQL Server 的差距。'
tags = ["PostgreSQL", "pgbench"]
+++
要針對 PostgreSQL Server 做基準測試的話，可以選擇內建的 `pgbench` 工具，大略筆記一下使用 `pgbench` 的方式、以及實際使用情境，測試使用 Load Balancing 將附載分配到兩台 PostgreSQL Server，比較與單一 PostgreSQL Server 的差距。
___

### `pgbench` 簡介

`pgbench` 是安裝 PostgreSQL 時會一起被安裝的小程式，用於對 PostgreSQL Server 做基準測試。

```text
pgbench [option...] [dbname]
```

使用上跟 `psql` 一樣，可能會需要指定連線的主機名稱、阜號或使用者，這些連線選項用法與 `psql` 是一樣的：

```shell
pgbench -h 10.0.1.100 -p 5678 -U developer postgres
```

`pgbench` 內建有測試腳本，要使用內建腳本需要先使用 `-i` 選項，針對要測試的資料庫進行初始化：

```shell
pgbench -h 10.0.1.100 -U developer -i postgres
```

也可以透過 `-s` *`scale_factor`* 來指定要生成多少倍的測試資料，預設是 1 ，會生成 100000 筆：

```shell
$ pgbench -h 10.0.1.100 -U developer -i -s 20 postgres

# 輸出結果：
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
2000000 of 2000000 tuples (100%) done (elapsed 22.38 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 36.58 s (drop tables 0.05 s, create tables 0.11 s, client-side generate 22.78 s, vacuum 6.92 s, primary keys 6.72 s).
```

在初始化完成後就可以使用內建腳本進行基準測試：

使用 `-b` *`scriptname[@weight]`* 這個選項可以指定使用的內建測試腳本，有 `tpcb-like` 、 `simple-update` 和 `select-only` 三種，若沒有指定則會預設使用 `tpcb-like`。

`@weight` 指的是執行權重，預設是 1。

```shell
$ pgbench -h 10.0.1.100 -U developer -T 30 -j 2 -c 20 -P 5 -M prepared postgres

# 輸出結果：
pgbench (14.6 (Debian 14.6-1.pgdg110+1))
starting vacuum...end.
progress: 5.0 s, 1576.6 tps, lat 12.404 ms stddev 3.566
progress: 10.0 s, 1205.4 tps, lat 16.108 ms stddev 35.918
progress: 15.0 s, 419.6 tps, lat 48.154 ms stddev 84.108
progress: 20.0 s, 362.8 tps, lat 56.091 ms stddev 86.328
progress: 25.0 s, 323.6 tps, lat 60.920 ms stddev 114.580
progress: 30.0 s, 403.4 tps, lat 50.102 ms stddev 91.868
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 20
query mode: prepared
number of clients: 20
number of threads: 2
duration: 30 s
number of transactions actually processed: 21477
latency average = 27.853 ms
latency stddev = 61.904 ms
initial connection time = 101.683 ms
tps = 717.207182 (without initial connection time)
```

使用到的參數：

* `-T` *`seconds`* 指定測試要執行多久的時間。
* `-j` *`threads`* 指定執行基準測試所使用的執行續數量，預設為 1。
* `-c` *`clients`* 指定要模擬多少使用者，也就是同步執行的 session 數量，預設為 1。
* `-P` *`sec`* 指定每幾秒顯示執行進度的報告。
* `-M` *`querymode`* 指定執行的模式，有 `simple`、`extended` 和 `prepared` 三種。

除了內建腳本之外，也可以使用自訂的腳本進行測試，使用 `-f` *`filename[@weight]`* 這個選項：

```shell
$ pgbench -h 10.0.1.100 -U developer -T 30 -P 5 -j 2 -c 20 -b select-only@1 -f custom-script.sql@3  postgres

# 輸出結果：
pgbench (14.6 (Debian 14.6-1.pgdg110+1))
starting vacuum...end.
progress: 5.0 s, 2772.9 tps, lat 6.953 ms stddev 28.840
progress: 10.0 s, 2929.0 tps, lat 6.821 ms stddev 28.608
progress: 15.0 s, 3542.1 tps, lat 5.657 ms stddev 26.976
progress: 20.0 s, 4050.2 tps, lat 4.938 ms stddev 25.173
progress: 25.0 s, 5362.8 tps, lat 3.708 ms stddev 21.746
progress: 30.0 s, 8634.2 tps, lat 2.299 ms stddev 15.844
transaction type: multiple scripts
scaling factor: 20
query mode: simple
number of clients: 20
number of threads: 2
duration: 30 s
number of transactions actually processed: 136475
latency average = 4.387 ms
latency stddev = 23.263 ms
initial connection time = 116.107 ms
tps = 4536.915483 (without initial connection time)
SQL script 1: <builtin: select only>
 - weight: 1 (targets 25.0% of total)
 - 34159 transactions (25.0% of total, tps = 1135.566924)
 - latency average = 16.393 ms
 - latency stddev = 44.368 ms
SQL script 2: custom-script.sql
 - weight: 3 (targets 75.0% of total)
 - 101368 transactions (74.3% of total, tps = 3369.833659)
 - latency average = 0.373 ms
 - latency stddev = 0.639 ms
```

這邊使用了 `-b` 指定內建的 `select-only` 腳本，使用 `-f` 指定了 `custom-script.sql` 這個檔案作為腳本。並設定 `weight` 按照 1 比 3 的比例執行。

自定義的腳本有一些語法、函式可以使用，詳細內容參考[文件](https://www.postgresql.org/docs/14/pgbench.html#custom-scripts:~:text=SELECT%20is%20issued.-,Custom%20Scripts,-pgbench%20has%20support)。
___

### 實際測試

測試目標：了解在單純查詢（`select-only`）的情形下，單一 PostgreSQL Server 與兩台做 Load Balancing 的差距。

簡易架構：基準測試執行環境以及兩台 PostgreSQL Server 都使用 2vCPU 8GB RAM 的 VM。

首先是針對單一 Server 做測試

```shell
$ pgbench -h 10.0.1.100 -U developer -T 60 -P 5 -j 2 -c 20 -b select-only postgres

# 輸出結果（省略部分）：
transaction type: <builtin: select only>
scaling factor: 30
query mode: simple
number of clients: 20
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 831628
number of failed transactions: 0 (0.000%)
latency average = 1.437 ms
latency stddev = 0.334 ms
initial connection time = 91.408 ms
tps = 13880.704271 (without initial connection time)
```

可以看到針對單一 PostgreSQL Server 做測試的結果，使用 `select-only` 的腳本的 tps 約 13000-14000。
接著透過 Load Balancer 讓查詢請求可以分配到兩台 PostgreSQL Server：

```shell
$ pgbench -h 10.0.1.2 -U developer -T 60 -P 5 -j 2 -c 20 -b select-only postgres

# 輸出結果（省略部分）：
transaction type: <builtin: select only>
scaling factor: 30
query mode: simple
number of clients: 20
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 1685231
number of failed transactions: 0 (0.000%)
latency average = 0.702 ms
latency stddev = 0.412 ms
initial connection time = 85.166 ms
tps = 28126.510289 (without initial connection time)
```

可以看到，在同等條件下，將查詢分配到兩台 Server 上能夠獲得超過兩倍的 tps 提升。
___

### 其他

執行 `pgbench` 時還有很多不同的選項，例如指定 transaction 的執行速率，或是設定每個 client 執行多少次而不是限制執行時間，初始化的步驟也有不同選項可以指定。

___

### 參考資料

* [pgbench](https://www.postgresql.org/docs/14/pgbench.html)
___
