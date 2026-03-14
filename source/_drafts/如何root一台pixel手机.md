---
abbrlink: 如何root一台pixel手机
categories: []
date: '2026-03-14T17:35:34.541334+08:00'
excerpt: root pixel手机过程与指令
tags:
- 教程
- 记录
title: 如何root一台pixel手机
updated: '2026-03-14T17:35:36.253+08:00'
---
手机打开开发者选项，开启OEM解锁与调试模式。

确保电脑有adb与fastboot环境并有对应驱动。


完成以上前置条件并连接电脑后可以开始输入指令了。

首先确认是否连接上电脑：

~~~
adb devices
~~~

若出现如下字样则连接成功

~~~List
List of devices attached
29031FDH2000A3  device
~~~

输入

~~~
fastboot devices
~~~

进入fastboot模式，手机显示对应模式后输入

~~~
fastboot devices
~~~

出现如下字样则正确

~~~
29031FDH2000A3   fastboot
~~~

接着输入

~~~
fastboot flashing unlock
~~~

注意，此命令会清空数据。

之后跟着提示走就可以。

开机后重新打开开发者选项与调试模式，此时OEM已经变灰不能关闭。

完成后前往[Google Play services](https://sspai.com/link?target=https%3A%2F%2Fdevelopers.google.com%2Fandroid%2Fimages)下载自己机型对应的线刷包，注意build版本需要一样




参考教程 [一日一技｜如何 root 一台 Pixel 手机 - 少数派](https://sspai.com/post/76276)
