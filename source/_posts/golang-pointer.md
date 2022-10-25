---
title: golang 雜記 - pointer
toc: true
categories: golang
date: 2022-10-21 17:15:52
tags: golang
---

雜記

<!-- more -->

位址運算子`&`可以取得變數地址。

```go
func main() {
	x := 10
	fmt.Println(x)
	fmt.Println(&x)

    var y &int

}
// output 10
// output 0xc0000220d8
```

間接運算子`*`可以從指標型態的變數取得指標所指的值。

```go
func main() {
	var x int
	x = 10
	pointerOfX := &x
	fmt.Println(*pointerOfX)
}
// output 10
```

`*`同時也是用來標示變數為指標變數的定義子，例如：

```go
func Sample(intput *int){
    //
}

func main(){
    a := new(int)
    Sample(a)
}
```

內建函數`new()`可以建立一個任意型態的指標回傳。

```go
func main() {
	x := new(int)
	*x = 10
	fmt.Println(x)
	fmt.Println(*x)
}
// output 0xc0000bc000
// output 10
```

golang在建立函式並傳遞變數給函式時，實質是複製了一個該變數的副本給函式使用；某種意義上這算是一種不可變變數的實現方式，取而代之的是如果該值需要在函式中可變，那golang提交的解答是給予一個指標變數過去讓函式可以透過間接運算子讀值/改值，雖然說傳遞指標變數給函式也是複製了一個指標變數，但兩個指標變數都指向同一個指標位置。

我覺得這是golang一個很大很大的優點，如果學過其他語言（例如C#）就會看到有些東西是call by value, 有些是call by ref。有時候它是在函數裡可變的，有時候不行；加上某些語法糖讓需要背的例子越來越多；golang簡化了一切，永遠是call by value，所有變數進入函數都會複製一份，包括了指標。

```go
func main() {
	x := new(int)
	fmt.Println(*x)
	fmt.Println(x)
	fmt.Println(&x)
	Sample(x)
	fmt.Println(*x)
}

func Sample(input *int) {
	fmt.Println(input)
	fmt.Println(&input)
	*input = 20
}

// 0
// 0xc0000bc000
// 0xc0000b6018
// 0xc0000bc000
// 0xc0000b6028
// 20
```

可以看到我宣告了一個指標變數`x`他的值是`0xc0000bc000`而存儲這個指標變數的記憶體位置是`0xc0000b6018`，我將這個值傳入函式，於是它複製了一份指標變數，而他的值一樣會是`0xc0000bc000`，但查看他的記憶體位置則是`0xc0000b6028`，意謂著它並不是同一個變數，只是他的值是一樣的。

但既然我都拿得到`x`的記憶體位置，所以我直接解參考去改它存在記憶體上的值是成立的。

這下不用記啥是call by value, 啥是call by ref，真棒golang.

> 除了slice跟map，這兩個是以指標變數實做的。

接著看看slice，有說到這個是以指標變數實做的，具體slice在複製給令一個函式時複製了三樣東西：
1. 值的指標位置
2. 長度
3. 容量

複本的slice跟本尊除了值的指標位置是同一個，剩下兩樣都不是同一個東西了，所以對它的操作會出現一些很詭異的現象。

```go
func main() {
	x := make([]int, 0, 3)
	x = append(x, 1, 2, 3)
	Sample(x)
	fmt.Println(x)
    //output [1,2,5]
}

func Sample(input []int) {
	input[2] = 5
	fmt.Println(input)
    //output [1,2,5]
}
```
因為複製過去的副本在值的部份複製的是指標，修改也是直接改了解參指標的值，所以我們沒有透過指標傳值一樣使得slice的內容發生了變化。

但有趣的事情來了

```go
func main() {
	x := make([]int, 0, 3)
	x = append(x, 1, 2, 3)
	Sample(x)
	fmt.Println(x)
    // output [1,2,3]
}

func Sample(input []int) {
	input = append(input, 2, 3, 4, 5)
	fmt.Println(input)
    // output [1,2,3,2,3,4,5]
}
```

當我在做append的時候因為slice定義的時候給定長度容量為3，當我複製了複本過去之後同樣複製了這個長度跟容量。但當我append的時候長度增加了，容量不夠了，於是go runtime給了一個新的slice來承載內容，所以函式中的slice已經不是函式外的slice的複本了，而是一個新的slice。

