---
title: 在hexo中书写latex公式
date: 2018-06-28 13:27:26
tags:
  - 教程
  - 工具
categories: Spark源码阅读计划
author: lishion
toc: true
---

昨天在写笔记的时候发现在 typora 中可以正确显示的 latex 公式 hexo 无法正确渲染。hexo 默认渲染器不支持 latex公式。需要安装插件才能使用。这里我采用了[hexo-renderer-markdown-it-plus](https://github.com/CHENXCHEN/hexo-renderer-markdown-it-plus) 替换默认的渲染器。其中公式渲染采用 [katex](https://khan.github.io/KaTeX/) 使用步骤如下:

1. 安装依赖:

   `npm uninstall hexo-renderer-marked`

   `npm install hexo-renderer-markdown-it-plus`

   `npm install markdown-it-katex`

2. 修改配置文件

   在`_config.yml`中添加:

   ```yaml
   markdown_it_plus:
       highlight: true
       html: true
       xhtmlOut: true
       breaks: true
       langPrefix:
       linkify: true
       typographer:
       quotes: “”‘’
       pre_class: highlight
   ```

3. 修改主题模板:

      主题的 head 模板位于: `themes/next-gux/layout/_partials/head.swig` 中。将

      ```
      <link href="https://cdn.bootcss.com/KaTeX/0.7.1/katex.min.css" rel="stylesheet">
      ```

      添加到`head.swig`文件末尾。

   完成以上步骤即可 hexo 即可正确的渲染 latex 公式。

