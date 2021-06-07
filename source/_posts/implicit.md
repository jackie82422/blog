---
title: 使用者定義轉換運算子(implicit, explicit operator)
toc: true
categories: C#
date: 2021-06-07 00:09:52
tags: C#
---

MSDN有時候就是有這種每個中文字我都看得懂，合在一起我就不知道在說三小的文章...

<!-- more -->

## Introduction
這兩東西照字面上來翻譯是隱含的(Implicit),顯式的(Explicit)；其實就是指C#裡面隱式轉換和顯示轉換用的關鍵字。

搭上了Operator的關鍵字也就變成

Implicit Operator(隱式轉換運算子), Explicit Operator(顯示轉換運算子).

概括地來說，隱式磚換可以簡單想像成當型別照著隱式轉換轉的時候，基本不會丟失任何資訊。

舉個例子

```cs
int myAge = 28;
double myPreciseAge = myAge;
```

我將型別從int => double，但不會丟失任何資訊。

但反過來做的話，事情就不對了。

```cs
double myPerciseAge = 28.4;
int myAge = (int)myPerciseAge;
// output = 28, 無條件捨去
```

指定型別(顯式轉換)的同時我也知道我會有資訊遺失的問題出現，畢竟Int沒有小數點的嘛。

好了，那Implicit Operator And Explicit Operator這兩個特殊的運算子就是來定義我這個類別如果要做顯式/隱式轉換的時候，它會發生甚麼事情。


## Sample Code

我們簡單實作一個 Student的Class，代碼如下
```cs
public class Student
{
    private string StudentName { get; set; }
    private int Id { get; set; }
    public Student(string studentName, int id)
    {
        StudentName = studentName;
        Id = id;
    }
    public string GetStudentName => StudentName;
    //當隱式轉換為string的時候會轉換出什麼內容
    public static implicit operator string(Student student) => student.StudentName;
    public static implicit operator int(Student student) => student.Id;
    //當顯示轉換int => student時會時實作內容
    public static explicit operator Student(int value) => new Student("Unknown Student", value);
}
```

執行呼叫

```cs
var student = new Student("ChachaLin",1);
string studentName = student;
int studentId = student;
Console.WriteLine($"Student Name is :{studentName}");
Console.WriteLine($"Student Id is :{studentId}");
Console.WriteLine($"Add Student withoutName : {((Student)2).GetStudentName}");
```

印出內容
![](A.png)

這樣我們就能看到當我New出一個Student實例的時候，如果我隱式轉換為String，那麼這個String的內容就會是Student Name，轉換成Int的時候以此類推。

而我將int顯式轉換為Student的時候，實際上是New了一個Student
>new Student("Unknown",2);

這就是implicit/explicit operator的具體用法了。