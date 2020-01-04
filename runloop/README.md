## Runloop

#### 参考

+ https://blog.ibireme.com/2015/05/18/runloop/

+ http://southpeak.github.io/2015/03/14/nsnotification-and-multithreading/

  

#### 说一下 Runloop 和线程的关系

runloop 跟线程的关系是一一对应关系，如果没有线程，runloop也没有存在的必要。主线程的runloop会自动创建，子线程的runloop需要手动创建。



#### Runloop 有几种 mode

+ NSRunLoopDefaultMode，App默认 mode，主线程就是运行在该 mode 下。
+ UITrackingRunloopMode，界面跟踪 mode，当滑动 UIScrollView 的时候，defaultMode 就会切换到 UITrackingMode。
+ UIInitializationRunloopMode，App 启动的时候获取的第一个 mode，启动完成后不再使用。
+ GSEventReceiveRunloopMode，接收系统内部事件，不经常使用。
+ kCFRunLoopCommonModes 占位 mode，common 属性，runloop 内容发生变化时，会自动把commonModes 里面的 source、observer、timer 同步到具有 common 属性的 mode 中。

#### Runloop Observer 的状态，Runloop 的内部实现？

Observer 的状态：

+ entry
+ beforeTimer
+ beforeSource
+ beforeWaiting
+ AfterWaiting
+ exit

#### RunLoop的作用是什么？它的内部工作机制了解么？（最好结合线程和内存管理来说）

内部的本质其实就是一个do-while循环。核心是基于mach port的。

+ 1、首先根据modeName找到对应的mode，如果mode中没有source、timer、observer，会直接返回。
+ 2、通知observer，runloop即将进入loop、
+ 3、通知observer，runloop即将触发timer回调。
+ 4、通知observer，runloop即将触发source0回调。
+ 5、触发source0回调
+ 6、如果有source1，直接处理这个source1，然后跳转去处理消息。
+ 7、通知observer，runloop即将进入休眠。
+ 8、线程即将进入休眠，直到被下面某个事件唤醒。
    + 一个基于port的source事件。
    + 一个timer时间到了
    + runloop自身超时时间到了
    + 被其他调用者手动唤醒。
+ 9、通知observer，runloop的线程刚刚被唤醒。
+ 10、通知observer，runloop即将退出。

#### Runloop 有关的内容

1、autorelease 释放相关

+ APP启动后，apple会在主线程注册两个observer，第一个observer监听的事件是entry，其回调会调用`_objc_autoreleasePoolPush()`创建自动释放池，优先级最高，保证在所有回调之前。
+ 第二个observer监听两个事件，beforeWaiting时调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池。exit 时调用 _objc_autoreleasePoolPop() 来释放自动释放池，这个优先级最低，保证所有回调之后。

关于事件响应 

+ apple 会注册一个 source1 用来接收系统事件，如果有事件发生，就会触发source1的回调函数，被事件封装成UIEvent进行分发，包括手势识别、屏幕旋转、按钮点击等等。

关于手势识别

+ 当识别一个手势的时候，会先 cancel 当前 touchbegin 等事件回调，将手势识别器标记为待处理，apple 注册的 observer 监听 beforWaiting 事件，回调函数就会获取内部被标记的待处理手势，然后执行。

关于界面更新

+ 当操作 UI 时，所有的改动，都被标记为待处理，并提交到一个全局容器中。apple注册了一个observer监听beforeWaiting和exit事件，回调函数内去遍历所有待处理的绘制和布局事件，去更新UI。

2、定时器

一个NSTimer注册到runloop后，Runloop会为其重复的时间点注册事件，为了节省资源，不会再准确的时间节点回调这个timer。

3、performSelector

其实当我们调用 `performSelecter:afterDelay` 和 `performSelector:onThread` 方法时，实际都会创建一个timer放到runloop中，如果当前线程没有runloop，则这个方法不会被执行。

4、关于GCD

只有调用 dispatch_async 时，libDispatch 才会向主线程发送runloop消息，runloop会被唤醒，并在回调里执行这个block。

5、关于网络请求

AFNetworking 2.x 版本的时候，内部还使用runloop来进行线程保活，因为NSURLConnection 的设计，当发送请求后，必须让线程保活来接收 connection delegate 回调回来的数据，所以设计了使用NSRunloop来进行线程保活。

#### NSNotificationCenter是在哪个线程发送的通知？

+ 它是一个线程安全类，可以在多线程环境下使用同一个 NSNotificationCenter 对象而不需要加锁。
+ 但是在多线程环境中，NSNotificationCenter 的 post 和转发 是在同一个线程中。

