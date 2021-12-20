---
title: Reveal配置
date: 2021-01-05 19:31:23
tags:
---

# 一、Reveal概述

Reveal是一款UI调试神器，对iOS开发非常有帮助。本文使用的是Version 4 (12917)的版本。
# 二、Reveal安装
1.mac电脑:
[下载安装Reveal](https://pan.baidu.com/s/16rlHINfqgHpTMIqPa2UP8w) 提取码: scir 

2.iPhone(越狱):
打开Cydia搜索Reveal Loader

# 三、环境配置
iPhone(越狱):

1. 打开系统设置页进入Reveal设置打开对应的App#
{% asset_img IMG_2274.PNG This is an example image %}
![头文件](./Reveal%e9%85%8d%e7%bd%ae/IMG_2274.PNG)

{% asset_img IMG_2275.PNG This is an example image %}
![头文件](./Reveal%e9%85%8d%e7%bd%ae/IMG_2275.PNG)


2.导入dylib文件
由于新版本的Reveal只有Framework文件，所以需要修改Framework的可执行文件为dylib库

找到相关库 Help-->Show Reveal Library in Finder-->iOS Library

{% asset_img RevealFrameWork.PNG This is an example image %}
![头文件](./Reveal%e9%85%8d%e7%bd%ae/RevealFrameWork.PNG)

在手机的/Library 目录下新建目录 mkdir RHRevealLoader
使用iFunBox 将RevealServer 复制成Library/RHReveal/LibReveal.dylib
或使用scp 命令 （将电脑中的可执行库拷贝到iPhone目录中$scp -r –P 12345 RevealServer root@127.0.0.1:/Library/RHRevealLoader/libReveal.dylib
）

重启手机APP

完成