## [LGTM]彻底理解服务端渲染[转]

页面的渲染其实就是浏览器将HTML文本转化为页面帧的过程。

原文：https://github.com/yacan8/blog/issues/30

### 渲染

页面的渲染其实就是浏览器将HTML文本转化为页面帧的过程。

#### FPS

FPS全称每秒传输帧数，英文释义为：Frames Per Second

60fps，是大部分设备的默认值，也是让我们视觉上舒适的值。那么对于浏览器，我们希望1m内渲染至少60次帧（图）是完美状态，低于60次就存在掉帧的可能。从而视觉上表现为卡顿。

**1m内渲染60次，相当于16ms内完成一次浏览器的渲染。**

#### 页面刷新频率

屏幕是由成千上万个像素点组成的大矩阵，而这个矩阵以极快的速度“从左往右，从上往下”刷新“扫描”每个像素点的值。而人眼感觉不到这种超快的“闪烁”，比如电脑刷新屏幕的物理频率大概是60Hz。当屏幕扫描完一横排像素就会产生“水平同步信号”，当屏幕扫描完所有横排即一屏的像素，就会产生“垂直同步信号”。**也就是“垂直同步信号”反映的是屏幕刷新的频率。**

不同帧率给人的直观感受：

- 帧率能够达到 50 ～ 60 FPS 的动画将会相当流畅，让人倍感舒适；
- 帧率在 30 ～ 50 FPS 之间的动画，因各人敏感程度不同，舒适度因人而异；
- 帧率在 30 FPS 以下的动画，让人感觉到明显的卡顿和不适感；
- 帧率波动很大的动画，亦会使人感觉到卡顿

要想页面给用户丝滑顺畅的感受，就要努力完成每秒60帧的目标💪，具体到每一帧就要控制在16.67ms之内，因为浏览器内部通常还会有不同的线程之间调度的工作，所以通常留给我们的V8的时间只有10ms，我们应该尽量保证在10ms的时间内完成js的执行。。。

#### 16ms

FPS 表示的是每秒钟画面更新次数，当今大多数设备的屏幕刷新率都是60次/秒，在两次硬件刷新之间浏览器进行两次重绘是没有意义的只会消耗性能，那么浏览器渲染动画或页面的每一帧的速率，也需要跟设备的刷新率保持一致。

也就是说，浏览器对每一帧画面的渲染工作需要在16ms（1000ms/60）之内完成，也就是说每一次渲染都要在 16ms才不会掉帧。

1m内渲染60次，相当于16ms内完成一次浏览器的渲染。

**在这16ms 内浏览器要完成的工作有：**

- 脚本执行（JavaScript）：脚本造成了需要重绘的改动，比如增删 DOM、请求动画等
- 样式计算（CSS Object Model）：级联地生成每个节点的生效样式。
- 布局（Layout）：计算布局，执行渲染算法
- 重绘（Paint）：各层分别进行绘制（比如 3D 动画）
- 合成（Composite）：将位图发送给合成线程

#### 渲染流程

1. 浏览器通过请求得到一个HTML文本
2. 渲染进程解析HTML文本，构建DOM树
3. 解析HTML的同时，如果遇到内联样式或者样式脚本，则下载并构建样式规则（stytle rules），若遇到JavaScript脚本，则会下载执行脚本。
4. DOM树和样式规则构建完成之后，渲染进程将两者合并成渲染树（render tree）
5. 渲染进程开始对渲染树进行布局，生成布局树（layout tree）
6. 渲染进程对布局树进行绘制，生成绘制记录
7. 渲染进程的对布局树进行分层，分别栅格化每一层，并得到合成帧
8. 渲染进程将合成帧信息发送给GPU进程显示到页面中

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/browser-render.png)

### 动态渲染/客户端渲染/CSR

而如今我们大部分WEB应用都是使用 JavaScript 框架（Vue、React、Angular）进行页面渲染的，也就是说，在执行 JavaScript 脚本的时候，HTML页面已经开始解析并且构建DOM树了，JavaScript 脚本只是动态的改变 DOM 树的结构，使得页面成为希望成为的样子，这种渲染方式叫动态渲染，也可以叫客户端渲染（client side rende）。

