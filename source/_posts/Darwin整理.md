---
layout: _draft
title: Darwin整理 (1)
date: 2017-09-01 14:24:24
tags: 碎片整理
categories: 碎片整理
---

读书的碎片整理
<!--more-->

```
install 
	openssh	
	core utilities
	Adv-cmds
	top

openssh login -> uname -a 
192.168.238.176


otool -arch i386 -tv /usr/lib/libSystem.B.dylib -p _chown | more


XNU {
	Mach (microkernel)
	BSD
	libKern
	I/O Kit
}

Mach(microkernel) {
	进程线程抽象
	虚拟内存管理
	任务调度
	进程通信和消息传递机制
}

BSD {
	UNIX 进程模型
	POSIX线程模型及相关同步原语
	UNIX 用户和组
	BSD Socket API
	文件访问
	设备访问
}

sysctl 修改查询内核变量值
kqueue 内核时间通知机制 {
	kevent
}

POSIX {
	poll/select
}

struct kevent ke;
EV_SET(&ke, pid, EVFILT_PROC, EV_ADD, NOTE_EXIT | NOTE_FORK | NOTE_EXEC, 0, NULL);


审计

MAC(Mandatory Access Control) 强制访问控制 沙盒

OpenLDAP(Lightweight Directory Access Protocol) 协议

用户和组的管理 {
	dscl . -read /Users/`whoami`
}

系统配置 {
	configd
	scutil>list
}

记录日志
syslog {
	处理文本消息 {
		facility -| ---  消息来源
				   => 分类
		severity -| ---  严重性
	} 
}

ASL (Apple System Log) 二进制格式 {
	向后兼容 syslogd
	网络协议 syslogd
	内核日志接口
	全新ASL接口
}

AppleScript {
	
}

AppleEvents {
	AEDebugSends
	AEDebugReceives
}

FSEvents {
	文件系统通知
		1.进程获得FSEvents机制的句柄
		2.ioctl调用
		3.请求者read
} --> (Linux -> inotify)

filemon 工具


通知 {      守护程序
	核心 -> notifyd    Darwin通知服务器
	核心 -> distnoted  分布式通知服务器
}


代码签名
苹果将其证书保存在 OS X 和 iOS的密钥中
系统拒绝运行任何没有签名的代码 没有签名会被内核杀掉

entitlement 权限
```

# 第4章 

```
Mach-o 格式 进程及线程内幕

setpgrp 将一个进程加入一个进程组

子进程返回的整数尤其父进程收集，进程将要返回的值传递给exit系统调用
主进程通过wait等待子进程，以便收集子进程返回的值

僵尸状态
进程的空壳，释放了所有的资源，但是依然占用着PID，以<defunct>或者Z状态出现在进程列表
最终被回收的情况 {
	1.父进程通过wait操作，收集子进程返回值
	2.父进程死亡，讲他们交与PID为1的进程
}

pid_suspend
pid_resume

pid_shutdown_sockets 系统调用可以再进程之外关闭这个进程的所有套接字

进程主要凭据 {
	创建者的用户id(UID)
	主进程组id(GID)
}

可执行文件
chmod + x 可以标记可执行文件 但不保证这个文件可以执行 这个标志是告诉操作系统内核将这个文件读入内存 然后寻找一个头签名 据此可以确定精确的可执行格式，这个头文件签名称为“魔数”


通用二进制格式
调用一个二进制文件时，Mach加载器回首页解析胖二进制文件头，确定其中可用的架构，然后加载最合适的架构的代码

{
	NXGetLocalArchInfo() 获得主机的架构信息
	NXFindBestFatArch()	 返回最匹配的架构
}
lipo -detailed_info /usr/bin/perl 查看二进制文件的胖头文件

Mach-O(bject) {
	魔数 MH_MAGIC/MH_MAGIC_64
	CPU类型及子类型字段
}

otool 命令

加载命令
类型-长度-值
加载过程在内核的部分负责新进程的基本设置{
	分配虚拟内存 LC_SEGMENT
	创建主线程
	处理可能的代码签名/加密的工作
}

LC_SEGMENT {
	主要加载命令
	指导内核如何设置新运行的进程的内存空间，这些‘段’直接从Mach-O二进制文件加载到内存中
}

LC_UNIXTHREAD {
	启动二进制程序的主线程
	列出所有初始化寄存器的状态，这些寄存器保存了程序入口点的地址
}

LC_THREAD {
	
}

LC_MAIN{
	设置程序主线程入口地址和栈大小
}

LC_CODE_SIGNATURE {
	Mach-O二进制文件的代码签名
	如果这个签名和代码不匹配，内核会立刻给进程发送SIGKILL信号，将进程杀死
}
```
## 4.4动态库
```
动态链接器 是 内核执行LC_DYLINKER 
链接器 {
	查找进程中所有符号和库的依赖关系，然后解决这些关系。是递归完成的，库还会依赖其他库
	dyld 是一个用户态的进程

}

二进制文件中使用了外部定义的函数和符号 那么在他的文本段会有一个名为__stubs的区，在这个区中存放的是这些本地未定义符号的占位符。编译器身材代码的时候会创建对符号桩的调用，链接器在运行的时候会解决对桩的调用。链接器的解决方法是在被调用的地址处放置一条JMP指令，JMP指令将控制权转交给真正的函数体。


nm 显示未决的符号

共享库缓存
dyld采用一个共享库预链接缓存
```

