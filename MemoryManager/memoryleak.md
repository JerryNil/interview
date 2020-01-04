# 内存泄漏

#### 内存泄漏形式

+ Leaked memory：未引用的内存不能使用或释放。

+ Abandoned memory：废弃掉的内存，但是App任然引用。
+ Cached memory：App任然使用的内存，可以再次获取已获得更好的性能。

#### OC 中出现泄漏的点

+ NSTimer
+ ViewController，delegate
+ Block
+ 对象之间的互相引用
+ NSURLSession，delegate 是 strong 类型。
+ 如果使用到 CF 内部 create 和 copy 的方法时，注意需要及时释放

#### 如何检测泄漏？

+ 静态检测，Analyze。
+ Instrument，Leaks，Allocations
+ 第三方工具 MLeaksFinder

#### 参考

+ https://www.jianshu.com/p/e9d989c12ff8
+ https://wereadteam.github.io/2016/02/22/MLeaksFinder/