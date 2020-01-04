## Associated Object

首先 runtime 为我们提供了三个关联对象的三个方法：

```
id objc_getAssociatedObject(id object, const void *key);

void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy);
```
三个方法的作用分别是：

+ 根据 key 获取关联对象。
+ 以键值对的形式添加关联对象。
+ 移除所有关联对象。


#### `objc_setAssociatedObject`

首先是 `objc_setAssociatedObject` 方法

```
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy) {
    _object_set_associative_reference(object, (void *)key, value, policy);
}
```

内部调用 `_object_set_associative_reference` 来完成关联对象的设置

```
void _object_set_associative_reference(id object, void *key, id value, uintptr_t policy) {
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        disguised_ptr_t disguised_object = DISGUISE(object);
        if (new_value) {
            if (i != associations.end()) {
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);

            }
        }
    }
}
```

上述的源代码可以发现几个新的类和数据结构：

+ AssociationsManager
+ AssociationsHashMap
+ ObjectAssociationMap
+ ObjectAssociation

我们来先看看这几个类或数据结构到底是用来干什么的？

#### AssociationsManager

`AssociationsManager`在代码中的定义如下：

```
spinlock_t AssociationsManagerLock;

class AssociationsManager {
    // associative references: object pointer -> PtrPtrHashMap.
    static AssociationsHashMap *_map;
public:
    AssociationsManager()   { AssociationsManagerLock.lock(); }
    ~AssociationsManager()  { AssociationsManagerLock.unlock(); }
    
    AssociationsHashMap &associations() {
        if (_map == NULL)
            _map = new AssociationsHashMap();
        return *_map;
    }
};
```

+ 它维护着 AssociationsHashMap 和 AssociationsManagerLock 两个单例。AssociationsManagerLock 是一个互斥锁。
+ 初始化的时候加锁，析构的时候解锁。
+ 通过 associations() 方法获取一个全局的 AssociationsHashMap。
+ 使用锁的机制来保证对 AssociationsHashMap 的操作是线程安全的。

#### AssociationsHashMap
`AssociationsHashMap` 的类型声明如下：

```
typedef hash_map<disguised_ptr_t, ObjectAssociationMap *> AssociationsHashMap;
```

它是一个散列表，是以 disguised_ptr_t 为键，ObjectAssociationMap 类型实例为值的映射方式来保存对象。

#### ObjectAssociationMap

`ObjectAssociationMap` 的类型定义如下：

```
typedef hash_map<void *, ObjcAssociation> ObjectAssociationMap;
```

保存了从 key 到 ObjcAssociation 的映射，这个数据结构保存了当前对象对应的所有关联对象。

#### ObjcAssociation

```
class ObjcAssociation {
   uintptr_t _policy;
   id _value;
public:
   ObjcAssociation(uintptr_t policy, id value) : _policy(policy), _value(value) {}
   ObjcAssociation() : _policy(0), _value(nil) {}

   uintptr_t policy() const { return _policy; }
   id value() const { return _value; }
   
   bool hasValue() { return _value != nil; }
};
```

该对象内包含了 policy 和 value。

现在我们重新来看 objc_setAssociatedObject 的具体实现，我们使用一张流程图表示：

![2016-06-08-objc-ao-associateobjcect](media/15736628242691/2016-06-08-objc-ao-associateobjcect.png-1000width)

这边有两种情况：

+ new_value != nil，添加和更新关联对象。
+ new_value == nil，删除一个关联对象。

设置关联对象 

```
  // create the new association (first time).
 ObjectAssociationMap *refs = new ObjectAssociationMap;
 associations[disguised_object] = refs;
 (*refs)[key] = ObjcAssociation(policy, new_value);
 object->setHasAssociatedObjects();
```

更新关联对象

```
// secondary table exists
 ObjectAssociationMap *refs = i->second;
 ObjectAssociationMap::iterator j = refs->find(key);
 if (j != refs->end()) {
     old_association = j->second;
     j->second = ObjcAssociation(policy, new_value);
 } else {
     (*refs)[key] = ObjcAssociation(policy, new_value);
 }
```

删除关联对象

```
AssociationsHashMap::iterator i = associations.find(disguised_object);
if (i !=  associations.end()) {
 ObjectAssociationMap *refs = i->second;
 ObjectAssociationMap::iterator j = refs->find(key);
 if (j != refs->end()) {
     old_association = j->second;
     refs->erase(j);
 }
}
```

