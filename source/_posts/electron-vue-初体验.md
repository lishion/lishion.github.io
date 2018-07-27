---
title: electron-vue 初体验
date: 2018-07-24 18:56:08
tags:
  - electron
  - 编程
categories: python笔记
author: lishion
toc: true
---

## 安装步骤

使用 electron-vue 可以方便的结合 eletron 与 vue。使用步骤如下:

1. 安装 vue-cli:

   `npm install vue-cli -g`

2. 使用 vue-cli 初始化 eletron-vue 环境

   `vue init simulatedgreg/electron-vue my-projec`

   过程中会输入一些信息。

3. 安装完成后:

   ```bash
     $ cd my-projec
     $ npm install
     $ npm run dev
   ```

   如过没有问题就能看到打开的 electron-vue 的欢迎界面

## 主要文件

我们只需要关注 src 文件夹下的几个子文件夹:

* main/index.js : electron 的入口，也就是主进程
* renderer: 有关渲染的文件夹:
  * assests : 存放静态资源
  * components : 存放组件
  * router : 路由
  * main.js : 渲染进程的入口

我们试着来编写一个hello world代码:

修改 router 下的 index.js 为:

```
import Vue from 'vue'
import Router from 'vue-router'

Vue.use(Router)

export default new Router({
  routes: [
    {
      path: '/',
      name: 'landing-page',
      component: require('@/components/LandingPage').default
    },
    {
      path:'/test',
      name: 'test',
      component:require('@/components/test').default
    },
    {
      path: '*',
      redirect: '/'
    }
  ]
})

```

 在 components 下新增 test.vue:

```vue
<template>
    <div>Hello World</div>
</template>
<script>
export default {
    name:"test"
}
</script>
```

修改 main/index.js 中的 winURL 为:

```
const winURL = process.env.NODE_ENV === 'development'
  ? `http://localhost:9080/#/test`
  : `file://${__dirname}/index.html`
```

然后 `npm run dev` 就可以看到打开的页面。

## 集成ElementUL

1. npm i element-ui 

2. 在renderer/main.js中添加:

   ```
   import ElementUI from 'element-ui';
   import 'element-ui/lib/theme-chalk/index.css';
   Vue.use(ElementUI);
   ```

   即可以使用 ElementUI 的组件

   ​
