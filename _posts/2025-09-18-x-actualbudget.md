---
categories: FN
toc: true
title: Actual Budget 记账 - Docker Compose 安装指南
---

## 概述

Actual Budget 是一个本地优先的开源个人理财应用，注重隐私和数据控制。本文档提供使用 Docker Compose 安装 Actual Budget 的详细指南。

![ui]({{ site.baseurl }}/assets/images/fn/actualbudget/ui.png)

## 前提条件

- 已安装 Docker
- 已安装 Docker Compose
- 基本的命令行操作知识

## 安装步骤

### 1. 创建项目目录

```bash
mkdir actual-budget && cd actual-budget
```



### 2. 创建 Docker Compose 文件

创建 `docker-compose.yml` 文件：

```yaml
services:
  actual_server:
    image: docker.io/actualbudget/actual-server:latest
    ports:
      # This line makes Actual available at port 5006 of the device you run the server on,
      # i.e. http://localhost:5006. You can change the first number to change the port, if you want.
      - '5007:5006'
    volumes:
      # Change './actual-data' below to the path to the folder you want Actual to store its data in on your server.
      # '/data' is the path Actual will look for its files in by default, so leave that as-is.
      - ./actual-data:/data
    healthcheck:
      # Enable health check for the instance
      test: ['CMD-SHELL', 'node src/scripts/health-check.js']
      interval: 60s
      timeout: 10s
      retries: 3
      start_period: 20s
    restart: unless-stopped
```



### 3. 启动 Actual Budget 服务

```bash
# 后台启动
docker compose up --detach
```



### 4. 访问 Actual Budget

服务启动后，在浏览器中访问：

```tex
http://localhost:5007
```



首次访问时会提示设置预算文件和进行初始配置。