#### nodejs CSR

nodejs的出现代表前端可以使用js来处理涉及后端的东西，也就代表着前端可以不依赖后端处理很多事情

nodejs出现后，慢慢实现了前后端分离，网页开始被当成了独立的应用程序（SPA，Single Page Application），前端团队接管了所有页面渲染的事，后端团队只负责提供所有数据查询与处理的API，大体流程是这样的：首先浏览器请求URL，前端服务器直接返回一个空的静态HTML文件（不需要任何查数据库和模板组装），这个HTML文件中加载了很多渲染页面需要的 JavaScript 脚本和 CSS 样式表，浏览器拿到 HTML 文件后开始加载脚本和样式表，并且执行脚本，这个时候脚本请求后端服务提供的API，获取数据，获取完成后将数据通过JavaScript脚本动态的将数据渲染到页面中，完成页面显示。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/csr.png)

这一个前后端分离的渲染模式，也就是客户端渲染（CSR）

#### CSR缺点

##### 白屏时间长

客户端渲染，需要先得到一个空的HTML页面，这个时候页面已经进入白屏，之后还需要经过加载并执行 JavaScript、请求后端服务器获取数据、JavaScript 渲染页面几个过程才可以看到最后的页面。特别是在复杂应用中，由于需要加载 JavaScript 脚本，越是复杂的应用，需要加载的 JavaScript 脚本就越多、越大，这会导致应用的首屏加载时间非常长，进而降低了体验感。

##### 不利于SEO

爬虫也分低级爬虫和高级爬虫。

- 低级爬虫：只请求URL，URL返回的HTML是什么内容就爬什么内容。
- 高级爬虫：请求URL，加载并执行JavaScript脚本渲染页面，爬JavaScript渲染后的内容。

也就是说，低级爬虫对客户端渲染的页面来说，简直无能为力，因为返回的HTML是一个空壳，它需要执行 JavaScript 脚本之后才会渲染真正的页面。而目前像百度、谷歌、微软等公司，有一部分年代老旧的爬虫还属于低级爬虫，使用服务端渲染，对这些低级爬虫更加友好一些。



### 服务端渲染/SSR

服务端渲染（server side render）？顾名思义，服务端渲染就是在浏览器请求页面URL的时候，服务端将我们需要的HTML文本组装好，并返回给浏览器，这个HTML文本被浏览器解析之后，不需要经过 JavaScript 脚本的执行，即可直接构建出希望的 DOM 树并展示到页面中。这个服务端组装HTML的过程，叫做服务端渲染。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/csr-and-ssr.png)

#### web1.0 ssr

在没有AJAX的时候，也就是web1.0时代，几乎所有应用都是服务端渲染（此时服务器渲染非现在的服务器渲染），那个时候的页面渲染大概是这样的，浏览器请求页面URL，然后服务器接收到请求之后，到数据库查询数据，将数据丢到后端的组件模板（php、asp、jsp等）中，并渲染成HTML片段，接着服务器在组装这些HTML片段，组成一个完整的HTML，最后返回给浏览器，这个时候，浏览器已经拿到了一个完整的被服务器动态组装出来的HTML文本，然后将HTML渲染到页面中，过程没有任何JavaScript代码的参与。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/web1.0-ssr.png)

在WEB1.0时代，服务端渲染看起来是一个当时的最好的渲染方式，但是随着业务的日益复杂和后续AJAX的出现，也渐渐开始暴露出了WEB1.0服务器渲染的缺点。

- 每次更新页面的一小的模块，都需要重新请求一次页面，重新查一次数据库，重新组装一次HTML
- 前端JavaScript代码和后端（jsp、php、jsp）代码混杂在一起，使得日益复杂的WEB应用难以维护

而且那个时候，根本就没有前端工程师这一职位，前端js的活一般都由后端同学 jQuery 一把梭。但是随着前端页面渐渐地复杂了之后，后端开始发现js好麻烦，虽然很简单，但是坑太多了，于是让公司招聘了一些专门写js的人，也就是前端，这个时候，前后端的鄙视链就出现了，后端鄙视前端，因为后端觉得js太简单，无非就是写写页面的特效（JS），切切图（CSS），根本算不上是真正的程序员。

