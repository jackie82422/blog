---
title: channel
toc: true
categories: C#
date: 2022-09-20 19:46:31
tags: C#
---


# Preface

最近在做某公司的面試試題，用到了System.Threading.Channels這個物件，覺得還蠻有趣的，這次來一起玩一下這個Channel.

<!-- more -->

# Introduction

Channel是net core 2.1時加入的feature，他在net core 2.1需要透過nuget安裝 `dotnet add package System.Threading.Channels`；在net core 3.1以後的版本(包括net 5, net 6)則是內建在框架之中了。

那Channel是甚麼呢？其實他是一個用於解決Producer/Consumer問題的實用feature.

我們先定義甚麼叫做Producer/Consumer，快速的理解這個設計模式是甚麼，它解決了甚麼問題。

## Producer Consumer Pattern

```mermaid
graph LR;
    Producer --> Queue
    Queue --> Consumer
```

想像你在參加一個大胃王比賽，比賽吃熱狗，你瘋狂的吃而廚師瘋狂的烤熱狗，廚師不會等你吃完才開始做下一條，因為你會靠北上菜太慢。所以廚師一次可能烤好幾條，同時上好幾條給你，而你不可能同時吃好幾條因為你會噎死，所以烤好但還沒吃的就先放在桌上不動。

這一系列的動作就是Producer Consumer Pattern想做的事情，一個/或多個廚師(Producer)烤熱狗給你(Consumer)或其他參賽者(Consumers)吃，烤了還沒吃的就先放在桌上(Queue)。

同時它會衍伸個問題：桌子並不是無限大的，烤得太快或者吃得太慢都會塞滿桌子，在等待被吃掉的過程中慢慢變冷，啊冷掉的熱狗一定爆幹難吃。

這時候廚師就會有幾個選擇

1. 我等到桌子開始有空間，我再開始繼續生產熱狗。
2. 我把最一開始上的熱狗給扔了，放剛烤好的，熱熱吃最好吃。
3. 我把最後的熱狗給扔了，放剛烤好的，滿頭問號的操作但也是一種策略。
4. 我把我剛做好的熱狗給扔了 (???)

而這些策略在Channel上都有實作，你只需要做選擇。

# What is Channel?

Channel的主要目的就是已實做的Producer Consumer Pattern，特別適合多個線程之間交換資料和併發操作，同時底層它用來存儲資料的儲存體有做了針對多執行緒的處理，因此它保障了Thread-Safe和FIFO(First in first out)等特性(但多線程的狀況下先被處理的可不見得先完成...)

> UnBoundedChannel底層使用ConcurencyQueue作為儲存體

> BoundedChannel底層使用Deque(Doblue end queue) + Monitor Lock作為儲存體

## Create Channel

如何創建Channel ?

Channel分為兩種，`UnboundedChannel`以及`BoundedChannel`，兩者差別就是用來區別暫存資料的存儲體是否有限制資料的數量上限而已。在一些情境下Consumer消耗資料的速度可能不夠快，如果無限堆積下去最後會造成Memory out of bound. 而限制Bounded的狀況就要另外考慮當資料來到設定的上限時該怎麼處理。

創建Channel可以簡單地透過Channel內置Provider來取得

```C#
 var boundedChannel = Channel.CreateBounded<T>(options: new BoundedChannelOptions(capacity: 30)
 {
     SingleWriter = false, // default is false
     SingleReader = false, // default is false 
     AllowSynchronousContinuations = false, // default is false
     FullMode = BoundedChannelFullMode.Wait
 });

 var unBoundedChannel = Channel.CreateUnbounded<T>(new UnboundedChannelOptions()
 {
     SingleWriter = false, // default is false
     SingleReader = false, // default is false 
     AllowSynchronousContinuations = false, // default is false
 });
 ```

> `BoundedChannelOptions`和`UnboundedChannelOptions`都繼承自`ChannelOptions`，`BoundedChannelOptions`僅是多了`FullMode`參數，而`UnboundedChannelOptions`所有參數都來自繼承的`ChannelOptions`。

BoundedChannel的`FullMode`參數定義了當資料達到限制的上限時應該做甚麼動作，有四者可以選擇。

```C#
    public enum BoundedChannelFullMode
    {
        /// <summary>Wait for space to be available in order to complete the write operation.</summary>
        Wait,
        /// <summary>Remove and ignore the newest item in the channel in order to make room for the item being written.</summary>
        DropNewest,
        /// <summary>Remove and ignore the oldest item in the channel in order to make room for the item being written.</summary>
        DropOldest,
        /// <summary>Drop the item being written.</summary>
        DropWrite
    }
```
> 四種操作如上所示

## Insert data to Channel

不論是BoundedChannel還是UnBoundedChannel，都是在呼叫內部的`ChennalWriter`屬性後呼叫以下方法：
1. TryWrite
2. WriteAsync
3. WaitToWriteAsync

雖然兩邊都是用一樣的方法，但BoundedChannel實際上的行為會複雜許多。

例如`TryWrite`，在BoundedChannel上除了一些基礎判斷以外，還另外做了`Monitor` Lock住要被寫入的物件，針對我們選擇的`FullMode`做了不同的處理。這邊展開可能會有點長，還是以入門為主；之後有空再深入Source Code(?)

但如果你只是要Write data to channel，用上WriteAsync就足矣；不管是`UnBoundedChannel`或是`BoundedChannel`都是如此，原因我們看一下源碼馬上就有答案。

