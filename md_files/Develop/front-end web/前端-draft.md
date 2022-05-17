## 前端-draft

### base

#### \<input>

##### placeholder

```html
<input type="text" autocomplete="off" placeholder="我是红色的" class="el-input__inner">
```

当表单控件为空时，控件中显示的内容.它还可以搭配css使用

```css
::placeholder {
  color: red;
  font-size: 1.5em;
}
```

效果如下：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/input-placeholder-css.jpg)

#### CSR js渲染的\<input>:happy:

这是我困惑好久的东西了哈哈哈，其实实现挺简单的，如下基于vue的实现

困惑点：页面-->Inspect 看到的input没有任何值的属性，但是页面上input框是填充了某个后端接口的返回值的。

原因：Inspect看到的是js渲染后的了~~~ Sources看到的input是有提示的

```html
    <div id="test">
       <p>{{ message }}</p>
       <input  v-model="message" />
    </div>
```

js代码

```bash
import * as Vue from 'vue'

const Test = {
  data() {
    return {
    ## 类比下这里只需掉后端接口就行 
      message: 'dodo'
    }
  }
}
Vue.createApp(Test).mount('#test')
```

就是如下的效果

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ssr-input.png)

但是页面的Sources还是显示了这个input绑定了v-mode

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/CSR-SITE-ALLOW-JS-2.png)

### DOM

#### 渲染

DOM渲染是浏览器展现给用户的DOM文档的生成的过程。

DOM渲染的演化过程：

　　①纯后端渲染

　　②纯前端渲染

　　③服务端的js渲染结合前端渲染

　　纯后端渲染：DOM树的生成完全是在后端服务器中完成，服务器的程序会把各种的数据拼成一个DOM树。采用这样的渲染方式，就是每一个页面中，在Chrome中展开得到的DOM，和服务器返回的DOM基本一致。这个过程会有少部分js代码，不过这并不影响DOM的主体是服务器返回的。

　　纯前端渲染：后端只返回一个DOM框架，DOM生成的主体逻辑在前端，js代码将页面的主体渲染到前端。

​		服务端的js渲染结合前端渲染：前面两种渲染中有一些任务，交给纯后端DOM渲染逻辑分离的不好，交给纯前端DOM渲染会造成较长的延迟，所以分离出来了一个独立DOM渲染阶段。从SEO角度来说很友好，但是加大了开发难度。

#### DOM 节点

根据 W3C 的 HTML DOM 标准，HTML 文档中的所有内容都是节点：

- 整个文档是一个文档节点
- 每个 HTML 元素是元素节点
- HTML 元素内的文本是文本节点
- 每个 HTML 属性是属性节点
- 注释是注释节点

HTML文本会被解析为DOM树, 树中的所有节点均可通过 JavaScript 进行访问。所有 HTML 元素（节点）均可被修改，也可以创建或删除节点。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ct_htmltree.png)



### Chrome调试

#### Inspect and sources  diff

“查看网页源代码”的代码内容是服务器发送到浏览器的原封不动的源代码，不包括页面动态渲染的内容；

“审查元素”包括源代码+js动态渲染的内容，即最终展示的html内容

当对一个CSR 页面，禁用和开启页面加载js时，审查元素/inspect的内容不一致

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/CSR-SITE-BLOCK-JS-1.png)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/CSR-SITE-ALLLOW-JS-1.png)

但是sources显示的都是一致的--

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/CSR-SITE-ALLOW-JS-2.png)

#### 

#### 键盘导航DOM树

点击页面元素选择DevTools中的Inspect图标。

在DOM树中选中元素，可以使用键盘导航DOM树。

使用**上下键**，顺序选择当前元素的前后兄弟节点。

通过**左右方向键**，展开和关闭当前节点的子节点。

**连续左方向键**，依次关闭已展开的DOM节点。

**连续右键**依次打开未展开的DOM元素。

#### 搜索节点

通过字符串、CSS选择器，或XPath选择器搜索DOM树。将光标对准[Element]面板。

使用快捷键：

```
Control+F或Command+ F（Mac）
```

