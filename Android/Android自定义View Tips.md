# Android 自定义 View Tips

1. `addStatesFromChildrenJava` 方法（xml 中也有该属性）可以实现子 View 的 drawable state 发生改变后，ViewGroup 的 drawable state 跟随改变
2. 抗锯齿并不适合任何场景，抗锯齿的原理是修改图形边缘像素颜色从而看起来更加平滑，但是在某些时候会造成看起来很模糊或者变粗，例如画 1 像素的东西的东西的时候，没有曲线、圆角的情况下不开抗锯齿效果也不错
3. `onMeasure` 可能调用多次，要取得准确的控件宽高，使用  `onSizeChanged` 获取
4. 对画布的操作，例如 `translate`、`rotate` 等操作是叠加的，例如连续两次调用在 X 轴正方向平移 50px 后中心点就是 X 轴正方向 100px 处了。组合利用各种画布操作会让坐标计算更简单方便
5. `canvas.skew(float sx, float sy)` ，是在 X 轴或 Y 轴上倾斜一定角度，参数 sx 和 sy 均为倾斜角度的 tan 值。公式为：`newX = x + sx * y` 和 `newY = y + sy * x`
6. `canvas.save` 和 `canvas.restore` 成对调用可以保存和恢复对画布的变换操作
7. `canvas.saveLayer` 是一个好方法，新建一个图层进行离屏渲染，但是这个方法代价太高，官方文档说他会花费超过两倍的时间去渲染内容，所以能避免使用就避免使用
8. `Picture` 是一个用来录制绘图操作的工具，当使用 ` canvas.drawPicture(Picture picture, Rect dst)` 时，picture 的绘制内容的宽高超过 dst 时，会进行缩放处理
9. `path.addArc` 方法会将最后个点移动到圆弧起点，而 `arcTo (RectF oval, float startAngle, float sweepAngle, boolean forceMoveTo)` 最后一个 `forceMoveTo` 参数为 `false` 的情况下就不会移动点，而是连接最后一个点和圆弧的起始点
10. `path.offset (float dx, float dy, Path dst)` 方法类似于画布的偏移，最后一个参数 `dst` 为空时就直接作用于 path，否则将结果输出到 `dst`
11. path 的添加点的操作方法中，`rXxx` 的方法是相对上一个点位置的，很多时候用起来更方便
12. path 中的 `op` 方法用来计算两个 path 之间的并集交集等，能够更方便地画出一些奇形怪状的图形，但是这个方法在 API19 及以上才能使用
13. `PathMeasure` 可以获得 Path 的长度，某一点的正切值，片段 path 等，在用来做动画时非常有用
14. 使用 `Region` 来判断一个坐标是否位于不规则图形中
15. 处理复杂的触摸操作时推荐 `GestureDetector`
16. `canvas.drawText` 时候指定的 y 其实是基线位置，如果要指定正确的位置，公式为 `baseLineY ＝ centerY － 1/2(ascent + descent)`
17. xfermode 在计算时是计算 src 与 dst 共同的叠加效果。API Demo 中有点误导人，下面以 DST_IN 模式为例。如果直接 drawCircle、drawRect 会发现最后的结果和 官方 Demo 不同，原因是 Demo 中其实是绘制到两个 Bitmap 再对两个 Bitmap 进行叠加的，并且两个 Bitmap 的位置和大小是一样的；而当我们自己 drawCircle 和 drawRect 的时候，参与叠加计算的面积并不是一样的，而作为 dst 的图片没有与 src 叠加的部分，就会直接地绘制出来，所以效果会与 API Demo 中有些不同。所以一定注意 dst 和 src 都一定要有最够大的区域，否则参与计算的面积不够会造成达不到预期的效果