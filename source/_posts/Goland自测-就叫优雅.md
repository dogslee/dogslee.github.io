---
title: Goland自测--就叫优雅
date: 2022-06-22 22:03:14
index_img:
banner_img:
categories:
tags:
---

日常工作中如何保证我们开发代码的质量，这正是测试所存在的意义，TDD-测试驱动开正是为了解决这个痛点

## go test 使用介绍

要开始一个单元测试，需要准备一个 go 源码文件，在命名文件时需要让文件必须以_test结尾。默认的情况下，go test命令不需要任何的参数，它会自动把你源码包下面所有 test 文件测试完毕，当然你也可以带上参数。

### go test 常用参数

- -bench regexp 执行相应的 benchmarks，例如 -bench=.；
- -cover 开启测试覆盖率；
- -run regexp 只运行 regexp 匹配的函数，例如 -run=Array 那么就执行包含有 Array 开头的函数；
- -v 显示测试的详细命令。

eg1:

```golang
package demo

import "testing"

func TestHello(t *testing.T) {
    Hello()
}

```

## gomock使用介绍

## gomokey使用介绍
