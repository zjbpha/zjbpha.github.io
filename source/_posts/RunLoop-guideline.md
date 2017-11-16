---
title: RunLoop Guideline 翻译
tags: 翻译
date: 2017-09-03 17:10:12
categories: 翻译
---



网上已经有很多总结，自己读一遍再理解比较，查看疏漏。

<!--more-->

Threading Programming Guide[^guideline]

# Run Loops

run loop 接收两种不同类型的事件。输入源 派发异步的事件，通常是来自于其他线程或者其他程序的消息。Timer sources 转发同步的事件，以计时器或者间隔重复触发的。这两种事件源通过特定 (application-specific) 的方式处理接受到的消息。

## Run Loop 解析

run loop 就像他的名字一样。这是一个循环你线程的入口，用于启动到来的事件的响应函数。你的代码提供了状态控制用于实现 run loop 的当前循环部分，换句话说，你的代码提供了一个 while 或者 for 的循环去驱动这个 run loop。在你的循环里，你用一个 run loop 对象去“启动”这个事件处理的代码，接受事件并且调用装载好的操作。

run loop 接受来自两种不同的源的事件。输入源派送异步事件，通常是来自其他线程或者其他程序的消息。 timer 源派同步事件，由定时或者间隔时间发生的。这两种源通过特定的例程去处理到来的事件。

3-1 展示了 run loop 和 一系列源的概念结构。输入源派发异步事件到相应的处理函数然后通过 runUntilDate：函数（有线程相关的 NSRunLoop 对象调用）退出。定时器源派发到处理例程的事件并不会导致 run loop 退出。

__3-1__ Structure of a run loop and its sources    
![runloop和源的结构](runloop结构.jpg)

除了接管 输入源之外，run loop 也产生一些关于他行为的通知。注册过的 run-loop 观察者可以接受到这些通知，并在线程里可以做一些其他的处理。你可以用 Core Foundation 去安装 run-loop 观察者到你的线程中。

下面展示了更多关于一个 run loop 的组件信息和他们执行时所在的模式的信息。也阐述了在不同处理消息时机产生的通知。


## Run Loop Modes

run loop mode 是包括一系列被监控的 输入源、定时器源和一系列将会被通知的 run loop 观察者的集合。每次你启动你的 run loop，你必须指定（不论直接或间接）一个特地的 'mode' 去运行。在这个 run loop 期间，只有和这个 mode 相关联的源才会被监控并且被允许去派发他们的事件。（同样的，观察者也是同样的）和其他 mode 相关的源将会暂存他们的新事件，直到轮到运行在与之相关的模式下。

在你的代码中，你需要用名字来标识你的 mode。在 Cocoa 和 Core Foundation 中都定义了一些常用默认的模式，都被标识了特定的名称。你可以简单的用自定义 mode 名字的方式定义一个 mode。虽然名字起的比较随意，但是 mode 中的内容不行。你必须添加一个或更多的 intput 源、定时器或者观察者到你的模式中。

<!--你用 mode 可以过滤掉你不需要源的信息。大多数情况下，你期望你的 run loop 在系统定义的默认 mode 下运行。一个模态是的样式，然而，可能会运行在一个“模态”的 mode 下。在这个-->

模式是以消息的来源区分的，而不是消息的类型。举个例子，你不可以用 mode 去匹配是鼠标点击的消息，还是键盘的消息。你可以用 mode 去监听不同的一组端口、临时挂起的定时器，或者反过来修改当前监控的源和观察者。


__3-1__  Predefined run loop modes 

|Mode|Name|Description|
|:--:|:-------|:--------------------|
|Default|NSDefaultRunLoopMode (Cocoa) <br>kCFRunLoopDefaultMode (Core Foundation)|The default mode is the one used for most operations. Most of the time, you should use this mode to start your run loop and configure your input sources.|
|Connection|NSConnectionReplyMode (Cocoa)|Cocoa uses this mode in conjunction with NSConnection objects to monitor replies. You should rarely need to use this mode yourself.|
| Modal |NSModalPanelRunLoopMode (Cocoa)|Cocoa uses this mode to identify events intended for modal panels|
|Event tracking|NSEventTrackingRunLoopMode (Cocoa)|Cocoa uses this mode to restrict incoming events during mouse-dragging loops and other sorts of user interface tracking loops|
| Common modes |NSRunLoopCommonModes (Cocoa)<br>kCFRunLoopCommonModes (Core Foundation)|This is a configurable group of commonly used modes. Associating an input source with this mode also associates it with each of the modes in the group. For Cocoa applications, this set includes the default, modal, and event tracking modes by default. Core Foundation includes just the default mode initially. You can add custom modes to the set using the CFRunLoopAddCommonMode function.|


## 输入源

