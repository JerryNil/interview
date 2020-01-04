# Autoreleasepool 实现原理

#### autoreleasepool

它是什么？直接使用 clang 让编译器重写这个文件

```
{
    __AtAutoreleasePool __autoreleasepool;
}
```

```
struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};

```

结构体 ` __AtAutoreleasePool ` 在初始化的时候调用 `objc_autoreleasePoolPush()`，在析构的时候调用 `objc_autoreleasePoolPop`。

上述两个函数的实现通过底层代码查看，该实现在 `NSObject.mm` 文件中：

```
void *
objc_autoreleasePoolPush(void)
{
    // 调用了AutoreleasePoolPage中的push方法
    return AutoreleasePoolPage::push();
}

void
objc_autoreleasePoolPop(void *ctxt)
{
    // 调用了AutoreleasePoolPage中的pop方法
    AutoreleasePoolPage::pop(ctxt);
}
```
#### AutoReleasePoolPage

线程的自动释放池是一个指针堆栈，每个指针要么是要释放的对象，要么是自动释放池边界。当自动释放进行pop时，边界对象之前的对象都要被释放。这个堆栈被分配成一个双链表页面，

```
    magic_t const magic;
    // 一个AutoreleasePoolPage中会存储多个对象
    // next指向的是下一个AutoreleasePoolPage中下一个为空的内存地址（新来的对象会存储到next处）
    id *next;
    // 保存了当前页所在的线程(一个AutoreleasePoolPage属于一个线程，一个线程中可以有多个AutoreleasePoolPage)
    pthread_t const thread;
    // AutoreleasePoolPage是以双向链表的形式连接
    // 前一个节点
    AutoreleasePoolPage * const parent;
    // 后一个节点
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;
```

+ magic，用于对当前 `AutoreleasePoolPage` 完整性的校验
+ next，指向的是下一个AutoreleasePoolPage中下一个为空的内存地址
+ tread，当前线程
+ parent，前一个节点
+ child，后一个节点

#### POOL_SENTINEL

哨兵对象只是 nil 的别名。在每个自动释放池初始化调用 `objc_autoreleasePoolPush` 的时候，都会把一个 `POOL_SENTINEL ` push 到自动释放池栈顶。并且返回这个对象。

通过 `AutoReleasePoolpage::pop` 操作来负责对pool中的 `autoreleased` 对象执行 `release` 操作，直到第一个哨兵对象。

#### Push

```
void *
objc_autoreleasePoolPush(void)
{
    // 调用了AutoreleasePoolPage中的push方法
    return AutoreleasePoolPage::push();
}
```

`objc_autoreleasePoolPush` 内部是调用 page 的 push 方法。

```
static inline void *push() 
{
    id *dest;
    // POOL_BOUNDARY就是nil
    // 首先将一个哨兵对象插入到栈顶
    if (DebugPoolAllocation) {
        // Each autorelease pool starts on a new pool page.
        dest = autoreleaseNewPage(POOL_BOUNDARY);
    } else {
        dest = autoreleaseFast(POOL_BOUNDARY);
    }
    assert(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
    return dest;
}
```

push 方法就是先创建一个新的 autoreleasepool，然后往 next 位置插入一个哨兵对象，并且返回插入的内存地址。

```
static inline id *autoreleaseFast(id obj)
{
    // hotPage就是当前正在使用的AutoreleasePoolPage
    AutoreleasePoolPage *page = hotPage();
    if (page && !page->full()) {
        // 有hotPage且hotPage不满，将对象添加到hotPage中
        return page->add(obj);
    } else if (page) {
        // 有hotPage但是hotPage已满
        // 使用autoreleaseFullPage初始化一个新页，并将对象添加到新的AutoreleasePoolPage中
        return autoreleaseFullPage(obj, page);
    } else {
        // 无hotPage
        // 使用autoreleaseNoPage创建一个hotPage,并将对象添加到新创建的page中
        return autoreleaseNoPage(obj);
    }
}
```

