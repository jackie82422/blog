---
title: golang 雜記 - goroutine
toc: true
categories: golang
date: 2022-10-26 15:31:10
tags: golang
---

golang - goroutine 雜記

<!-- more -->

goroutine大概是我接觸golang以前最常聽到人家稱讚golang的原因了，平心而論其實C#在後面對concurrency的處理也很不錯，絕大多數的狀況下都不用自己去管理thread而是交由dotnet runtime去管理thread pool，用的也是很開心，在處理parallelism也有一些不錯的函式庫可以使用。

到了golang重新複習一次這些東西。

在系統中每一個proccess都是program的實例，os會提供一些資源（ex memory)給予process，同時確保這些資源不會被其他的proccess給使用到。而一個process會由一個到多個的thread所組成，thread會共享process裡面的資源，而cpu可以同時處理一個或多個thread的指令。

在golang中goruntime會建立一些thread並且起啟動一個goruntine來處理程式，而程式中建立的所有thread會交給goruntime的scheduler來安排這些thread. 他的好處在於goruntime在操作thread的時候並不是從os底層開始做，而是由goroutine來處理。

他的速度會比os level的速度快，也更有記憶體效率，在thread的切換也更快，同時它也會自動做最佳化。

呼叫上方便到極致，

```go
func SayHelloWorld(s string) {
	for i := 0; i < 5; i++ {
		fmt.Printf("Hello World %v\n", s)
	}
}

func main() {
	go SayHelloWorld("0")
	SayHelloWorld("1")
	time.Sleep(time.Second * 1) 
}
```

只要在呼叫的函式上面加個`go`關鍵字就可以了。

跟其他語言一樣，能夠做到paralism是很強大的功能但同時也會因此出現一些奇奇怪怪的問題，最常見的是同時操作同一個記憶體位置導致程式沒有產出你預想的結果：

```go
func main() {
	var result []int
	var wg sync.WaitGroup

	wg.Add(100)
	for i := 0; i < 100; i++ {
		go func(i int) {
			defer wg.Done()
			result = append(result, i)
		}(i)
	}
	wg.Wait()
	fmt.Println(len(result))
	fmt.Println(result)
}
// output:
// 69
// [43 29 30 31 0 1 2 3 4 5 6 39 40 8 9 10 66 12 13 14 55 56 17 18 19 20 21 22 23 24 25 26 48 44 45 67 46 68 69 50 70 49 72 73 84 74 75 76 77 81 82 83 51 52 87 89 86 88 90 53 99 91 92 93 94 95 96 97 98]
```

所以有一句老話是

> don't communicate by sharing memory, share memory by communicating.

透過golang的channel可以使這個資料流變得清晰，channel是一個概念而golang自帶的lib有該概念的實做。

## Channel

channel需要一個sender和一個reader，在C#中有提供sender, reader等介面；在golang用箭頭operator來表示，channel同map/slice一樣是參考型態

```go
func main() {
	var channel = make(chan int)
	channel <- 1
	a := <-channel
	fmt.Println(a)
}
```

被寫入channel的值都只能被讀取一次，如果同時有多個goroutine再讀取亦同。

`channel <- 1`表示將1寫入channel中，而`a := <- channel`表示從channel中提取一個int並咐值給a。但這個範例這個操作是會出現error的，錯誤訊息：

> fatal error: all goroutines are asleep - deadlock!

這是因為對一個開啟的unbufferd channel寫入後，它會等待有另外一個goroutine進行讀取為止。而我們的範例是一個單線程的程序，當channel被寫入1後就handle住了。可以透過`go`開啟一個goroutine去處理。

```go
func main() {
	var channel = make(chan int)
	go func() {
		channel <- 1
	}()
	a := <-channel
	fmt.Println(a)
}
//output: 1
```

有unbufferd channel當然對立的也會有bufferd channel，建立的方法大同小異。

```go
func main() {
	var channel = make(chan int, 5)
	channel <- 1
	a := <-channel
	fmt.Println(a)
}
//output 1
```

用上bufferd chan會發現一樣的程式碼在unbufferd channel不能跑但現在卻可以了，這是因為unbufferd chan跟bufferd chan的一些行為是不一樣的。

寫入時：
| unbufferd channel	  | bufferd channel	  |
|  ----  			  | ----  			  |
| 暫停，直到有東西被讀取  | 當bufferd滿了，暫停 |

讀取時：
| unbufferd	channel	  | bufferd	channel	  |
| ---				  | ---				  |
| 暫停，直到有東西被寫入  | 如果沒東西就暫停	|

而chan是可以由sender去做關閉的，這很合理，producer才知道啥時已經沒有東西可以送了，而consumer則是有產品才自行決定要買還是不買。

```go
func CloseChanBySender(ch chan<- int) {
	close(ch)
}

func CLoseChanByReader(ch <-chan int) {
	close(ch)
}
// error
```
對於已經關閉的chan再進行關閉會發生panic，對已經關閉的chan進行send也會發生panic

```go
func main() {
	var channel = make(chan int, 5)
	close(channel)
	close(channel)
}
//panic: close of closed channel
```

這點不管bufferd chan or unbufferd chan都是一樣的，為了避免種狀況也可以用`ok`寫法處理這種狀況：

