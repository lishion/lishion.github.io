---
title: 使用travis自动部署hexo到github
date: 2018-06-20 20:37:50
tags:
  - 工具
  - 教程
categories: 工具
author: lishion
toc: true

---

在上一章[使用hexo+github搭建自己的博客](https://lishion.github.io/2018/06/07/%E4%BD%BF%E7%94%A8git+hexo%E5%BB%BA%E7%AB%8B%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2/)介绍了如何使用 hexo 以及 github 搭建自己的博客。但是随后我们就遇到了一个问题: 如何我们想在多台电脑上编辑我们的博客应该怎么办。当然，我们可以选择新建一个仓库|分支利用 github 在多台电脑上同步。但是这种方式有一点繁琐，每次写作之前我们要保证源文件的同步 还要保证生成新的资源并部署到 github。例如一次典型的写作我们需要:

```bash
git pull origin resource:resource # 假设我们的资源文件存储在 origin 分支
hexo new xxx
git push origin resource
hexo g
hexo d
```

使用 travis 就能让我们省去渲染和部署的命令，变成:

```shell~~
git pull origin resource:resource # 假设我们的资源文件存储在 origin 分支
hexo new xxx
git push origin resource
```

~~难道就只能节省两个命令吗?~~

~~哇，一共就五个命令。节省了两个等于节省了百分之四十啊!你说赛高不赛高。~~

## 使用 travis 进行自动部署

​	看看廖雪峰大佬对 [travis](http://www.ruanyifeng.com/blog/2017/12/travis_ci_tutorial.html) 的介绍:

> Travis CI 提供的是持续集成服务（Continuous Integration，简称 CI）。它绑定 Github 上面的项目，只要有新的代码，就会自动抓取。然后，提供一个运行环境，执行测试，完成构建，还能部署到服务器。

简单的说 travis 提供了一个平台，这个平台可以监听你 Github 上仓库的变化，如果发生改变。则会将你的代码拉取到 travis 的服务器上，并执行你事先定义好的操作。

### 初识 travis

只要你拥有 github 的账户就可以直接登录 [travis官网](https://www.travis-ci.org/)使用。如果没有。。。。那就去注册一个。登录之后的页面大概是这样的:

{% asset_img travis-mainpage.png travis主页 %}

当然如果你没有使用过那左边的仓库栏和右边的信息栏都是空吧的，这个时候点击仓库栏上面的+号，添加一个仓库。

{% asset_img travis-add.png 新增关联仓库 %}

当然这里我们要选择的是 hexo 部署到的仓库。

到这里，关联我们的仓库到 trvias 就完成了。但是我们还需要创建一个分支来保存我们的资源文件，因此，在我们的博客的根目录运行:

```bash
git check out -b resource
git add *
git commit -m "add resource" 
git add remote xxx # 这里的xxx就是博客对应的仓库
git push -u origin resource
```

这样把我们的资源文件同步到 github 上。

### 配置文件

当然，一个仓库有很多分支，我们还需要在根目录下配置一个 .travis.yml 文件来让 travis 知道我们需要构建哪一个分支。._config.yml 同时还告诉了 travis 在构建前需要做什么，构建后需要做什么。例如，我的 .travis.yml 如下:

```yaml
language: node_js　# 使用什么语言
node_js:
  - "10" # 语言版本
branches:
  only:
  - resource # 只构建 resource 分支
script:
  - hexo g # 构建时执行的命令
install:
  - npm install  hexo # 由于 travis 的服务器默认是不带 hexo 所以需要安装
  - npm install  hexo-cli
  - npm install hexo-deployer-git
  - sed -i'' "s~git@github.com:~https://${TOKEN}@github.com/~" _config.yml
after_success:
  - hexo d # 构建完成后执行的命令 这里就是进行部署
notifications: # 设置通知项
  email:
    - 544670411@qq.com
~                       
```

### 配置授权码

现在还存在一个问题。travis 是需要从 github 拉取到他的服务器上，并且还需要通过 hexo 将构建后的文件 push 到 github，然而在 travis 的服务器上是不存在我们的 ssh 公钥的。也就会说目前 travis 还对无法操作我们的 github 仓库。所以我们还需要生成一个 github 授权码:

{% asset_img author.png 新增授权码 %}

然后在 travis 中配置这个授权码:

{% asset_img token-add.png 配置授权码 %}

其中名字可以自定义。但是配置了这个授权码如何使用呢?答案就是在 .travis.yml 中配置:

```yaml
install:
  ...
  - sed -i'' "s~git@github.com:~https://${TOKEN}@github.com/~" _config.yml
```

这里的 ${TOKEN} 需要将 TOKEN 替换为你刚才设置的名字。

### 开始使用 

到这里，整个设置部分就完成了。现在只需要 git pull origin resource 。然后登录 travis 就能看到项目正在构建了。从日志可以看到构建成功还是失败。以后每次写文章就只需要:

```
git pull origin resource:resource
hexo new xxx
git push orgin resource
```

travis 就能帮你完成构建和部署的功能，是不是方便多了?






