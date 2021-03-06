#### 内存管理

iOS 使用引用计数来进行内存管理。当调用 reatin 的时候引用计数+1，当调用release 的时候引用计数-1，当引用计数为0的时候，对象销毁。

ARC，自动引用计数是编译器特性，会在编译时在适当的位置添加retain、release操作。

#### 内存管理原则

**总则：谁持有谁释放。**

所有权修饰符包括如下：

+ `__strong`：强引用，持有对象。__
+ `__weak`：弱引用，不持有对象，对象销毁，指向该对象的属性置为nil。_
+ `__unsafe_unretain`：不安全修饰符，不持有对象，对象销毁，指向对象的属性不会置为nil，容易出现悬垂指针，发生奔溃，但是效率比weak高。
+ `_autorelease` 添加到autoreleasepool 的对象都会进行延迟释放。

不是以alloc、new、copy、mutableCopy创建的对象，会自动将返回值的对象注册到 autoreleasepool。

#### 访问 __weak 修饰的变量，是否已经被注册在了 @autoreleasePool 中？为什么？

访问weak修饰符的变量必须访问注册到autorelease中，因为weak持有对象的弱引用，而在访问对象的过程中，该对象可能被废弃，如果注册到autorelease中，那么在autorelease释放之前都能保证对象的存在。

id指针或对象的指针在没有显示指定时会被附加上_autorelease修饰符。

#### ARC 内存管理规则

+ 不能使用retain、release、retainCount
+ 不能使用NSDeallocateObject
+ 不用显式调用dealloc
+ 使用@autorelease代替NSAutoreleasepool
+ 不能使用zone
+ 显示转换 id 和 void*

#### 悬垂指针和野指针区别

悬垂指针指的是，指向的内存已经释放了，但是指针还在。
野指针：未初始化的指针称为野指针。

#### BAD_ACCESS 声明情况下回出现？如何调试和？

访问了已经被销毁的内存空间，就会报出这个错误。 根本原因是有 悬垂指针 没有被释放。

1、比较老的方法是打开Xcode 的 enabled zoombie。
2、设置全局断点看能否快速定位问题。
3、Address sanitizer。
4、或者使用 Instruments Zombies.

+ https://devma.cn/blog/2016/11/10/ios-beng-kui-crash-jie-xi/

#### 被weak修饰的对象在被释放的时候会发生什么？是如何实现的？知道sideTable么？里面的结构可以画出来么？（内存管理方便，考底层weak实现）

对象在释放的时候会调用 dealloc 方法，内部会调用 objc_dispose，最后调用 objc_destoryInstance，然后再 objc_destoryInstance 内部会调用 weak_entry_remove 函数，通过在weak表中找到对象指针找到对应的entry，然后遍历entry中所有指向对象指针的weak变量指针，同时置为nil，最后把entry从weak表中移除。

#### 关联对象有什么应用，系统如何管理关联对象？其被释放的时候需要手动将所有的关联对象的指针置空么？（底层runtime方面，考底层associated）

1、为分类手动实现成员变量的setter和getter方法，用于关联对象。

2、底层使用 AssociatedManager 管理关联对象，内部其实有一个哈希表来 AssociationMap 存储所有关联对象的。相当于把所有关联对象存在一个哈希map中，map的key是这个对象的指针地址，而map的value是另一个哈希map（objectAssociationmap），里面保存关联对象的KV对。objectAssociationmap 内部是 value 和 policy，根据指定的 policy 进行内存管理。

3、释放的时候不需要手动设置指针为nil，因为 runtime 在对象销毁函数destructInstance中会判断这个对象有没有关联对象，如果有会自动做关联对象的清除。