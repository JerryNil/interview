# weak 实现原理

#### 修饰符定义

+ 当__weak修饰的变量，在所引用的对象被废弃时，则变量自动设置为nil。
+ 使用__weak修饰的变量，即是注册到 autoReleasePool 中的对象。

#### SideTable

首先我们来了解一下SideTable

```
struct SideTable {
    spinlock_t slock;
    RefcountMap refcnts;
    weak_table_t weak_table;

    SideTable() {
        memset(&weak_table, 0, sizeof(weak_table));
    }

    ~SideTable() {
        _objc_fatal("Do not delete SideTable.");
    }

    void lock() { slock.lock(); }
    void unlock() { slock.unlock(); }
    void forceReset() { slock.forceReset(); }

    // Address-ordered lock discipline for a pair of side tables.

    template<HaveOld, HaveNew>
    static void lockTwo(SideTable *lock1, SideTable *lock2);
    template<HaveOld, HaveNew>
    static void unlockTwo(SideTable *lock1, SideTable *lock2);
};
```

+ sideTable 是一个结构体，内部有一个互斥锁，这把锁的作用是在多线程环境下防止资源访问产生的竞态条件。
+ refcountMap 是用于管理对象的引用计数。
+ weak_table_t 是用来管理弱引用依赖表。

####  `weak_table_t`

```
struct weak_table_t {
    weak_entry_t *weak_entries;
    size_t    num_entries;
    uintptr_t mask;
    uintptr_t max_hash_displacement;
};
```

+ 它是全局的弱引用表，存储对象的 ids 作为 key，weak_entry_t 结构体对象作为值。
+ weak_entries 保存了所有指向指定对象的 weak 指针。
+ num_entries 是存储空间，表明存储 weak_entry_t 的数量。

```
// 添加一个 <object, weak pointer> 到弱引用表中。
id weak_register_no_lock(weak_table_t *weak_table, id referent, 
                         id *referrer, bool crashIfDeallocating);
// 从弱引用表中移除一个 <object, weak pointer>
void weak_unregister_no_lock(weak_table_t *weak_table, id referent, id *referrer);

// 将所有的弱引用指针置为nil
void weak_clear_no_lock(weak_table_t *weak_table, id referent);
```

#### `weak_entry_t`

```
struct weak_entry_t {
    DisguisedPtr<objc_object> referent;
    union {
        struct {
            weak_referrer_t *referrers;
            uintptr_t        out_of_line_ness : 2;
            uintptr_t        num_refs : PTR_MINUS_2;
            uintptr_t        mask;
            uintptr_t        max_hash_displacement;
        };
        struct {
            // out_of_line_ness field is low bits of inline_referrers[1]
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
    };

    bool out_of_line() {
        return (out_of_line_ness == REFERRERS_OUT_OF_LINE);
    }

    weak_entry_t& operator=(const weak_entry_t& other) {
        memcpy(this, &other, sizeof(other));
        return *this;
    }

    weak_entry_t(objc_object *newReferent, objc_object **newReferrer)
        : referent(newReferent)
    {
        inline_referrers[0] = newReferrer;
        for (int i = 1; i < WEAK_INLINE_COUNT; i++) {
            inline_referrers[i] = nil;
        }
    }
};
```

#### `weak_register_no_lock`

weak 修饰符的属性整个注册调用栈如下：

```
- _object_setIvar
    - objc_storeWeak
        - storeWeak
            - weak_register_no_lock
```

整个添加流程如下：

+ storeWeak 内部先初始化两个 sideTable，初始化 sideTable 的同时，会创建弱引用表。然后判断是否存在旧值，存在就先清除，调用 `weak_unregister_no_lock`；然后添加新值，调用 `weak_register_no_lock`。
+ 接下来是添加步骤
    + 如果以对象的指针能找到对应的 entry，就在 entry 中添加引用对象；反之进行下述操作。
    + 以指向对象的指针作为 key，weak修饰符变量作为 value，创建新的 entry；超过75%之后，开始以原来的2倍进行扩容；最后添加到 weak table 中。
    
