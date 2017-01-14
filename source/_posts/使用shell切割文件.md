---
title: 使用shell切割文件
date: 2013-03-07 13:49:44
tags:
- shell
- linux
---

最近工作中需要使用shell，从远程rsync数据过来预处理后提交到hdfs中，再调用pig脚本在hadoop集群上处理数据，完了fs -get下来结果文件并进行进一步处理，再推送给其他系统使用。其间需要将pig作业的结果文件合并并且均分为10个文件推送给远程服务器上的应用加载。因为结果文件比较大，远程应用拿到结果文件后使用多线程加载，所以需均分为10个小文件。虽然mr作业出来的文件结果也是part-00000、part-00001，但若pig脚本中不指定reduce任务数，产生的结果文件个数是3个，而且下下来之后需要进行重命名。与其这样还不如自己处理。

```java
rm -rf $TODAY_ALL_INDUSKEY
	for allName in `find $TODAY_ALL_TMP_DIR -name "part-*"`
		do
			INFO "Processing result file" $allName
			cat $allName >> $TODAY_ALL_INDUSKEY   #把结果文件重定向到一个文件
	done

	ALL_INDUSKEY_FILE_NUM=10    #拆分的文件数量
	ALL_KEY_LINES=0             #结果文件行数
	INFO "Split $TODAY_ALL_INDUSKEY into $ALL_INDUSKEY_FILE_NUM files"
	for str in `wc -l $TODAY_ALL_INDUSKEY`;	do
		t=`expr match $str "[1-9][0-9]*$"`;
		if [ $t -gt 0 ]; then
			ALL_KEY_LINES=$str         #获取结果文件行数
			INFO "Line of $TODAY_ALL_INDUSKEY is $ALL_KEY_LINES"
		fi
	done
	if [ $ALL_KEY_LINES -ne 0 ]; then
		tmpLine=`echo "scale=2;$ALL_KEY_LINES/$ALL_INDUSKEY_FILE_NUM"|bc`    #每个小文件的行数，保留两位小数
		INFO "$ALL_KEY_LINES/$ALL_INDUSKEY_FILE_NUM=$tmpLine"
		subFileLines=`echo $((${tmpLine//.*/+1}))`        #向上取整
		INFO "Per subfile lines:$subFileLines"
		split -l $subFileLines -a 1 -d $TODAY_ALL_INDUSKEY $TODAY_ALL_INDUSKEY"_"      #拆分文件
	fi

	if [ -f $TODAY_ALL_INDUSKEY ]; then
		touch $TODAY_ALL_INDUSKEY.done       #创建done文件
		rm -rf $TODAY_ALL_TMP_DIR
		INFO "Process result file dir $TODAY_ALL_TMP_DIR done!"
	fi
```

