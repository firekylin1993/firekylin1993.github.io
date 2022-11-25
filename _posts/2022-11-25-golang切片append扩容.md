---
layout:     post
title:      golang切片append
subtitle:   golang切片append
date:       2022-11-25
author:     果果
header-img: img/home-bg-o.jpg
catalog: true
tags:
- 数据结构与算法
- golang
---

## golang切片append

每次 slice 进行扩容的时候，要经历以下几个步骤：

1. 为最终的容量申请到足够的内存空间；
2. 从原有内存地址中将元素 copy 到新的内存中；
3. 销毁原有内存地址中的元素；

```go
package main

import "fmt"

func main() {
	a := []int{1}
	a = append(a, 2)
	a = append(a, 3)
	
	b := append(a, 4)
	c := append(a, 5)
	
	fmt.Println(a)
	fmt.Println(b)
	fmt.Println(c)
}

[1 2 3]
[1 2 3 5]
[1 2 3 5]
```

一开始 a 的元素为 5，在经历过两次 append 操作之后，a 的 cap 变为了 4，这时候append 4 和 5 这两个操作是分别为 a 添加了 4 和 5 的操作且并没有触发扩容，
但是这两个操作的结果并没有赋值给 a，后一个操作覆盖了前一个操作，且最终的 a b c 指向的都是同一个地址，所以会得出这样的结果。