### 4.4.2 库的运行时加载
```
代码付现
{
	dlopen 
	ldopen_preflight
	dlsym
	dladdr
	dlerror
}

弱定义的符号
__attribute__(weak_import)

dyld 特性 {
	1.两级名称空间
	2.函数拦截 
}
```

## 4.5 进程地址空间
用户态的一个优点在于虚拟内存的隔离，进程独享一个私有的地址空间
### 4.5.1 进程入口点
### 4.5.2 地址空间布局随机化
```
ASLR {
	进程每一次启动时，地址空间都会被简单地随机化--只是偏移
}

内存空间的段 {
	__PAGEZERO   
	__TEXT				代码区     
	__LINKEDIT			dyld使用，包含了字符串表，符号表以及其他数据
	__IMPORT			i386的二进制文件导入表
	__DATA				可读可写数据
	__MALLOC_TINY		小于1个页的内存分配
	__MALLOC_SMALL		几个页的内存分配
	__MALLOC_LARGE		1MB以上的内存分配
}

commpage {
	一组内核导出给所有用户态的页面，包含了各种和CPU以及平台相关的函数
}

```
## 4.6 进程内存分配（用户态）
```
用户态 内存处理 {
	栈
	堆
}

内核中内存处理规约为 {
	页面
}
```

### 4.6.1 alloca()
```
栈 {
	1.保存自动变量
	2.程序员动态分配内存 alloca {
		函数原型和malloc 一样的 、
		区别在于
			1.alloca返回的是栈上的指针
			2.malloc返回的是堆上的指针
	}
	优点 {
		1. 栈分配空间 修改栈指针寄存器
		2. 空间自动释放
	}
	缺点 {
		栈空间的受限
	}
}
```

### 4.6.2 堆分配
```
堆（数据结构--二叉堆(现在使用更为复杂的结构)） {
	分配区域(每个区域有各自的分配器) {
		tiny
		small
		large
		huge
	}
}
```

`malloc_zone_t`     
`malloc_create_zone`   
`malloc_zone_register`   
`malloc_zone_malloc`

`malloc_get_all_zones`

iOS中VM压力释放机制依赖于JetSam(实现didReceiveMemoryWarning方法)

### 4.6.3 虚拟内存

`vm_stat` 显示内核内部的虚拟内存技术器     
`sysctl`  查看和修改内核变量的标准UNIX命令     
`dynamic_pager` 不在内核层次管理交换空间，由一个专门的用户进程处理所有的交换请求


## 4.7 线程

1. `POSIX` 线程
2. `GCD`
不要从线程和线程函数的角度思考，从功能块的角度思考
能够自动地随着逻辑处理器的个数而扩展


#5 进程跟踪和调试

DTrace 应用于OS X 和 iOS 模拟器的调试平台    
Instruments 是对 D 脚本的分装

```
DTrace {
	探针
	脚本
}

```

### 5.1.2 dtruss
允许跟踪系统调用时打印出C风格的形式，显示系统调用、参数及返回值。      
dtruss支持的三种使用模式   
> 1. 通过dtruss运行一个进程
> 2. 附加到某个正在运行的进程实例
> 3. 附加到命名的进程

-c、-d、-e、和-o参数可以方便进行性能剖析     

## 5.2 其他剖析机制

> CHUD   
> AppProfileFamily

## 5.3 进程信息

> ps  
> lsof   
> netstat   
> ...

### 5.3.1 sysctl

`sysctl` 提供一些现实进程统计的数据变量     
`kern` 名称空间在`CTL_KERN`下暴露了 `KERN_PROCARGS`和`KERN_PROCARGS2`


## 5.4 进程和系统快照
> system_profiler 系统信息app的命令行版本    
> sysdiagnose 完整的诊断工具     
> allmemory 捕获用户进程的所有内存使用情况   
> stackshot 获得进程执行状态的快照--stack_snapshot 捕获指定进程中所有线程的状态

## 5.5 kdebug

### 5.5.1 基于kdebug的工具
> sc_usage   显示进程的系统调用信息   
> fs_usage   显示于文件、套接字和目录相关的系统调用    
> latency    显示中断和调度延迟值   

## 5.6 应用程序崩溃
进程奔溃的真正原因来自于内核，内核发现进程无法继续执行，生成这个信号作为最后的补救办法   

+ 核心转移   
	- `ulimit -c`
	- 核心文件巨大   
+ Crash Reporter   
	- 进程异常终止时自动触发

# 8 内核架构




