输入源同步派发时间到你的线程。有两种 输入源会发送消息过来。一种是基于端口的监控你程序的 Mach 端口。自定义源监视你的自定义源事件。一旦你关注到你的 run loop了，你必须了解他的 输入源到底是基于端口的还是自定义的。两种方式系统都做了定义你可以直接使用。他们之间的不同在于他们是如何被触发的。基于端口的源是被内核自动出发，自定义的源是被其他线程触发。

当你创建一个 输入源时，你必须给他指定一个或多个运行的 mode。这回影响到哪些 输入源将会在特定的时间被监控。大多时候你的 run loop 是处于默认 mode 下，当然你也可以指定到自定义。如果 输入源不在当前的监控 mode 下，他所产生的所有事件将会保留到下个所属的 mode 下。

### 基于端口的源

Cocoa 和 Core Foundation 提供了创建基于端口 输入源的相关对象和方法。例如，在 Cocoa 中，你不需要直接创建一个 输入源。你只要简单的创建一个端口对象，然后用 NSPort 的方法把他添加到 run loop 中。端口对象接管了你所需要的对 输入源的创建和配置。

在 Core Foundation 中，你必须手动创建端口和他的 run loop 源。在这两个步骤中你需要使用相关的（CFMac和PortRef，CFMessagePortRef，CFSocketRef）端口类型去创建关联对象。

### 自定义源

创建自定义源，你必须用 Core Foundation 框架中的 CFRunLoopSourceRef 类型。使用一些回调方法来配置自定义源。Core Foundation 会在不同的时期调用这些方法去配置源、处理将要到来的消息，并写在这个源在他从 run loop 中移除的时候。

除了定义消息到来时的处理行为之外，你必须定义好消息分发的机制。源的这一部分在一个独立的线程中，负责提供 输入源的数据和当数据准备好时触发处理流程。消息分发机制取决于你但是不需要过分复杂。

### Cocoa Perform Selector 源

除了基于端口的源， Cocoa 定义了自定义的 输入源，允许你在任意线程中执行一个 selector。和基于端口的源一样，执行 selector 的请求必须在目标线程中序列化过，缓解一些可能导致很多方法在同一个线程同步的问题。和基于端口源不一样的是，在 selector 执行完之后，他就自动从 run loop 中移除。

当在其他线程中执行 selector 时候，目标线程必须激活 run loop。对于你创建的线程，这意味着明确的启动了 run loop。由于主线程的 run loop 是自己启动的，不管怎样，你可以直接在程序调用完  applicationDidFinishLaunching 之后，在该线程调用。run loop 在一次循环中处理队列中所有的 selector，而不是每一次循环处理一个。


__3-2__  Performing selectors on other threads

|Methods|Description|
|:--:|:--:|
|performSelectorOnMainThread:withObject:waitUntilDone:<br>performSelectorOnMainThread:withObject:waitUntilDone:modes:|Performs the specified selector on the application’s main thread during that thread’s next run loop cycle. These methods give you the option of blocking the current thread until the selector is performed.|
|performSelector:onThread:withObject:waitUntilDone:<br>performSelector:onThread:withObject:waitUntilDone:modes:|Performs the specified selector on any thread for which you have an NSThread object. These methods give you the option of blocking the current thread until the selector is performed|
|performSelector:withObject:afterDelay:<br>performSelector:withObject:afterDelay:inModes:|Performs the specified selector on the current thread during the next run loop cycle and after an optional delay period. Because it waits until the next run loop cycle to perform the selector, these methods provide an automatic mini delay from the currently executing code. Multiple queued selectors are performed one after another in the order they were queued.|
|cancelPreviousPerformRequestsWithTarget:<br>cancelPreviousPerformRequestsWithTarget:selector:object:|Lets you cancel a message sent to the current thread using the performSelector:withObject:afterDelay: or performSelector:withObject:afterDelay:inModes: method.|



## Timer Source

定时器源在未来特定的时间里同步派发事件到你的线程中。定时器是一种线程提醒自己做一些任务的方式。     
尽管他产生的是基于定时器的通知，但是定时器也不是一个实时机制，像 输入源，定时器和你选择的运行 mode 息息相关。如果定时器不在当前的模式的监控中，他不会起作用，直到你的 run loop 在支持定时器的 mode 中。如果定时器触发的时机，在 run loop 调用定时器处理例程执行到一半的时候，那么该定时器会轮到下一次循环的时候才会调用到。如果 run loop 根本没有启动，那么定时器不会被触发。

你可以把定时器配置成触发一次或者某个时间间隔触发。重复触发的定时器自动按自己第一次应该触发的时间安排自己，而不是真实的触发时间。比如一个定时器被安排在一个特定的时间触发并且在之后的每隔5秒触发一次，那么这个安排的触发时间将会每一次都在原来计划的时间上每隔5秒调用一次，哪怕真实的触发时间晚了。如果出发时间晚了很多导致错过了一些触发定时器的点，那么定时器只会触发一次去弥补丢失的那段时间。之后，定时器会重新为下一次触发时间在安排。