将打开位于DOM树底部的搜索栏。输入：

关键字、CSS选择器、XPath选择器

#### 定位后端接口

点网络->直接ctrl+f 搜索前端页面显示的信息，入"MYSQL_ROOT"

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/chrome-devtool-1.jpg)

前端接口定位方法：

* 先查到提供数据的接口，再去前端源代码那里搜索那里调用的这个后端接口，从而看它的操作逻辑
* 选择页面一个固定文字或者label去代码里搜索

### 动态渲染/CSR

就酱：https://www.cxyzjd.com/article/lyclyc_/109962598

#### AngularJS ng-model

https://vimsky.com/examples/usage/angularjs-ng-model-directive.html

js代码可以使用后端取的接口数据，给这个input 渲染值

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ng-mode.jpg)

类似下面代码：

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<script src="https://cdn.staticfile.org/angular.js/1.4.6/angular.min.js"></script> 
</head>
<body>

<form ng-app="" name="myForm" ng-init="myText = 'test@runoob.com'">

Email:
<input type="email" name="myAddress" ng-model="myText" required>
<p>编辑邮箱地址，查看状态的改变。</p>

</form>
```

效果图：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/ng-mode-2.png)



### Vue例子

#### config server

```bash
$ cat vue.config.js
const { defineConfig } = require('@vue/cli-service')
module.exports = defineConfig({
  transpileDependencies: true,
  # 开启运行时编译
  runtimeCompiler: true,
  # port 
  devServer: {
     port: 9000,
  }
})

```

运行时编译可以解决，浏览器 -> F12 ->console的如下报错提示

Component provided template option but runtime compilation is not supported in this build of Vue. Configure your bundler to alias "vue"



#### demo: mount(#app)

Vue脚手架中生成的`main.js`中`mount(#app)`

```bash
import { createApp } from 'vue'
import App from './App.vue'

createApp(App).mount('#app')

```

这里创建的vue实例没有el属性，而是在实例后面添加了一个$mount(’#app’)方法。

$mount(’#app’) 是手动挂载到id为app的dom中的意思

该方法是直接挂载到入口文件index.html 的 id=app 的dom 元素上的

```bash
# cat ../public/index.html
<!DOCTYPE html>
<html lang="">
  <head>
    ...
  </head>
  <body>
    <noscript>
      <strong>We're sorry but <%= htmlWebpackPlugin.options.title %> doesn't worenable it to continue.</strong>
    </noscript>
    <div id="app"></div>
    <!-- built files will be auto injected -->
  </body>
</html>

```

#### 双向绑定input demo

```bash
# 脚手架初始化应用
$ vue create hello-world
$ cd hello-world
## 修改为如下内容
$ cat ./src/App.vue
$ cat ./src/main.js
import * as Vue from 'vue'

const Test = {
  data() {
    return {
      message: 'dodo'
    }
  }
}
Vue.createApp(Test).mount('#test')
$ cat ./public/index.html
<!DOCTYPE html>
<html lang="">
  <head>
  ...
  </head>
  <body>
   ## site 禁用 js时会显示
    <noscript>
      <strong>We're sorry but <%= htmlWebpackPlugin.options.title %> doesn't work properly without JavaScript enabled. Please enable it to continue.</strong>
    </noscript>
    <div id="test">
       <p>{{ message }}</p>
       <input  v-model="message" />
    </div>
    <!-- built files will be auto injected -->
  </body>
</html>
## 修改vue 配置文件
$ cat vue.config.js
const { defineConfig } = require('@vue/cli-service')
module.exports = defineConfig({
  transpileDependencies: true,
  runtimeCompiler: true,
  devServer: {
     port: 9000,
  }
})

```

App.vue为空即可，暂时不知道它的用途、

当site设置禁止加载JavaScript时，会显示`<noscript>`标签的内容

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/site-deny-js.png)

### 引用

1. https://blog.51cto.com/u_15301829/3087702
2. https://leohxj.gitbooks.io/front-end-database/content/html-and-css-basic/learn-dom-tree.html