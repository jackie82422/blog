---
title: 包一個Benchmark container.
toc: true
categories: C#
date: 2021-06-08 06:24:19
tags: [C#,Docker,Tools]
---

雲環境加上Docker Container部屬，那我本機跑Benchmark哪會準呢？乾脆我把bechmark包進Contianer，然後再雲環境直接做效能檢測吧？

起心動念來弄一下，順便紀錄一下這件事情該怎麼做。

<!-- more -->

# Introduction

## BenchmarkDotNet

Benchmark是一個用來評估函數執行時間的工具，如果你厭倦了StopWatch + Console.WirteLine的無限循環，可以考慮用Benchmark來解決這些麻煩事；同時它可以做到重複測試，並提供了許多統計指標讓你評估受測函數的性能，測試結果也可以用多種方式儲存，寫報告找不到靈感？Benchmark直接糊他一頁，整個專業度都變高了。

這個套件同時有 .Net Foundation的支持，廣大的社群確保了你不會有出了問題找不到人求助的窘境。

<a href="https://github.com/dotnet/BenchmarkDotNet">BenchmarkDotNet Github</a>


# Implement

我們採用最簡單的例子，來自官方的字串編碼做測試案例。

## Install Benchmark

安裝Benchmark可以透過以下指令
> dotnet add \<project Name> package benchmarkdotnet -v \<version>

![或者透過GUI去安裝](A.png)

## Sample Code 

```cs
public class Md5VsSha256
{
    private SHA256 sha256 = SHA256.Create();

    private MD5 md5 = MD5.Create();

    private byte[] data;   

    [Params(1000, 10000)]
    public int N;    
    
    [GlobalSetup]
    public void Setup()
    {
        data = new byte[N];
        new Random(42).NextBytes(data);
    }    
    
    [Benchmark]
    public byte[] Sha256() => sha256.ComputeHash(data);    
    
    [Benchmark]
    public byte[] Md5() => md5.ComputeHash(data); 
}
```

這串sample code在官網的範例上就有，原文照著貼而已，但拿掉了一些針對不同framework的測試，只去測試專案使用的framework版本，這邊是用net core 3.1

接著在程式的進入點加入這行

```cs
static void Main(string[] args)
{
    BenchmarkRunner.Run<Md5VsSha256>();
}
```

這樣就會針對Md5VsSha256這個Class去做測試，會尋找這個class裡面帶有[Benchmark]標籤的方法測試。

接著將執行方式改為Release
![](B.png)

按下執行，執行結束後就可以看到執行結果了。
![](C.png)

Release後的檔案也可以透過以下指令去做執行

> dotnet \<projectName>.dll

![](C2.png)

說穿了我們也是用這種方式寫在Dockerfile去包成Image而已。

這樣一來關於程式的部分就結束了，接著來處理Dockerfile.

## Dockerfile

在Project目錄下新增Dockerfile

內容如下
```
FROM mcr.microsoft.com/dotnet/sdk:3.1
WORKDIR /src
COPY . .
RUN dotnet restore
RUN dotnet build BenchmarkWithDocker.csproj
RUN dotnet publish BenchmarkWithDocker.csproj -c Release -o /src/bin/publish
WORKDIR /src/bin/publish
ENTRYPOINT ["dotnet", "BenchmarkWithDocker.dll]
```

接著執行
> docker build -t benchmarktest .

包好image之後可以選擇上傳到私有的dockerhub或者任何地方，在受測環境Pull下來後執行

> docker run benchmarktest

就可以得到一樣的統計報表了
![](E.png)

當然你也可以改成輸出成html報表，只要掛volumn出來即可。