## Observers 

相比较被相关同步或者一部的消息发生的触发的事件源,run loop 的观察者在run loop执行的特定时机被出发。你可以用观察者去 __准备__ 你的线程去处理一个事件或者在线程将要休眠之前去 __准备__ 一些东西。你可以把观察者和下面几个事件相关联  
 
	- 进入run loop          
	- run loop 准备处理一个timer     
	- run loop 准备处理一个 输入事件源    
	- run loop 准备休眠    
	- run loop 被唤醒，但是没有处理唤醒他的事件     
	- run loop 退出      


## Run Loop 事件的顺序

1. 通知观察者进入 run loop 了
2. 通知观察者 any ready timer 将要被触发了
3. 通知观察者 不是基于 port 的 输入源事件 将被触发
4. 触发 非基于 port 的事件
5. 如果有基于 port 的事件准备好等待出发，立刻处理，Go to 步骤9
6. 通知观察者 线程将要休眠了
7. 将线程休眠直到有如下事件发生
	- 基于 port 的事件源到达
	- 定时器触发了
	- run loop 超时了
	- run loop 被唤醒了
8. 通知观察者线程刚被唤醒了
9. 处理没有判决的事件
	- 如果用户定义的定时器触发了，处理定时器事件接着重启 loop，Go to 步骤2
	- 如果 输入事件源触发了，分发事件
	- 如果 run loop 被唤醒了，但是还没有超时，重启 loop，Go to 步骤2
10. 通知观察者 run loop 退出了



因为上面 2 和 3 观察者得到的通知时机早于这些时机真实发生的时候，这里可能存在一个时间上的间隔。如果事件对于时间处理特别敏感，你可以用 loop 进入睡眠和从睡眠中被唤醒的通知来帮助你关联事件之间的时间间隔。


## 你合适需要使用 Run Loop

- 用 port 或者 自定义的 输入事件源 和 其他线程通信
- 在这个线程上使用timer计时器
- 在Cocoa的程序中使用类似 performSelector... 这样的方法
- 让线程保活处理周期性的任务

## 使用 Run Loop 对象

### 获得 Run Loop 对象

- 在 Cocoa 的程序中 用 `currentRunLoop` 这个类方法
- 用 `CFRunLoopGetCurrent` 方法

尽管他们不是 toll-free bridge 的类型，你依然可以从 NSRunLoop 的对象中获得 CFRunLoopRef。因为他们指向的是同一个 run loop。

### 配置 Run Loop

因为你在次要的线程中起了 run loop。所以你必须添加至少一个 输入源 或者 timer 到 run loop 里面去。用 `CFRunLoopObserverRef` 方法创建一个观察者，同事用 `CFRunLoopAddObserver`方法添加到你的 run loop 当中。观察者必须是用 Core Foundation 创建的，哪怕是 Cocoa 的程序。

当为有长时间保活需求的线程配置 run loop 的时候，比较好的是添加至少一个 输入源去接受消息。尽管你也可以用一个timer定时器添加到 run loop 中，一旦定时器被触发过了，定时器就失效了，这个就会导致 run loop 的退出。添加一个重复触发的定时器可以保证 run loop 在一个长时间的周期中，但是这样会周期性的调用触发的定时器去唤醒线程。相比之下，放置一个 输入的事件源保持线程休眠直到事件到达，这种做法是比较好的。


### 启动 Run Loop

有多种办法可以启动 run loop ，包括如下方式
1. 无条件的
2. 一个时间限制
3. 一个特地的模式

无条件的是一个最简单的方案，但是也是描述的最少的。如果你是用这种方式的话，那么你就把你的线程放到了一个无限循环的 loop 中了，你仅仅有少些的控制 loop 的能力。你可以添加或者移除 输入源和定时器，但是唯一可以停止他的办法就是杀死他。当然 run loop 也不能运行在自定义的模式中。

运行在一定有效时间内比无条件运行稍微好一点。这种方式下，run loop 在事件到达或者有效时间过了之后会退出。事件到达时，事件会被派发下去给 handler 处理接着 run loop 退出。你也可以重启 run loop 处理下面的事件。当有效时间到达的时候，你可以简单的重启一下 run loop。

最后一种特定的模式。就是在上一种方式的基础上添加上运行模式。


### 退出 Run Loop

有两种方法可以使 run loop 在处理过消息之后退出

- 在配置 run loop 的时候添加超时时间
- 给 run loop 下发停止命令

如果你可以控制 run loop ，那么设置超时时间是一种比较好的方法。设置一个超时时间可以让 run loop 结束他的正常处理流程，包括在退出之前派发消息给观察者。