不管是`UnBoundedChannel.Writer` or BoundedChannel.Writer 實際都是依賴於同一個抽象類別`ChannelWriter<T>`，細節再各自繼承去展開。在`WriteAsync`函數中是這樣寫

```C#
public virtual ValueTask WriteAsync(T item, CancellationToken cancellationToken = default)
{
    try
    {
        return
            cancellationToken.IsCancellationRequested ? new ValueTask(Task.FromCanceled<T>(cancellationToken)) :
            // ###### here ######
            TryWrite(item) ? default :
            // ###### here ######
            WriteAsyncCore(item, cancellationToken);
    }
    catch (Exception e)
    {
        return new ValueTask(Task.FromException(e));
    }
}
```

第一個`TryWrite`在這邊了，所以不管你是不是用`TryWrite`還是`WriteAsync`在判斷外部是不是有傳入`CancellationToken`之後其實第一個跑的都是`TryWrite`。但要注意的是`TryWrite`回傳的是`bool`而`WriteAsync`只是回傳`TaskValue`，你只能知道`WriteAsync`結束了，但並不知道到底值是不是你這個行為傳進去的，也有可能是由其他Writer寫入的。

接著到`WriteAsyncCore`

```
C#
private async ValueTask WriteAsyncCore(T innerItem, CancellationToken ct)
{
    while (await WaitToWriteAsync(ct).ConfigureAwait(false))
    {
        if (TryWrite(innerItem))
        {
            return;
        }
    }
    throw ChannelUtilities.CreateInvalidCompletionException();
}
```

算是相當簡單粗暴了，如果我寫不進去，我就繼續寫他直到我寫進去為止。而與Write相關的第三個函數也在這邊了`WaitToWriteAsync`。所以實際`WriteAsync`在跑的時候三個函數都是調用的，但知道了它內部是如何調用之後就可以針對自己的業務邏輯做一些before write, after write之類的操作，而在Single Producer情況下就是完全無腦用`WriteAsync`就可以了。

```C#
var producer = Task.Run(async () =>
{
    for (var i = 0; i <= 100; i++)
    {
        await unBoundedChannel.Writer.WriteAsync(i);
        Console.WriteLine($"Insert value to channel, value is {i}");
    }
});
```

如果已經將所需要寫入的資料都寫入完了，且後續不會再寫入；那可以加入`Writer.Complete()`讓其他Producer知道這個Channel Writer的工作已經結束了。後續有Producer嘗試寫入的時候便會自動終止。


## Read data from Channel

在Reader中一樣提供了四種的跟讀取有關的函數：

1. ReadAsync
2. ReadAllAsync
3. TryRead
4. WaitToReadAsync

這邊的整個結構跟上面Write的結構很相似，一樣在做`ReadAsync`的時候會先`TryRead`，失敗後會去做`WaitToReadAsync`跑While直到取得值，或者收到Cancel Tone或者儲存體裡已經沒有值。

因為很相像，這邊就不另外展開；但在這基礎上多了一個函數`WaitToReadAsync`，先留著等等再說。

跟Writer的概念一樣，我們再多數情境下可以直接使用`ReadAsync`，除非有甚麼特殊的處理那可以透過其他函數去實作`ReadAsync`的內容並加上自己的業務邏輯。

所以可以這樣寫：

```C#
var consumer = Task.Run(async () =>
{
    while (unBoundedChannel.Reader.WaitToReadAsync() != default)
    {
        var result = await unBoundedChannel.Reader.ReadAsync();
        Console.WriteLine($"get value from channel, value is {result}");
    }
```});

其中While的Condition我這邊是直接用`WaitToReadAsync`處理，只要是true便代表儲存體裡面仍有資料。

## **或者**

你可以使用`ReadAllAsync`所回傳的`IAsyncEnumerable<T>`來操作，雖然它一樣後面是做了`WaitToReadAsync`，但搭配C#8的`await IAsyncEnumerable<T>`比較酷炫是不是！

```C#
var consumer = Task.Run(async () =>
{
    await foreach (var item in unBoundedChannel.Reader.ReadAllAsync())
        Console.WriteLine($"get value from channel, value is {item}");
});
```

最後完整的Code如下：

```C#
var unBoundedChannel = ChannelCreateUnbounded<int>(new UnboundedChannelOptions()
{
    SingleWriter = false, // default is false
    SingleReader = false, // default is false 
    AllowSynchronousContinuations = false, // default is false
});

var producer = Task.Run(async () =>
{
    for (var i = 0; i <= 100; i++)
    {
        await unBoundedChannel.Writer.WriteAsync(i);
        Console.WriteLine($"Insert value to channel, value is {i}");
    }
});

var consumer = Task.Run(async () =>
{
    await foreach (var item in unBoundedChannel.Reader.ReadAllAsync())
        Console.WriteLine($"get value from channel, value is {item}");
});
await Task.WhenAny(producer,Task.Delay(-1)).ContinueWith(_ => unBoundedChannel.Writer.Complete());
await consumer;
```
Console Result :
```cmd
...
Insert value to channel, value is 68
get value from channel, value is 10
Insert value to channel, value is 69
get value from channel, value is 11
get value from channel, value is 12
Insert value to channel, value is 70
get value from channel, value is 13
Insert value to channel, value is 71
Insert value to channel, value is 72
...
```
# conclusion

Channel大概就介紹到這邊了，在處理類似的問題還有一個`BlockingCollection`在dotnet framework 4之後加入的，而Channel是在Net Core 2.1之後加入，執行的速度略高於`BlockingCollection`而且在使用上也蠻容易使用的。

如果有遇到類似的情境不彷使用看看，蠻有趣的。