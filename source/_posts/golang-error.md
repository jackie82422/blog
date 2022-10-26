---
title: golang 雜記 - error
toc: true
date: 2022-10-25 18:09:19
categories: golang
tags: golang
---

Error type雜記

<!-- more -->

在golang中error是滿足了error interface的實作。

```go
// The error built-in interface type is the conventional interface for
// representing an error condition, with the nil value representing no error.
type error interface{
    Error() string
}
```

從註釋上可以知道當函式/方法回傳的error為nil時代表的是沒有出現error，以這個方式判斷程式是否出錯算是golang語言的通則。

在建立上可以透過new或者fmt.Errorf函式建立。

```go
func main() {
	fmt.Println(returnError())
	fmt.Println(returnErrorWithFmt())
}

func returnError() error {
	return errors.New("return a error")
}

func returnErrorWithFmt() error {
	return fmt.Errorf("return a %v", "error")
}
```

透過實做interface是一個好方法：

```go
const (
	InvalidPermission = iota
	InvalidPath
	ResourceNotFound
)

type StatusErr struct {
	errorCode    int
	errorMessage string
}

func (s StatusErr) Error() string {
	return s.errorMessage
}

func GetData(path string) (bool, error) {
	if len(path) == 0 {
		return false, StatusErr{
			InvalidPath,
			"InvalidPath",
		}
	}

	return true, nil
}

func main() {
	result, err := GetData("")

	if err != nil {
		fmt.Println(err.Error())
	}
	fmt.Println(result)
}

```

但要特別注意的是沒有error時應該回傳nil而不是回傳自訂的error實體，就算沒有特別去附值後續在判斷是否為nil時也不會是nil的，除非回傳的型態跟他的底層型態都是nil才會是nil。