---
layout:     post
title:      github action推送镜像
subtitle:   github action
date:       2022-08-15
author:     果果
header-img: img/post-bg-mma-5.jpg
catalog: false
tags:
- CICD
---

最近看到同事在用github action帮忙生成镜像，所以闲来无事自己也简单的写了一个workflow，每一行的作用都写了注释，方便理解和记忆

仓库地址：[docker-build](https://github.com/firekylin1993/docker-build)

DOCKERHUB_USERNAME 和 DOCKERHUB_TOKEN 可以登陆dockerhub获取

```yaml
# docker-image.yml
name: Push Docker image   # workflow名称，可以在Github项目主页的【Actions】中看到所有的workflow

on:   # 配置触发workflow的事件
  push:
    branches:   # master分支有push时触发此workflow
      - 'master'
    tags:       # tag更新时触发此workflow
      - 'v*'
  schedule:
    - cron: '50 0 * * *'

jobs:
  push_to_registry:  # job的名字
    name: Push Docker image to Docker Hub
    runs-on: ubuntu-latest   # job运行的基础环境

    steps:  # 一个job由一个或多个step组成
      - name: Check out the repo
        uses: actions/checkout@v2   # 官方的action，获取代码

      - name: Log in to Docker Hub
        uses: docker/login-action@v1  # 三方的action操作， 执行docker login
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build And Push Docker image
        env:
          # 指定自己dockerhub用户名（不要动）
          docker_repo: firekylin93
          # 指定dockerhub仓库名称
          image_name: go-app
          # 指定镜像标签
          tag: latest
        run: | 
          # 查看docker 版本
          docker version
          # 使用Dockerfile构建镜像
          docker build . -f Dockerfile -t $docker_repo/$image_name:$tag
          # 推送镜像到镜像仓库
          docker push $docker_repo/$image_name:$tag
```
