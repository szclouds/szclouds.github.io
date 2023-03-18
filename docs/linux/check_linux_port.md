---
title: 端口状态查看
layout: default
tags: linux shell
permalink: /linux_port
parent: Linux
---
# 端口状态查看
- `nc -zvw 3 ip/domain port` 查看指定 ip/domain+port 是否连通 响应如下则为连通状态
   
   ``` shell
   Connection to ip/domain port port [tcp/https] succeeded!
   ```
   
- `netstat -an | grep LISTEN | grep 3306` 查询 3306 端口是否处于 LISTEN 状态
  
   ``` shell
   tcp4       0      0  127.0.0.1.3306         *.*                    LISTEN
   tcp4       0      0  127.0.0.1.33060        *.*                    LISTEN
   ```
   
- `lsof -i:3306` 用于列出当前系统中打开的文件和网络连接等信息。其中，-i 选项用于指定要显示的网络连接信息，后面可以跟上端口号或者服务名等网络连接相关信息
