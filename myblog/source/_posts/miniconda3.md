---
title: miniconda3
date: 2025-04-21 18:40:55
tags:
categories: python
---
> conda是一个包和环境管理工具，用于创建、管理和切换Python的虚拟环境

# 安装
```shell
mkdir -p ~/miniconda3
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ~/miniconda3/miniconda.sh
bash ~/miniconda3/miniconda.sh -b -u -p ~/miniconda3
rm ~/miniconda3/miniconda.sh
source ~/miniconda3/bin/activate
```
# 使用
```shell
1. conda --version #查看conda版本，验证是否安装
2. conda update conda #更新至最新版本，也会更新其它相关包
3. conda update --all #更新所有包
4. conda update package_name #更新指定的包
5. conda create -n env_name package_name #创建名为env_name的新环境，并在该环境下安装名为package_name 的包，可以指定新环境的版本号，例如：conda create -n python2 python=python2.7 numpy pandas，创建了python2环境，python版本为2.7，同时还安装了numpy pandas包
6. source activate env_name #切换至env_name环境
7. source deactivate #退出环境
8. conda info -e #显示所有已经创建的环境
9. conda create --name new_env_name --clone old_env_name #复制old_env_name为new_env_name
10. conda remove --name env_name –all #删除环境
11. conda list #查看所有已经安装的包
12. conda install package_name #在当前环境中安装包
13. conda install --name env_name package_name #在指定环境中安装包
14. conda remove -- name env_name package #删除指定环境中的包
15. conda remove package #删除当前环境中的包
16. conda env remove -n env_name #采用第10条的方法删除环境失败时，可采用这种方法
```



两个环境，一个有request一个没有，隔离作用


# 镜像源
```shell
# 查看镜像源
conda config --show-sources
# 添加镜像源
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
# 从镜像源中安装包时显示来源
conda config --set show_channel_urls yes
# 删除镜像源
conda config --remove channels https://XXX
# 删除配置的镜像源，使用默认镜像源
conda config --remove-key channels
```

# 打包运行环境
```shell
pip install conda-pack
conda pack -n my_env_name -o out_name.tar.gz
tar -zxvf 2.7.tar.gz -C 2.7
conda info -e
source activate my_env_name
```

