# Git

## 查看当前项目的远程地址

> git remote -v

## 添加文件

添加当前目录下所有文件到暂存区
> git add .

添加一个或多个文件到暂存区
> git add [file1] [file2]

添加指定目录到暂存区，包括子目录
> git add [dir]

## 生成SSH Key

> ssh-keygen -t rsa -C “备注”

## 初始化项目

> git init

## 生成.gitignore文件

> touch .gitignore

## 修改用户信息

> 修改当前项目   
> git config user.name fool   
> git config user.email fool@fool.com  
> 全局修改   
> git --global config user.name fool     
> git --global config user.email fool@fool.com

## 拉取,提交失败等情况处理方法

### 拉取提示:Your account has been blocked

1. 删除当前远程地址

> git remote remove origin

2. 重新添加元辰地址

> git remote add origin [项目地址]

3. 设置当前master分支

> git branch --set-upstream-to=origin/master master

或者
> git remote set-url origin [项目地址]




