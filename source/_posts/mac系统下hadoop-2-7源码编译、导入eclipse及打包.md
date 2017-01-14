---
title: mac系统下hadoop-2.7源码编译、导入eclipse及打包
date: 2015-05-18 10:00:44
tags:
- hadoop
- eclipse
categories:
- Hadoop
---

编译环境要求
--

> JDK1.7+  
> MAVEN 3.0以上版本  
> 如果需要编译native code，还需要CMake 2.6、Zlib devel、openssl devel（mac下一般安装了xcode后应该都会有这些包）。

编译方法
--

解压源码包hadoop-2.7.0-src.tar.gz，iterm下进入文件夹hadoop-2.7.0-src，然后根据需要执行相应的mvn命令就可以了。

> 仅编译：mvn compile  
> 打包生成jar：mvn package  
> 生成eclipse项目：eclipse:eclipse -DskipTests，加上-DskipTests可跳过test阶段。

期间遇到几个问题，记录如下。

问题记录
--

先执行mvn eclipse:eclipse -DskipTest生成eclipse项目，执行到一半时，提示下面的报错：

<font color='red'>‘protoc –version’ did not return a version -> [Help 1]</font>

意思也就是是找不到protoc命令，安装protocolbuffer后重试，又提示错误：

<font color='red'>protoc version is ‘libprotoc 2.6.1′, expected version is ’2.5.0′</font>

看上去是protocolbuffer版本问题，hadoop需要的版本是2.5.0，而系统安装的是2.6.1，查了很多资料，都是说protocolbuffer版本太低后来升到2.5的，而我这是2.6.1的版本，难不成还得降回去，不至于吧。因此，猜测这个版本限制是在pom.xml中写死的，于是grep了一下，发现果然在hadoop-project/pom.xml中配置了编译时使用的pb版本。

<font color='red'>\<protobuf.version>2.5.0\</protobuf.version></font>

把以上配置项改为2.6.1，再重新执行生成eclipse项目的命令就OK了。

导入eclipse及打包
--

生成eclipse项目后，从eclipse里import existing project into workspace，选择hadoop-2.7.0-src目录，就会把所有代码模块导入eclipse了。接下来就可以看代码并修改了，比如增加一些日志信息等。

代码修改完毕后，可以再打出一个新的hadoop-distribution包来验证代码修改效果。

在hadoop-2.7.0-src目录下执行命令：<font color='red'>mvn package -Pdist -Ptar -Pdocs -skipTests </font>

等上漫长的一段时间，编译成功后，可以到hadoop-dist/target下找到新的jar包。

![](https://raw.githubusercontent.com/maohong/picture/master/20150518-hadoop%E7%BC%96%E8%AF%91%E6%89%93%E5%8C%85/1.png)

补充说明
--

编译过程需要从maven中央仓库下载大量依赖包，我使用的是oschina的库。

```java
    <mirror>
      <id>CN</id>
      <name>OSChina Central</name>
      <url>http://maven.oschina.net/content/groups/public/</url>
      <mirrorOf>central</mirrorOf>
    </mirror>
      <profile>
    	<id>oschina</id>
    	<repositories>
    		<repository>
    			<id>nexus</id>
    			<name>local private nexus</name>
    			<url>http://maven.oschina.net/content/groups/public/</url>
    			<releases>
    				<enabled>true</enabled>
    			</releases>
    			<snapshots>
    				<enabled>false</enabled>
    			</snapshots>
    		</repository>
    	</repositories>
    	<pluginRepositories>
    		<pluginRepository>
    			<id>nexus</id>
    			<name>local private nexus</name>
    			<url>http://maven.oschina.net/content/groups/public/</url>
    			<releases>
    				<enabled>true</enabled>
    			</releases>
    			<snapshots>
    				<enabled>false</enabled>
    			</snapshots>
    		</pluginRepository>
    	</pluginRepositories>
      </profile>
```