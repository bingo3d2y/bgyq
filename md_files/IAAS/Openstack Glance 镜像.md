## Openstack Glance 镜像

需求：

默认在uis上创建的虚机，开启castools工具提供的时间同步功能。

我的猜想：

云平台通过调用uis接口创建虚拟机时，通过添加某个参数是的vm开启时间同步的功能。

这个猜想的原因是：我在uis虚拟机管理页面点击修改可以选择打开或者关闭时间同步功能

事实：

云平台调用createvm接口很简单就是使用哪个镜像和使用什么规格就没。要想默认vm实例开启castool的时间同步，要具备两个条件：

* 镜像中安装了castool工具
* 镜像中默认设置了开启时间同步

这个时间同步对应的元数据是`sync_time`，直接更新镜像就行。

uis界面修改时间同步功能算是修改基于镜像启动的instance了，是另一个接口了。

对镜像设置sync_time=true元数据看下

```bash
glance image-update   镜像id     --property sync_time=true 
```

