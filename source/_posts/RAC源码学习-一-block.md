---
title: RAC源码学习(一) block
date: 2017-11-14 16:59:35
categories: 源码阅读
tags: 源码阅读
---

学习 RAC 源码之前，对block再次学习...

<!--more-->

## block

### 基本概念
> 闭包     
> lambda    
> block     
> 箭头函数      
> 匿名函数     

去年到今年的整个过程中，我接触的一些编程语言。都会出现一些类似的特性：‘跨’作用域的访问变量。这个在 iOS 中也是有类似的方法实现。我就想把他们整理一下。可能对概念的划分有点不准确或者遗漏,还望指正。

如果要划分谁的 scope 比较大，这个其实我并不那么纠结，没有什么意义，主要看理解。硬说的话，可能匿名函数的范畴更大一点

来自 WIKIPEDIA   
匿名函数&emsp;&emsp;&emsp;是指一类无需定义标识符(函数名)的函数或子程序                      
闭包&emsp;&emsp;&emsp;&emsp;&emsp;又称词法闭包或者函数闭包，是引用了自由变量的函数。这个被引用的自由变量将和这个函数一同存在，即使已经离开了创造它的环境。

其他的你可能会遇到的描述就是：在Python中用 lambda 来创建匿名函数之类的。

C++ 中的 lambda 表达式
   
``` 
[capture](parameters) mutable -> return-type{statement}
```

JavaScript 中的箭头( => )函数

```
ES6 的特性
(param1, param2, …, paramN) => { statements }
```

Python 中的 lambda 表达式

```
lambda parameters: return [statement],variable
```

Object-C <sub>[1]</sub> 中的块( block )表达式

```
As a local variable :
returnType (^blockName)(parameterTypes) = ^returnType(parameters) {...};

As a property :
@property (nonatomic, copy, nullability) returnType (^blockName)(parameterTypes);

As a method parameter :
- (void)someMethodThatTakesABlock:(returnType (^nullability)(parameterTypes))block;

As an argument to a method call :
[someObject someMethodThatTakesABlock:^returnType (parameters){...}];

As a typedef :
typedef returnType (^TypeName)(parameterTypes);
TypeName blockName = ^returnType(parameters){...};

```


这些表达式给我呈现的最直观的感受是:           
1. 书写格式比较简洁        
2. 捕获变量

### 用法和注意

用法可能就像上面定义描述的那样，作为值传递、回调、参数去使用。这里可能比较常见的注意点就是 iOS 里面一直会说的 retain cycles（引用循环）。      
<img src="./retain_cycle.png" width = "319" height = "190" alt="引用循环" align=center/>

如何避免这样的循环引用？       
<img src="./weak_cycle.png" width = "319" height = "190" alt="引用循环" align=center/>


## 那么 block 是什么

在回答这个问题之前我们要先去看一个日常开发中的场景：

```
@property (nonatomic,strong) Variable *vari_;

{	
   Variable *_vari = [Variable new];
    NSLog(@"%s --- %p --- %p",__FUNCTION__,_vari,&_vari);
    [self just_need_a_variable:_vari];
}

- (void)just_need_a_variable:(NSObject *)obj {
	NSLog(@"%s --- %p --- %p",__FUNCTION__,obj,&obj);
}

你可以先猜一下打印出什么
console :

-[RWViewController viewDidLoad] --- 0x1c00144c0 --- 0x16ee291c0
-[RWViewController just_need_a_variable:] --- 0x1c00144c0 --- 0x16ee28eb8

```

这里就扯到堆栈的概念了，我们一般 new、alloc、copy 这样的操作，都是将变量分配在堆上，然后局部变量是在栈帧上对堆地址的引用。而在函数调用作为参数传递的是这个堆上的地址，本身这个栈地址是跟着这个栈帧变化了。   

<img src="./stack_frame.png" width = "386" height = "204" alt="内存分部图" align=center/>

伪代码

