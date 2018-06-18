---
title: 使用git + hexo 建立个人博客
categories: 工具
date: 2018-06-07 10:42:45
tag: 
  - 工具
  - 教程
---

## 安装 hexo

1. 安装`node`
2. 安装`npm`
3. 安装`hexo: npm install -g hexo-cli`


## 建立一个新博客

1. `hexo init blog`:使用该命令会在blog目录下建立一个博客并且在`source/_posts/hello-world.md`生成一篇名为`hello world`的文章，随后你可以选择删除它并新建自己的文章。
2. `cd blog`
3. `hexo g`:使用该命令将 `source/_posts/hello-world.md` 渲染为`html、css、js`静态资源
4. `hexo s`:开启服务器。然后http://localhost:4000/

## 关联至github

1. 新建仓库`xxx.github.io`，这里 xxx 可以是你想要取的名字，但是必须以`github.io`结尾

2. 此时可以访问`https://xxx.github.io`，但是没有内容

3. 修改`blog`目录下配置文件`_config.yml`，找到`deploy`选项，修改(新增)为:

   ```yaml
   deploy:
     type: git
     repository: git@github.com:xxx/xxx.github.io.git 
     branch: master
   ```

4. 安装插件`npm install hexo-deployer-git --save`

5. `hexo d -g` 生成内容后部署

6. 访问`https://xxx.github.io`，应该要延迟一段时间才能看到效果

## 更换主题

由于初始的主题不怎么好看，可以选择更换一下主题。官方主题地址为[hexo-themes](https://hexo.io/themes/)。本教程中采用[maupassant-hexo](https://www.haomwei.com/technology/maupassant-hexo.html#%E6%94%AF%E6%8C%81%E8%AF%AD%E8%A8%80)为主题

1. 由于大部分的主题都托管在`github`上，在`blog`目录下运行:

   `git clone https://github.com/tufu9441/maupassant-hexo.git themes/maupassant` 

   `themes/xxx`是`hexo`存放`xxx`主题的目录

2. `npm install hexo-renderer-pug --save`

3. `npm install hexo-renderer-sass --save --registry=https://registry.npm.taobao.org`

4. 修改`_config.yml`中主题为`theme: maupassant`

5. `hexo g`重新生成

6. `hexo s`开启服务器

主题还有许多可用的配置，请参照上面给出的链接进行设置

## 目录、tag

需要`归档`和`tag`只需要在`markdown`上加上一些`YAML`头部信息:

```
---
title: hello
categories: 杂谈
tag: 杂七杂八
---
```

即可