#### Nodejs SSR

基于nodejs CSR 模式由于JavaScript 脚本也不断的臃肿起来，使得首屏渲染相比于 Web1.0时候的服务端渲染，也慢了不少

于是前端团队又选择了使用 nodejs 在服务器进行页面的渲染，进而再次出现了服务端渲染。大体流程与客户端渲染有些相似，首先是浏览器请求URL，前端服务器接收到URL请求之后，根据不同的URL，前端服务器向后端服务器请求数据，请求完成后，前端服务器会组装一个携带了具体数据的HTML文本，并且返回给浏览器，浏览器得到HTML之后开始渲染页面，同时，浏览器加载并执行 JavaScript 脚本，给页面上的元素绑定事件，让页面变得可交互，当用户与浏览器页面进行交互，如跳转到下一个页面时，浏览器会执行 JavaScript 脚本，向后端服务器请求数据，获取完数据之后再次执行 JavaScript 代码动态渲染页面。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/nodejs-ssr.png)



#### SSR缺点

并不是所有的WEB应用都必须使用SSR，这需要开发者自己来权衡，因为服务端渲染会带来以下问题：

- 代码复杂度增加。为了实现服务端渲染，应用代码中需要兼容服务端和客户端两种运行情况，而一部分依赖的外部扩展库却只能在客户端运行，需要对其进行特殊处理，才能在服务器渲染应用程序中运行。
- 需要更多的服务器负载均衡。由于服务器增加了渲染HTML的需求，使得原本只需要输出静态资源文件的nodejs服务，新增了数据获取的IO和渲染HTML的CPU占用，如果流量突然暴增，有可能导致服务器down机，因此需要使用响应的缓存策略和准备相应的服务器负载。
- 涉及构建设置和部署的更多要求。与可以部署在任何静态文件服务器上的完全静态单页面应用程序 (SPA) 不同，服务器渲染应用程序，需要处于 Node.js server 运行环境。

所以在使用服务端渲染SSR之前，需要开发者考虑投入产出比，比如大部分应用系统都不需要SEO，而且首屏时间并没有非常的慢，如果使用SSR反而小题大做了。



### 同构

#### 同构的定义

在服务端渲染中，有两种页面渲染的方式：

- 前端服务器通过请求后端服务器获取数据并组装HTML返回给浏览器，浏览器直接解析HTML后渲染页面
- 浏览器在交互过程中，请求新的数据并动态更新渲染页面

这两种渲染方式有一个不同点就是，一个是在服务端中组装html的，一个是在客户端中组装html的，运行环境是不一样的。所谓同构，就是让一份代码，既可以在服务端中执行，也可以在客户端中执行，并且执行的效果都是一样的，都是完成这个html的组装，正确的显示页面。也就是说，一份代码，既可以客户端渲染，也可以服务端渲染。

#### 同构的条件

为了实现同构，我们需要满足什么条件呢？首先，我们思考一个应用中一个页面的组成，假如我们使用的是`Vue.js`，当我们打开一个页面时，首先是打开这个页面的URL，这个URL，可以通过应用的`路由`匹配，找到具体的页面，不同的页面有不同的视图，那么，视图是什么？从应用的角度来看，视图 = `模板` + `数据`，那么在 Vue.js 中， 模板可以理解成`组件`，数据可以理解为`数据模型`，即响应式数据。所以，对于同构应用来说，我们必须实现客户端与服务端的路由、模型组件、数据模型的共享。

### Demo

#### 基于nodejs服务端渲染

首先，模拟一个最简单的服务器渲染，只需要向页面返回我们需要的html文件。

```js
const express = require('express');
const app = express();

app.get('/', function(req, res) {
    res.send(`
        <html>
            <head>
                <title>SSR</title>
            </head>
            <body>
                <p>hello world</p>
            </body>
        </html>
    `);
});

app.listen(3001, function() {
    console.log('listen:3001');
});
```

启动之后打开localhost:3001可以看到页面显示了hello world。而且打开网页源代码。如下，所示block site js仍然能正常显示页面

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/demo-ssr.png)

