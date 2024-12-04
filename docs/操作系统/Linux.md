# Windows安装Ubuntu子系统

## 安装包安装

参考链接

>[Ubuntu设置、非C盘安装及重装_ubuntu必须安装在c盘吗_EcrayyarcE的博客-CSDN博客](https://blog.csdn.net/Echo__Yi/article/details/120987822)

# 远程SSH免密访问方法

本机通过ssh-keygen生成密钥

```shell
ssh-keygen -t rsa
```

默认会在C:\Users\用户名/.ssh/下生成id_rsa，id_rsa.pub两个文件



# 无法使用kill杀死的进程

## 产生原因

无法杀死线程,可能是由于kill的是子线程

## 解决办法

使用 ps aux 查看进程状态,找到STAT一栏,如果为Z那么就是僵尸进程

通过 ps -ef | grep 僵尸进程ID  找到父进程ID,先kill掉父进程

# CPU占用过高排查

1. 使用 top -c 命令查询CPU占用过高进程

   ```
     PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
       1 root      20   0 3153388  29572  15996 S 399.2   1.5   7:40.41 java -jar application.jar
      27 root      20   0    2392    756    688 S   0.0   0.0   0:00.01 /bin/sh
      35 root      20   0    9748   3428   2992 R   0.0   0.2   0:00.00 top -c
   ```

2. 根据第一步查询到的CPU占用过高的进程ID, 使用 top -Hp 1 查询进程下各个线程CPU占用情况 

   ```
     PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
      23 root      20   0 3153388  29572  15996 R  81.7   1.5   2:50.39 Thread-1
      25 root      20   0 3153388  29572  15996 R  80.7   1.5   2:55.73 Thread-3
      24 root      20   0 3153388  29572  15996 R  80.1   1.5   2:55.78 Thread-2
      22 root      20   0 3153388  29572  15996 R  79.7   1.5   2:47.36 Thread-0
      26 root      20   0 3153388  29572  15996 R  76.7   1.5   2:50.62 Thread-4
       1 root      20   0 3153388  29572  15996 S   0.0   1.5   0:00.03 java
       8 root      20   0 3153388  29572  15996 S   0.0   1.5   0:00.11 java
       9 root      20   0 3153388  29572  15996 S   0.0   1.5   0:00.00 java
      10 root      20   0 3153388  29572  15996 S   0.0   1.5   0:00.00 java
      11 root      20   0 3153388  29572  15996 S   0.0   1.5   0:00.00 java
      12 root      20   0 3153388  29572  15996 S   0.0   1.5   0:00.00 java
      13 root      20   0 3153388  29572  15996 S   0.0   1.5   0:00.00 VM Thread
      14 root      20   0 3153388  29572  15996 S   0.0   1.5   0:00.00 Reference Handl
      15 root      20   0 3153388  29572  15996 S   0.0   1.5   0:00.00 Finalizer
      16 root      20   0 3153388  29572  15996 S   0.0   1.5   0:00.00 Signal Dispatch
      17 root      20   0 3153388  29572  15996 S   0.0   1.5   0:00.00 C2 CompilerThre
      18 root      20   0 3153388  29572  15996 S   0.0   1.5   0:00.00 C2 CompilerThre
      19 root      20   0 3153388  29572  15996 S   0.0   1.5   0:00.02 C1 CompilerThre
      20 root      20   0 3153388  29572  15996 S   0.0   1.5   0:00.00 Service Thread
      21 root      20   0 3153388  29572  15996 S   0.0   1.5   0:00.14 VM Periodic Tas
   ```

3. 根据进程ID和进程下CPU占用最高线程ID的十六进制数,使用 jstack 1 | grep ‘0x17’ -C5 --color 查询具体的堆栈信息

   ```
   # jstack 1 | grep '0x17' -C5 --color
      java.lang.Thread.State: RUNNABLE
           at com.fool.service.InfiniteCycle.lambda$execute$0(InfiniteCycle.java:14)
           at com.fool.service.InfiniteCycle$$Lambda$1/455659002.run(Unknown Source)
           at java.lang.Thread.run(Thread.java:748)
   
   "Thread-1" #10 prio=5 os_prio=0 tid=0x00007f88ec289800 nid=0x17 runnable [0x00007f88ce84c000]
      java.lang.Thread.State: RUNNABLE
           at com.fool.service.InfiniteCycle.lambda$execute$0(InfiniteCycle.java:14)
           at com.fool.service.InfiniteCycle$$Lambda$1/455659002.run(Unknown Source)
           at java.lang.Thread.run(Thread.java:748)
   ```

通过以上信息可以精确到具体哪个方法



# 设置开机启动

使用systemctl

在/etc/systemd/system文件夹下创建 <服务名>.service文件

```shell
vi  <服务名>.service
```

编辑<服务名>.service文件

```shell
[Unit]
Description=<服务名>
After=network.target

[Service]
# 工作目录
WorkingDirectory=/usr/local/ynpacs-main
# 用户名
User=root
# 用户组
Group=root
# 类型
Type=fork
#ExecStart=/usr/bin/java -jar -Dspring.profiles.active=pro /usr/local/ynpacs-main/ynpacs-main.jar
# 启动命令
ExecStart=/usr/local/ynpacs-main/startup.sh
SuccessExitStatus=143
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

执行开机启动命令

```shell
systemctl enable <服务名>
```



# 命令

## 服务

查看服务列表

```shell
systemctl list-unit-files
```

查看服务日志

```shell
journalctl -f -u <service>
```



## 其他

重启防火墙



后台运行jar

```shell
nohup java -jar xxx.jar >output.log &
```

给sh文件执行权限

```shell
chmod 777 startup.sh
```

查看磁盘使用情况

```shell
df -h
```

查看内存使用情况

```shell
cat /proc/meminfo
```

查询端口占用命令

```shell
netstat -tunlp|grep 8080
```

查看所有进程

```shell
ps aux
```

查看磁盘空间

```shell
df -h
```

查看文件信息

```shell
stat <文件名称>
```

在文件末尾添加字符串

```shell
echo "hello">>test.txt
```

查找文件

```shell
find / -name <文件名称>
```

查看当前文件夹路径

```shell
pwd
```

查看shell解释器类型

```shell
ls -l /bin/sh
```

修改解释器类型

```shell
dpkg-reconfigure dash
```

删除文件/文件夹

```shell
rm -rf <文件夹名称>   
rm -f <文件名称>
```

解压压缩文件

```shell
tar -zxvf 压缩文件名.tar.gz
```

移动文件到

```shell
mv [source] [target]
```

复制文件到

```shell
cp [source] [target]
```



# vi编辑

查询文本

> /查询内容

保存并退出

> :wq

不保存退出

> :q

强制不保存退出

> :q!







