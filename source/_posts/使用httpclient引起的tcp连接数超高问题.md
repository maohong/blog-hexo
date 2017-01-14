---
title: 使用httpclient引起的tcp连接数超高问题
date: 2014-03-28 17:44:58
tags:
- httpclient
- tcp连接数
categories: 
- 问题分析
---

组内的一个系统新上线了通过图片url上传图片到图片存储平台的功能。其中使用了httpclient，通过向图片存储平台发送MultipartPostMethod上传图片。当业务量较大时，10个处理线程满负荷运行，上传图片时，发现应用系统服务器的tcp连接数陡然升高，<font color='red'>峰值能达到几万个tcp连接数！</font>

排查系统代码并结合分析httpclient的源码发现，应用系统每次上传图片时，都会做new HttpClient()操作，这个操作内部默认使用的是SimpleHttpConnectionManager来管理http连接，而SimpleHttpConnectionManager有个默认字段alwaysClose=false，表示当外部程序调用了HttpMethod.releaseConnection()时并不会立即释放连接，而是保持这个连接并尝试用于后续的请求，在连接空闲一段时间后（默认3秒）才真正释放。

因此，当业务量较大，<font color='red'>系统高并发发送post请求时，new出来的HttpClient对象会很多，而这个对象使用完毕后，而当中建立的client对象在短时间内并不会立即释放连接</font>，因此，随着时间的积累，tcp连接数保持居高不下。

通过查看官方文档，建议在高并发环境下使用MultiThreadedHttpConnectionManager来管理httpclient，因此，我们将httpclient改为单例后，tcp连接数回复正常水平。

通过管理httpclient的代码如下：

```java
private static HttpClient initHttpClient()
{
    HttpConnectionManagerParams params = new HttpConnectionManagerParams();
    //指定向每个host发起的最大连接数，默认是2，太少了
    params.setDefaultMaxConnectionsPerHost(1000);
    //指定总共发起的最大连接数，默认是20，太少了
    params.setMaxTotalConnections(5000);
    //连接超时时间-10s
    params.setConnectionTimeout(60*1000);
    //读取数据超时时间-60s
    params.setSoTimeout(60*1000);
 
    MultiThreadedHttpConnectionManager manager = new MultiThreadedHttpConnectionManager();
    manager.setParams(params);
    return new HttpClient(manager);
}
```