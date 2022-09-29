---
layout:     post
title:      goland配置golangci-linter代码扫描
subtitle:   golangci-linter
date:       2022-02-12
author:     果果
header-img: img/post-bg-mma-6.jpg
catalog: false
tags:
    - Golang
    - cicd
---


## 1.写在前面
go作为一门静态语言。运行静态代码分析作为golang代码审查的做法防御的第一线，其作用不言而喻。

因此golangci-lint就诞生了，在安装之后他会在终端的问题中显示所有的代码不规范的地方及优化提示。

实用且廉价，安全又方便


## 2.如何安装
>2.1 安装golangci-linter
>>2.1.1  从GitHub找到对应系统的编译文件进行下载（推荐） <a href="https://github.com/golangci/golangci-lint/releases/tag/v1.43.0" target="_blank">地址摸我</a>
![pic](/img-post/202202/golinter-1.png "pic")
>>2.1.2  当然，也可以使用命令行安装
>>```go
>># Go 1.16+
>>go install github.com/golangci/golangci-lint/cmd/golangci-lint@v1.43.0
>>
>># 或者
>>
>>curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh sh -s -- -b $(go env GOPATH)/bin v1.43.0
>>```
>>2.1.3   命令执行完毕会在$GOPATH/bin目录下生成名为golangci-lint的二进制文件（windows的后缀为.exe）
>>2.1.4   检查是否配置环境变量，没有的需要自行配置。配置方法自行百度
>>```go
>>golangci-lint --version     #检查是否安装成功
>>```
>2.2 从goland插件商店下载Go Linter插件（访问商店需开启梯子，没有梯子的可本地下载后导入goland）
![pic](/img-post/202202/golinter-2.png "pic")
>2.3  从goland配置Go Linter插件
![pic](/img-post/202202/golinter-3.png "pic")
>>path配置2.1下载或者自己编译生成的二进制文件
>>
>>project root配置项目地址
>>
>>点击apply，OK。到此安装完毕

## 3.简单看看效果
>3.1 例子一：当引入有风险的包，会自动提示异常
![pic](/img-post/202202/golinter-4.png "pic")
>3.2 例子二：啰嗦的语句变清爽
![pic](/img-post/202202/golinter-5.png "pic")