## 分类

#### 分类和扩展有什么区别？可以分别用来做什么？分类有哪些局限性？分类的结构体里面有哪些成员？(考底层category实现方面)

https://techbird.me/2016/05/16/ios-category-use-merit-and-demerit/
https://tech.meituan.com/2015/03/03/diveintocategory.html

分类的作用：分类就是对类进行功能扩展。为现有的类添加新方法。

优缺点（局限性）：

+ 把类的实现拆分到多个category文件中。
  + 减少单个文件体积。
  + 可以把不同功能组织到不同的category
  + 按需加载需要的category
+ 模拟多继承
+ 把framework的私有方法公开。

category 缺点：

+ 只能给某个已有的类添加方法，不能添加成员变量。
+ 分类中可以添加属性，但是不能合成存取方法和成员变量。
+ 如果分类中的方法和类中的方法同名，分类中的方法会覆盖掉原来类中的方法，所以避免这种情况一般分类中的方法都需要添加前缀。
+ 如果多个分类存在同名方法，运行时调用哪个方法需要编译器确定。

category和extension的区别

+ extension 是一个匿名的 category，它属于类的一部分属于编译期确定，可以添加实例变量。
+ category 是在运行期决定的，无法添加实例方法，因为在运行期的时候对象的布局已经确定，如果添加实例变量会破坏类的内存布局。

3、category源码分析

```
typedef struct category_t {
    const char *name;  //类的名字
    classref_t cls;  //类
    struct method_list_t *instanceMethods;  //category中所有给类添加的实例方法的列表
    struct method_list_t *classMethods;  //category中所有添加的类方法的列表
    struct protocol_list_t *protocols;  //category实现的所有协议的列表
    struct property_list_t *instanceProperties;  //category中添加的所有属性
} category_t;
```

+ 所有的OC对象和类，在runtime底层都是结构体表示，category就是用category_t表示。
+ 类的名字、类、给类添加的实例方法列表、所有类方法列表、所有协议列表、添加的所有属性。但是无法添加实例变量。
+ 运行时，objc_class的大小是固定的，不能向这个结构体中添加数据，只能修改。ivars指向的是一个固定区域，只能修改成员变量的值，不能增加成员变量的个数。
+ methodLists是一个二维数组，可以修改*methodLists的值来添加成员变量方法。

4、 category如何加载

+ category 的加载是依赖 OC 的 runtime 的。
+ 把 category 的实例方法、协议以及属性添加到类中。
+ 把 category 的类方法和协议添加到类的 metaclass 中。

category 的方法没有完全替换掉原来类已经有的方法，类的方法列表中其实存在多个添加的方法。
category 的方法被放到前面，而原来类的方法被放到了后面，因为运行时在查找方法的时候是顺着方法列表查找的，一找到名字就罢休了。

5、category和关联对象

关联对象是存在什么地方呢？如何存储？对象销毁时如何处理关联对象？

关联对象是由 AssociationsManager 管理，它的内部是由一个静态 AssociationsHashMap 来存储所有关联对象的，相当于存在一个全局map中。map的key是这个对象的指针地址，而value是另外一个associationsHashMap,里面保存了关联对象的kv对。

嗯，runtime的销毁对象函数objc_destructInstance里面会判断这个对象有没有关联对象，如果有，会调用_object_remove_assocations做关联对象的清理工作。