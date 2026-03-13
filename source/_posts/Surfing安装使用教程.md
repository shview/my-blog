---
abbrlink: Surfing安装使用教程
categories: []
date: '2026-03-13T21:19:39.287322+08:00'
excerpt: Surfing安装使用教程
tags:
- 教程
- 记录
title: Surfing安装使用教程
updated: '2026-03-13T21:19:41.906+08:00'
---
本文所讨论的是Magisk模块Surfing

项目地址[https://github.com/GitMetaio/Surfing](https://github.com/GitMetaio/Surfing)

前置条件：

1.安卓手机已root，我使用的是Magisk

2.有至少一个可用的梯子订阅链接

接下来进入教程

1.下载好zip后进入Magisk从本地安装，其中有按音量上下键选择配置，按下键或等10s按默认的就可以，完成后会出现如下图的两个模块，如若仅使用保留第一个即可，若要使用完整功能需两个均保留。

![](https://github.com/shview/my-blog/raw/main/source/images/2026/03-13-image.png)

2.接下来会发现桌面上多出了一个名为Web的软件，先去配置订阅。打开文件`/data/adb/box_bll/clash/config.yaml`，找到“主要地址”，在url中的双引号内的文本替换为自己的链接，注意双引号不能删。

3.之后进行重启，默认直接打开。现在我们可以进入Web软件进行进一步配置。默认的配置是规则模式，也可以按软件配置。

教程到这里就结束了。


如何电脑上查看更改部分设置？

1.Web-设置-后端，可以看到默认地址是`http://127.0.0.1:9090`,复制下来。

2.确保电脑有adb环境，连接手机确保可以调试，打开cmd或powershell，输入

~~~adb
adb forward tcp:9090 tcp:9090
~~~

等待自动进行完成，电脑端浏览器输入`http://127.0.0.1:9090`,看到

```
{"hello":"mihomo"}
```

说明成功了。而现在我们还需要一个图形化界面。

3.访问[yacd.metacubex.one](https://yacd.metacubex.one/) 或 [metacubex.github.io/metacubexd](https://metacubex.github.io/metacubexd/)即可以看到了。


其余参考：

[Magisk模块“Surfing”使用教程，实现全局透明代理 | BoJack' Blog](https://bojack0428.online/archives/Qj2VXchk)

上面的还有同时防广告教程，不过是以Clash Meta为基础的。