+ autoreleaseFast 函数在执行一个具体的插入操作时，分别对三种情况进行不同的处理。
    + 当前page存在且没有满的时候，直接将对象添加到当前page中，即next指向的位置。
    + 当前page存在且满的时候，创建一个新的page，并且将对象添加到新的page中。
    + 当个page不存在的时候，创建一个新的page，并将对象添加到新创建的page中。

#### autorelease操作

+ autorelease操作其实就是调用AutoReleasePoolPage中的autorelease函数。
+ autorelease函数中其实也调用了autoreleasefast函数，跟push操作很类似。
+ autorelease操作插入的是一个具体的autoreleased对象。

#### pop

```
void
objc_autoreleasePoolPop(void *ctxt)
{
    // 调用了AutoreleasePoolPage中的pop方法
    AutoreleasePoolPage::pop(ctxt);
}
```

`objc_autoreleasePoolPop` 调用的就是 page 的 pop  方法。

```
static inline void pop(void *token) {
    AutoreleasePoolPage *page = pageForPointer(token);
    id *stop = (id *)token;

    page->releaseUntil(stop);

    if (page->child) {
        if (page->lessThanHalfFull()) {
            page->child->kill();
        } else if (page->child->child) {
            page->child->child->kill();
        }
    }
}
```

+ pop 中的 token 就是 push 函数的返回值。通过 `pageForPointer ` 找到所在的 page，然后内存地址在 token 之后的所有对象都会被 release，直到 page 的 next 指针指向边界对象。
+ 调用 `releaseUntil` 方法释放栈中的对象，直到 stop。
+ 调用 chil 的 kill 方法。

```
// 释放AutoreleasePoolPage中的对象，直到next指向stop
while (this->next != stop) {
    // Restart from hotPage() every time, in case -release 
    // autoreleased more objects
    // hotPage可以理解为当前正在使用的page
    AutoreleasePoolPage *page = hotPage();

    // fixme I think this `while` can be `if`, but I can't prove it
    // 如果page为空的话，将page指向上一个page
    // 注释写到猜测这里可以使用if，我感觉也可以使用if
    // 因为根据AutoreleasePoolPage的结构，理论上不可能存在连续两个page都为空
    while (page->empty()) {
        page = page->parent;
        setHotPage(page);
    }

    page->unprotect();
    // obj = page->next; page->next--;
    id obj = *--page->next;
    memset((void*)page->next, SCRIBBLE, sizeof(*page->next));
    page->protect();

    // POOL_BOUNDARY为nil，是哨兵对象
    if (obj != POOL_BOUNDARY) {
        // 释放obj对象
        objc_release(obj);
    }
}
```

```
// 删除双向链表中的每一个page
void kill() 
{
    // Not recursive: we don't want to blow out the stack 
    // if a thread accumulates a stupendous amount of garbage
    AutoreleasePoolPage *page = this;
    // 找到链表最末尾的page
    while (page->child) page = page->child;

    AutoreleasePoolPage *deathptr;
    // 循环删除每一个page
    do {
        deathptr = page;
        page = page->parent;
        if (page) {
            page->unprotect();
            page->child = nil;
            page->protect();
        }
        delete deathptr;
    } while (deathptr != this);
}
```

#### 什么时候调用autoreleasepool

+ 编译器检查是否以alloc、new、copy、mutableCopy开始，如果不是则自动将返回值对象注册到autoreleasepool
+ 以 __weak 修饰的对象，会注册到autoreleasepool
+ 调用Foundation 对象的类方法会注册到autoreleasepool中。
+ id的指针或对象的指针，在没有显示的指定修饰符，会被默认附加上__autoreleaseing 修饰符。


#### autorelease对象在何时释放

+ 在没有手动加入 autoreleasepool 的情况下，autorelease 对像是在当前runloop迭代结束后释放的，而它能够释放的原因是系统在每个runloop迭代中加入自动释放池的push和pop操作。