```go
func main() {
	var channel = make(chan int, 5)
	close(channel)
	v, ok := <-channel
	fmt.Println(v)
	fmt.Println(ok)
}
//output: 0
//output: false
```

應該很快會想到一件事情，當我有多個sender在執行時我要如何去避免寫入一個已經關閉的chan? `ok`寫法是從channel裡面提取值來判斷是不是空了，但對sender來說無法這樣做；尤其大多數語言都會建議使用者將sender跟reader分開給予function/method/... others.

golang提供了一個關鍵字`select`可以做到檢查的操作，當然它不只如此，先說明`select`的特性。

```go

func main() {
	ch := make(chan int, 5)
	fmt.Println(IsChannelClosed(ch))
	close(ch)
	fmt.Println(IsChannelClosed(ch))

}

func IsChannelClosed(ch <-chan int) bool {
	select {
	case <-ch:
		return true
	default:
	}

	return false
}
//output false
//output true
```

這個`select`關鍵字的結構跟`switch`很像，但它是for channel的，每個case都是針對chan做的操作，你也可以取得值之後做一些別的操作，例如`case a := <- ch`後續拿a找事。

對於每個case，select會判斷是否可以執行，如果有複數以上的case是可以執行的，則會隨機地從裡面抽一個case執行；如果沒半個可以執行它就會卡著等到有一任何一個可以執行，但如果有default case，則執行default case。

它用來解決[飢餓](https://zh.m.wikipedia.org/zh-hant/%E9%A5%A5%E9%A5%BF_(%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F))問題，但在這個範例中這恰好可以拿來替sender檢查該chan是否關閉。

第一個case我們從chan中取值，如果可以取值則回傳`true`。

但老實說這個方法並不是好的，第一個是顯而易見的，當我執行了`case <- ch`同時代表我將值給取走了，資料不連續了。其次是就算知道chanel is not closed也只是在這個當下not closed，在下一個瞬間它可能就closed了，這個判斷並不足以讓你可以安全的避掉panic發生。

一個通用的規則是如果你確定這個sender是最後一個sender，那你可以安心關閉channel，如果你有多個sender則你不應該隨意的關閉channel。

但通用規則常常就是拿來打破的，我可以很容易想到很多個情境說明我必須關閉channel，縱使這個channel擁有多個sender.

最簡單的方式應該是用`recover()`函數，這個函數可以在觸發panic之後讓goroutine重新接管程序，你可以當作C#的final block，無論exception是啥，它最後都會執行這個final block。

```go
func main() {
	ch := make(chan int, 5)
	result, err := sendData(ch, 1)
	fmt.Println(result)
	fmt.Println(err)
	close(ch)

	result, err = sendData(ch, 1)
	fmt.Println(result)
	fmt.Println(err)
}

func sendData(ch chan<- int, data int) (result bool, err error) {
	defer func() {
		if recover() != nil {
			err = fmt.Errorf("send on closed channel")
			result = false
		}
	}()

	ch <- data
	result = true
	err = nil
	return result, err
}
// output:
// true
// <nil>
// false
// send on closed channel
```

> 建議參考這篇，有更多詳細的說明和方法來處理這類的事情 - [How to Gracefully Close Channels
](https://go101.org/article/channel-closing.html)

但老實說有些狀況下用channel真的是有點... 雖然資料流變簡單了，但程式碼複雜了。以第一個例子為例，單純append數字到slice裡面，如果要改寫成channel的模式的話

```go
func main() {
	const totalCount = 100
	ch := make(chan int)
	var dataSet = make([]int, 0, 100)
	go AddNum(ch, totalCount)

	for x := range ch {
		dataSet = append(dataSet, x)
	}

	fmt.Println(dataSet)
	fmt.Println(len(dataSet))
	//output:
	// [28 43 29 30 31 32 33 34 35 36 37 38 39 40 41 42 61 51 52 53 54 55 56 57 58 59 60 17 0 1 2 3 4 5 6 7 8 9 18 19 20 21 22 50 44 45 46 47 48 49 13 10 11 12 24 25 26 27 99 76 77 78 79 80 81 82 23 83 84 92 91 85 86 87 88 89 90 93 68 94 96 97 98 62 63 64 65 66 95 67 14 15 16 71 69 70 75 74 73 72]
	// 100
}

func AddNum(ch chan<- int, num int) {
	wg := sync.WaitGroup{}
	wg.Add(num)
	for i := 0; i < num; i++ {
		go func(val int) {
			defer wg.Done()
			ch <- val
		}(i)
	}
	wg.Wait()
	close(ch)
}
```

如果只是單純讀取和寫，而且不會對該對象做太多操作，不需要去了解操作對象的goroutine屬於哪一支時，用`mutex`會簡單非常多。

```go
func main() {
	const totalCount = 100
	var dataSet = make([]int, 0, 100)
	lock := sync.Mutex{}
	for i := 0; i < totalCount; i++ {
		go func(val int) {
			lock.Lock()
			defer lock.Unlock()
			dataSet = append(dataSet, val)
		}(i)
	}

	time.Sleep(time.Second)
	fmt.Println(dataSet)
	fmt.Println(len(dataSet))
}
```
這邊的time.Sleep只是單純因為開啟其他goroutine後主程序先跑完了，所以強迫它等一下等其他routine都完成作業；這是個簡單的例子，實務上這種情境是不應該另外開goroutine去處理的。