---
title: 如何给git设置代理 (HTTPS/SSH)
categories:
  - 专业
  - 工具
  - git
tags:
  - git
  - github
  - ssh
index_img: https://cdn.jsdelivr.net/gh/simonyangchao/resources@image/github.png
banner_img: https://cdn.jsdelivr.net/gh/simonyangchao/resources@image/github.png
---

## 解决问题

开发者经常需要在终端和github之间进行repository的push/pull等操作，但是由于GFW的存在，不得不通过代理保障github访问的流畅度（太不友好..）。本文简单介绍如何给github配置HTTPs和SSH代理，让开发者对代理无感知丝滑使用。

<!-- more -->

## 分辨需要设置的代理

- HTTP 形式：
  
   > git clone https://github.com/owner/git.git
- SSH 形式：
  
   > git clone git@github.com:owner/git.git

## HTTP 形式
### 走 HTTP 代理

```bash
git config --global http.proxy "http://127.0.0.1:8080"
git config --global https.proxy "http://127.0.0.1:8080"
```

### 走 socks5 代理（如 Shadowsocks）

```bash
git config --global http.proxy "socks5://127.0.0.1:1080"
git config --global https.proxy "socks5://127.0.0.1:1080"
```

### 取消设置

```bash
git config --global --unset http.proxy
git config --global --unset https.proxy
```

## SSH 形式

修改 `~/.ssh/config` 文件（不存在则新建）：

```
# 必须是 github.com
Host github.com
   HostName github.com
   User git
   # 走 HTTP 代理
   # ProxyCommand socat - PROXY:127.0.0.1:%h:%p,proxyport=8080
   # 走 socks5 代理（如 Shadowsocks）
   # ProxyCommand nc -v -x 127.0.0.1:1080 %h %p
```
