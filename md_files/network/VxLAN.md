## VxLAN

### Why VxLAN

VLAN地址可以重叠吗？？？



我还是不理解，vxlan的场景啊，它底层还是vlan，vlan不就4096个？？？

但是，不同机房vlan可以重复？

vxlan那么多的vni给谁用？

vxlan vni最终映射到vlan id，实现底层物理传输。

好处就是，vm直接可以画更多的vxlan，这样？？？

从VXLAN的封装看到的优势：

1. 比VLAN有着更多的二层段。从VXLAN头部VNID的24bit可以算出，VXLAN支持的数量是2^24=16777216个(1600W多个)。
2. 比VLAN有着更广的扩展性。VXLAN的MAC over UDP封装模型是可以穿越三层网络，比VLAN有更好的扩展性。
3. 提供L2隧道技术。从封装模型看，VXLAN是MAC over UDP模型结构。

### VxLAN
#### VxLAN网络架构组件
##### 实体组件
##### 逻辑组件



### 引用
1. https://www.zhihu.com/question/51675361/answer/263300150
2. http://www.3542xyz.com/?p=1331
3.  https://www.cnblogs.com/mlgjb/p/8087612.html

