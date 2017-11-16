---
title: weak 源码跟踪
date: 2017-08-30 17:03:22
categories: 源码阅读
tags: 源码阅读
---

weak 跟踪学习笔记

<!-- more -->
# weak 的源码跟踪

## 开始DEBUG
> main 中代码

{% asset_img main代码.png main代码 %}


``` code
2017-08-29 09:22:27.252377+0800 debug-objc[1763:112943] addr is 0x100703db0

```

### 调用堆栈

{% asset_img weak调用堆栈.png 调用堆栈 %}        

怎么markdown解析有问题？    


{% asset_img 代码块.png %}

上面是调用的传参类型    
         

这边 `obj` 的地址 和 `objc_storeWeak` 的第一个参数的地址 做个对比 
`setIvar--> objc_storeWeak` 这边，我参考了 `_object_setIvar` 帮助理解 
                 
 
### 地址偏移计算函数 


``` code
ivarOffset = ivar_getOffset(ivar); //偏移单位 
.
//typedef struct objc_object *id   //id 是一个接受任何类型对象的指针的类型
id *location = (id *)((char *)obj + offset); //location 就是 指针的指针
.
objc_storeWeak(location, value)         

```

                     
接着 `objc_storeWeak` 这里入参就是 __地址的地址__ 和 __新的存储值__ 。            


``` code
XXView *view = [XXView new];
XXObject *vv = [XXObject new];
Ivar iva = class_getInstanceVariable([vv class], "_view");
object_setIvar(vv, iva, view);
XXView *view2 = [XXView new];
object_setIvar(vv, iva, view2);
```

调用修改



## SideTable和SideTables

[关于c++的引用][4]


``` code
'&': 在 C 中是取地址符这个好理解
   在 C++ 中这个是引用

```


### 关于 分拆锁 分离锁

__分拆锁__ 和 __分离锁__ 不是锁 ，他是使用__锁__的一种方式 不摆链接了 搜一下很多


### SideTables

单看字面意思，感觉像是上面`SideTable`的数组的数据结构。     
内心OS ：到`SideTables`中 按地址得到对应的`SideTable`？__这里就是一个简单的数组吗?!__

`alignas` : [位对齐][3]    
先找到对应的 `Table` 如下：          


``` code
oldTable = &SideTables()[oldObj];
|
SideTables()
他返回的是
return *reinterpret_cast<StripedMap<SideTable>*>(SideTableBuf);

```
这里有个c++转换符         

`reinterpret_cast`  ：   [类型转换][2] 辅助`hash`函数            
   
先注意返回值 `StripedMap<SideTable>*` 返回的是 `StripedMap` 对象。在他里面对 `[]` 做了运算符重载。

{% asset_img hash表实现.png hash表实现 %}

这里发现就是 `hash` 表的运算


`SideTableBuf` 是什么?


``` code
alignas(StripedMap<SideTable>) static uint8_t 
    SideTableBuf[sizeof(StripedMap<SideTable>)];
```


通过上面得到的地址到 `SideTables` 中获取对应的 `SideTable`    



### SideTable  
 找到`old`查找流程
 
####  step 1 查找 Table 流程         

 
 ``` code
 SideTable *oldTable = &SideTables()[oldObj];
 struct SideTable {
    spinlock_t slock;//有把自旋锁 
    RefcountMap refcnts;//一个继承与 DenseMapBase 的类 主要是有一个
    std::pair<KeyT, ValueT> BucketT; 这样的键值对 
    weak_table_t weak_table;// `weak_table` 全局的一个表 next
    ...
 }
 ```
 
 
#### step 2 查找 Table 对应的 weak_entry_t 

 
 ``` code
 //Table 中的weak_table结构
 /**
 * The global weak references table. Stores object ids as keys,
 * and weak_entry_t structs as their values.
 */
 这里说了 以obj的ids 作为keys值 以结构体作为value值 全局表
 这里weak_entries就是weak_entry_t数组
 struct weak_table_t {
    weak_entry_t *weak_entries;// next 
    size_t    num_entries;
    uintptr_t mask; // 掩码
    uintptr_t max_hash_displacement;
};

//退出循环的条件
..
 weak_table->weak_entries[index].referent != referent
...
 //数组中存的单元
 struct weak_entry_t {
    DisguisedPtr<objc_object> referent;// 最后对比的指针
    union {
        struct {
            weak_referrer_t *referrers;
            uintptr_t        out_of_line_ness : 2;
            uintptr_t        num_refs : PTR_MINUS_2;
            uintptr_t        mask; // bit掩码（bit Mask）
            uintptr_t        max_hash_displacement;
        };
        struct {
            // out_of_line_ness field is low bits of inline_referrers[1]
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
    };

 ...
 };
 
 while (weak_table->weak_entries[index].referent != referent/*这里*/) {
        index = (index+1) & weak_table->mask;
        if (index == begin) bad_weak_table(weak_table->weak_entries);
        hash_displacement++;
        if (hash_displacement > weak_table->max_hash_displacement) {
            return nil;
        }
    }
    
    return &weak_table->weak_entries[index];
 
 ```
 
 
#### step 3 删除对应的old  ( Clean up old value, if any. )
 
 `remove_referrer(entry, referrer);`      

对 `weak_entry_t` 中因为是union的结构需要判断是那种结构，根据 `out_of_line` 返回值。
这里注释讲的比较清楚


``` code
// out_of_line_ness field overlaps with the low two bits of inline_referrers[1].
// inline_referrers[1] is a DisguisedPtr of a pointer-aligned address.
// The low two bits of a pointer-aligned DisguisedPtr will always be 0b00
// (disguised nil or 0x80..00) or 0b11 (any other address).
// Therefore out_of_line_ness == 0b10 is used to mark the out-of-line state.
```


直接把对应的 `entry->inline_referrers[i] = nil; ` 或者 `entry->referrers[index] = nil;` 

 

#### step 4 赋新值 （ Assign new value, if any ）


`weak_register_no_lock(&newTable->weak_table, (id)newObj, location, 
                                  crashIfDeallocating)`
                                  

经过一系列判断,刚刚删除旧的变量的引用的时候也有这样的判断忘记写了


``` code
...
append_referrer(entry, referrer);
or
weak_entry_t new_entry(referent, referrer);
weak_grow_maybe(weak_table);
weak_entry_insert(weak_table, &new_entry);
...

```

到这有点晃神 回去看看 

#### 对 isa_t 的分析 (在x86上)


``` code
struct {
        uintptr_t nonpointer        : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 44; // MACH_VM_MAX_ADDRESS 0x7fffffe00000
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;  弱引用
        uintptr_t deallocating      : 1;  释放
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 8;  引用计数
#       define RC_ONE   (1ULL<<56)
#       define RC_HALF  (1ULL<<7)
    };
    
```


下一篇看 `Runloop` 对 `weak` 引用释放的时机 还有 引用计数添加 对 `isa_t` 的操作

[0]: https://www.zybuluo.com/mdeditor?url=https://www.zybuluo.com/static/editor/md-help.markdown#cmd-markdown
[1]: http://www.jianshu.com/p/ef6d9bf8fe59
[2]: http://www.cnblogs.com/ider/archive/2011/07/30/cpp_cast_operator_part3.html
[3]: http://zh.cppreference.com/w/c/language/_Alignas
[4]: http://www.cnblogs.com/Mr-xu/archive/2012/08/07/2626973.html
[5]: https://github.com/RetVal/objc-runtime


