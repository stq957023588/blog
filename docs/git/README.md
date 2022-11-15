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

## Git解决冲突后仍提示Merging解决办法

```shell
#在 android studio Terminal窗口下，查看当前git的状态
git status
#红色字体显示当前冲突的文件，如delete.......filename.....
#根据提示，使用rm命令删除该文件即可
git rm /app/src/......filename....

```



## 拉取,提交失败等情况处理方法

### 拉取提示:fatal: refusing to merge unrelated histories

在命令后面添加``--allow-unrelated-histories``

```shell
git pull --allow-unrelated-histories
```

### 拉取提示:Your account has been blocked

1. 删除当前远程地址

> git remote remove origin

2. 重新添加元辰地址

> git remote add origin [项目地址]

3. 拉取分支信息

> git pull

4. 设置当前master分支

> git branch --set-upstream-to=origin/master master

或者
> git remote set-url origin [项目地址]

## 上传本地项目到GitHub

1. GitHub上新建仓库

2. 初始化本地项目,在项目根目录下执行

   ```shell
   git init
   ```

3. 添加远程版本仓库

   ```shell
   git remote origin [远程仓库地址]
   ```

4. 提交代码

   ```shell
   git add .
   git commit -m "提交注释"
   git push origin main
   ```

   
