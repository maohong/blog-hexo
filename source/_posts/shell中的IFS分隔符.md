---
title: Shell中的IFS分隔符使用
date: 2013-01-29 20:41:54
tags: 
- shell
categories: 
- Shell
---

在linux中，shell把每个 $IFS 字符对待成一个分隔符，且基于这些字符把其他扩展的结果分割。
  
工作中需要处理一个文件datafile，文件中有好几列，列与列之间以‘\3′分割，如下(终端下’\3′显示为方块)：

![](https://raw.githubusercontent.com/maohong/picture/master/20130129/shell.png)

我需要拿到文件中<font color='blue'>第三列为1</font>的数据行再做具体的处理，比如取其中的某一列数据再去其他文件grep数据等等。简单点，直接逐行cat数据吧。 
 
**脚本如下：**

```c
for line in `awk -F"\3" '{if($3==1) print $0}' datafile`
    do
        echo $line
done
```

**结果如下：**  

![](https://raw.githubusercontent.com/maohong/picture/master/20130129/shell2.png) 

本来是想要逐行打印出来的，可结果却不是我想要的，究其原因，是因为在shell的for循环中，列出集合的item时，默认是以<space>或<tab>或<newline>为分隔符，我们的数据文件中有空格，因此它就以空格分割打印了。
  
可以通过显式设置IFS的值来达到我们要的效果，修改后的脚本如下：  

```c
oldifs=$IFS
IFS=$'\n'    #change seperator to '\n' to get a line
for line in `awk -F"\3" '{if($3==1) print $0}' datafile`
    do
        echo $line
done
IFS=$oldifs #reset seperator
```

通过先保存当前的IFS变量的值到一个临时变量，再显式设置为我们想要的行分隔符$’\n’，然后在for循环结束后，再重置IFS的值即可。  

**结果如下：**  
![](https://raw.githubusercontent.com/maohong/picture/master/20130129/shell3.png)  
