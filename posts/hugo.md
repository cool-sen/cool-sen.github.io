# Hugo搭建个人博客（一）


## 1 安装Hugo

我在windows和ubuntu下安装过hugo，简要介绍下我的安装过程，其他方式可以参考[官方文档](https://gohugo.io/getting-started/installing)。

### 1.1 windows下安装

Hugo的安装方式有两种，一种是直接下载编译好的Hugo二进制文件。另一种方式是获取Hugo的源码，自己编译。

如windows使用二进制安装：

1. 下载[Hugo二进制文件](https://github.com/spf13/hugo/releases),下载下来后，解压，将解压后的文件夹名称和文件夹里面的.exe文件都改为同一个名称，否则hugo无法运行。
2. 配置计算机环境变量，右击计算机-属性-高级系统设置-高级-环境变量-系统变量，找到path，添加hugo路径。
3. 在终端进行 `hugo version` 进行验证是否安装正确。

### 1.2 ubuntu下安装

第一种方式：从[Github](https://github.com/gohugoio/hugo/releases)下载deb文件，然后使用dpkg命令安装，这样可以自由选择版本，我使用的是这种方法。

过程如下：

```bash

wget https://github.com/gohugoio/hugo/releases/download/v0.69.0/hugo_0.69.0_Linux-64bit.deb


```

第二种方式：

```bash
sudo apt-get install hugo
```

但是版本比较老，官方也不推荐使用

其他方式可以参考[官方文档](https://gohugo.io/getting-started/installing)

## 2 建站

### 2.1 创建项目

```bash
hugo new site my_website
cd my_website
```

这样就建立了新的站点。此时我们的新站点无法启动，需要安装主题。

### 2.2 安装主题

可以从[官方主题库中](https://themes.gohugo.io/)选择，里面有上百种主题。我使用的主题是[LoveIt](https://github.com/dillonzq/LoveIt),感觉风格简约，并且功能齐全。

安装方式有两种，第一种：

直接把这个主题克隆到 `themes` 目录:

```bash
git clone -b master https://github.com/dillonzq/LoveIt.git themes/LoveIt
```

或者, 初始化项目目录为 git 仓库, 并且把主题仓库作为网站目录的子模块:

```bash
git init
git submodule -b master add https://github.com/dillonzq/LoveIt.git themes/LoveIt
```

第二种方式：下载主题的 [最新版本 .zip 文件](https://github.com/dillonzq/LoveIt/releases) 并且解压放到 `themes` 目录.特别适合在国内的我们，因为Github下载clone速度太慢了，尤其对于大文件。

## 2.3 基础配置

在根目录中 `config.toml` 进行配置，填写主题名称

```
baseURL = "http://example.org/"
languageCode = "zh-CN"
title = "My New Hugo Site"
theme = "LoveIt"
```






