# 性能优化

#### 说一下UITableViewCell的卡顿你是怎么优化的？

CPU

+ 重用 cell。

+ 缓存 cell 高度。
+ 异步计算每个 cell 的高度，缓存高度。
+ 减少视图的添加和移除，使用 hidden 操作。
+ 使用纯代码布局。
+ 异步加载图片，图片在后台线程进行解码，resize 图片大小。
+ 尽量把耗时任务放到子线程中处理。

GPU

+ 减少图层数。opaque = YES。
+ 减少离屏渲染。
+ 采用特殊图层进行渲染。
+ 图片过多可使用 Core Graphics 进行图片合成。

#### 渲染流程

![](https://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/sketch-images/ios-core-animation-pipeline-steps.png)



+ Layout
+ Display
+ Prepare
+ Commit

#### 离屏渲染产生原因

+ cornerRadius + maskToBounds
+ shadow
+ mask
+ shouldRasterize

#### 首页启动优化

从点击 App 图标开始到首页渲染显示出来的整个过程。总共分为三个阶段：

+ main 函数执行前。

+ main 函数执行后。

+ 首屏渲染后。

  

main 函数执行前（dyld -> rebase -> bind -> objc setup -> initialize）:

+ 加载可执行文件。

+ 加载动态链接库，进行rebase指针调整、bind符号绑定。

+ OC 运行时的初始化，包括OC类注册、category注册、selector唯一性检查。

+ 初始化，包括了执行 +load() 方法、attribute((constructor)) 修饰的函数的调用、创建 C++ 静态全局变量。

+ 当类和类别加入到runtime的时候，load方法就会被调用。

+ 调用所有framework中的初始化方法。

+ 调用所有load方法。

+ 调用c++静态初始化以及C++中的构造函数

+ 调用所有链接到目标文件的framework中的初始化方法。

  

针对 main 函数执行前，启动优化的几个点：

+ 减少动态库加载。

+ 减少加载启动后不会去使用的类或方法。

+ load 方法的内容可以放到首屏渲染之后在执行，可以使用initialize 方法替换掉。

  

Main 函数执行之后，首页渲染之前：

+ 初始化项调整。



首页渲染：

+ 首屏初始化所需配置文件的读写操作。
+ 首屏列表大数据的读取。
+ 首屏渲染的大量计算。