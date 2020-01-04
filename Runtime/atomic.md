## Atomic

#### 讲一下atomic的实现机制；为什么不能保证绝对的线程安全（最好可以结合场景来说）？（多线程方面，底层atomic的实现）

使用atomic修饰的属性，编译器会默认设置读写方式为原子操作，并添加互斥锁进行保护。这把锁是os_fair_lock。

```
// 属性设置方法
id objc_getProperty(id self, SEL _cmd, ptrdiff_t offset, BOOL atomic) {
    // Atomic retain release world
    spinlock_t& slotlock = PropertyLocks[slot];
    slotlock.lock();
    id value = objc_retain(*slot);
    slotlock.unlock();
    
    // for performance, we (safely) issue the autorelease OUTSIDE of the spinlock.
    return objc_autoreleaseReturnValue(value);
}
```

底层维护的一个属性锁数组，它的内部都是互斥锁，通过传入对象的哈希值来去数组中获取相应的互斥锁。

```
// 属性获取方法
static inline void reallySetProperty(id self, SEL _cmd, id newValue, ptrdiff_t offset, bool atomic, bool copy, bool mutableCopy)
{
    if (!atomic) {
        oldValue = *slot;
        *slot = newValue;
    } else {
        spinlock_t& slotlock = PropertyLocks[slot];
        slotlock.lock();
        oldValue = *slot;
        *slot = newValue;        
        slotlock.unlock();
    }

    objc_release(oldValue);
}
```