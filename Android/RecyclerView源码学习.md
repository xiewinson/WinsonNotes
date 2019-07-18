# RecyclerView 源码学习

`RecyclerView` 的使用方法本文就不赘述了，先明确一下其涉及的 6 个类的职责：

* `ViewHolder`：`RecyclerView` 用来缓存的基本单位，每项 item 的 `View` 的承载
*  `Adapter` ：创建、绑定 ViewHolder
* `LayoutManager`：管理 item 的布局
* `ItemAnimator`：处理 item 的动画效果
* `ItemDecoration` ：装饰 item，例如添加分割线和进行额外的绘制
* `Recycler`：负责管理回收与复用



