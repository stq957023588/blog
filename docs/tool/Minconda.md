# Miniconda

用于python包管理（包括python本身）

# 安装配置

1. [官网下载](https://docs.conda.io/en/latest/miniconda.html#)

2. 打开命令终端：Anaconda Prompt

3. 配置

   ```shell
   conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
   conda config --set show_channel_urls yes
   ```

4. 创建环境

   ```shell
   conda create -n [环境名称] python=3.6
   ```

5. 激活环境

   ```shell
   conda activate [环境名称]
   ```

6. 退出环境

   ```shell
   conda deactivate
   ```

   

# 命令

* 下载安装包

  ```shell
  conda install pandas
  ```

* 查看已安装的包

  ```shell
  conda list
  ```

* 