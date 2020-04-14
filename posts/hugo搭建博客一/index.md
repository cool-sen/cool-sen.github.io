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
dpkg -i hugo_0.69.0_Linux-64bit.deb
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

然后hugo会自动生成这样一个目录结构：

|               |                          |
| ------------- | ------------------------ |
| ▸ archetypes  | default.md为模板         |
| ▸ content     | 放的是你写的markdown文章 |
| ▸ layouts     | 网站的模板文件           |
| ▸ static      | 图片、css、js等资源      |
| ▸ config.toml | 网站的配置文件           |

这样就建立了新的站点。但此时我们的新站点无法启动，需要安装主题。

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

### 2.3 基础配置

在根目录中 `config.toml` 进行配置，填写主题名称

```
baseURL = "http://example.org/"
languageCode = "zh-CN"
title = "My New Hugo Site"
theme = "LoveIt"
```

注意：**在key:和值之间，有且只有一个空格**，参考[front-matter](https://gohugo.io/content-management/front-matter/)。

### 2.4 创建博客

创建第一篇博客

```bash
hugo new posts/first_blog.md
```

{{< admonition >}}
默认情况下, 所有文章和页面均作为草稿创建. 如果想要渲染这些页面, 请从元数据中删除属性 `draft: true`, 或者设置属性 `draft: false`.
{{< /admonition >}}

### 2.5 在本地启动网站

```bash
hugo server
```

也可以在启动server时应用主题。

```bash
hugo server --theme=LoveIt --watch
```

参数说明：

```
* --theme 用于选择主题，如果在配置文件中选择了主题，这里就不需要使用了
* --buildDrafts 用于是否显示草稿文章
* --watch 用于实时监控变化，方便调试
```

访问：http://localhost:1313/  

![image-20200413235014478](/images/hugo%E6%90%AD%E5%BB%BA%E5%8D%9A%E5%AE%A2%E4%B8%80/image-20200413235014478.png)



### 2.6 构建网站

在项目根目录下直接使用 `hugo` 命令，会生成 public 目录，该目录下都是关于我们的 markdown 编译完成的 html 静态页面。博客安装好之后，就该进行部署了，可以部署到自己的网站，也可以部署到Git Page。我使用的是Git Page，后面会具体介绍如何部署到Git Page。



## 3 配置相关问题（常见坑总结）

### 3.1 图片路径

如下面的config.toml文件，截取了部分

```TOML
# 文章页面配置
  [params.home]
    # 主页信息设置
    [params.home.profile]
      enable = true
      # 主页显示头像的 URL
      avatarURL = "/images/avatar.png"
```

图片的路径都是相对于baseurl而言的，以static为根目录，所以这里的/images/avatar.png,是在static文件夹下。当初我就被这个坑了，的确是需要注意的。

### 3.2 本地和站点图片路径不一致

在 Typora 中编辑文章插入图片能够显示，而发布后网页中的图片不能正常显示（路径错误）。或者使用站点根目录（`/`）引用图片可以正常加载显示，但是无法在 Typora 编辑器中显示图片。

有以下几种方法解决。

1. 可以设置uglyURLs 来解决，但是这样url就会加上.html，可以参考[博文](http://www.maitianblog.com/hugo.html)。

2. 个人不是很喜欢，因此使用了另一种方法。更改Typora 设置

   具体步骤:

   * 将插入文档中图片默认保存在hugo的“static\images\文章名称”文件夹下

     <img src="/images/hugo%E6%90%AD%E5%BB%BA%E5%8D%9A%E5%AE%A2%E4%B8%80/image-20200413235848868.png" alt="image-20200413235848868" style="zoom:80%;" />

   * 在博客文章中加入typora-root-url，如：

     ```
     title: "typora test"
     draft: false
     typora-root-url: ../../static
     ```

   * 设置图片根目录”既可以设定为绝对路径，也可以设置为相对路径，这里建议使用相对路径，便于跨系统的迁移后也能够重现结果。进行上述的设定后，任何新插入的图片默认都会保存在“static\images\文章名称”文件夹下，“Typora”会使用“static”作为根目录，在文章内使用相对于根目录的路径连接插入进来的图片。

3. 此外还看到过一种方法，在github上开一个repository，专门用于存放图片，然后网站引用地址。不过我没有尝试，大家有兴趣可以试下。

###  3.3 评论区的设置

评论区我使用的是Valine，是一款快速、简洁且高效的无后端评论系统。具体可以参考[valine配置](https://www.smslit.top/2018/07/08/hugo-valine/)。这步重点是将自己的appId，appKey写入到config.toml中。

还有最重要的一点是，要更改环境。我就被这个坑惨了，当上面的配置都好之后，却发现然后没有评论区，搜索了很多，终于找到了关键。因为使用的是development环境。hugo server 的默认环境是 development，hugo 的默认环境是 production。

解决方法：

1. 设置```HUGO_ENV=production```
2. 本地启动时：```hugo --environment production server```

### 3.4 分类问题

需要自定义分类，比如我想再首页增加一个分类栏目，增加关于栏目。

首先需要知道的是，Hugo默认会产生 tags 和 categories 的分类，如果只需要这两个，可以不用在 config.toml 中声明。

其他的类别，需要在config.toml中增加配置，如series：

```toml
[taxonomies] 
tag = "tags" 
series = "series" 
category = "categories"
```

以及增加配置,只列举了一个，如下：

```toml
[[menu.main]] 
identifier = "categories" 
pre = "" 
name = "分类" 
url = "categories" 
title = "" 
weight = 3 
```

### 3.5 增加关于界面

关于界面只有一个页面，与上面的分类有所不同。

步骤：

1.新建了一个about.md文件在post同级目录下。

```bash
hugo new about.md
```

2.在config.toml中增加配置。

```toml
[[menu.main]]
    identifier = "about"
    pre = ""
    name = "关于"
    url = "about"
    title = ""
    weight = 4	//weight控制它在菜单栏的前后位置,根据情况设置
```

**注意**：about因为只是单个页面，所以，不能添加到[taxonomies]（网站所有的分类标签）目录中，要不然就不会显示。
