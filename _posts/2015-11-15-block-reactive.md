---
layout: post
title: Block 和 ReactiveCocoa的理解（一）
categories: [Objective-C]
tags: [Block, ReactiveCocoa]
fullview: false
shortinfo: 虽然接触iOS已经8个月了，block 作为Objective C中对于回调(callback)的实现，理解起来还是有点模棱两可.在《Pro Multithreading and Memory Management for iOS and OS X》书中，Kazuki Sakamoto 对block的定义

note: “Block是拥有自动变量(可以在block声明的语义环境里捕捉变量的状态)的匿名(使函数体(code)成为和数据(data)一样的一等公民，作为函数调用时输入的实参(argument))回调函数。”




---

## 1. block的最大用处：回调(callback)

虽然接触iOS已经8个月了，block 作为Objective C中对于回调(callback)的实现，理解起来还是有点模棱两可。在《Pro Multithreading and Memory Management for iOS and OS X》书中，Kazuki Sakamoto 对block的定义是：

>拥有自动变量（可以在block声明的语义环境里捕捉变量的状态）的匿名（使函数体(code)成为和数据(data)一样的一等公民，作为函数调用时输入的实参（argument））函数。

我发现，在大部分应用block的场景，这样的定义不够sharp，使人模棱两可。我用我自己的理解重定义block：
>拥有自动变量（可以在block声明的语义环境里捕捉变量的状态）的匿名(使函数体(code)成为和数据(data)一样的一等公民，作为函数调用时输入的实参（argument））**回调**（callback,等待被调用）函数。 

回调(callback)两个字把block在大部分场景应用时的扮演的角色给准确描述。下面我举一个简单的例子来解释回调。NSArray的实例方法 *- enumerateObjectsUsingBlock:(void (^ )(id obj, NSInteger idx, BOOL *stop))block 是一个用block实现数组枚举的方法。下面是该方法的简单应用：

    NSArray *cities = @[@"Beijing", @"Shanghai", @"Guangzhou", @"Shenzhen"，@"Hong Kong"];
    [cities enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL * stop) {
        NSLog(@"index: %lu, obj: %@, stop: %d",(unsigned long)idx,obj,*stop);
        if(idx == 3){
            *stop = YES;
        }
    }];

    //这是一个再简单不过的数组枚举。输出为：

    2016-01-06 22:52:18.143 Chapter00_Block implmenetaion[18596:780185] index: 0, obj: Beijing, stop: 0
    2016-01-06 22:52:18.144 Chapter00_Block implmenetaion[18596:780185] index: 1, obj: Shanghai, stop: 0
    2016-01-06 22:52:18.144 Chapter00_Block implmenetaion[18596:780185] index: 2, obj: Guangzhou, stop: 0
    2016-01-06 22:52:18.144 Chapter00_Block implmenetaion[18596:780185] index: 3, obj: Shenzhen, stop: 0

那么，如何理解上面block的新定义里的回调呢。由于apple 自己里大部分类的实现是保密的，我们来自己定义一个方法来简单实现下这个- enumerateObjectsUsingBlock：方法的功能。首先来给NSArray添加一个名为Enumeration的category，加入以下代码。

    NSArray+Enumeration.h

    #import <Foundation/Foundation.h>
    @interface NSArray (Enumeration)
    - (void)customizedEnumerateWithBlock:(void(^)(id obj, NSInteger index, BOOL *stop))block;
    @end

    NSArray+Enumeration.m

    #import "NSArray+Enumeration.h"
    @implementation NSArray (Enumeration)
    -(void)customizedEnumerateWithBlock:(void (^)(id, NSInteger, BOOL *))block{
        BOOL stop = false; for(int i = 0; i < self.count; ++i){
            block(i,[self objectAtIndex:i],&stop);
            if(stop) break;
        }
     }  
    @end

如果你把cities的数组枚举方法换成这个，那么输出是一样的。

    NSArray *cities = @[@"Beijing", @"Shanghai", @"Guangzhou", @"Shenzhen", @"Hong Kong"];
    [cities customizedEnumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL * stop) {
        NSLog(@"index: %lu, obj: %@, stop: %d",(unsigned long)idx,obj,*stop);
        if(idx == 3){
            *stop = YES;
        }
    }];

    //输出为：

    2016-01-06 22:52:18.143 Chapter00_Block implmenetaion[18596:780185] index: 0, obj: Beijing, stop: 0
    2016-01-06 22:52:18.144 Chapter00_Block implmenetaion[18596:780185] index: 1, obj: Shanghai, stop: 0
    2016-01-06 22:52:18.144 Chapter00_Block implmenetaion[18596:780185] index: 2, obj: Guangzhou, stop: 0
    2016-01-06 22:52:18.144 Chapter00_Block implmenetaion[18596:780185] index: 3, obj: Shenzhen, stop: 0

由此可见，我们自己实现的数组枚举能实现原本的数组枚举方法。在- customizedEnumerateWithBlock: 的实现里，回调了block这个函数。因此若有方法里有block体作为参数，则意味著这个方法在调用其他函数的同时，这个block在等待将来被调用。那么该block什么时候被调用呢（这里是for 循环语句里），在实际应用block场景里，block的调用更多地是简化网络请求，delegate等。

**因此“回调”是block我个人认为最大的用处。**

## 2. ReactiveCocoa的subscription原理

作为响应式函数编程的iOS实现，ReactiveCocoa将iOS编程带入了一个新纪元，其最大的贡献在于：
>统一消息传递机制：iOS 开发中有着各种消息传递机制，包括 KVO、Notification、delegation、block 以及 target-action 方式。

ReactiveCocoa的入门可以参考
 http://www.raywenderlich.com/62699/reactivecocoa-tutorial-pt1
http://www.raywenderlich.com/62796/reactivecocoa-tutorial-pt2

这里我着重要说的是ReactiveCocoa （订阅）subscription的实现原理--block回调的应用。在ReactiveCocoa里，RACSignal， RACSubscriber, RACDisposal 是最核心的三个类。下面我们就一个简单的例子深入理解subscription的实现原理，见下面代码：

    -(RACSignal *)signInSignal {
      // part 1:[RACSignal createSignal]来获得signal
      return [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
               [self.signInService signInWithUsername:self.usernameTextField.text
                                             password:self.passwordTextField.text
                                             complete:^(BOOL success) {
      // part 3: 进入didSubscribe，通过[subscriber sendNext:]来执行next block
                 [subscriber sendNext:@(success)];
                 [subscriber sendCompleted];
                }];
                return nil;
              }];
    }

    // part 2 : [signal subscribeNext:]来获得subscriber，然后进行subscription
    [[self signInSignal] subscribeNext:^(id x) {
        NSLog(@"Sign in result: %@", x);
    }];
