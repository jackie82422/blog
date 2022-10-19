---
title: golang 雜記
toc: true
categories: golang
date: 2022-10-19 13:33:59
tags: golang
---

block 相關雜記

<!-- more -->


## Block

與c#不同，當子block內用的變數與外部block變數名稱一致時，內部的變數會遮蔽（shadow）外部區塊建立的變數。

```go
func main(){
    x := 10
    if x > 5 {
        fmt.Println(x)
        // output 10
        x := 5
        fmt.Println(x)
        // output 5
    }
    fmt.Println(x)
    // output 10
}
```

與C#比較，C#的子block基本就不能命名的跟外部的變數名稱一樣，所以參考的還是同一個記憶體位址。

```C#
public class Program
{
	public static void Main()
	{
		var x = 10;
		if(true){
			Console.WriteLine(x);
            // output 10
			x = 5
			Console.WriteLine(x);
            // output 5
		}
		Console.WriteLine(x);
        // output 5
	}
}
```

所以golang在宣告相同變數的時候基本是產了兩個不同的記憶體位置來存儲。同時也說明`:=`的宣告式在程式碼閱讀上在一些情景並不是很適合。

在比較一段程式碼：

```go
func main(){
    x := 10
    if x > 5 {
        fmt.Println(x)
        // output 10
        x = 5
        fmt.Println(x)
        // output 5
    }
    fmt.Println(x)
    // output 10
}
```

這邊的`x`在if block裡面就不是在宣告一個，而是拿上層block的`x`；所以站在閱讀性上需要特別注意這點，避免內外層宣告相同名稱變數，在C#這邊編譯器會檢查並且返回編譯失敗，在Go則是合法的。

甚至包括外部引入的package也可以遮蔽

```go
func main(){
    fmt.Println("Hello world")
    fmt := "oops";
    fmt.Println("Hello world")
    // error : fmt.Println undefined (type string has no field or method Println)
}
```

但所幸在golang也有提供工具可以提早檢查lint，可參考
[vet](https://pkg.go.dev/cmd/vet)

可是關鍵字的遮蔽連lint都檢查不到，要特別注意

``` go
fmt.Println(true)
//output true
true := 10
fmt.Println(true)
//output 10
```


## for

golang沒有while, while do，取而代之都是以for操作。

### for-range

```go
evenVals := []int{1,3,5,7,9}
for i,v := range evenVals{
    fmt.Println(i,v)
}
//output 0,1
//output 1,3
//output 2,5
//output 3,7
//output 4,9
```

特別注意的是`for-range`所取得的value是call by value的，對其值的更改不會影響到原有的結構體。

另外`for-range`在處理字串時golang自己做了一些特別的操作，上篇文說到字串是byte陣列組成，但在做`for-range`時的對象並不是[]byte，而是[]rune。

```go
func main() {
	s := "哈囉狗練"
	for _, v := range s {
		fmt.Println(string(v))
	}

    // output
    // 哈
    // 囉
    // 狗
    // 練
}
```

## switch

跟大多數語言差不多，差別在複數case要做同一個處理時，用`,`分開condition而不是多個case。

case block沒有內容就啥都不處理，然後有個空switch可以不放要判斷的變數，不過大多數情況這樣可讀性不太高，慎用。

```go
func main() {
loop:
	for i := 0; i < 10; i++ {
		switch {
		case i == 1, i == 2, i == 3:
			fmt.Println("1 <-> 3")
		case i == 4:
			fmt.Println("4")
		case i == 5:
		case i == 9:
			fmt.Println("break loop")
			break loop
		default:
			fmt.Println(i)

		}
	}
}

// 0
// 1 <-> 3
// 1 <-> 3
// 1 <-> 3
// 4
// 6
// 7
// 8
// break loop
```


