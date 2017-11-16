---
title: React Native 笔记 一
date: 2017-11-15 15:56:31
categories: 源码阅读
tags: 源码阅读
---

# RN 源码跟踪

前一阵子做过 React Native 相关的项目，和原来同项目的安卓同事讨论遇到的问题和实现方式时，跟踪源码找解决方案，这里做个整理回顾。

<!--more-->

## 入口

由 RCTRootView 生成一个 rootView。这里遇到了一个重要的类 RCTBridge __TODO:这是一个什么设计模式的?__ ,他接受了刚刚创建 rootView 的 url 和 launchOptions 两个参数。

对于 这个 Bridge 最后调用了初始化函数

```
- (instancetype)initWithDelegate:(id<RCTBridgeDelegate>)delegate
                       bundleURL:(NSURL *)bundleURL
                  moduleProvider:(RCTBridgeModuleProviderBlock)block
                   launchOptions:(NSDictionary *)launchOptions
{
  if (self = [super init]) {
    _delegate = delegate;
    _bundleURL = bundleURL;
    _moduleProvider = block;
    _launchOptions = [launchOptions copy];

    [self setUp];
    //reload key 'R'
    RCTExecuteOnMainQueue(^{ [self bindKeys]; });
  }
  return self;
}
```
对于 RCTExecuteOnMainQueue 这个是在主线程中，当运行设备是模拟器的时候，绑定 ‘R’ 键作为 reload 的作用。__TODO:这个可以再细致一点__

我们主要看下 `[self setUp];` 这个方法。有一些基础的类的创建和 bundle 的相关处理。然后这里我们看到它又创建了一个 BatchedBridge。
 
```
[self createBatchedBridge];
[self.batchedBridge start];
```

在 createBatchedBridge 函数中，它有创建了一个 RCTBatchedBridge 类型的属性变量 batchedBridge ，并且弱引用这自己。这个 RCTBatchedBridge 是 RCTBridge 的子类。在他的初始化方法中，我们看到了一个比较感兴趣的东西 RCTDisplayLink 这个类，还维护了一个 保存 dispatch_block_t 的_pendingCalls 的数组。在创建并初始化完成之后，他就把这个 batch 设给了一个静态的变量。

在 RCTDisplayLink 中我们发现了 RunLoop CADisplayLink 这些和刷新或者线程状态有关的类。在做初始化的时候，也只是创建了一个集合和 CADisplayLink 的相关设定，根据 Sel 的名字 _jsThreadUpdate 可以猜测是一个在屏幕固定的刷新时机，去调用 js 的执行。 这个一会来看 ...

回到上面创建完 batched，那下面直接调用了 batchedBridge 的 start 函数。

```
- (void)start
{
  1.发送一个 RCTJavaScriptWillStartLoadingNotification 的消息给 root 的 bridge。
  2.创建一个并行的队列 com.facebook.react.RCTBridgeQueue，然后异步的去加载源代码。
  3.这里我们看到 -loadSource:onProgress 函数中，出现了新的类 RCTJavaScriptLoader ，看名字猜意思，大概作用就是加载 js 吧。看到里面做了一些检查最后把 bundle 指向的文件转成 data 了,赋值给了 sourceCode。
  4.然后接着就同步的去初始化所有的不能懒加载的 native 模块。此时的 delegate 是 nil、modelProvider 也是 nil。
  5.在bridgeQueue中异步初始化 JS executor 统一建立一个 group
  6.异步对模块进行配置
  7.最后上面的做完，开始执行 sourceCode
}
```

### Step 4

这里在第4步去初始化本地代码的时候, 此时的 delegate 是 nil、modelProvider 也是 nil。所以也就跳过了上面的循环函数。去初始化 \_javaScriptExecutor ，它是一个 RCTJSCExecutor 的类。扫了一下这个类的属性和引用，发现它引用了 JSCore、包含了一个线程变量\_javaScriptThread、一个RCTJavaScriptContext 变量 \_context，看作者对类的描述 使用了一个 JavaScriptCore 的上下文来作为执行的引擎。看了一下这个js线程的优先级适合主线程一样的，用添加源的方式启动了该线程的 runloop。

