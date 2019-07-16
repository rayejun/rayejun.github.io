---
title: IntelliJ IDEA用JRebel插件实现热部署，破解JRebel
date: 2019-07-16 16:37:31
categories: 
 - [IntelliJ IDEA]
tags: 
 - IntelliJ IDEA
---
## IDEA安装JRebel插件
File->Settings->Plugins->Browse repositories
{% asset_img jrebel-install.png %}
点击Install安装JRebel

## 激活JRebel
下载代理工具ReverseProxy 地址：https://github.com/ilanyu/ReverseProxy/releases 运行
{% asset_img jrebel-click.png %}
{% asset_img jrebel-activation.png %}
{% asset_img jrebel-activation-success.png %}
这里就激活成功了

以后运行项目用JRebel插件按钮运行项目，改代码直接Build一下就行了，不用在重启项目，省了好多浪费的时间