```
typedef Addr address ;// 地址类型变量

in heap
Addr heap_addr = &(heap intance);
 
in stack 
Addr stack1_addr = &(stack instance);
(stack instance).data = heap_addr;

define function delivery(Addr addr);
delivery(stack1_addr);

```

这里函数参数的传递是值拷贝的指针传递。函数传递参数的三种方式：        
1. pass by value          
2. pass by pointer        
3. pass by reference         

照例放一个内存分部图      
<div align=center>
<img src="./内存分部图.png" width = "427" height = "405" alt="内存分部图" align=center/>
</div>


那回到原来的问题，block 是什么？     
              
我们程序会经历一个从预编译、编译、汇编、链接的过程。这里我们用到一个工具 Clang。它是一个C语言、C++、Objective-C、Objective-C++语言的轻量级编译器。Clang将支持其普通lambda表达式、返回类型的简化处理以及更好的处理constexpr关键字。Clang我仅了解一下常见的调试命令，感兴趣的可以自行查阅<sub>2</sub>。这里我们只要知道他是一个前端可以转化我们的OC成C语言。

<img src="./llvm.png" width = "604" height = "376" alt="内存分部图" align=center/>

下面开始用 clang 改写

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...
        void(^blo)(void) = ^() {
            NSLog(@"hello");
        };
        blo();
    }
    return 0;
}

console :
rewriteoc main.m 
1. 会在同级目录下生成一个 cpp 文件，
2. rewriteoc 是我在 .bash_profile 中事先 alias 好的 clang 的命令
```

将生成的 cpp 文件中多余的代码去除，剩下了关键的内容

```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
   NSLog((NSString *)&__NSConstantStringImpl__var_folders_kz_xtgqw0n96yj08ggnvknj700r0000gn_T_main_1b6637_mi_0);
}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};

int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { 
    __AtAutoreleasePool __autoreleasepool; 
        void(*blo)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
        ((void (*)(__block_impl *))((__block_impl *)blo)->FuncPtr)((__block_impl *)blo);
    }
    return 0;
}

```
这里 main 函数已经被改写了，autoreleasepool 我们就不谈了，不是我们的重点。我看看到原来的 block 的定义也被改写了。这里定一个了一个 \_\_main\_block\_impl\_0 的结构体，这个结构体有两个属性和一个初始化函数。在 main 函数中，通过刚刚的结构体初始化函数生成了一个 blo。他的初始化参数是一个\__main_block_func_0 静态函数 和 一个静态的已经初始化好的结构体。调用这个初始化函数做了哪些操作？

```
impl.isa = &_NSConcreteStackBlock;
impl.Flags = flags;
impl.FuncPtr = fp;
Desc = desc;

```
对这个结构体的两个属性进行赋值。
impl.isa = &\_NSConcreteStackBlock; 说明他是一个栈上的 block，你可能还在其他地方听过分配在堆上的 block 或者全局的 block。他们就是依照这个 isa 来区分的对应的值是 NSConcreteMallocBlock NSConcreteGlobalBlock 。 
impl.FuncPtr = fp;这个记录了这个 block 的执行函数。
再看一下 FuncPtr 传入的函数，他是带一个参数的 struct \_\_main\_block\_impl\_0 \*\_\_cself 。到这里看到这个参数名称，其实我们可以猜到他是什么了 cself 一个对自己的引用。看下代码实现 ((void (\*)(\_\_block_impl \*))((\_\_block\_impl \*)blo)->FuncPtr)((\_\_block\_impl *)blo);跟我们猜的一样。

上面的 block 是一个空操作，如果我们访问外部变量呢？我们稍微改一下 main 函数的代码

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...
        int value = 10;
        void(^blo)(void) = ^() {
            NSLog(@"value is %d",value);
        };
        blo();
    }
    return 0;
}
```
同样用 Clang 改写


