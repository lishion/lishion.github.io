---
title: Vue.js入门
date: 2018-07-31 10:43:15
tags:
  - vue
  - 编程
categories: 前端
author: lishion
toc: true
---
最近用 electron + vue 做了一个小东西，这里把有关vue的东西记录一下。

# 入门

使用 vue 搭建一个hello world是非常简单的，如下面的代码:

```html
<script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
<div id="app">
    {{ message }}
</div>
<script>
    var app = new Vue({
        el:'#app',
        data:{
            message:'hello world'
        }
    })
</script>
<!--打开这个html，你就会看到 hello world-->
```

这里首先引入了 vue.js 的依赖，随后将一个Mustache”语法 (双大括号) 的文本插值插入到`<div>`标签中，接着实例化了一个vue实例。这个时候message已经和渲染的Dom关联起来了，如果你在控制台中修改`app.message=5`，那么你会看到对应的渲染结果也会改变。

## 语法
### 文本插值

可以使用双大括号使用文本插值，例如入门中提到的内容。需要注意的是文本插值只能显示文本，所有的 html 标签都会按照原样输出。如果你想插入 html 标签，你可以使用`v-html`

### v-html

让我们修改一下入门实例来展示 v-html 的用法:

```html
<script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
<div id="app" v-html="message">
</div>
<script>
    var app = new Vue({
        el:'#app',
        data:{
            message:'<span style="color:red">hello world</span>'
        }
    })
</script>
<!--打开这个html，你就会看到红色的 hello world-->
```

### 指令

在某些时候，不但需要修改dom的值或者html，还需要修改dom的属性，这时可以使用v-bind指令，例如:

```html
<script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
<div id="app" v-html="message" v-bind:id="id">
</div>
<script>
    var app = new Vue({
        el:'#app',
        data:{
            message:'<span style="color:red">hello world</span>',
            id:'test'
        }
    })
</script>
<!--最开始打开时，div的id='test'，可以在控制台中改变app.id的值，随后再查看id的只已经发生变化-->
```

`v-bind:id` 可以简写为 `:id`。

当然还可以绑定修改css的值。例如:

```html
<script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
<div id="app" :style="style">
    {{message}}
</div>
<script>
    var app = new Vue({
        el:'#app',
        data:{
            message:'hello world',
            style:{
                 'color':"blue",
                'font-size':'3em'
            }
        }
    })
</script>
```

这里的 :style 相当于 v-bind:style。传入的style数据为一个json对象，将css转化为key:value，例如 'color':'red' 。

这里还有一些其他的指令，例如:

```html
<div v-if="seen"></div> <!--seen为false则不会显示-->
<div v-else-if="otherSeen"></div>
<div v-else></div>
  
<div v-on:click="click"></div> <!--将点击事件绑定到click-->
```

### 列表渲染

列表渲染使用的是v-for指令，一个简单的例子如下:

```html
<script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>

<ul id="app">
    <li v-for="item in items">
         {{item}}
    </li>
</ul>

<script>
    var app = new Vue({
        el:'#app',
        data:{
           items:["a","b","c","d"]
        }
    })
</script>
```

在某些时候我们需要知道迭代的index，还可以这样写:

```html
<script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>

<ul id="app">
    <li v-for="(item,index) in items">
        {{index+item}}
    </li>
</ul>

<script>
    var app = new Vue({
        el:'#app',
        data:{
           items:["a","b","c","d"]
        }
    })
</script>
```

还可以迭代一个对象的属性:

```html
<script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>

<ul id="app">
    <li v-for="(key,value) in items">
        {{key}}:{{value}}
    </li>
</ul>

<script>
    var app = new Vue({
        el:'#app',
        data:{
           items:{
               firstName:"li",
               lastName:"lo",
               age:30,
           }
        }
    })
</script>
```

同时，还可以同访问item.length来判断数组的长度，这样就能正对第一个或最后一个做出特殊处理:

```html
<script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>

<div id="app">
    <div v-for="(item,index) in items">
        <div v-if="index === 0">
            first {{item}}
        </div>
        <div v-else-if="index === items.length-1">
            last {{item}}
        </div>
        <div v-else>
            other {{item}}
        </div>
    </div>
</div>

<script>
    var app = new Vue({
        el: '#app',
        data: {
            items: [1, 2, 3, 4]
        }
    })
</script>
```



这里需要注意的是，并不是所有数据的变动都能被检测到，例如使用下标直接对数组进行赋值的变动就不能被检测到。可以使用:

```js
vm.$set(array,index,value)
//替代
array[index] = value
//同样
vm.$set(dict,key,value)
//替代
array[key] = value
```

