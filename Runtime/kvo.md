## KVO



#### KVO的底层实现？如何取消系统默认的KVO并手动触发（给KVO的触发设定条件：改变的值符合某个条件时再触发KVO）？

https://tech.glowing.com/cn/implement-kvo/

+ 原理 isa-swizzling

当被观察对象的属性值发生变化时，观察者对象就会收到通知。

KVO是基于runtime的isa-swizzling技术实现。为一个对象的属性设置观察者时，被观察的对象的isa指针会被修改，指向一个中间类，这个中间类是原来类的子类，重写了父类的setter方法，在重写的setter方法前后通知观察者对象值的改变，最后把这个isa指针指向新创建的类，对象也变成了新创建类的子类。

+ KVOController的源码分析
  https://github.com/draveness/analyze/blob/master/contents/KVOController/KVOController.md
  https://satanwoo.github.io/2016/02/27/FBKVOController/

#### 使用的坑

+ 必须在合适的时间，需要手动移除观察者，不然可能会引起crash。
+ 注册观察者的代码和事件发生处的代码上下文不同，会分布在不同的方法中。
+ 需要复写很长的方法，比较麻烦。
+ 如果有多个观察属性，就需要在处理方法中进行if else的判断，可读性不高。