```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int value;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _value, int flags=0) : value(_value) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int value = __cself->value; // bound by copy
  NSLog((NSString *)&__NSConstantStringImpl__var_folders_kz_xtgqw0n96yj08ggnvknj700r0000gn_T_main_512dfa_mi_0,value); 
}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        int value = 10;
        void(*blo)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, value));
        ((void (*)(__block_impl *))((__block_impl *)blo)->FuncPtr)((__block_impl *)blo);
    }
    return 0;
}

```

生成的文件基本一样除了几个地方。       
\_\_main\_block\_impl\_0 这个结构体中多了一个 value 的属性。结构体的初始化函数也对 value 做了初始化工作。" ："是 C++初始化函数的写法.      
在 block 的执行的那个静态函数 \_\_main\_block\_func\_0 中,有个同名的变量 value ，它是对上面结构体的 value属性的引用。  
main 函数中，将 value 作为初始化参数，去调用结构体的初始化函数。
总结来说就是，main 函数中的 value 变量和我们 block 中打印的 value 已经不是同一个变量了。

我们在稍微修改一下我们的代码，对main中的 value 进行操作。

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...
        __block int value = 10;
        void(^blo)(void) = ^() {
            value += 1;
        };
        blo();
    }
    return 0;
}
```
用 Clang 改写

```

struct __Block_byref_value_0 {
  void *__isa;
__Block_byref_value_0 *__forwarding;
 int __flags;
 int __size;
 int value;
};

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __Block_byref_value_0 *value; // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_value_0 *_value, int flags=0) : value(_value->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  __Block_byref_value_0 *value = __cself->value; // bound by ref
    (value->__forwarding->value) += 1;
}
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->value, (void*)src->value, 8/*BLOCK_FIELD_IS_BYREF*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {
_Block_object_dispose((void*)src->value, 8/*BLOCK_FIELD_IS_BYREF*/);
}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};

int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        __attribute__((__blocks__(byref))) __Block_byref_value_0 value = {(void*)0,(__Block_byref_value_0 *)&value, 0, sizeof(__Block_byref_value_0), 10};
        void(*blo)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_value_0 *)&value, 570425344));
        ((void (*)(__block_impl *))((__block_impl *)blo)->FuncPtr)((__block_impl *)blo);
    }
    return 0;
}

```
这次代码明显变多了好多。增加了 \_\_Block\_byref\_value\_0 、\_\_main\_block\_copy\_0 、 \_\_main\_block\_dispose\_0 这几个实现。

我们首先看 \_\_Block\_byref\_value\_0 这个结构体的内部结构。我们主要看 \_\_forwarding 和 value 这两个属性。在 main 函数中，我没看到首先他声明了一个 \_\_Block\_byref\_value\_0 这样的类型的结构体，并且初始化。而 \_\_forwarding 对应的我们看到是一个对自己的引用（c++中&有引用的意思，可以理解为别名，对别名的操作是直接对原对象的操作），value 对应的就是我们原来 main 函数的 10。接着看 \_\_main_block\_impl\_0 的实现，同样调用初始化函数指定 block 的调用函数，特别注意的是这边由上面一个例子中的直接传值，变成了传递 \_\_Block\_byref\_value\_0 这个结构体实例的引用。最后调用 blo 中的 FuncPtr 函数，调用参数是 blo 本身。在FuncPrt 指向的函数中，他取到了 blo 中 \_\_Block\_byref\_value\_0 类型的 value 值。然后对这个 value 的 \_\_forwarding 属性(其实也就是他自己)进行 +1 的操作。

回顾刚刚的3个版本，从没有外部访问，到对外部变量的访问，最后到修改外部变量。其实我们最后是用了指针来包裹变量来实现了作用域的扩大。这其实和我们最开始讨论的日常开发场景是类似的。这也是我觉得 block 中的思想。 

这里对 value 的操作，并不是直接操作。而是通过对属性 \_\_forwarding 的引用（其实是对自己的），再去访问 value 这个值。为什么要这样呢？我们知道上面的 block 都是分配在栈上的。我们有的时候会将它 copy 到堆上。如下代码

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...
        __block int value = 10;
        void(^blo)(void) = ^() {
            value += 1;
        };
        blo();
        void(^clo)(void) = [blo copy];
        clo();
    }
    return 0;
}
```

