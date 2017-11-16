---
title: Masonry笔记
date: 2017-06-12 12:02:01
tags: 源码阅读
categories: 源码阅读
---


统一的类别名 ``MAS_VIEW`` 是 ``UIView``   
一些常用的宏见daily    
通过为UIView添加类别的形式扩展了mas_的属性 和 主要的约束变更修改方法   

```
	mas_makeConstraints:       
	mas_updateConstraints:
	mas_remakeConstraints:
```

<!-- more -->

类的结构关系

{% asset_img masonry-1.png 大概图 %}

我们一般使用 

```
	UIView *view = [UIView new];
    [self.view addSubview:view];
    [view mas_makeConstraints:^(MASConstraintMaker *make) {
        make.top.mas_equalTo(self.view.mas_top);
        make.left.bottom.right.mas_equalTo(self.view);
    }];
```

view 的 ``mas_makeConstraints`` 方法 是 MAS 扩展的类`` View+MASShorthandAdditions.h `` 提供的。他其实是 `` View+MASAdditions `` 一个补充，提供一个便捷的访问方式。   

具体看 ``mas_makeConstraints`` 方法        

```
- (NSArray *)mas_makeConstraints:(void(^)(MASConstraintMaker *))block {
	//autolayout 的开关
    self.translatesAutoresizingMaskIntoConstraints = NO;
    //构建者
    MASConstraintMaker *constraintMaker = [[MASConstraintMaker alloc] initWithView:self];
    //调用外部添加的约束 (见p1)
    block(constraintMaker);
    return [constraintMaker install];
}

```

p1: 处执行的代码  

```
make.top.mas_equalTo(self.view.mas_top);
make.left.bottom.right.mas_equalTo(self.view);
```

这里`` make ``就是刚刚p1处的创建的构建者，他的top属性在``MASConstraintMaker`` 中实际上是创建了一个对象


```
step: 1
- (MASConstraint *)top {
    return [self addConstraintWithLayoutAttribute:NSLayoutAttributeTop];
}
step: 2
- (MASConstraint *)addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
    return [self constraint:nil addConstraintWithLayoutAttribute:layoutAttribute];
}
step:3
这个方法是在构造者的对象中
- (MASConstraint *)constraint:(MASConstraint *)constraint addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute{
	//这里根据开始传进来的view 创建`MASViewAttribute`
	//接着包装成`MASViewConstraint` 这个类是`MASConstraint` 的子类
	//最后把这个对象放到maker这个构造者的`constraints`数组当中
}

```

上面make.top返回了一个``MASConstraint ``的对象，接着调用了``mas_equalTo``这个方法，``mas_equalTo``是一个宏``equalTo(MASBoxValue((__VA_ARGS__)))``     

其中``MASBoxValue ``一会看    
``equalTo()``可能出现的参数大概有几种


|index |value  | type |
|:----:|:-----:|:----:|
|1     |@(1)  | NSNumber |
|2     |view.mas_top  | MASViewAttribute |
|3     | view  | MAS_VIEW |


那么刚刚``MASBoxValue``做了什么？       

```
tips:
	@encode
	编译器指令，返回一个给定类型编码为一种内部表示的字符串
	类似于typeof
extend:
@ ---> An object (whether statically typed or typed id)
# ---> A class object (Class)

sample code: 
A *a = A.new;
@encode(typeof(a))--->@
@encode(typeof(A))--->{ A =# }

```


``MASBoxValue``对数值型、坐标型的``__VA_ARGS__``进行了``NSValue``的包装    
回去看``equalTo()``方法，他是``MASConstraint ``的实力调用的，因为在创建的时候是返回的一个``MASViewConstraint``的父类指针引用，所以最终他还是调用的子类``MASViewConstraint``的``equalTo``方法

```
in file MASConstraint.m 
- (MASConstraint * (^)(id))equalTo {
    return ^id(id attribute) {
        return self.equalToWithRelation(attribute, NSLayoutRelationEqual);
    };
}

- (MASConstraint * (^)(id, NSLayoutRelation))equalToWithRelation { MASMethodNotImplemented(); }
```

返回的是一个函数，这个函数可以返回``MASConstraint``对象


```
in MASViewConstraint.m file
- (MASConstraint * (^)(id, NSLayoutRelation))equalToWithRelation {
    return ^id(id attribute, NSLayoutRelation relation) {
    	// |==============================================|
    	// |make.height.equalTo(@[greenView, blueView]);  |
    	// |多个view的约束可能会传进数组                      |
    	// |循环做了else的操作                              |
    	// |==============================================|
        if ([attribute isKindOfClass:NSArray.class]) {
            NSAssert(!self.hasLayoutRelation, @"Redefinition of constraint relation");
            NSMutableArray *children = NSMutableArray.new;
            for (id attr in attribute) {
                MASViewConstraint *viewConstraint = [self copy];
                viewConstraint.layoutRelation = relation;
                viewConstraint.secondViewAttribute = attr;
                [children addObject:viewConstraint];
            }
            MASCompositeConstraint *compositeConstraint = [[MASCompositeConstraint alloc] initWithChildren:children];
            compositeConstraint.delegate = self.delegate;
            [self.delegate constraint:self shouldBeReplacedWithConstraint:compositeConstraint];
            return compositeConstraint;
        } else {
            NSAssert(!self.hasLayoutRelation || self.layoutRelation == relation && [attribute isKindOfClass:NSValue.class], @"Redefinition of constraint relation");
            self.layoutRelation = relation;
            self.secondViewAttribute = attribute;
            return self;
        }
    };
}

```


这里 ``MASCompositeConstraint`` 等一下看  
就把约束和关系加到 ``MASConstraint``中了

构建者maker 最后在调用``install`` 当中会根据调用函数是``remake、update、make``来做约束``uninstall``的更新。在遍历当前``constraints``数组中的约束调用他的``install`` (同``equalTo``)类似，是子类``MASViewConstraint`` 中

下次写   
下班。。。

