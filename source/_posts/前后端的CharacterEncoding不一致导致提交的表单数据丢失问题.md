---
title: 前后端的CharacterEncoding不一致导致提交的表单数据丢失问题
date: 2014-03-20 11:09:03
tags:
- web开发
- java
categories: 
- 问题分析
---

最近在开发分布式任务调度系统的web端时，遇到一个坑，记录如下：

在页面上新增和修改任务，提交后，任务的属性在后端怎么都接收不到，但是在另一个协同开发的同学那边本地调试就OK，在我的本地和公共开发环境都不行，这不合理啊。。。。。

排查了很多地方，js、setter等等，一直没发现问题在哪。跟负责前端的同学交流了下，发现前端post的数据确实是修改过的，也就是后端接收有问题。

于是把最新版本和历史版本对比，发现最新版本新增了一个LogFilter，用于记录pagedelay的，仔细一看，logFilter里面是

```java
request.setCharacterEncoding(“UTF-8″);
response.setContentType(“text/html;charset=UTF-8″);
```

但页面上是GBK编码，所以导致数据在这个filter中编码出错，造成数据丢失，后端接收到的数据为null。

<font color='red'>解决方法：</font>

把logFilter里的UTF-8改为GBK，就一切正常了。

<font color='red'>疑问：</font>

1. 为何历史本没问题呢，因为历史版本中的logFilter配在struts2Filter之后，请求根本走不到logFilter里去。。。。

2. 为何协同开发的同学本地调试没问题呢，那是因为他把web.xml里的LogFilter的filtermapping注掉了。。。。

好一个歪萝卜大烂坑。。。