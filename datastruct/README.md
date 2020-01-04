#### 1、链表和数组的区别是什么？插入和查询的时间复杂度分别是多少？

+ 数组是一块连续的内存。随机访问性强，查找速度快。插入和删除效率低，可能浪费内存，内存空间要求高，必须有足够的连续内存空间。插入和删除是On，查询是O1
+ 链表插入删除速度快，内存利用率高，不浪费内存，大小没有固定，容易扩展。插入和删除是O1，查询是On

#### 2、哈希表是如何实现的？如何解决地址冲突？
#### 3、排序题：冒泡排序，选择排序，插入排序，快速排序（二路，三路）能写出那些？
https://www.cnblogs.com/onepixel/p/7674659.html

#### 4、链表题：如何检测链表中是否有环？如何删除链表中等于某个值的所有节点？
#### 5、数组题：如何在有序数组中找出和等于给定值的两个元素？如何合并两个有序的数组之后保持有序？
#### 6、二叉树题：如何反转二叉树？如何验证两个二叉树是完全相等的？

知识详解：https://blog.ibireme.com/2015/05/18/runloop/#base

+ runloop和thread是一一对应的，对应的关系保存在一个全局的dictionary。
+ apple不允许直接创建runloop，它只提供了两个自动获取的函数。
+ 线程刚创建的时候并没有runloop，如果你不获取就不会存在。
+ runloop的创建时发生在第一次获取时，销毁是发生在线程结束时。

+ 每个runloop包含若干个mode，每个mode包含若干个source、timer、observer。每次调用都只能指定其中一个mode。若要切换mode，只能退出当前loop，重新指定一个mode再进入。
+ source、timer、observer都是mode item，一个item可以被同时加入多个mode，如果一个mode中一个item都没有，则runloop会直接退出，不进入循环。
+ 主线程里面有预置的两个mode：default 和 UITrackingRunLoopMode。当你创建一个timer并添加到defaultMode时，Timer会得到重复回调，但是滑动TableView的时候，runloop mode切换为TrackingRunloopMode，这时候timer就不会被调用，也不会影响到滑动操作。

+ CFRunloopSourceRef是产生事件的地方，source有两个版本source0和source1
    + source0只包含一个回调，它并不能主动触发事件，使用时，你需要先标记为待处理，然后手动调用wakeup唤醒runloop。
    + source1包含一个mach_port和一个回调，被用于通过内核和其他线程相互发送消息，这种source能主动唤醒runloop的线程。
+ CFRunloopTimerRef是基于事件的触发器，它包含了一个时间长度和一个回调，当其加入到runloop时，runloop会注册对应的时间点，当时间点到达时候，runloop会被唤醒执行回调。
+ CFRunloopObserverRef是观察者，每个observer都包含了一个回调，让runloop发生状态变化时，观察者就能通过回调接受到这个变化。
    + entry，即将进入loop
    + beforeTimers，即将处理timers
    + beforeSources，即将处理sources
    + beforeWaiting，即将进入休眠
    + afterwaiting，刚从休眠中唤醒
    + exit，即将推出loop

#### 20、如何设计一个网络请求库？
#### 21、说一下多线程，你平常是怎么用的？
#### 22、说一下UITableViewCell的卡顿你是怎么优化的？
#### 23、看过哪些三方库？说一下实现原理以及好在哪里？


#### 25、设计一套缓存策略。

NSUserDefault
NSDictionary key=>value
NSCache

#### 26、设计一个检测主线和卡顿的方案。

主线程对应的是主runloop，卡顿就是无响应，所以根据对主runloop的状态监控判断某段事件内的方法调用超过一个阈值。


#### 3、load和initialize方法分别在什么时候调用的？

从点击 App 图标开始到首页渲染显示出来的整个过程。总共分为三个阶段：

1、main 函数执行前。
2、main 函数执行后。
3、首屏渲染后。

main 函数执行前：

```
+ 加载可执行文件。
+ 加载动态链接库，进行rebase指针调整、bind符号绑定。
+ OC 运行时的初始化，包括OC类注册、category注册、selector唯一性检查。
+ 初始化，包括了执行 +load() 方法、attribute((constructor)) 修饰的函数的调用、创建 C++ 静态全局变量。
+ 当类和类别加入到runtime的时候，load方法就会被调用。
+ 调用所有framework中的初始化方法。
+ 调用所有load方法。
+ 调用c++静态初始化以及C++中的构造函数
+ 调用所有链接到目标文件的framework中的初始化方法。


``
针对 main 函数执行前，启动优化的几个点：
```

+ 减少动态库加载。
+ 减少加载启动后不会去使用的类或方法。
+ load 方法的内容可以放到首屏渲染之后在执行，可以使用initialize 方法替换掉。

```
main 函数执行后

首页业务代码要在这个阶段，也是首屏渲染前执行的
```
+ 首屏初始化所需配置文件的读写操作。
+ 首屏列表大数据的读取。
+ 首屏渲染的大量计算。
```

load 加载调用顺序

父类load -> 子类load -> 分类load

+ 父类的 load 调用在子类的 load 调用之前。
+ 分类的 load 在所有父类和子类 load 调用完成之后再调用。
+ 分类的 load 调用不分子类和父类，根据 compile sources 中的顺序依次执行。
+ 没有实现 load 方法就不调用。
+ load 中禁止调用super load，由系统加载，不需要手动调用。

initialize

在这个类接收第一条消息之前调用。

+ 父 initialize 先于 子 initialize。
+ 子没有实现 initialize，父的 initialize 会被调用。总共会被调用两次。
+ 父类没有实现，只会调用子类的 initialize 方法。
+ 任何父类或者子类的 category 都会覆盖父类或子类的 initialize 实现。