使用 CFRunLoopStop 函数停止 run loop 和设置超时时间的效果相似。run loop 发送出剩下的消息然后退出。不同点在于你可以用这种方式结束那些用无条件开启的 run loop。

尽管将 输入源和timer定时器从 run loop 中移除也会导致他退出，但是这并不是一个可靠的退出方式。有些系统例程会添加 输入源到 run loop 中去处理消息。你代码可能不知道这些 输入源，可能会没有移除他们，从而导致 run loop 没有退出。

### 线程安全 和 Run Loop 对象

线程安全取决于你使用什么样的 API 去操控线程的 run loop。Core Foundation中的方法通常是线程安全的，可以在任意线程里调用。如果你正在执行修改 run loop 配置的操作，无论怎么样，当在可能的情况下用拥有 run loop 的线程去这样做是比较好的。 

Cocoa 的 NSRunLoop 类并不是线程安全的。如果你使用 NSRunLoop 去修改你的 run loop，你必须在拥有这个 run loop 的线程中操作他。在其他线程中给本线程添加 输入源 或者 定时器会导致你的代码 crash 或者其他异常。

## 配置 Run Loop 源

### 定义一个自定义的 输入源

创建一个自定义源涉及到下面几个方面

- 你希望自定义源处理的信息
- 一个调度的例程好让其他端知道如何和你的 输入源通信
- 一个处理的例程去完成其他段发过来的请求
- 一个取消的例程用来让自定义源作废

因为你创建了一个自定义的 输入源来处理自定义的消息，配置操作非常灵活。调度例程、处理例程、取消例程就是比较关键的例程对于你自定义的 输入源。然而其余大部分 输入源的行为，不在这些处理例程的范围之内。像如何定义处理发过来的数据和如何和其他线程通信的机制是取决于你的。

下面展示了一个自定义 输入源的例子。在这个例子中，程序的主线程保留了一个 输入源(工作线程 run loop)的引用，一个自定义的命令缓冲区（为 输入源准备的）引用和一个安装了上面 输入源的 run loop 引用。当主线程有一个任务需要转发给工作线程的时候，主线程发送了一条命令到命令缓冲区并携带一些任务开始可能需要的信息。（因为主线程和工作线程同事拥有对命令缓冲区的操作权限，那么每次操作应该是同步的。）一旦命令发出了，主线程发送信号给 输入源同时唤醒工作线程的 run loop。接受到唤醒命令之后，工作线程的 run loop 调用 输入源的处理方法，去处理那些已经发送到命令缓冲区的指令。

<div align=center>
![3-2 Operating a custom input source](custominputsource.jpg)
</div>


### 定义 输入源

定义一个自定义的 输入源需要使用 Core Foundation 的例程去配置你的 run loop 源然后把他添加到你的 run loop 中。尽管一些基础的处理方法是基于C的函数，但是这个不影响你用Objective-C 或者 C++去包装这些方法到你的代码中。

上图 3-2 上的中使用了 Object-C 对象管理命令缓冲区并协助 run loop。下面 3-3 代码会展示这些对象的定义。 RunLoopSource 对象管理命令缓冲区并用以接受其他线程发过来的消息。下面代码也展示了 RunLoopContext 对象，一个包裹对象用来传递 RunLoopSource 对象并且提供run loop的引用给程序主线程。      


__3-3 自定义 输入源对象的定义__

```code
@interface RunLoopSource : NSObject
{
    CFRunLoopSourceRef runLoopSource;
    NSMutableArray* commands;
}
 
- (id)init;
- (void)addToCurrentRunLoop;
- (void)invalidate;
 
// Handler method
- (void)sourceFired;
 
// Client interface for registering commands to process
- (void)addCommand:(NSInteger)command withData:(id)data;
- (void)fireAllCommandsOnRunLoop:(CFRunLoopRef)runloop;
 
@end
 
// These are the CFRunLoopSourceRef callback functions.
void RunLoopSourceScheduleRoutine (void *info, CFRunLoopRef rl, CFStringRef mode);
void RunLoopSourcePerformRoutine (void *info);
void RunLoopSourceCancelRoutine (void *info, CFRunLoopRef rl, CFStringRef mode);
 
// RunLoopContext is a container object used during registration of the input source.
@interface RunLoopContext : NSObject
{
    CFRunLoopRef        runLoop;
    RunLoopSource*        source;
}
@property (readonly) CFRunLoopRef runLoop;
@property (readonly) RunLoopSource* source;
 
- (id)initWithSource:(RunLoopSource*)src andLoop:(CFRunLoopRef)loop;
@end

```


虽然 Object-C 代码已经处理了 输入源的的数据，把 输入源添加到 run loop 中需要基于 C 的回调方法。这些方法第一个回调的时机是当你把 run loop 源添加到你的 run loop 中，如下面 3-4 所示。因为这些 输入源只有一个客户（主线程），他用例程方法发送一个消息其中包含了程序的委托去注册。当委托想和 输入源通信的时候，他就使用 RunLoopContext 对象中的信息。       

