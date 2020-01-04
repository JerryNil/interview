#### 为什么一定要在主线程里面更新UI？

#### 你们应用的崩溃率是多少？
#### 哈希表 Hash Table

NSDictionary底层实现其实是一个哈希表。哈希表的本质是一个数组，数组的每一个元素是一个箱子，箱子中存放的是键值对。

使用数组存储元素，一个索引对应一个数据，元素很多也叫桶 buckets。

```
哈希表存储过程：

+ 传入 key， 根据哈希函数算出它的哈希值 index。index 必须是整数。
+ 假设箱子的个数为 n，那么这个键值对应该放在第 index % n 的箱子中。
+ 哈希函数计算出的 index 相同就会产生哈希冲突，就是用开放寻址法或拉链发解决冲突。
+ 开放寻址：按照一定规则向其他地址探测，直到遇到空桶。
+ 再哈希法、拉链法通过链表将所有相同 index 元素串起来。有可能链表转成红黑树。
+ 如果使用双向链表容易浪费内存，多了一个指针。

使用拉链法解决哈希冲突时，每个箱子其实是一个链表，属于同一个箱子的键值对都会排在链表中。

还有一个负载因子，表示哈希表的空满程度。
```

> https://juejin.im/post/5b56155e6fb9a04f8b78619b#heading-1

#### NSDictionary的实现原理是什么？

HashTable 哈希表实现的。

#### 什么是指针常量和常量指针？

+ 指针常量本质是一个常量，表示是一个指针常量，指针说明常量类型。
+ 常量指针本质是一个指针，表示指针指向的内容是一个常量。指向的内容不允许修改。

#### 不借用第三个变量，如何交换两个变量的值？要求手动写出交换过程。

```
void swap(int &a, int& b)
{
	a = a ^ b;
	b = a ^ b;
	a = a ^ b;
}
```

#### iOS 事件传递和响应
[](http://smnh.me/hit-testing-in-ios/)

![](http://d33wubrfki0l68.cloudfront.net/2321d34cb19b0a4dbefa426561c761e10672fcc7/237b8/images/hit-test-flowchart.png)

[](https://stackoverflow.com/questions/4961386/event-handling-for-ios-how-hittestwithevent-and-pointinsidewithevent-are-r/4961484#4961484)

iOS 事件流有两条线，事件传递（分发）和事件响应

1、事件传递

当用户触摸了设备的屏幕，此时系统会产生一个 UIEvent 被添加到 UIApplication 管理的事件队列，等到下一个 runloop 到来之时添加到 runloop 中。然后从 UIApplication 的事件队列取出事件交由 UIWindow 主窗口开始处理，接下来就是 hitTest 的主要流程。

hitTest：

1、UIWindow 调用 pointInSide:withEvent: 来判断触摸点是否在当前视图上。
2、如果返回NO，则 hitTest 返回nil。如果返回YES，则向当前视图的子视图发送 hitTest 事件。
3、子视图从上至下遍历视图，依次判断触摸点是否在当前视图，如果是继续遍历。
4、如果最底层的子视图有返回非空对象，则hitTest就返回该对象。
5、如果返回的都是nil，则 hitTest 返回自身。

hitTest 找能响应事件的视图时，有三个限制条件防止hitTest继续往下找。

1、hidden=YES
2、userInteractionEnabled=NO
3、alpha<0.01

2、事件响应

事件产生之后由点击的视图首先判断能否处理，如果处理不了就会往视图的父视图进行传递，如果父视图还不能响应继续传递给根视图，如果还不响应传递给控制器，然后传递给主窗口，最后传递给UIApplication，如果还不能处理就直接废弃。

```
First Responser --> The Window --> The Application --> nil（丢弃）
```

#### NSCache 和 NSDictionary 的区别

1、NSCache 会在内存紧张的时候自动清除缓存。
2、NSCache 是线程安全类，可以在多线程情况下不需要使用锁就可以添加和删除操作。
3、NSDictionary 的 key 是遵循 NSCopying 协议的。

#### synthesize 和 dynamic 分别有什么作用

property 有两个对应的词，一个是 synthesize，一个是 dynamic。

1、synthesize 是自动合成，编译器会自动为我们合成属性、存取方法。
2、dynamic 是告诉编译器，属性方法由自己实现，不需要自动合成。
3、dynamic 在使用的时候，需要注意的是没有实现属性方法，程序去调用的时候会crash。

#### 默认关键字

1、常量类型： atomic、readwrite、assign
2、对象类型：atomic、readwrite、strong

#### 使用copy关键字声明属性，为什么？

1、保护类的封装性，类的属性不受外部干扰，copy 之后生成的是一个不可变副本，也就是个不可变对象。
2、如果使用 strong，类的封装性被破坏，在不确定因素下存在被外部修改的可能。

#### 深拷贝、浅拷贝

1、深拷贝是内容拷贝
2、浅拷贝是指针拷贝

1、对于非集合类，对不可变对象copy是浅拷贝，其余都会深拷贝
2、对于集合类，不可变对象copy是浅拷贝，其余都是单层深拷贝

#### @synthesize合成规则是如何的？

+ 实例变量=成员变量=ivar
+ 使用属性的话，编译器是自动编写setter和getter方法，此过程叫自动合成，编译器还会自动向类中添加适当类型的实例变量。
+ 四条规则如下：
    + 如果指定了成员变量的名称，会生成一个指定名称的成员变量。
    + 如果这个成员变量已经存在，就不再生成。
    + 如果是 @synthesize foo; 还会生成一个名称为foo的成员变量。
    + 如果是 @synthesize foo = _foo; 就不会生成成员变量了.
    
#### 在有了自动合成属性实例变量之后，@synthesize还有哪些使用场景？

+ 什么情况下不会自动合成。
    + 1、同时重写了setter和getter
    + 2、重写了只读属性的getter
    + 使用了@dynamic
    + 在@protocol中使用的属性
    + 在category中定义的所有属性
    + 重载的属性
    
    如果子类重载了父类的属性，必须使用@synthesize进行手动合成ivar。

#### 在 protocol 和 category 中如何使用property
    
    只会生成 setter 和 getter 方法声明。
    
    1、protocol 声明的属性，遵循该协议的类属性重写或者通过 @syncthesize 来自动合成。
    2、category 声明的属性，需要使用associate object来关联对象。
    
#### 常见的Exception

1、SIGSEGV 重复释放对象
2、SIGABRT 收到abort信号，插入nil到数组中会遇到此类错误
3、SEGV 代表无效指针，比如空指针
4、SIGBUS 总线错误，地址对齐问题
5、SIGILL 尝试非法指令

#### Autolayout

Auto Layout 拥有一套 Layout Engine 引擎，由它来主导页面的布局。App启动后，主线程的Run Loop会一直处于监听状态，当约束发生变化后会触发Deffered Layout Pass（延迟布局传递），在里面做容错处理（约束丢失等情况）并把view标识为dirty状态，然后Run Loop再次进入监听阶段。当下一次刷新屏幕动作来临（或者是调用layoutIfNeeded）时，Layout Engine 会从上到下调用 layoutSubviews() ，通过 Cassowary算法计算各个子视图的位置，算出来后将子视图的frame从Layout Engine拷贝出来，接下来的过程就跟手写frame是一样的了。
