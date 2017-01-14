---
title: 使用hexo+gitpage搭建博客
date: 2014-09-02 19:50:40
tags: 
- hexo
- gitpage
categories: 
- 工具
---

环境准备
--
系统：mac osx  
软件：Node.js，npm，git，hexo  
具体安装以及git与github打通的配置就不详述了，可以google到各种方法。  

hexo命令
--
hexo init &lt;folder&gt;  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; #表示执行init命令初始化hexo到你指定的目录  
<font color="red">以下命令需要在&lt;folder&gt;目录下执行：</font>  
hexo generate  &nbsp;&nbsp;&nbsp;#自动根据当前目录下文件,生成静态网页  
hexo server &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;#运行本地服务  
  
启动服务后，就可以通过访问 http://localhost:4000 来看看效果了。  
接下来，可以使用以下命令来创建一篇新博文：  
hexo new "test blog 1"  
创建一个名为test blog 1的博客页面，对应的md文件路径是&lt;folder&gt;/source/_posts\test blog 1.md  
  
接下来就可以在这个md文件中写文章了，我使用的是MacDown来编辑md文件，支持实时查看页面效果，还是挺好用的。

发布博客
--
文章写好后，通过以下方式发布到github上。  
1.编辑./_config.yml文件，修改以下部分，配置本地内容同步至github：  

>  deploy:  
>  &nbsp;&nbsp;type: git  
>  &nbsp;&nbsp;repository: git@github.com:maohong/maohong.github.io.git  
>  &nbsp;&nbsp;branch: master  

2.执行hexo generate(hexo g)生成html内容  
3.执行hexo deploy(hexo d)讲更新内容发布至guthub  
  
然后就可以访问主页查看效果了，可以使用github帐户名.github.io进行访问, 也可以设置个性域名。
