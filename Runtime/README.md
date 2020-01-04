## Runtime

#### 参考

+ https://halfrost.com/objc_runtime_isa_class/
+ https://halfrost.com/objc_runtime_objc_msgsend/
+ https://halfrost.com/how_to_use_runtime/

#### 向一个nil对象发送消息会发生什么？

运动时不会有任何影响，是有效的。

如果一个方法返回值是一个对象，发送给nil的消息将返回0。如果向一个nil对象发送消息，首先寻找对象的isa指针就是0地址返回了。所以不会出现任何错误。

#### self 和 super 的区别

+ self 是类的隐藏参数，指向当前调用方法的类的实例。
+  super 是编译器标识符，和self指向的同一个消息接受者。
+ super 会告诉编译器去父类找相应的方法，而不是在本类中。

#### objc_msgSend 和 [objc foo] 关系？

编译之后就变成 objc_msgSend，第一参数是消息的接受者，第二个参数消息的选择子。

#### 说一下OC的反射机制？

OC 是一门动态语言，可以在运行时阶段为已有类添加方法列表、可以动态创建类并且添加成员变量等等。还可以在程序执行阶段，通过多种反射方法，以字符串的形式获取对应的类和选择子，还可以判读当前类是否是某个类的子类或类等等。

常用的例子：通过 runtime 给模型自动设置属性值。通过 runtime 给对象自动实现归档、拷贝功能。

#### Obj-C 对象、类的本质是通过什么数据结构实现的？

都是结构体。

+ 实例对象 objc_object 内部只有一个 isa 指针
+ 类对象 objc_class 内部有isa、cache、superClass、bits等，元类跟类对象差不多。

objc_object 内部是一个 isa_t 类型指针。

```
struct objc_object {
private:
    isa_t isa;
}
```

objc_object 继承自 objc_class

```
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags

    class_rw_t *data() { 
        return bits.data();
}
```

#### 一个 NSObject 对象占用多少内存空间？

16字节。一个 isa 占8位，不满16补足16位。

#### `class_ro_t` 和 `class_rw_t` 的区别？

+ `ObjC` 类中的属性、方法还有遵循的协议等信息都保存在 `class_rw_t` 中
+ `rw` 中有一个指向常量的指针 `ro`。
+ `ro` 存储了当前类在编译期就已经确定的属性、方法以及遵循的协议。

通过底层 `realizeClass` 方法将 `class_ro_t` 常量赋值给 `class_rw_t` 中的 `ro` 常量。然后通过 `methodizeClass` 将 ro 的方法列表、属性列表、协议列表添加到 `class_rw_t` 中对应的字段中，并且最后还将分类中的方法、属性、协议添加到对应的字段中。

```
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name;
    method_list_t * baseMethodList; // 方法列表
    protocol_list_t * baseProtocols; // 协议列表
    const ivar_list_t * ivars; // 成员变量列表

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties; // 属性列表
```

```
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro; // 指向常量 ro

    method_array_t methods; // 方法数组
    property_array_t properties; // 属性数组
    protocol_array_t protocols; // 协议数组

    Class firstSubclass;
    Class nextSiblingClass;
}
```

#### 能否向编译后的类中添加实例变量？能否向运行时创建的类中添加实例变量？为什么？

`class_ro_t` 中有如下定义：

```objc-c
const uint8_t * ivarLayout;
const char * name;
method_list_t * baseMethodList;
protocol_list_t * baseProtocols;
const ivar_list_t * ivars;
```

`ivars` 声明的是常量，也就是说在编译期的时候，类的 `ivars` 和 `ivarLayout` 就已经确定完成，不可再添加实例变量。

运行时可以通过 `class_allocationClassPair` 和 `class_registerClassPair` 创建类，并且通过 `class_addIvar` 添加实例变量。

#### category 如何加载？

```
struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods; // 实例方法
    struct method_list_t *classMethods; // 类方法
    struct protocol_list_t *protocols; // 协议列表
    struct property_list_t *instanceProperties; // 属性列表
    // Fields below this point are not always present on disk.
    struct property_list_t *_classProperties;

    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }

    property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
};
```

category 是在运行时，通过 realizedClass 和 methodizeClass 将 category 中的方法、属性、协议和所属类合并的。

category 是在编译之后，runtime 进行objc setup的阶段，调用_objc_init，然后通过 `read_images` 读取所有类的相关信息，然后调用 `realizeClass` 和 `methodizeClass` 进行和原有类的方法列表、协议列表进行合并，最后添加到 `class_rw_t` 结构体中。

#### category为什么不能添加实例变量？

因为在编译阶段就已经确定了类的内存布局结构，并且category中并没有ivar_list的声明。

#### category 编译过后，在什么时机跟原有类结合在一起

