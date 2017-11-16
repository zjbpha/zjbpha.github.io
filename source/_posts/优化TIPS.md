---
title: 优化TIPS
date: 2017-11-14 17:40:40
categories: 碎片整理
tags: 碎片整理
---

日常优化整理笔记
<!--more-->

## 语言或平台层面
1. 循环引用
2. 加速启动时间 
	1. pre-main {加载应用可执行文件、加载dyld、加载dylib}
	2. main {dyld调用main()、调用UIApplicationMain、WillFinishLaunching、didFinishLaunching}
	3. 首屏渲染动画


## 业务逻辑处理的优化

1. 对于局部内存分配较多的地方，选择 autoreleasepool 去处理。
2. 不要 block 主线程
3. 数组采用快速枚举
4. 网络传输中默认的 gzip 压缩
5. 大量 JS 代码分段执行
6. 单次定位
7. 单次计算，建立数据模型



## UI 层面的优化

1. 图片的解码线程
2. UITableView 的优化处理
  1. Cell 的复用
  2. 减少图层
  3. 对于圆角
  4. 图片的像素点对齐
  5. 统一的单次计算，赋值和计算分离
  6. opaque no blending
3. 懒加载和重用
4. 异步绘制
5. 滑动时按需加载
6. cache
7. 避免某些函数的多次调用（日期格式转化）
8. view 的 opaque（view 不透明）不这么弄会有 blend、还有shadow
9. 在次线程做图像缩放
10. 当内存警告的时候，回收一些内存

## 量化
1. 主要是工具 Instrument

## 网络
1. 合适的数据传输格式 
2. 适当的合并网络请求
3. HTTP/2 NSURLSession 对比 HTTP/1.1 PIPE

## 体积
1. 去重（图片、代码）
2. 删除没有连接的资源
3. linkMap 分析
4. AppThinning

## 硬件

1. 减少无效的 CPU 计算，节省电量。
2. 支付宝有个节省屏幕亮度的方案
3. 网络请求

待补充...