我们对 blo做了一个 copy 操作，接着用 Clang 改写

```
struct __Block_byref_value_0 {
  void *__isa;
__Block_byref_value_0 *__forwarding;
 int __flags;
 int __size;
 int value;
};

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __Block_byref_value_0 *value; // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_value_0 *_value, int flags=0) : value(_value->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  __Block_byref_value_0 *value = __cself->value; // bound by ref
	(value->__forwarding->value) += 1;
}
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->value, (void*)src->value, 8/*BLOCK_FIELD_IS_BYREF*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->value, 8/*BLOCK_FIELD_IS_BYREF*/);}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        __attribute__((__blocks__(byref))) __Block_byref_value_0 value = {(void*)0,(__Block_byref_value_0 *)&value, 0, sizeof(__Block_byref_value_0), 10};
        void(*blo)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_value_0 *)&value, 570425344));
        ((void (*)(__block_impl *))((__block_impl *)blo)->FuncPtr)((__block_impl *)blo);
        void(*clo)(void) = (void (*)())((id (*)(id, SEL))(void *)objc_msgSend)((id)blo, sel_registerName("copy"));
        ((void (*)(__block_impl *))((__block_impl *)clo)->FuncPtr)((__block_impl *)clo);
    }
    return 0;
}
```

main 代码中，我们看到 objc\_msgSend((id)blo,sel_registerName("copy")) ，这是调用了 blo 中的注册过的 copy 方法。这里其实我找了一下最后在 libclosure的 runtime.c 中找到了对应的说明<sub>3</sub>。调用 block 的 copy 方法的时候，最后调用了 \_Block\_copy 方法

```
// Copy, or bump refcount, of a block.  If really copying, call the copy helper if present.
void *_Block_copy(const void *arg) {
    struct Block_layout *aBlock;
    if (!arg) return NULL;
    // The following would be better done as a switch statement
    aBlock = (struct Block_layout *)arg;
    if (aBlock->flags & BLOCK_NEEDS_FREE) {
        // latches on high
        latching_incr_int(&aBlock->flags);
        return aBlock;
    }
    else if (aBlock->flags & BLOCK_IS_GLOBAL) {
        return aBlock;
    }
    else {
        // Its a stack block.  Make a copy.
        struct Block_layout *result = malloc(aBlock->descriptor->size);
        if (!result) return NULL;
        memmove(result, aBlock, aBlock->descriptor->size); // bitcopy first
        // reset refcount
        result->flags &= ~(BLOCK_REFCOUNT_MASK|BLOCK_DEALLOCATING);    // XXX not needed
        result->flags |= BLOCK_NEEDS_FREE | 2;  // logical refcount 1
        _Block_call_copy_helper(result, aBlock);
        // Set isa last so memory analysis tools see a fully-initialized object.
        result->isa = _NSConcreteMallocBlock;
        return result;
    }
}

static void _Block_call_copy_helper(void *result, struct Block_layout *aBlock)
{
    struct Block_descriptor_2 *desc = _Block_descriptor_2(aBlock);
    if (!desc) return;

    (*desc->copy)(result, aBlock); // do fixup
}

```
我们进入的最后一个 else 中分配内存空间并做一些初始化工作。之后调用了 \_Block\_call\_copy\_helper 这个函数。在这个函数里，我们看到了调用了 desc 的 copy 方法。回到上面的定义中

我们在 blo 中有个\_\_main\_block\_desc\_0类型的属性 Desc。他是在 blo 初始化的时候，是由传入的 \_\_main\_block\_desc\_0\_DATA 静态变量指定的。而这个结构体有两个方法 copy 和 dispose。我们看一下 copy 的实现。它也是对应了一个静态方法 \_\_main\_block\_copy\_0 ,这个方法实现是