__3-4__   Scheduling a run loop source    
 
 

```code
void RunLoopSourceScheduleRoutine (void *info, CFRunLoopRef rl, CFStringRef mode)
{
    RunLoopSource* obj = (RunLoopSource*)info;
    AppDelegate*   del = [AppDelegate sharedAppDelegate];
    RunLoopContext* theContext = [[RunLoopContext alloc] initWithSource:obj andLoop:rl];
 
    [del performSelectorOnMainThread:@selector(registerSource:)
                                withObject:theContext waitUntilDone:NO];
}
```

其中一个重要的回调例程是用来处理自定义消息，在你的 输入源收到信号的时候。3-5 的代码展示了用 RunLoopSource 处理的回调例程。这个方法简单的转发这个请求到 sourceFired 方法，去处理在命令缓存中出现的指令。

__3-5__ Performing work in the input source

```
void RunLoopSourcePerformRoutine (void *info)
{
    RunLoopSource*  obj = (RunLoopSource*)info;
    [obj sourceFired];
}
```

如果你每次把你的 输入源从他的 run loop 中用 CFRunLoopSourceInvalidate 方法移除，系统会调用你的 输入源的取消例程。你可以用这个例程去通知你的客户端，这个 输入源已经不再有效了，客户端需要移除 输入源的各种引用。在 3-6 中展示了取消例程，当初向 RunLoopSource 对象注册的。这个方法发送了另一个 RunLoopContext 对象到程序的委托，这一次是告诉委托去删除刚刚的引用。

```
void RunLoopSourceCancelRoutine (void *info, CFRunLoopRef rl, CFStringRef mode)
{
    RunLoopSource* obj = (RunLoopSource*)info;
    AppDelegate* del = [AppDelegate sharedAppDelegate];
    RunLoopContext* theContext = [[RunLoopContext alloc] initWithSource:obj andLoop:rl];
 
    [del performSelectorOnMainThread:@selector(removeSource:)
                                withObject:theContext waitUntilDone:YES];
}
```

#### 装载 输入源到 Run Loop 上

3-7 的代码段展示了 RunLoopSource 类中的 init 和 addToCurrentRunLoop 方法。init 方法创建的 CFRunLoopSourceRef 类型必须要添加到 run loop 中。他把 RunLoopSource 对象当做上下文这样回调例程才有一个指向对象的指针。Installation of the input source does not occur until the worker thread invokes the addToCurrentRunLoop method, at which point the RunLoopSourceScheduleRoutine callback function is called. Once the input source is added to the run loop, the thread can run its run loop to wait on it

__3-7__ Installing the run loop source

```
- (id)init
{
    CFRunLoopSourceContext    context = {0, self, NULL, NULL, NULL, NULL, NULL,
                                        &RunLoopSourceScheduleRoutine,
                                        RunLoopSourceCancelRoutine,
                                        RunLoopSourcePerformRoutine};
 
    runLoopSource = CFRunLoopSourceCreate(NULL, 0, &context);
    commands = [[NSMutableArray alloc] init];
 
    return self;
}
 
- (void)addToCurrentRunLoop
{
    CFRunLoopRef runLoop = CFRunLoopGetCurrent();
    CFRunLoopAddSource(runLoop, runLoopSource, kCFRunLoopDefaultMode);
}

```

#### 和客户端协调 输入源

你的 输入源可以用了，你需要操作他从其他线程发信号。关于 输入源最重要的一点是将源相关的线程休眠直到有什么事情需要他处理。这就必然在你的程序中有其他的线程知道这个 输入源并且可以通过一种方式和他通信。

一个把你的 输入源告知客户端的方式是发送注册请求，在你的 输入源第一次添加配置到他的 run loop 上的时候。你可以注册任意多的客户端，或者你可以简单的交给代理去做。 3-8 的代码展示了通过程序代理注册方法，当 RunLoopSource 对象的例程调用的时候执行。这个方法接收一个 RunLoopContext 的对象，然后把它添加到源队列中。这段代码也展示了当 输入源从他的 run loop 中移除的时候用来解除绑定的例程。

__3-8__ Registering and removing an 输入source with the application delegate

```
- (void)registerSource:(RunLoopContext*)sourceInfo;
{
    [sourcesToPing addObject:sourceInfo];
}
 
- (void)removeSource:(RunLoopContext*)sourceInfo
{
    id    objToRemove = nil;
 
    for (RunLoopContext* context in sourcesToPing)
    {
        if ([context isEqual:sourceInfo])
        {
            objToRemove = context;
            break;
        }
    }
 
    if (objToRemove)
        [sourcesToPing removeObject:objToRemove];
}
```