准备了3个容器来装载

```
  NSMutableArray<Class> *moduleClassesByID = [NSMutableArray new];
  NSMutableArray<RCTModuleData *> *moduleDataByID = [NSMutableArray new];
  NSMutableDictionary<NSString *, RCTModuleData *> *moduleDataByName = [NSMutableDictionary new];

```

上面只有 \_javaScriptExecutor 被添加进去。

然后就对注册的 module 进行添加操作

```
for (Class moduleClass in RCTGetModuleClasses()) {
    NSString *moduleName = RCTBridgeModuleNameForClass(moduleClass);
    // Check for module name collisions
    RCTModuleData *moduleData = moduleDataByName[moduleName];
    ...

    // Instantiate moduleData (TODO: can we defer this until config generation?)
    moduleData = [[RCTModuleData alloc] initWithModuleClass:moduleClass
                                                     bridge:self];
    moduleDataByName[moduleName] = moduleData;
    [moduleClassesByID addObject:moduleClass];
    [moduleDataByID addObject:moduleData];
  }
```

这里 RCTGetModuleClasses 操作的是一个静态数组，它对 标志了 RCT_EXPORT_MODULE 这个宏的类在 load 的时候，添加到静态数组中。

```
#define RCT_EXPORT_MODULE(js_name) \
RCT_EXTERN void RCTRegisterModule(Class); \
+ (NSString *)moduleName { return @#js_name; } \
+ (void)load { RCTRegisterModule(self); }

void RCTRegisterModule(Class moduleClass);
void RCTRegisterModule(Class moduleClass)
{
  ...
   // Register module
  [RCTModuleClasses addObject:moduleClass];
}

```

这里来看一下封装了 moduleClass 和 bridge 引用的一个类 RCTModuleData。这个类提供两个初始化方法分别对应实例参数的初始化和类参数的初始化。在两个初始化方法中都调用了 setUp 方法。用来判断是否重写了 init 和 constantsToExport 方法。如果重写了就意味着可能需要在主线程中去初始化这个实例，因为他可能用到了 UIKit 相关的东西。


回到👆上面，在存储了模块之后，去做一些预初始化的事情

```
for (RCTModuleData *moduleData in _moduleDataByID) {
	这里的判断条件 就是刚刚在 setUp 中提供的
    if (moduleData.hasInstance &&
        (!moduleData.requiresMainQueueSetup || RCTIsMainQueue())) {
            (void)[moduleData instance];
    }
  }
  
...

[self setUpInstanceAndBridge]; 
对刚刚通过类参数创建 RCTModuleData 的实例进行一系列的判断，创建一个类参数的实例

RCTProfileHookInstance(_instance);
这个函数，目前还不会进入。但是函数中用运行时创建了一个 _instance 所属类的子类。并替换了 _instance 的类类型。__TODO__:这个先放一下

[self setBridgeForInstance];
最后用 KVC 设定了 _instance 对 bridge 的引用。

[self prepareModulesWithDispatchGroup:dispatchGroup];
在主线程中执行有需要的类实例。
```

### Step 5

再回到 -start 函数的第5点，执行了 [weakSelf setUpExecutor]; 这里的 _javaScriptExecutor 他是 RCTJSCExecutor 类型的。接着调用了他的 setUp 方法，主要是创建了 JS 的执行环境 context。没有用custom的 JSLib，所以最后调用了 `facebook::react::systemJSCWrapper() `。然后在 context 上注册了一些常用的方法。

```
NSMutableDictionary *threadDictionary = [[NSThread currentThread] threadDictionary];
if (!threadDictionary[RCTFBJSContextClassKey] || !threadDictionary[RCTFBJSValueClassKey]) {
      threadDictionary[RCTFBJSContextClassKey] = JSC_JSContext(contextRef);
      threadDictionary[RCTFBJSValueClassKey] = JSC_JSValue(contextRef);
    }
```
 
把 JSContext 的存在线程字典中。

<div align=center>
<img src="./数据走向.png" width = "1015" height = "586" alt="数据走向" align=center/>
</div>


待补充...

