更多请看[注意事项](https://cn.vuejs.org/v2/guide/list.html#%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9)。

### 获取表单输入

```html
<script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>

<div id="app">
    <input v-model="value" type="text">
    <p>{{value}}</p>
</div>

<script>
    var app = new Vue({
        el: '#app',
        data: {
            value: ""
        }
    })
</script>
<!--显示值会随着输入改变-->
```

使用 v-model 可以获取表单输入。

## 组件

组件是一个可以复用的Vue实例，用如下的代码可以定义并使用一个组件:

```javascript
<script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>

<div id="app">
    <button-counter></button-counter>
</div>

<script>
  	let t = '<button v-on:click="count++">You clicked me {{ count }} times.</button>'
    Vue.component('button-counter', {
        data: function () {
            return {
            	count: 0
            }
        },
        template: t
    })
    var app = new Vue({
        el: '#app',
    })
</script>
```

在组件内同样有一些绑定的数据需要初始化，这里同样data属性。但是注意，这里的 data 属性**必须是一个函数**，这是为了使得每一个模板对象的数据都是**独立的**。例如不管使用多少次`<button-counter></button-counter>`模板，它的点击次数都是相互独立的。

### 注册

在刚的例子中，组件是**全局注册的**，也就是说它们在注册之后可以用在任何新创建的 Vue 根实例 (`new Vue`) 的模板中。在很多时候我们使用webpack对应用进行打包，可以这样写:

```html
<!--xxx.vue-->
<div>
	xxx
</div>

<!--当需要使用的使用-->
<div>
  <xxx></xxx>
</div>

<script>
import xxx from './xxx.vue'
export default {
  components: {
    xxx
  }
}
</script>
```

### 使用props向子组件传递数据

例如我们写了这样一个子组件,它获取一个数组，并将数组的内容展开，这个时候我们希望通过父组件传入数据可以使用 props:

```html
<!--my-list.vue-->
<ul>
  <li v-for="for item in items">
    {{item}}
  </li>
</ul>
<script>
  export default{
    name:"myList"
    props:['items']
  }
</script>
```
在父组件中将子组件中申明的props作为自定义属性传入，例如:
```html
<div id="app">
  <myList items="items"></myList>
</div>
<script>
	import myList from './xxx.vue'
    export default {
      components: {mylist},
      data:function(){
        return {
          items:[1,2,3,4]
        }
      }
    }
</script>
```

这里的对应关系:

```html
子组件 如 myList:
<script>
  export default{
   	//xxx
    props:['items']
    //xxx
  }
</script>

父组件:
<myList items="data"></myList>
```

### 使用slot向子组件传递html结构

使用props可以向子组件传递数据，但是如果某些时候我们想往子组件传递我们自己编写的html结构，就可以使用`slot`：

```html
<!--myList.vue-->
<ul>
  <li v-for="for item in items">
    <slot></slot>
  </li>
</ul>

<!--父组件-->
<div>
  <myList>
  	<p>hello</p>
  </myList>
</div>
<script>
	import myList from './xxx.vue'
    export default {
      components: {mylist},
      data:function(){
        return {
          items:[1,2,3,4]
        }
      }
    }
</script>

<!--渲染后的myList.html-->
<ul>
  <li v-for="for item in items">
    <p>hello</p>
  </li>
</ul>
```

如果我们需传递多个结构:

```html
<!--myList.vue-->
<ul>
  <li v-for="for item in items">
    <slot name="header"></slot>
    ...
    <slot name="main"></slot>
  </li>
</ul>

<!--父组件-->
<div>
  <myList>
  	<template slot="header">
    	<h1>Here might be a page title</h1>
  	</template>
    <template slot="main">
    	<p>Here's some contact info</p>
  	</template>
  </myList>
</div>
<script>
	import myList from './xxx.vue'
    export default {
      components: {mylist},
      data:function(){
        return {
          items:[1,2,3,4]
        }
      }
    }
</script>

<!--渲染后的myList.html-->
<ul>
  <li v-for="for item in items">
    <h1>Here might be a page title</h1>
    ...
    <p>Here's some main info</p>
  </li>
</ul>
```

思考这样一个需求，我们写了一个通用的列表展示的组件todoList，这组件用来展示带办事项。

```html
<ul>
  <li
    v-for="todo in todos"
    v-bind:key="todo.id"
  >
    <slot v-bind:todo="todo">
      {{ todo.text }}
    </slot>
  </li>
</ul>
```

现在有这样一个要求，对于未完成事项，需要在前面加上一个x，而完成的事件需要在前面加上一个o。我们可以对组件进行这样的修改:

```html
<ul>
  <li
    v-for="todo in todos"
    v-bind:key="todo.id"
  >
    <slot v-bind:todo="todo">
      {{ to.isComplete ? 'o'+todo.text:'x'+todo.text }}
    </slot>
  </li>
</ul>
```

但是这是一个十分不优雅的实现方式，因为它限制了组件的通用范围。例如有的用户并不想利用 x 或 o 作为一个标志而想使用一个更优美的图表。又例如某些时候to-list也许还需要按照事项的重要程度做出一定的标志，这个组件明显是无法满足的。这个时候可以使用 slot-scope 使得父组件获取子组件的数据。例如:

```html
<!--todo-list.vue-->
<ul>
  <li
    v-for="todo in todos"
    v-bind:key="todo.id"
  >
    <!-- 我们为每个 todo 准备了一个插槽，-->
    <!-- 将 `todo` 对象作为一个插槽的 prop 传入。-->
    <slot v-bind:todo="todo">
      {{ todo.text }}
    </slot>
  </li>
</ul>

<!--父组件-->
<todo-list :todos="todos"> <!--子组件需要申明todos为props--> 
  <!-- 将 `slotProps` 定义为插槽作用域的名字 -->
  <template slot-scope="slotProps">
    <span v-if="slotProps.todo.isComplete">✓</span>
    <span v-else-if="slotProps.todo.level==='normal'">！         </span>
    <span v-else-if="slotProps.todo.level==='important'">!!!     </span>
    {{ slotProps.todo.text }}
  </template>
</todo-list>
```

这样优化之后便可以在父组件中根据子组件的数据做出不同的渲染效果，使得组件的通用性更高。事实上，副组件还可以如下简化:

```html
<todo-list :todos="todos"> <!--子组件需要申明todos为props--> 
  <!-- 将 `slotProps` 定义为插槽作用域的名字 -->
  <template slot-scope="{{todo}}">
    <span v-if="todo.isComplete">✓</span>
    <span v-else-if="todo.level==='normal'">!</span>
    <span v-else-if="todo.level==='important'">!!!</span>
    {{todo.text }}
  </template>
</todo-list>
```

具体参照:[解构 `slot-scope`](https://cn.vuejs.org/v2/guide/components-slots.html#%E8%A7%A3%E6%9E%84-slot-scope)

### props以及slot

在我开始使用vue的时候不清楚props和slot的使用场景，觉得这两个很相似，都可以网子组件传递数据，但是其实差别还是很大的:

1. props 传递**数据**很简单，同时也可以传递**html结构**，前提是**html结构**是从数据生成或渲染出的的，例如**html结构字符串**。也就是说无法传递**自己编写的html**，除非你以字符串的形式编写 html。例如:

   ``` html
   <child item='[1,2,3,4]'></child>            ✓
   <child item='["<p>xxx</p>","<p>xxx</p>","<p>xxx</p>","<p>xxx</p>"]'></child>         不建议
   ```

2. slot 既可以传递数据也可以传递html结构，但是一般用来传递自己编写的html结构。

   ​

## 使用中遇到的一点问题

以下是为实际工程中遇到的问题，解决方案大多数也是在网上搜索的结果，因此不能保证它们是最优的解决方案。只能保证可以解决我所遇到的问题。

### 向父组件传递事件

当一个子组件包含button时，需要将点击时间传递给父组件，也许还需要传递点击的哪一个按钮，可以使用`$emit`。

```html
<!--子组件-->
<button @click="click(someValue)"></button>
export default{
    methods:{
        click(value){
			this.$emit('event-name',value)
        }
    }
}
<!--父组件可以使用$event获取到子组件传递的值-->
<child @event-name="$event"></child>
<!--也可以将其传递给一个处理方法-->
<child @event-name="eventHandle($event)"></child>
```

### 向兄弟组件传递事件

首先定义一个 vue 实例作为中间件:

```javascript
//bus.js
import Vue from "vue";

export default new Vue();
```

在两个不相关的组件中:

````javascript
//compemnt a 发起一个事件
import Bus from 'your-path/bus.js'
Bus.$emit("event",value)

//component b 接收一个事件
import Bus from 'your-path/bus.js'
Bus.$on("event",(value)=>{
  	//do some thing
})
````

### __dirname

在项目中涉及到一个本地数据库的路径问题。最开始写的路径为相对路径:

`../../xxx.db`

随后发现桌面上的快捷方式与根目录的快捷方式生成的数据文件位置不一样，才知道相对路径是相对**执行时**的路径，而使用`__dirname`的到的才是**执行文件所在的绝对路径**。因此最后改为:

`path.join(__dirname,'../../xxx.db')`

### export 与 export_default

export default 每一个js文件只能有一个，import 引用的时候不需要{}。例如:

```javascript
//a.js
export default a = 5
//b.js
import a from 'a.js'
```

export 可以有很多个，引入的时候需要{}:

```javascript
//a.js
export var a = 5
export var b =6

//b.js
import {a,b} from 'a.js'
```

### 引入静态资源

在使用webpack打包的时候无法使用相对路径，这是后有两种解决办法:

```html
<!--方法1-->
<img :url='url'></img>
<script>
	import pic from 'your-path/xxx'
    export default{
      data:function(){
        return {
          url:pic
        }
      }
    }
</script>
<!--方法2-->
<img :url='url'></img>
<script>
  export default{
    data:function(){
      return {
        url:require(your-path/xxx)
      }
    }
  }
</script>
```

在需要动态的改变url时使用第二种方法更好。