#### 通知 输入源

当客户端发完他的数据，他必须通知源同事唤醒源的 run loop。通知源让 run loop 知道源准备处理事情了。由于信号到来的时候，源的线程可能处于休眠状态，你必须把他唤醒。如果没有这么做可能会导致处理上的延迟。

__3-9__  Waking up the run loop

```
- (void)fireCommandsOnRunLoop:(CFRunLoopRef)runloop
{
    CFRunLoopSourceSignal(runLoopSource);
    CFRunLoopWakeUp(runloop);
}
```


### 配置定时器源

为了创建一个定时器源，你只要创建一个定时器对象并且把它添加到你的 run loop中去。在 Cocoa 中，你可以使用 NSTimer 类去创建新的定时器对象，在 Core Foundation 中 你可以使用 CFRunLoopTimerRef 。NSTimer 是 Core Foundation 的一个扩展并提供了一些便捷的特性。

在 Cocoa 中，你可以创建并安排一个定时器通过下面任意一种方法

- scheduledTimerWithTimeInterval:target:selector:userInfo:repeats:
- scheduledTimerWithTimeInterval:invocation:repeats:

这些方法创建的定时器并添加到当前线程的 run loop 的 default mode (NSDefaultRunLoopMode) 中。你可也以通过 addTimer:forMode: 手动把一个定时器添加到 run loop 中。这两个是一回事，只是给了你配置 timer 的不同的控制权限。

__3-10__  Creating and scheduling timers using NSTimer

```
NSRunLoop* myRunLoop = [NSRunLoop currentRunLoop];
 
// Create and schedule the first timer.
NSDate* futureDate = [NSDate dateWithTimeIntervalSinceNow:1.0];
NSTimer* myTimer = [[NSTimer alloc] initWithFireDate:futureDate
                        interval:0.1
                        target:self
                        selector:@selector(myDoFireTimer1:)
                        userInfo:nil
                        repeats:YES];
[myRunLoop addTimer:myTimer forMode:NSDefaultRunLoopMode];
 
// Create and schedule the second timer.
[NSTimer scheduledTimerWithTimeInterval:0.2
                        target:self
                        selector:@selector(myDoFireTimer2:)
                        userInfo:nil
                        repeats:YES];

```

3-11 展示了用 Core Foundation 配置一个定时器。

__3-11__ Creating and scheduling a timer using Core Foundation

```
CFRunLoopRef runLoop = CFRunLoopGetCurrent();
CFRunLoopTimerContext context = {0, NULL, NULL, NULL, NULL};
CFRunLoopTimerRef timer = CFRunLoopTimerCreate(kCFAllocatorDefault, 0.1, 0.3, 0, 0,
                                        &myCFTimerCallback, &context);
 
CFRunLoopAddTimer(runLoop, timer, kCFRunLoopCommonModes);

```


### 配置一个机遇端口的 输入源

在 Cocoa 和 Core Foundation 中都提供了机遇端口的对象，用于线程或者进程之间的通信。下面就展示怎么样去建立不同端口的通信方式。

#### 配置一个 NSMachPort 对象

为了建立一个和 NSMachPort 对象的本地链接，你要创建一个端口对象，把他添加到你的主线程的 run loop 中。当你启动你的次要线程的时候，你把刚刚这个对象发送给线程的入口函数。次要线程就可以使用同样的对象去发回消息到你的主线程。


__实现主线程代码__

3-12 展示了主线程为了启动次要工作线程的代码。因为 Cocoa的框架做了很多中间步骤，所以 launchThread 方法明显比 Core Foundation 中要短。但是效果差不多，一个区别是一个是发送的端口的名字，一个是直接发送端口对象。


__3-12__  Main thread launch method  

```
- (void)launchThread
{
    NSPort* myPort = [NSMachPort port];
    if (myPort)
    {
        // This class handles incoming port messages.
        [myPort setDelegate:self];
 
        // Install the port as an input source on the current run loop.
        [[NSRunLoop currentRunLoop] addPort:myPort forMode:NSDefaultRunLoopMode];
 
        // Detach the thread. Let the worker release the port.
        [NSThread detachNewThreadSelector:@selector(LaunchThreadWithPort:)
               toTarget:[MyWorkerClass class] withObject:myPort];
    }
}
```

为了在线程间建立两条通信通道，你可能希望工作线程把他的端口发送给主线程表示通过回调的消息。收到回调的消息让你的主线程知道一切正常，同时也增加了一条消息的通道


__3-13__  Handling Mach port messages


```
#define kCheckinMessage 100
 
// Handle responses from the worker thread.
- (void)handlePortMessage:(NSPortMessage *)portMessage
{
    unsigned int message = [portMessage msgid];
    NSPort* distantPort = nil;
 
    if (message == kCheckinMessage)
    {
        // Get the worker thread’s communications port.
        distantPort = [portMessage sendPort];
 
        // Retain and save the worker port for later use.
        [self storeDistantPort:distantPort];
    }
    else
    {
        // Handle other messages.
    }
}

```

