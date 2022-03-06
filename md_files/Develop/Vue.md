## Vue

### 界面逻辑and业务逻辑



### MVC

MVC design pattern **splits** an application into three main aspects: Model, View and Controller.

MVC 实现前后端分离--

(1) Model 无法将数据“发送”给 View，因为它根本不知道 View 的存在，数据应该是由 Controller 持有，并显示出 View。

 (2) 因此，用户也不是直接操作 Controller，即使是输入 URL，也可以认为那是由 View 触发的（就像在 View 上点击了一个链接）。
因此，MVC 的处理流程是 V -> C -> M -> C -> V

- 视图（View）：用户界面。
- 控制器（Controller）：业务逻辑
- 模型（Model）：数据保存

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/mvc-1.png)

#### 前端MVC

MVC web框架其实不是真正的MVC设计模式，而是JSP Model2设计模式。

前端主要是页面展示和用户交互，因此MVC中重点在视图V，有些场景下承担业务逻辑和交互的控制器C与视图V完全放在了一起，而模型M即与服务端交互的接口及浏览器本地的存储操作则作为单独的一部分。



#### 后端MVC

后端 mvc 中的 v 是前端 mvc 的全部。

我在米哈游也是只写Model 和 Controller...

服务端MVC的例子，页面上有一个按钮（View），点击之后会触发一个事件调用一个JavaScript函数（Controller），然后这个JavaScript函数进行业务处理（Model），处理之后修改DOM（View）。

M=数据对象+数据访问+业务逻辑，必要时可以分层

C=路由+视图逻辑

V=视图，如果是接口开发这层可以不要

Fat model, thin controller

之所以让model层负责更多的业务，主要原因是遍于重构，代码复用，在一个view层经常变更的场景下，controller相应的也会变，但因为业务层独立，可以保证做到最少的代码变更。

尤其是只提供接口时，MVC中就会更侧重模型M，视图V概念则弱化为了对外开放的接口具体到实现上甚至就是一系列的接口列表，而控制器C则承担接口的实现及部分服务端操作如定时任务等功能，成为较厚重的一层，模型M承担数据库操作和从其他服务获取数据的功能则作为第三层

#### 客户端MVC

客户端的MVC则一般都较为均衡，但是界面展示与业务逻辑很多时候还是会强绑定，造成V与C结合在一起，例如纯代码开发IOS时，每个页面的展示及交互逻辑分层很明显，但是就整个项目来说，视图V并没有形成单独的一层，是成为了控制器C层的一部分。

### MTV

MVP 模式实际上就是 MVC，只不过这里面的 C 主要负责的不再是业务逻辑，而是界面逻辑了，比如何时显示/隐藏某个选项卡，绑定 View 事件等。



MTV模式本质上和MVC是一样的，也是为了各组件间保持松耦合关系，MTV分别是值：

**M** 代表模型`Model`： 负责业务对象和数据库的关系映射。负责相关的所有事务： 如何存取、如何验证有效，是一个抽象层，用来构建和操作你的web应用中的数据，模型是你的数据的唯一的、权威的信息源。
 **T** 代表模板 `Template`：负责如何把页面展示给用户(html)。
 **V** 代表视图`View`： 负责业务逻辑，并在适当时候调用Model和Template。

除了以上三层之外，还需要一个URL分发器，将一个个URL的页面请求分发给不同的View处理，View再调用相应的Model和Template，MTV的响应模式如下所示：

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/mtv.jpg)

用户通过浏览器向服务器发起一个请求(request)，请求访问视图函数，如果不涉及到数据调用，视图函数返回一个模板也就是一个网页给用户），视图函数调用模型，模型去数据库查找数据，然后逐级返回，视图函数把返回的数据填充到模板中空格中，最后返回网页给用户。

### MVP

MVP 模式将 Controller 改名为 Presenter，同时改变了通信方向。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/mvp.png)

 Unlike view and controller, view and presenter are **completely decoupled** from each other’s and communicate to each other’s by an interface.

完全解耦--

**Key Points about MVP Pattern:**

- User interacts with the View.
- There is one-to-one relationship between View and Presenter means one View is mapped to only one Presenter.
- View has a reference to Presenter but View has not reference to Model.
- Provides two way communication between View and Presenter.





### MVVM

https://objccn.io/issue-13-1/

**Model**

The Model represents a set of classes that describes the business logic and data. It also defines business rules for data means how the data can be changed and manipulated.

**View**

The View represents the UI components like CSS, jQuery, html etc. It is only responsible for displaying the data that is received from the controller as the result. This also transforms the model(s) into UI.

**View Model**