```
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {
	_Block_object_assign((void*)&dst->value, (void*)src->value, 8/*BLOCK_FIELD_IS_BYREF*/);
}
```

这里又出现了一个新的操作 block 的方法。最后我在 libclosure 中找到了他的方法实现  

```
struct __Block_byref_value_0 {
  void *__isa;
__Block_byref_value_0 *__forwarding;
 int __flags;
 int __size;
 int value;
};
这里的 destArg 和 object 是 __Block_byref_value_0 结构体类型
void _Block_object_assign(void *destArg, const void *object, const int flags) {
    const void **dest = (const void **)destArg;
    switch (os_assumes(flags & BLOCK_ALL_COPY_DISPOSE_FLAGS)) {
      ...
      case BLOCK_FIELD_IS_BYREF | BLOCK_FIELD_IS_WEAK:
      case BLOCK_FIELD_IS_BYREF:
        /*******
         // copy the onstack __block container to the heap
         // Note this __weak is old GC-weak/MRC-unretained.
         // ARC-style __weak is handled by the copy helper directly.
         __block ... x;
         __weak __block ... x;
         [^{ x; } copy];
         ********/

        *dest = _Block_byref_copy(object);
        break;
        ...
    }
}


static struct Block_byref *_Block_byref_copy(const void *arg) {
    struct Block_byref *src = (struct Block_byref *)arg;
	这个 Block_byref 结构体和上面 __Block_byref_value_0 结构体内部结构基本一致 少了一个 value 属性
    if ((src->forwarding->flags & BLOCK_REFCOUNT_MASK) == 0) {
        // src points to stack
        struct Block_byref *copy = (struct Block_byref *)malloc(src->size);
        copy->isa = NULL;
        // byref value 4 is logical refcount of 2: one for caller, one for stack
        copy->flags = src->flags | BLOCK_BYREF_NEEDS_FREE | 4;
        copy->forwarding = copy; // patch heap copy to point to itself
        src->forwarding = copy;  // patch stack to point to heap copy
        copy->size = src->size;

        if (src->flags & BLOCK_BYREF_HAS_COPY_DISPOSE) {
            // Trust copy helper to copy everything of interest
            // If more than one field shows up in a byref block this is wrong XXX
            struct Block_byref_2 *src2 = (struct Block_byref_2 *)(src+1);
            struct Block_byref_2 *copy2 = (struct Block_byref_2 *)(copy+1);
            copy2->byref_keep = src2->byref_keep;
            copy2->byref_destroy = src2->byref_destroy;

            if (src->flags & BLOCK_BYREF_LAYOUT_EXTENDED) {
                struct Block_byref_3 *src3 = (struct Block_byref_3 *)(src2+1);
                struct Block_byref_3 *copy3 = (struct Block_byref_3*)(copy2+1);
                copy3->layout = src3->layout;
            }

            (*src2->byref_keep)(copy, src);
        }
        else {
            // Bitwise copy.
            // This copy includes Block_byref_3, if any.
            memmove(copy+1, src+1, src->size - sizeof(*src));
        }
    }
    // already copied to heap
    else if ((src->forwarding->flags & BLOCK_BYREF_NEEDS_FREE) == BLOCK_BYREF_NEEDS_FREE) {
        latching_incr_int(&src->forwarding->flags);
    }
    
    return src->forwarding;
}


```

最后我们看到了 block 的 isa 由原先的 Stack 被修改成了 Malloc 。

这里也就解释为什么是 forwarding->value 而不是直接访问 value。因为有从 stack 拷贝到 heap 上来的这个操作，指针变化了。


下面我们就开始来讲一个纯函数式的语言 Haskell ...






[1]:[http://fuckingblocksyntax.com/]
[2]:[http://www.nagain.com/activity/article/4/]
[3]:[http://www.galloway.me.uk/2013/05/a-look-inside-blocks-episode-3-block-copy/]