__实现次要线程的代码__

对于次要线程，你必须配置他并使用收到的端口用来和主线程通信。   
3-14 展示了配置工作线程。在创建完自动释放池之后，该方法创建了一个工作对象去驱动线程的执行。


__3-14__  Launching the worker thread using Mach ports

```
+(void)LaunchThreadWithPort:(id)inData
{
    NSAutoreleasePool*  pool = [[NSAutoreleasePool alloc] init];
 
    // Set up the connection between this thread and the main thread.
    NSPort* distantPort = (NSPort*)inData;
 
    MyWorkerClass*  workerObj = [[self alloc] init];
    [workerObj sendCheckinMessage:distantPort];
    [distantPort release];
 
    // Let the run loop process things.
    do
    {
        [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode
                            beforeDate:[NSDate distantFuture]];
    }
    while (![workerObj shouldExit]);
 
    [workerObj release];
    [pool release];
}
```

当使用 NSMachPort 时，本地和远端的线程可以使用同一个端口对象来单路的通信。也就是，其中一个线程的本地端口对象也成为了另一个线程的远端端口对象。     
3-15 展示了次要线程的收到返回消息的例程。

__3-15__  Sending the check-in message using Mach ports

```
// Worker thread check-in method
- (void)sendCheckinMessage:(NSPort*)outPort
{
    // Retain and save the remote port for future use.
    [self setRemotePort:outPort];
 
    // Create and configure the worker thread port.
    NSPort* myPort = [NSMachPort port];
    [myPort setDelegate:self];
    [[NSRunLoop currentRunLoop] addPort:myPort forMode:NSDefaultRunLoopMode];
 
    // Create the check-in message.
    NSPortMessage* messageObj = [[NSPortMessage alloc] initWithSendPort:outPort
                                         receivePort:myPort components:nil];
 
    if (messageObj)
    {
        // Finish configuring the message and send it immediately.
        [messageObj setMsgId:setMsgid:kCheckinMessage];
        [messageObj sendBeforeDate:[NSDate date]];
    }
}

```

#### 配置一个 NSMessagePort 对象

为了用 NSMessagePort 建立一个本地连接，你不能就简单的把端口对象发给其他线程。必须通过名字来获取。在 Cocoa 中你需要用一个名字来注册你的端口，然后把这个名字发送给远端线程，这样他才能获得这个端口对象。


__3-16__  Registering a message port

```
NSPort* localPort = [[NSMessagePort alloc] init];
 
// Configure the object and add it to the current run loop.
[localPort setDelegate:self];
[[NSRunLoop currentRunLoop] addPort:localPort forMode:NSDefaultRunLoopMode];
 
// Register the port using a specific name. The name must be unique.
NSString* localPortName = [NSString stringWithFormat:@"MyPortName"];
[[NSMessagePortNameServer sharedInstance] registerPort:localPort
                     name:localPortName];

```

#### 在 Core Foundation 中配置一个基于端口的 输入源

这一段展示了如何在 Core Foundation 下建立两条的通信通道。


__3-17__  Attaching a Core Foundation message port to a new thread

```
#define kThreadStackSize        (8 *4096)
 
OSStatus MySpawnThread()
{
    // Create a local port for receiving responses.
    CFStringRef myPortName;
    CFMessagePortRef myPort;
    CFRunLoopSourceRef rlSource;
    CFMessagePortContext context = {0, NULL, NULL, NULL, NULL};
    Boolean shouldFreeInfo;
 
    // Create a string with the port name.
    myPortName = CFStringCreateWithFormat(NULL, NULL, CFSTR("com.myapp.MainThread"));
 
    // Create the port.
    myPort = CFMessagePortCreateLocal(NULL,
                myPortName,
                &MainThreadResponseHandler,
                &context,
                &shouldFreeInfo);
 
    if (myPort != NULL)
    {
        // The port was successfully created.
        // Now create a run loop source for it.
        rlSource = CFMessagePortCreateRunLoopSource(NULL, myPort, 0);
 
        if (rlSource)
        {
            // Add the source to the current run loop.
            CFRunLoopAddSource(CFRunLoopGetCurrent(), rlSource, kCFRunLoopDefaultMode);
 
            // Once installed, these can be freed.
            CFRelease(myPort);
            CFRelease(rlSource);
        }
    }
 
    // Create the thread and continue processing.
    MPTaskID        taskID;
    return(MPCreateTask(&ServerThreadEntryPoint,
                    (void*)myPortName,
                    kThreadStackSize,
                    NULL,
                    NULL,
                    NULL,
                    0,
                    &taskID));
}
```