The View Model is responsible for exposing methods, commands, and other properties that helps to maintain the state of the view, manipulate the model as the result of actions on the view, and trigger events in the view itself.

**Key Points about MVVM Pattern:**

- User interacts with the View.
- There is many-to-one relationship between View and ViewModel means many View can be mapped to one ViewModel.
- View has a reference to ViewModel but View Model has no information about the View.
- Supports two-way data binding between View and ViewModel.

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/mvvm.png)

唯一的区别是，它采用双向绑定（data-binding）：View的变动，自动反映在 ViewModel，反之亦然。

#### 数据双向绑定

https://ibclee.github.io/2017/10/23/%E5%89%8D%E7%AB%AF%E6%A1%86%E6%9E%B6%E4%B8%AD%E7%9A%84%E5%8F%8C%E5%90%91%E6%95%B0%E6%8D%AE%E7%BB%91%E5%AE%9A/

### 模式总结

#### 1

M,V之间是Observer模式,即V直接依赖M,M间接依赖V.M,C之间是C直接依赖M.这两点是MVC中最广泛认可,同时也是MVC成为一个解决方案模式的关键:视图和逻辑分离.

理想的MVC模式中V,C之间没有直接依赖(没有单向依赖),但现实中做不到.

Native应用要一般由view分发事件给controller,controller要决定那些view用户可见.

Web应用中情况好一点.用户可以直接通过url直接访问controller,不需要view知道controller,但是controller还负责路由view.前端复杂化后,页面上与controller交互更频繁,controller也很难只通过url来实现了.

事实上MVC有三个问题:
1.V和M之间不匹配,用户界面和流程要考虑易用性,用户体验优化同时考虑业务流程的精确和无错.
2.C和M之间界线不清,什么样的逻辑是界面逻辑,什么样的逻辑是业务逻辑,很难定义清楚.
3.V的变化不能完全由Model控制,即observer模式不足以支持复杂的用户交互.这其实要求VC之间要有依赖.

后来的MVP与MVVM都是为了优化V,C之间的关系而提出的.
MVP认为VC之间强绑定不可避免, 但可以加强P的能力,V变成只显示,P提供数据给V,把双向依赖简化为P直接依赖于V.
由于V的数据由P提供,则MV之间的Observer关系转移到MP之间.

MVP主要解决1,3两个问题,但代价是加重了问题2.P更重,而且与Model耦合无法框架化.但实践中view很难完全被动
化,它总是会随用户的事件变化,这部分成为P的负担.

而MVVM则是另一个方面来解决问题.MVVM认为view应该是事件驱动,模型变化只是一种事件.C应该只处理view与模型不配合的问题,而问题3可以通过分离用户交互与界面构造.C退化为一个View的模型VM(ViewModel),它与view之间由框架提供事件-数据的绑定.

MVVM与MVP相比主要彻底解决了问题1,重新定义了问题2,在MVVM中VM的功能比较明确,把M翻译成View需要的数据.
VM有一定视图的属性,view与VM(ViewModel)有对应关系,也就解决一部分问题3.但是由于各角色职责已经定义,需要引入第四个组件来解决这个问题.

我们可以打个分,直接依赖计为1,间接依赖计为0.5全部互相依赖是6
理想MVC,MV=1.5,MC=1共2.5
现实MVC根据VC之间的依赖情况可能是0.5到2,实际是3到4.5
MVP,MP=1.5,VP=1,MV=0共2.5与理想情况一致.
MVVM,MVM=1.5,VM+V=0.5或1,共2到2.5,即不高于理想MVC

#### 2

http://stonefishy.github.io/blog/2015/02/06/understading-mvc-mvp-and-mvvm-design-patterns/

正好这三个模式我都认识。我是自学 Rails 的，Rails 就是 MVC 的，刚开始学 Rails 时，把逻辑都放进了 Views 里，虽然 Rails 提供了 Helper，但是感觉不 OO，就没用。后来发现　Views 逻辑太多了，而想对网站外观进行修改时就很费力，放进 Model 中的话，在 Model 中生成 URL 等就很复杂了，需要包含各种 Rails Helper。后来发现了 Github 上的 Presenter，就是把不知道该放进 View 还是 Model 的东西放进去，View 中不再与 Model 交互，今天才知道这叫 MVP。MVVM 是我看的 Rails 作者的一篇文章中写得，英文不好，当时看个大概，没怎么明白，今天才明白了

### DOM事件

### 参考

1. https://www.jianshu.com/p/ebbd091339dd
2. https://www.ruanyifeng.com/blog/2015/02/mvcmvp_mvvm.html