当浏览器拿到服务器返回的这一段HTML源代码的时候，不需要加载任何JavaScript脚本，就可以直接将hello world显示出来。

#### 基于vue客户端渲染

我们用 `vue-cli`新建一个vue项目,修改`index.html`、`App.vue`和`main.js`

清空`App.vue`，并在脚手架生成的index.html中添加如下内容

```html
    <div id="test">
       <p>{{ message }}</p>
       <input  v-model="message" />
    </div>
```

main.js代码

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

并修改vue的配置文件

```bash
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

## 启动应用,因为是容器环境所以对外端口是31011
$ npm run server
```

当site禁止加载js时的页面效果

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/CSR-SITE-BLOCK-JS-1.png)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/CSR-SITE-BLOCK-JS-1.png)

当site allow js时

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/CSR-SITE-ALLLOW-JS-1.png)

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/CSR-SITE-ALLOW-JS-2.png)



除了简单的兼容性处理 noscript 标签以外，只有一个简单的id为test的div标签，通过双向绑定给input提供初始值，并随着input输入修改\<p>标签显示内容。而当加载了下面的 script 标签的 JavaScript 脚本之后，页面开始这行这些脚本，执行结束，input 正常显示。

#### Inspect and sources 

“查看网页源代码”的代码内容是服务器发送到浏览器的原封不动的源代码，不包括页面动态渲染的内容；

“审查元素”包括源代码+js动态渲染的内容，即最终展示的html内容

### Vue.js

Vue.js 是一个构建客户端应用的框架。默认情况下，作为其输出，Vue 组件会在浏览器中生产并封装 DOM。然而，我们也可以在服务器端把同样的组件渲染成 HTML 字符串，然后直接将其发送给浏览器，并最终将静态标记“激活”为完整的、可交互的客户端应用。

既然是Vue的服务端渲染，那么**首先得有一个单页面的Vue应用实例**，继而**在这个单页面的Vue实例上进行同构**，**服务端解析这个Vue实例并渲染成一段html字符串**，最后**交给前端浏览器直接显示出来**。

```js
const app = new Vue({
  template: `<div><button @click="doClick">点击</button></div>`
  methods: {
    doClick() {
        alert(1);
    }
  }
});
app.$mount("#app", true); // 强制进行客户端激活
```

假设现在**服务器通过解析Vue实例并渲染成了一段html字符串**，如:

```html
// 服务器解析Vue实例渲染生成了如下字符串

<div id="app"><button>点击</button></div>
```

浏览器拿到服务端渲染出来的字符串之后**可以看到该按钮**。但是存在一个问题，**那就是按钮上的事件丢失了**，因为**服务端渲染的结果仅仅是HTML字符串**，**并没有包含处理事件的JS文件**。解决的办法就是**客户端激活**，也就是**让整个单页面的Vue实例重新走一遍**。

```xquery
<script src="https://cdn.bootcdn.net/ajax/libs/vue/2.6.9/vue.min.js"></script>

<div id="app"><button>点击</button></div>

<script>
const app = new Vue({
  template: `<div><button @click="doClick">点击</button></div>`
  methods: {
    doClick() {
        alert(1);
    }
  }
});
app.$mount("#app", true); // 强制进行客户端激活
</script>
```

可以发现按钮事件可以生效了，重新执行一遍Vue单页面实例由于**Vue实例中配置了template模板**，所以**Vue渲染的时候其实会忽略服务端渲染出来的字符串**，导致**页面重新渲染并挂载**，从而**浪费性能**，我们**应该复用服务端渲染出来的字符串**。解决方法就是，**我们给服务端渲染的结果添加一个标识**。

```xml
<div id="app" data-server-rendered="true"><button>点击</button></div>
```

data-server-rendered="true"表示**让客户端 Vue 知道这部分 HTML 是由 Vue 在服务端渲染的**，并且**应该以激活模式进行挂载**。**在没有 data-server-rendered 属性的元素上，还可以向 $mount 函数的第二个参数位置传入true，来强制使用激活模式(hydration)**。

```arcade
app.$mount("#app", true); // 强制进行客户端激活
```

end