+ 使用 old_association 创建一个临时的 ObjcAssociation 对象。
+ 通过 acquireValue 函数对 new_value 进行 retain 或 copy 操作。
+ 初始化一个 AssociationsManager，获取唯一关联对象的哈希表 AssociationsHashMap。
+ 先使用 DISGUISE(object) 作为 key 寻找对应的 ObjectAssociationMap。
+ 如果没找到，初始化一个 ObjectAssociationMap，再实例化 ObjcAssociation 对象添加到 map 中。并调用 setHasAssociatedObjects 表明当前对象含有关联对象。
+ 如果找到对应 ObjectAssociationMap，就看 key 是否存在，来决定是更新原有的关联对象还是增加一个。
+ 到最后原来的关联对象有值的话，会调用 ReleaseValue 释放关联对象的值。

#### objc_getAssociatedObject

```
id _object_get_associative_reference(id object, void *key) {
    id value = nil;
    uintptr_t policy = OBJC_ASSOCIATION_ASSIGN;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        disguised_ptr_t disguised_object = DISGUISE(object);
        AssociationsHashMap::iterator i = associations.find(disguised_object);
        if (i != associations.end()) {
            ObjectAssociationMap *refs = i->second;
            ObjectAssociationMap::iterator j = refs->find(key);
            if (j != refs->end()) {
                ObjcAssociation &entry = j->second;
                value = entry.value();
                policy = entry.policy();
                if (policy & OBJC_ASSOCIATION_GETTER_RETAIN) {
                    objc_retain(value);
                }
            }
        }
    }
    if (value && (policy & OBJC_ASSOCIATION_GETTER_AUTORELEASE)) {
        objc_autorelease(value);
    }
    return value;
}
```

+ 获取全局的哈希表 AssociationsHashMap
+ 通过 DISGUISE(object) 为key，获取哈希表中存储对应的 ObjectAssociationMap。
+ 以 void *key 为 key 查找 ObjcAssociation。
+ 根据 policy 进行相应的内存管理。

```
if (policy & OBJC_ASSOCIATION_GETTER_RETAIN) {
    objc_retain(value);
}

if (value && (policy & OBJC_ASSOCIATION_GETTER_AUTORELEASE)) {
        objc_autorelease(value);
}
```
5、最后返回关联对象的值。

#### objc_removeAssociatedObjects

```
void objc_removeAssociatedObjects(id object) 
{
    if (object && object->hasAssociatedObjects()) {
        _object_remove_assocations(object);
    }
}
```

+ 会先通过 hasAssociatedObjects 来判断是否有关联对象。没有则调用结束。
+ 调用`_object_remove_assocations`方法。

```
void _object_remove_assocations(id object) {
    vector< ObjcAssociation,ObjcAllocator<ObjcAssociation> > elements;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        if (associations.size() == 0) return;
        disguised_ptr_t disguised_object = DISGUISE(object);
        AssociationsHashMap::iterator i = associations.find(disguised_object);
        if (i != associations.end()) {
            // copy all of the associations that need to be removed.
            ObjectAssociationMap *refs = i->second;
            for (ObjectAssociationMap::iterator j = refs->begin(), end = refs->end(); j != end; ++j) {
                elements.push_back(j->second);
            }
            // remove the secondary table.
            delete refs;
            associations.erase(i);
        }
    }
    // the calls to releaseValue() happen outside of the lock.
    for_each(elements.begin(), elements.end(), ReleaseValue());
}
```

+ 获取全局 AssociationsHashMap。
+ 判断哈希表大小是否为 0。
+ 以 DISGUISE(object) 为 key 查找 ObjectAssociationMap。
+ 查找到所有关联对象，存放在 elements 中。
+ 最后 elements 中的对象调用 ReleaseValue，释放所有不需要的值。


#### 小结

+ 关联对象其实就是 ObjcAssociation 对象。
+ 关联对象由 AssociationsManager 管理并在 AssociationsHashMap 存储。
+ 对象的指针及其对应的 ObjectAssociationMap 以键值对的形式存储在 AssociationsHashMap 中。
+ ObjectAssociationMap 则是用于存储关联对象的数据结构。
+ 每个对象都有一个标记为 has_assoc 表示对象是否含有关联对象。

#### 引用 

+ https://draveness.me/ao
  