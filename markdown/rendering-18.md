渲染之十八 实时全局光、探针体和LOD组
======

 * 支持实时全局光照
 * 计算自发光对全局光照的贡献
 * 使用光照探针代理体
 * 将 LOD 组与全局光照结合
 * 交叉淡入淡出 LOD 组

这篇是关于渲染的系列教程的第18部分。在第17部分完成烘焙全局照明后，我们继续支持实时全局光照，之后，我们还将支持光照探针代理体和交叉淡入淡出LOD组。

从现在开始，本系列教程使用 Unity 2017.1.0f3 制作，不适用于旧版本，因为我们最终将使用新的着色器功能。

![](https://catlikecoding.com/unity/tutorials/rendering/part-18/tutorial-image.jpg)  
*静态 LOD 组和实时全局光的结合*

# 1 实时全局光

烘焙光照对于静态几何体非常有效，并且借助光照探针也非常适合动态几何体。但是，它无法处理动态光源。混合模式下的光源可以进行一些实时调整，但是调整过多会使烘焙的间接光严重不协调。因此，当你有户外场景时，太阳必须保持不变，它不能像现实生活中那样东升西落，因为这就需要逐渐改变全局光照。所以场景必须保持在某个时间点不变。

为了使间接光照与移动的太阳一起工作，Unity使用Enlighten系统来计算实时全局光照。除了在运行时计算光照贴图和探针外，它的工作方式与烘焙间接光一样。

弄清楚间接光需要知道光如何在静态表面之间反弹。问题是哪些表面可能受到哪些表面的影响，以及其程度。弄清楚这些关系工作量太大，不能实时完成，因此，这些数据由编辑器处理并存储以便在运行时使用。 Enlighten 使用这些数据来计算实时光照贴图和探针。即使这样，它也只适用于低分辨率光照贴图。
