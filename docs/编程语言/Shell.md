# Shell

在Linux上运行的语言,文件后缀未.sh

## 文件内容替换

> sed -i "s/需要被替换的内容/替换的内容/g" 文件名

## 关闭对应端口进程

代码:

```shell
#!/bin/bash
echo will kill the port:$1
for i in `netstat -nltp|awk '{ print $4  " " $7 }'|grep -E ":::$1|0.0.0.0:$1|127.0.0.1:$1}"|awk  '{print $2}'`;
do
echo $i;
index=`expr index $i "/"`;
#echo $index;
pid=${i:0:index-1};
echo $pid;
echo "kill the process named [${i:index}]";
kill -9 $pid
done;
```

使用(文件名:killport.sh)
> ./killport.sh 7099 