当端口建立好、线程也已经启动了，在等待其他线程回发消息时，主线程可以继续他的周期工作。当回发消息到达，那就分发到主线程的 MainThreadResponseHandler 方法上（见 3-18）。这个函数提取了工作线程的端口名字并创建一个管道为日后通信。


__3-18__  Receiving the checkin message

```
#define kCheckinMessage 100
 
// Main thread port message handler
CFDataRef MainThreadResponseHandler(CFMessagePortRef local,
                    SInt32 msgid,
                    CFDataRef data,
                    void* info)
{
    if (msgid == kCheckinMessage)
    {
        CFMessagePortRef messagePort;
        CFStringRef threadPortName;
        CFIndex bufferLength = CFDataGetLength(data);
        UInt8* buffer = CFAllocatorAllocate(NULL, bufferLength, 0);
 
        CFDataGetBytes(data, CFRangeMake(0, bufferLength), buffer);
        threadPortName = CFStringCreateWithBytes (NULL, buffer, bufferLength, kCFStringEncodingASCII, FALSE);
 
        // You must obtain a remote message port by name.
        messagePort = CFMessagePortCreateRemote(NULL, (CFStringRef)threadPortName);
 
        if (messagePort)
        {
            // Retain and save the thread’s comm port for future reference.
            AddPortToListOfActiveThreads(messagePort);
 
            // Since the port is retained by the previous function, release
            // it here.
            CFRelease(messagePort);
        }
 
        // Clean up.
        CFRelease(threadPortName);
        CFAllocatorDeallocate(NULL, buffer);
    }
    else
    {
        // Process other messages.
    }
 
    return NULL;
}
```

随着主线程配置完，唯一剩下的事情就是为刚刚的工作线程创建一个对应的端口然后回调给工作线程。3-9 展示了工作线程的入口函数。这个函数提取主线程的端口名称，用它来创建一个和主线程连接的通道。然后函数创建了一个本地端口，配置本地端口，发送一个登入信息给抓线程包括他刚刚创建的端口名称

__3-19__  Setting up the thread structures

```
OSStatus ServerThreadEntryPoint(void* param)
{
    // Create the remote port to the main thread.
    CFMessagePortRef mainThreadPort;
    CFStringRef portName = (CFStringRef)param;
 
    mainThreadPort = CFMessagePortCreateRemote(NULL, portName);
 
    // Free the string that was passed in param.
    CFRelease(portName);
 
    // Create a port for the worker thread.
    CFStringRef myPortName = CFStringCreateWithFormat(NULL, NULL, CFSTR("com.MyApp.Thread-%d"), MPCurrentTaskID());
 
    // Store the port in this thread’s context info for later reference.
    CFMessagePortContext context = {0, mainThreadPort, NULL, NULL, NULL};
    Boolean shouldFreeInfo;
    Boolean shouldAbort = TRUE;
 
    CFMessagePortRef myPort = CFMessagePortCreateLocal(NULL,
                myPortName,
                &ProcessClientRequest,
                &context,
                &shouldFreeInfo);
 
    if (shouldFreeInfo)
    {
        // Couldn't create a local port, so kill the thread.
        MPExit(0);
    }
 
    CFRunLoopSourceRef rlSource = CFMessagePortCreateRunLoopSource(NULL, myPort, 0);
    if (!rlSource)
    {
        // Couldn't create a local port, so kill the thread.
        MPExit(0);
    }
 
    // Add the source to the current run loop.
    CFRunLoopAddSource(CFRunLoopGetCurrent(), rlSource, kCFRunLoopDefaultMode);
 
    // Once installed, these can be freed.
    CFRelease(myPort);
    CFRelease(rlSource);
 
    // Package up the port name and send the check-in message.
    CFDataRef returnData = nil;
    CFDataRef outData;
    CFIndex stringLength = CFStringGetLength(myPortName);
    UInt8* buffer = CFAllocatorAllocate(NULL, stringLength, 0);
 
    CFStringGetBytes(myPortName,
                CFRangeMake(0,stringLength),
                kCFStringEncodingASCII,
                0,
                FALSE,
                buffer,
                stringLength,
                NULL);
 
    outData = CFDataCreate(NULL, buffer, stringLength);
 
    CFMessagePortSendRequest(mainThreadPort, kCheckinMessage, outData, 0.1, 0.0, NULL, NULL);
 
    // Clean up thread data structures.
    CFRelease(outData);
    CFAllocatorDeallocate(NULL, buffer);
 
    // Enter the run loop.
    CFRunLoopRun();
}
```


一旦进入 run loop，所有发到现场端口的事件将会被 ProcessClientRequest 方法处理。方法的实现取决于工作线程的类型，这里没有展示。   





有些地方翻译的不溜，看一下代码就明了了。





[^guideline]: https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW1