---
title: 浅谈go defer 执行过程
date: 2022-02-14 21:30:00
index_img: /img/golang/go.jpg
banner_img:
categories: 
- golang
tags:
- golang
---

## defer 一般用途

1. 关闭文件句柄
2. 释放锁
3. 释放资源

go 语言的defer功能强大，对于资源管理非常方便，但是如果没用好，也会有陷阱。

## defer 执行流程

defer 是先进后出的数据结构，如果有多个defer表达式，调用顺序类似于栈，越后面的defer表达式越先被调用。

不过如果对defer的了解不够深入，使用起来可能会踩到一些坑，尤其是跟带命名的返回参数一起使用时。

defer是在return之前执行的。这个在 [官方文档](https://go.dev/ref/spec#defer_statements)中是明确说明了的。
要使用defer时不踩坑，最重要的一点就是要明白，return xxx这一条语句并不是一条原子指令!

defer 语句可以转换为如下语句顺序：

```go
返回值 = xxx
调用defer函数
空的return
```

以下面几个例子为例：

### 例子1

```go
func f() (result int) {
    defer func() {
        result++
    }()
    return 0
}
```

这个例子返回的数值是1，可以转译为如下

```go
func f() (result int) {
     result = 0  //return语句不是一条原子调用，return xxx其实是赋值＋ret指令
     func() { //defer被插入到return之前执行，也就是赋返回值和ret指令之间
         result++
     }()
     return
}
// 所以返回值为1
```

### 例子2

```go
func f() (r int) {
     t := 5
     defer func() {
       t = t + 5
     }()
     return t
}
```

可以转化为如下：

```go
func f() (r int) {
     t := 5
     r = t //赋值指令
     func() {        //defer被插入到赋值与返回之间执行，这个例子中返回值r没被修改过
         t = t + 5
     }
     return        //空的return指令
}
```

所以输出结果为5

### 例子3

```go
func f() (r int) {
    defer func(r int) {
          r = r + 5
    }(r)
    return 1
}
```

可以转换为如下：

```go
func f() (r int) {
     r = 1  //给返回值赋值
     func(r int) {        //这里改的r是传值传进去的r，不会改变要返回的那个r值
          r = r + 5
     }(r)
     return        //空的return
}
```
这里稍不一样 func 中的r操作是值拷贝 不是闭包 不会改变r的值 所以仍然是1

> 以上为defer 的陷阱的浅析 0.0

## Refrence

[深入解析Go](https://tiancaiamao.gitbooks.io/go-internals/content/zh/01.0.html)
[延迟调用defer](https://www.topgoer.com/%E5%87%BD%E6%95%B0/%E5%BB%B6%E8%BF%9F%E8%B0%83%E7%94%A8defer.html)