+ 程序启动后，通过编译之后，Runtime 会进行初始化，调用 _objc_init。
+ 然后会 map_images。
+ 接下来调用 map_images_nolock。
+ 再然后就是 read_images，这个方法会读取所有的类的相关信息。
+ 最后是调用 reMethodizeClass:，这个方法是重新方法化的意思。
+ 在 reMethodizeClass: 方法内部会调用 attachCategories: ，这个方法会传入 Class 和 Category ，会将方法列表，协议列表等与原有的类合并。最后加入到 class_rw_t 结构体中。

#### method swizzling 是否使用过，说一下在实际开发中的什么场景下使用过。

1、UIViewController 生命周期的 PV 打点。
2、点击事件的打点等等。
3、防 crash。

#### 什么时候会报 unrecognized selector 的异常？

[](https://juejin.im/post/5e0afa3b6fb9a047ed618bea?utm_source=gold_browser_extension#heading-0)

+ OC的方法调用的本质是消息的传递。
+ 当调用一个对象上的某个方法，而该对象没有实现这个方法的时候，可以进行”消息转发“机制。
+ OC是动态语言，每个方法的调用在运行会被动态转为消息发送，即objc_msgsend

#### 消息查找流程

+ 消息的传递是通过 `objc_msgSend` 进行发送。
+ 查找的顺序，通过 isa 找到类，通过类的 superClass 找父类。
+ 先找缓存，后遍历方法列表。
+ 找到即返回，并且添加缓存。
+ 如果没找到走消息转发。

#### 消息转发流程

objc在向一个对象发送消息时，runtime库会根据对象的isa指针找到该对象实际所属的类，然后在该类中的方法列表以及其父类方法列表中寻找方法运行，如果，在最顶层的父类中依然找不到相应的方法时，程序在运行时会挂掉并抛出异常 `unrecognized selector sent to XXX` 。但是在这之前，objc的运行时会给出三次拯救程序崩溃的机会：

+ 动态方法解析，resolveInstanceMethod 或者 resolveClassMethod 在类中动态添加方法实现。如果没有实现，继续往下走。
+ 快速转发，fast forwarding
    + 在 `forwardingTargetForSelector` 该方法中会查找能够实现消息的新对象，如果没找到就返回nil，如果找到就返回这个新对象，消息的接受者也变成这个新对象。切记不可返回self，不然会死循环。
+ 慢速转发，normal forwarding
    + `methodSignatureForSelector` 根据SEL生成方法签名，封装了参数和返回值类型。如果methodSignatureForSelector返回nil，runtime就会发出doesNotRecognizeSelector消息。通过方法签名创建一个 `NSInvocation` 对象并发送 `forwardingInvocation` 消息给能处理消息的对象。
+ `forwardingInvocation` 中也没有对象可以处理该消息，就会调用 `doesNotRecognizedSelector` 抛出异常。

#### _objc_msgForward 是做什么的？直接调用会发生什么？

这个主要用作消息转发。

_objc_msgForward 消息转发流程，整体流程同上。

如果直接调用 `_objc_msgForward` 会直接走转发流程，不会走查找方法缓存和寻找方法。


#### 一个objc对象如何进行内存布局？测试对象图的结构

+ 所有父类的成员变量和本类的成员变量都会存在该对象的存储空间中。
+ 每一个对象内部都有一个isa指针，指向它的类对象，类对象又存放着本类的实例方法列表、成员变量列表、属性列表。
+ 类对象本身内部也有一个isa指针，指向它的元对象，元对象内部存储着类方法列表。
+ 类对象内部还有一个superclass指针，指向它的父类对象。
+ 根对象是NSObject，它的superclass指针指向nil。
+ 根元对象的superclass指向NSObject，isa指针指向自己。

#### 讲一下对象，类对象，元类，跟元类结构体的组成以及他们是如何相关联的？为什么对象方法没有保存的对象结构体里，而是保存在类对象的结构体里？

实例对象的 isa 指向类对象，类对象的 isa 指向元类，元类的 isa 指向根元类， 根元类的 isa 指向 自己。

#### iOS 中内省的几个方法？class方法和objc_getClass方法有什么区别?

[](https://www.jianshu.com/u/3171707d8892)

```
- (Class)class {
    return object_getClass(self);
}

Class object_getClass(id obj)
{
    // 对象存在，获取对象isa指针指向的对象
    if (obj) return obj->getIsa();
    else return Nil;
}
```

```
Class objc_getClass(const char *aClassName)
{
    if (!aClassName) return Nil;

    // NO unconnected, YES class handler
    return look_up_class(aClassName, NO, YES);
}
```

+ `object_getClass` 获取的是对象的isa指针所指向的对象

+ objc_getClass` 根据类名名称获取 Class

#### 在运行时创建类的方法objc_allocateClassPair的方法名尾部为什么是pair（成对的意思）？

这个方法的作用是动态创建类，因为类里面含有实例方法和类方法，实例方法是存放在类对象中，类方法是存放在元类中，所以其实创建新类是同时创建类和它的metaclass。所以尾部方法是pair成对的意思