#### `weak_clear_no_lock`

对象被废弃的整个调用栈如下：

```
- dealloc
    - _objc_rootDealloc
        - rootDealloc
            - object_dispose
                - objc_destructInstance
                    - clearDeallocating
                        - sidetable_clearDeallocating
                            - weak_clear_no_lock
```

`objc_destructInstance` 内部实现：

```
// Read all of the flags at once for performance.
bool cxx = obj->hasCxxDtor();
bool assoc = obj->hasAssociatedObjects();

// This order is important.
if (cxx) object_cxxDestruct(obj);
if (assoc) _object_remove_assocations(obj);
obj->clearDeallocating();
```

+ 调用C++的析构器。
+ 判断有没有关联对象，移除关联对象引用。
+ 调用 `clearDeallocating`

```
SideTable& table = SideTables()[this];

// clear any weak table items
// clear extra retain count and deallocating bit
// (fixme warn or abort if extra retain count == 0 ?)
table.lock();
// 遍历引用计数表
RefcountMap::iterator it = table.refcnts.find(this);
if (it != table.refcnts.end()) {
   // 找有有弱引用标志的 weak table
   if (it->second & SIDE_TABLE_WEAKLY_REFERENCED) {
       // 清除弱引用
       weak_clear_no_lock(&table.weak_table, (id)this);
   }
   // 移除整个计数表
   table.refcnts.erase(it);
}
table.unlock();
```

然后通过 `weak_clear_no_lock` 的源码来分析下具体的释放流程是怎样的？

```
void 
weak_clear_no_lock(weak_table_t *weak_table, id referent_id) 
{
    objc_object *referent = (objc_object *)referent_id;

    weak_entry_t *entry = weak_entry_for_referent(weak_table, referent);
    if (entry == nil) {

        return;
    }

    // zero out references
    weak_referrer_t *referrers;
    size_t count;
    
    if (entry->out_of_line()) {
        referrers = entry->referrers;
        count = TABLE_SIZE(entry);
    } 
    else {
        referrers = entry->inline_referrers;
        count = WEAK_INLINE_COUNT;
    }
    
    for (size_t i = 0; i < count; ++i) {
        objc_object **referrer = referrers[i];
        if (referrer) {
            if (*referrer == referent) {
                *referrer = nil;
            }
            else if (*referrer) {
            }
        }
    }
    
    weak_entry_remove(weak_table, entry);
}
```

+ 先通过赋值对象的指针在 table 中，找到所有存储weak修饰符变量指针的 entry。
+ 然后判断该 entry 有没有扩容，去动态数组中查找referrers，否则从静态数组中查找referrers。
+ 通过遍历找到所有的弱引用 referrer，同时置为nil。
+ 把存储的 entry 从 table 中删除。
+ 最后从引用计数表中删除废弃对象为 key 的所有记录。

#### 小结

首先在 runtime 中维护了一个 weak 表，其实是一个 hash 表。用于存储指向某个对象的所有weak指针，key是指对象的地址，value是weak指针的地址（这个地址的值是所指对象的地址）数组。

整个过程分为三步：

+ `objc_initWeak`，初始化创建 SideTable，然后通过 SideTable 创建对应的 weak
table。
+ `objc_storeWeak`， 添加引用时，旧对象解除注册操作，新对象添加注册操作。把引用对象的地址作为 key，weak 修饰符的变量地址作为 value 存储到 weak table 中。
+ `dealloc`，对象释放时，先通过对象指针找到存储 weak 修饰符变量指针的 entry；然后找出所有的 referrers；再通过遍历 referrers 获取到每一个 referrer，同时置为 nil；接着把存储的 entry 从 table 中移除；最后从引用技术表中删除废弃对象为 key 的所有记录。

#### 参考

+ http://www.cocoachina.com/articles/18962
+ https://juejin.im/post/5defc49951882512290f1e87?utm_source=gold_browser_extension