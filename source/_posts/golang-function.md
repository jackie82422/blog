---
title: golang 雜記 - func
toc: true
categories: golang
date: 2022-10-19 14:37:59
tags: golang
---

雜記

<!-- more -->

## Func


### 宣告
function包含四個部份，關鍵字`func`、`func 名稱`、`輸入參數`、`回傳型態`。

```go
func sample(name string, id int) int(){
    return name + strconv.Itoa(id)
}
```

golang的function可以做到不定數量(variadic)參數傳入，而不用透過array, slice去模擬，但實際上傳入的是個slice，算是語法糖。

```go
func main() {
	result := Sum(2, 3, 4, 5, 6)
	fmt.Println(result)
    // output 20
}

func Sum(val ...int) int {
	var result int
	for _, v := range val {
		result += v
	}

	return result
}
```

須注意不定數量參數只能放在最後一個。

### 回傳複數值
同時golang的function也有提供tuple的功能：

```go
func main() {
	result, count := Sum(2, 3, 4, 5, 6)
	fmt.Println(result, count)
}

func Sum(val ...int) (int, int) {
	var result int
	for _, v := range val {
		result += v
	}

	return result, len(val)
}
```

跟C#一樣也可以給定名稱：

```go
func Sum(val ...int) (sum int, len int) {
	var result int
	for _, v := range val {
		result += v
	}

	return result, len(val)
}
```

但跟C#不同的是在C#中這個具名是真的給個名字而已，而在golang裡面這個具名是function內部會建立這個變數且先給予default值，只是沒有嚴格要求你一定要用到而已。

```go
func Sum(val ...int) (sum int, count int) {
	var result int
	for _, v := range val {
		result += v
	}

	fmt.Println(sum)
	fmt.Println(count)
    // output 0
    // output 0

	return result, len(val)
}
```

因為是真的存在的變數，所以一樣會發生shadow的行為。

```go
func Sum(val ...int) (sum int, len int) {
	var result int
	for _, v := range val {
		result += v
	}

	return result, len(val)
    // error : invalid operation: cannot call non-function len (variable of type int)
}
```

雖然明確給定了回傳變數的名稱有助可讀性的增加，但卻會有遮蔽的風險，而且給定明確名稱也不會在操作該function的時候可以靠智能語法拿到名稱。

尤其是golang並不限制你必須回傳你明確指定的回傳值，而是允許你return任何你想要的，像上面的例子我都是回傳其他的變數回去。但當你單純做blank return(空回傳)時，你拿到的是你明確名稱的變數。

```go
func main() {
	result, count := Sum(2, 3, 4, 5, 6)
	fmt.Println(result, count)
    //output 10 10
}

func Sum(val ...int) (sum int, count int) {
	sum = 10
	count = 10

	return
}
```

整體而言我認為在golang上明確回傳變數的名稱並沒有帶來特別多的效益，反而會導致程式碼可能存在難以trace的資料流。

### Function is Value !

在golang中function是一級公民(first-class)且是higher-order function，意味著我們可以做到將函式當值，將函式傳給令一個函式或者從函式中取得函式回傳。我們也可以定義function型態，可以拿來規範某些功能的function必須滿足的傳入跟傳出。

```go
type calculator func(firstNum int, secNum int) int

func main() {
	var calculate = make(map[string]calculator)
	calculate["+"] = func(a int, b int) int { return a + b }
	calculate["-"] = func(a int, b int) int { return a - b }
	calculate["*"] = func(a int, b int) int { return a * b }
	calculate["/"] = func(a int, b int) int { return a / b }
	fmt.Println(calculate["-"](1, 2))
    // output -1
}
```

在這邊我們先定義了函式的型態，給放到`calculate`的函式給個規範，後面製造`map`值則是直接給了`func`當值。

### 匿名函式

函式不但可以傳遞，還可以在函式裡面定義函式；其實跟`javascript`有點相像。

```go
func main() {
	for i := 0; i < 2; i++ {
		func(j int) {
			fmt.Println(j)
		}(i)
	}

	nameDecorator := func(name string) string { return "Dear " + name }
	fmt.Println(nameDecorator("Cha"))

    //output :
    // 0
    // 1
    // Dear Cha
}
```

### 函式回傳函式

```go
func main() {
	fmt.Println(Sum()(1, 2))
}

func Sum() func(int, int) int {
	return func(a int, b int) int {
		return a + b
	}
}
```

### Closure

這個概念我記得我在javascript的時候看了很久，然後專心寫後端之後我就把這事忘得一乾二淨了。

Closure指的是在函式裡面宣告的函式，這個函式可以read/write外層的函式所宣告的變數。意味著當它傳給了其他函式，可以在其他的函式裡面去調用這個一起被傳過來的變數，但這個傳過來的變數並非原本的變數了，golang會製造一個該變數的副本。

越說越模糊，上code

```go
func main() {
    c := 1
	f := Sample(c)
	fmt.Println(f(1))
	fmt.Println(f(1))
    fmt.Println(&c)
    fmt.Println(c)
}

func Sample(x int) func(int) int {
	fmt.Println(&x)
	return func(y int) int {
		fmt.Println(&x)
		x = x + 1
		return x + y
	}
}

// 0xc00001c030
// 0xc00001c030
// 3
// 0xc00001c030
// 4
// 0xc00001c038
// 1
```

拆解一下這段code, 我宣告了一個函式`Sample`，它會回傳一個匿名函式，匿名函是的內容是印出`x`的記憶體地址；這邊已經可以看到內部函式是可以取得外部函式的變數了，因為匿名函式中印出的指標位置跟外層函式的指標位置一致。

之後`x = x + 1` 試試可不可以修改，最後回傳`x + y`。

從output可以看到`0xc00001c030`重複出現，代表位址未曾變過，已經存在了函式中。而後印出匿名函式的結果，帶入`y=1`所以結果是`x(1) + 1 + y(1) = 3`，再來是`x(2) + 1 + y(1) = 4`。

最後看看一開始定義`c`的位置跟值，會發現`c`不會因為傳入而有所改變，甚至位址都不是同一個；這是因為這個值傳入函式後被複製了一份，之後用的都是這個複本。

> map, slice 傳入後並不會複製，因為這兩個結構體是指標(ref)，其他型態如果需要對值做修改則要傳入指標位置。

也可以透過這個方式慢慢的將多參數變小，完成`Curring（f(g(x))` (ps. C#做這個神tm麻煩...)




### defer

跟其他語言的destructor有點類似，在函式結束後調用，就像C#會用using結構包住程式讓他在結束時調用`IDispose`一樣，defer的用途很多時候也是拿來做差不多的事情，譬如關閉資料庫連線、關閉IO...etc

差別在於defer可以定義多次，執行的順序則是FILO (first in last out)。