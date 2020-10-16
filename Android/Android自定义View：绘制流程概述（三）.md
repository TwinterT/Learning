**`View`的绘制流程从顶级`View（DecorView）`的`ViewGroup`开始，一层一层从`ViewGroup`至子`View`遍历测绘**

![travel](img/travel_view_tree.jpeg?raw=true)

* 绘制的流程 = `measure`过程、`layout`过程、`draw`过程，具体如下

![详情](img/travel_detail.jpeg?raw=true)

![解释](img/travel_explain.jpeg?raw=true)

