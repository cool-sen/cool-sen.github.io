# Hugo搭建博客（二）—– Hugo+Github Pages搭建博客


![image-20200414125449742](/images/hugo%E6%90%AD%E5%BB%BA%E5%8D%9A%E5%AE%A2%E4%BA%8C/image-20200414125449742.png)

使用Hugo已经把博客搭建好了，那应该部署到哪里呢？可以使用VPS、云服务器等，我使用的是Github Pages，免费而且也很好用！

我使用过两种方式来部署，第一种部署简单，但是每次发表博客很麻烦，第二种部署虽然步骤较多，但是发表博客非常便捷，管理起来很非常方便。下面介绍一下在部署过程的步骤和解决的问题。

## 1 最简单的部署

### 1.1 创建Repository

在 Github 上创建一个 用户名.github.io 的 Repository。

### 1.2 **生成 public 目录**

```bash
hugo --baseUrl="http://cool-sen.github.io/
```

当然，也可以在config.toml中设置base URL，这样就不用每次设置base URL了，直接使用```hugo```命令即可

### 1.3 **推送到Github远程仓库**

这步就是将public目录添加到远程仓库。如下面的命令：

```bash
$ cd public 
$ git init  
$ git remote add origin https://github.com/cool-sen/cool-sen.github.io.git // 添加远程仓库
$ git add -A  // 跟踪所有文件
$ git commit -m "first commit" // 提交
$ git push -u origin master // 推送到远程仓库
```

注意：push出错时语句改为

```bash
$ git push -u origin master -f  //强制覆盖
```

此时访问 用户名.github.io就可以在线查看你的网站了。

这种方式，每次都需要手动使用```hugo```命令去生成public目录，不是很方便。下面介绍一种自动化的方式。

## 2 使用Github Actions自动构建博客

本地添加文章，提交到Github，之后会自动触发Github Actions帮助我们把刚刚添加的文章通过Hugo发布到Github Pages进行托管。之后即可通过 Github 给 Pages 生成的 URL 访问即可。

### 2.1 创建两个仓库

这里需要创建两个仓库，一个是 Github Pages仓库，也就是usename.github.io；另一个是Hugo文章仓库，来存放Hugo文章。

* 创建Github Pages仓库

  在你的 Github 账号里新建一个 Repository ，仓库名必须为 [你的用户名]-github.io，必须使用 master 分支，这个就是你建立 Github Pages 的 Repository.

* 创建Hugo文章仓库

  仓库名为：名称随意都可以，我设置是cool-sen.github.io.myBlog,可以设置为Private，看个人需求。

### 2.2 为两个仓库绑定 SSH Key

因为当我们在通过Git提交源码之后，Github Actions会编译生成静态文件并通过Git Push到 **`username.github.io`**，因此这一步需要 Git 账户认证。

#### 2.2.1 生成提交代码用的 *ssh key*

打开cmd或者其他命令行工具

```bash
ssh-keygen -t rsa -b 4096 -C user.email
```

![image-20200414132435869](/images/hugo%E6%90%AD%E5%BB%BA%E5%8D%9A%E5%AE%A2%E4%BA%8C/image-20200414132435869.png)

这种方式可以输入自定义的路径来保存SSH Key，这样就不会影响到电脑中旧的SSH Key。

将会得到以下这两个文件:

```
id_rsa_hugo_deploy.pub (public key)
id_rsa_hugo_deploy     (private key)
```

#### 2.2.2 填写密钥

假设 部署的项目为 `[你的用户名].github.io`，Hugo 文章的 Repository 名字是 `myBlog`。

1. 将 **Public Key**填写到[你的用户名].github.io

![image-20200414133132266](/images/hugo%E6%90%AD%E5%BB%BA%E5%8D%9A%E5%AE%A2%E4%BA%8C/image-20200414133132266.png)

红色箭头指的`Allow write access` 一定要勾上，否则会无法部署。

2. 将 **Private Key** 添加到 `[你的用户名].github.io.myBlog仓库`

![image-20200414133414017](/images/hugo%E6%90%AD%E5%BB%BA%E5%8D%9A%E5%AE%A2%E4%BA%8C/image-20200414133414017.png)

这里 Secrets 变量名要一定是：  `ACTIONS_DEPLOY_KEY`, 后面会用到。

### 2.3 本地准备

这一步主要是本地和仓库进行同步，假设Hugo 文章的 Repository 名字是 `myBlog`，将myBlog仓库克隆到本地，开始初始化 Hugo 系统，如果本地已经有Hugo 系统，只需要步到myBlog仓库即可。

### 2.4 编写 Github Actions 脚本

通过 Github 自动为我们仓库生成，注意是为 `[你的用户名].github.io.myBlog`仓库配置 Actions。

![image-20200414134244095](/images/hugo%E6%90%AD%E5%BB%BA%E5%8D%9A%E5%AE%A2%E4%BA%8C/image-20200414134244095.png)

new workflow后，会出现一个 yml 文件的编辑器。第一次使用出现的是`skip this : Set up a workflow yourself`。

接下来参考[peaceiris/actions-hugo](https://github.com/peaceiris/actions-hugo) 和 [peaceiris/actions-gh-pages](https://owovo.xyz/post/peaceiris/actions-gh-pages) 项目，编写自己的 workflow。如下面代码，修改完用户名信息，可以直接粘贴过去。

```yml
name: Github Pages #自动化的名称
on:
  push: # push 的时候触发
     branches: # 那些分支需要触发
      - master
jobs:
  build:
    runs-on: ubuntu-latest # 镜像市场
    steps:
      - name: checkout # 步骤的名称
        uses: actions/checkout@master #软件市场的名称
        # with: # 参数
         # submodules: true
      - name: Setup Hugo # 安装 Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest' # 使用Hugo最新版
          extended: true
      - name: Build # 编译
        run: hugo --minify
      - name: Deploy # 部署
        uses: peaceiris/actions-gh-pages@v2
        env:
         ACTIONS_DEPLOY_KEY: ${{ secrets.ACTIONS_DEPLOY_KEY }}
         EXTERNAL_REPOSITORY: 你的用户名/你的用户名.github.io
         PUBLISH_BRANCH: master
         PUBLISH_DIR: ./public
```

修改好之后，点击右上角 commit 提交即可。以后可以本地重新commit，更加方便。

### 2.5 推送和访问

搭建就结束后，我们可以访问Github为`[你的用户名]/[你的用户名].github.io` 仓库生成的域名： `https://[你的用户名].github.io/` 查看效果。

之后每次写完博客，直接在本地仓库push就可以访问了。也可以写个脚本，直接一键push。如：

```shell
msg="rebuilding site `date`"
echo $msg
if [ $# -eq 1 ]
then msg="$1"
fi

git add -A
git commit -m "$msg"
git push origin master
```

执行上面的代码后，Github 收到PUSH后Actions 就会自动开始构建了，等待结束大约1分钟不到即可打开网站域名。

快点试试吧...
