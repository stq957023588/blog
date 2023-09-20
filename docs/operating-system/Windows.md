# 指定安装路径安装Ubuntu

1. 微软商店安装Ubuntu
2. 打开Ubuntu，并设置账号密码（username:root,password:123456）
3. 查看wsl运行状态

```shell
wsl -l -v
```

4. 关闭wsl

```shell
wsl --shutdown
```

5. 导出当前Linux镜像

```shell
wsl --export Ubuntu-22.04 H:\ubuntu22.04.tar 
```

其中，"Ubuntu-22.04"是使用"wsl -l -v"查看到的linux系统的名字和版本,H:\ubuntu22.04.tar 是导出路径

6. 注销微软商店安装的Ubuntu系统

```shell
wsl --unregister Ubuntu-22.04
```

7. 导入刚刚导出的系统到指定位置

```shell
wsl --import Ubuntu-22.04 H:\Ubuntu2204 H:\ubuntu22.04.tar 
```

Ubuntu-22.04 Linux系统名称和版本
H:\Ubuntu2204 安装路径
H:\ubuntu22.04.tar 镜像所在位置

8. 配置先前设置的默认登陆用户

```shell
ubuntu2204.exe config --default-user root
```

9. 通过window卸载微软商店安装的Ubuntu

> 参考：[WSL2安装ubuntu及修改安装位置 - 简书 (jianshu.com)](https://www.jianshu.com/p/6f3195bad5f1)