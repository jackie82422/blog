---
title: 試試在Net core上用AWS TimeStream?
toc: true
date: 2021-06-03 23:02:38
categories: AWS
tags: [AWS,C#]
---


## Introduction

AWS TimeStream 是一個時序型資料庫，跟其他的時序型資料庫（influxdb, Prometheus）用途差不多，主要是用來處理帶時間標籤，具有依照時間順序變化的資料，例如machine log.
<!-- more -->

目前(2021/06/03)TimeStream可使用的區域較少，僅以下五個
1. 美國東部(維吉尼亞州北部) us-east-1
2. 美國東部(俄亥俄) us-east-2
3. 美國西部(奧勒岡) us-west-2
4. 歐洲(法蘭克福) eu-central-1
5. 歐洲(愛爾蘭) eu-west-1

可以預想得到的如果在台灣使用Tokyo Regin去做資料增修的話得跨半個地球，詳細的測速資料文末會提供。

在創建上設計得很簡單，有兩種設定可以選擇
1. Standard Database
2. Sample Database

兩者的差異在是否可以對TimeStream的資料做加密。

## Sample Code

要使用Aws TimeStream，需要添加幾個package

```cs
dotnet add package AWSSDK.Core
dotnet add package AWSSDK.TimestreamWrite
dotnet add package AWSSDK.TimestreamQuery
```

在連接TimeStream的時候可以採用將credentials放在user資料夾的方式去取得，這樣就不必特別在code裡面補上aws_secret_access_key.

最基本的設定如下，其中RegionEndPoint是看你的TimeStream開在哪裡。


```cs
_amazonTimeStreamWriteClient = new AmazonTimestreamWriteClient
(new AmazonTimestreamWriteConfig
            {
                Timeout = TimeSpan.FromSeconds(2), 
				RegionEndpoint = RegionEndpoint.EUWest1, 
				MaxErrorRetry = 3
            });
```

透過AmazonTimeStreamWriteClient這個物件可以輕鬆的創建database, table, description..etc

例如Create Database的方式是這樣:

```Cs
/// <summary>
/// CreateDatabase.
/// </summary>
/// <param name="databaseName"></param>
/// <returns></returns>
public async Task<bool> CreateDatabase(string databaseName)
{
    var createDatabaseRequest = new CreateDatabaseRequest {DatabaseName = databaseName};
    var result = await _amazonTimeStreamWriteClient.CreateDatabaseAsync(createDatabaseRequest);
    return result.HttpStatusCode == HttpStatusCode.OK;
}
```

Create Table如下，參數的部分可以設定硬碟儲存多少天，TimeStream是一個時序型資料庫，大多情境下這種資料庫的資料是有效期的，譬如說記錄的是機器log，如果去看一年前的機器log基本上是沒什麼幫助的。

而table這邊沒有特別去設定property，可以當做timestream也類似於一個dynamic db.

```cs
/// <summary>
/// create table.
/// </summary>
/// <param name="databaseName"></param>
/// <param name="tableName"></param>
/// <returns></returns>
public async Task<bool> CreateTable(string databaseName, string tableName)
{
    var createTableRequest = new CreateTableRequest
    {
        DatabaseName = databaseName,
        TableName = tableName,
        RetentionProperties = new RetentionProperties
        {
            // 硬碟儲存多久
            MagneticStoreRetentionPeriodInDays = 7,
            // Memory儲存多久
            MemoryStoreRetentionPeriodInHours = 24
        }
    };
    var response = await _amazonTimeStreamWriteClient.CreateTableAsync(createTableRequest);
    return response.HttpStatusCode == HttpStatusCode.OK;
}
```

試著insert一筆資料
```cs
/// <summary>
/// insert data
/// </summary>
/// <param name="database"></param>
/// <param name="tableName"></param>
/// <returns></returns>
public async Task<bool> InsertData(string database, string tableName)
{
    var now = DateTimeOffset.UtcNow;
    var currentTimeString = now.ToUnixTimeMilliseconds().ToString();
    var dimensions = new List<Dimension>
    {
        new Dimension {Name = "RowData", Value = "TestData"},
        new Dimension {Name = "ClientId", Value = "ChachaLin"},
    };
    var records = new List<Record> {new Record
    {
        Dimensions = dimensions,
        MeasureName = "cou temp",
        MeasureValue = "100",
        MeasureValueType = MeasureValueType.BIGINT,
        Time = currentTimeString
    }};
    var writeRecordsRequest = new WriteRecordsRequest
    {
        DatabaseName = database, TableName = tableName, Records = records
    };
    var response = await _amazonTimeStreamWriteClient.WriteRecordsAsync(writeRecordsRequest);
    return response.HttpStatusCode == HttpStatusCode.OK;
}
```

資料的結構是aws提供的

首先定義dimension（維度）需要哪些東西，然後定義record的內容

Record的內容固定要有Dimensions, MeasureName, MeasureValue, MeasureValueType, Time

因為這類db處理的問題主要是log，機器log, Sensor Log..etc，Measure（度量）特定的指標隨時間的變化是這類db被設計來使用的主要因素，除此之外才去記錄這個指標的其他維度（dimesion）。

## Performance

以Insert為例，資料大小小於1kb的情況下利用docker在EC2(Tokyo Region)連接愛爾蘭的Timestream Db，透過bechmark蒐集到的數據如下:
![](performance.png)
