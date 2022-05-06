## [LGTM]Chrome性能调试

原文：https://cloud.tencent.com/developer/article/1624660

http://blog.itpub.net/24475491/viewspace-2795705/

### Demo

#### step 1: 隐身模式打开chrome

目的是避免缓存以及不必要的问题

#### step 2: 打开测试地址

谷歌性能测试地址 https://googlechrome.github.io/devtools-samples/jank/

由于有些用户的设备 cpu 性能很高，无法很好的分析移动端，或者发现低级设备的性能问题，所以我们要降速 找到控制台中的 performance 项，找到 CPU 选项，选择降低 4 倍性能或 6 倍性能

### 了解 performance 各模块

#### step 1：了解 Fps

fps：是指页面每秒帧数

 fps = 60 性能极佳 

fps < 24 会让用户感觉到卡顿，因为人眼的识别主要是 24 帧

#### setp 2： 低玩定位耗时代码

查看帧信息，图中区域可以“上下左右”的拖动

* 上下拖动：可以看到Frame、Main、Raster等
* 左右拖动：可以调整录制的平面帧时间

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/chrome-performance-frame.png)

查看一个frame的概述

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/chrome-perfermance-frame-summary.png)

开启页面的frame stats 监控

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/chrome-perfermance-frame-stats.png)

查看一个帧的代码调用情况

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/chrome-performance-call-tree.png)

end

#### Step 3： 高玩

http://blog.itpub.net/24475491/viewspace-2795705/

