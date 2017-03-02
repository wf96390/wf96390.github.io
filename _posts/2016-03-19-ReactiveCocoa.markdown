---
layout:     post
title:      ReactiveCocoa学习总结
author:     慢慢
tags: 		ReactiveCocoa iOS
subtitle:  	
category:  blog
---
<!-- Start Writing Below in Markdown -->

ReactiveCocoa是一个FRP的思想在Objective-C中的实现框架，主要为了改善以下几个问题：

1 传统iOS开发过程中，状态以及状态之间依赖过多的问题
2 传统MVC架构的问题：Controller比较复杂，可测试性差
3 提供统一的消息传递机制
 
在我们现在的开发工作中，RAC主要是为了实现MVVM框架，因为RAC的信号机制很容易将某一个变量的变化与界面元素关联，所以非常容易应用Model-View-ViewModel 框架。通过引入ViewModel层，然后用RAC将ViewModel与View关联，View层的变化可以直接响应ViewModel层的变化，这使得Controller变得更加简单。目前如果要实现iOS的MVVM，需要实现数据绑定的功能，因此RAC目前是不可缺少的。
 
我们可能会在ViewModel里写的代码
```
- (RACCommand *)loadCommand
{
    @weakify(self);
    if (!_loadCommand) {
        _loadCommand = [[RACCommand alloc] initWithSignalBlock:^RACSignal *(id input) {
            @strongify(self);
            return [[self.model fetchResultInfoSignal] doNext:^(ResultInfo *resultInfo) {
                @strongify(self);
                self.resultInfo = resultInfo;
            }];
        }];
    }
    return _loadCommand;
}
 
 
- (RACSignal *)fetchResultInfoSignal
{
    @weakify(self);
    return [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        @strongify(self);
        [self fetchResultInfoFinished:^(ResultInfo *resultInfo, NSError *error) {
            if (error) {
                [subscriber sendError:error];
            } else {
                [subscriber sendNext:resultInfo];
                [subscriber sendCompleted];
            }
        }];
        return nil;
    }];
}
[[self.viewModel.loadCommand execute:nil] subscribeCompleted:^{
    // do next Operation
}];
```
而背后的原理是怎样的呢，可以看一下ReactiveCocoa的源码实现

## RACCommand
在RACCommand.m中，我们可以看到初始化方法
```
- (id)initWithSignalBlock:(RACSignal * (^)(id input))signalBlock {
    return [self initWithEnabled:nil signalBlock:signalBlock];
}
```
初始化方法中调用了- (id)initWithEnabled:(RACSignal *)enabledSignal signalBlock:(RACSignal * (^)(id input))signalBlock
初始化了command中的signalBlock，并且初始化了activeExecutionSignals、executing、errors、enabled等 
在- (RACSignal *)execute:(id)input 中
```
RACSignal *signal = self.signalBlock(input);
RACMulticastConnection *connection = [[signal subscribeOn:RACScheduler.mainThreadScheduler] multicast:[RACReplaySubject subject]];
```
使用了self.signalBlock，使用了connection进行连接
```
[self addActiveExecutionSignal:connection.signal];
[connection.signal subscribeError:^(NSError *error) {
    @strongify(self);
    [self removeActiveExecutionSignal:connection.signal];
} completed:^{
    @strongify(self);
    [self removeActiveExecutionSignal:connection.signal];
}];
```

并把connection.signal加入到activeExecutionSignal中，执行完了移除信号
我们可以看到RACCommand执行，返回值类型为RACSignal，我们可以继续对RACSignal进行操作

## RACSignal

RAC的核心就是RACSignal，那什么是RACSignal呢
RACSignal是RACStream的子类，RACStream是一个抽象类，描述了值的流动。 RACSignal通过createSignal进行创建，该方法会调用RACDynamicSignal子类的createSignal方法
createSignal的参数为一个block，block的参数是支持RACSubscriber协议的id类型，返回值为RACDisposable

```
+ (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe {
    return [RACDynamicSignal createSignal:didSubscribe];
}
```
RACDynamicSignal中会把block保存到didSubscribe中，didSubscribe和参数类型一样的block
```
+ (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe {
    RACDynamicSignal *signal = [[self alloc] init];
    signal->_didSubscribe = [didSubscribe copy];
    return [signal setNameWithFormat:@"+createSignal:"];
}
```
而doNext方法做了什么事情呢
```
- (RACSignal *)doNext:(void (^)(id x))block {
    NSCParameterAssert(block != NULL);
    return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
        return [self subscribeNext:^(id x) {
            block(x);
            [subscriber sendNext:x];
        } error:^(NSError *error) {
            [subscriber sendError:error];
        } completed:^{
            [subscriber sendCompleted];
        }];
    }] setNameWithFormat:@"[%@] -doNext:", self.name];
}
```
我们可以看到doNext方法是把原来的Signal生成一个新的Signal，新的Signal的didSubscribe会订阅原来的信号，并且执行doNext中的block

## RACSubscriber
RACSubscriber协议里有4个方法，分别是
```
- (void)sendNext:(id)value;
- (void)sendError:(NSError *)error;
- (void)sendCompleted;
- (void)didSubscribeWithDisposable:(RACCompoundDisposable *)disposable;
```
在执行sendXXX的方法的时候，实际上执行的是
```
@property (nonatomic, copy) void (^next)(id value);
@property (nonatomic, copy) void (^error)(NSError *error);
@property (nonatomic, copy) void (^completed)(void);
```
对应的block，block的内容是在
```
+ (instancetype)subscriberWithNext:(void (^)(id x))next error:(void (^)(NSError *error))error completed:(void (^)(void))completed {
    RACSubscriber *subscriber = [[self alloc] init];
    subscriber->_next = [next copy];
    subscriber->_error = [error copy];
    subscriber->_completed = [completed copy];
    return subscriber;
}
```
中进行赋值的，而subscriberWithNext是在什么时候调用的呢
```
- (RACDisposable *)subscribeNext:(void (^)(id x))nextBlock error:(void (^)(NSError *error))errorBlock completed:(void (^)(void))completedBlock {
    RACSubscriber *o = [RACSubscriber subscriberWithNext:nextBlock error:errorBlock completed:completedBlock];
    return [self subscribe:o];
}
```
是在Signal被订阅的时候被调用，调用的时候发现是RACSubscriber的类方法，也就是说RACSubscriber不仅是一个协议，也存在一个同名的类
RACSubscriber类继承自NSObject，实现了<RACSubscriber>协议
那block是如何执行的呢
```
- (void)sendNext:(id)value {
    @synchronized (self) {
        void (^nextBlock)(id) = [self.next copy];
        if (nextBlock == nil) return;
        nextBlock(value);
    }
}
```
如果block不为空，就执行block中的内容，参数为sendNext的参数
sendXXX方法是在Signal在createSignal的时候保存在didSubscribe中，而didSubscribe的执行时机呢
这个时机就是在RACSignal订阅的时候返回的[self subscribe:o]中执行了didSubscribe，
```
- (RACDisposable *)subscribe:(id<RACSubscriber>)subscriber {
    NSCAssert(NO, @"This method must be overridden by subclasses");
    return nil;
}
```
RACSignal中的该方法是空，需要子类实现，我们需要查看RACDynamicSignal中的该方法
```
- (RACDisposable *)subscribe:(id<RACSubscriber>)subscriber {
    NSCParameterAssert(subscriber != nil);
    RACCompoundDisposable *disposable = [RACCompoundDisposable compoundDisposable];
    subscriber = [[RACPassthroughSubscriber alloc] initWithSubscriber:subscriber signal:self disposable:disposable];
    if (self.didSubscribe != NULL) {
        RACDisposable *schedulingDisposable = [RACScheduler.subscriptionScheduler schedule:^{
            RACDisposable *innerDisposable = self.didSubscribe(subscriber);
            [disposable addDisposable:innerDisposable];
        }];
        [disposable addDisposable:schedulingDisposable];
    }
    return disposable;
}
```
其中的didSubscribe就是createSignal时保存的block，所以只有信号被订阅的时候，创建时的block才会被执行

## RACDisposable

上一个方法的返回值是RACDisposable，也就是createSignal中的返回值，该对象封装了订阅的拆卸和清理工作，RACDisposable如何使用呢
```
__block int count = 0;
RACSignal *signal = [self signInSignal];
__block RACDisposable *dd = nil;
dd = [signal subscribeNext:^(id x) {
    ++count;
    if (count == 5) {
        [dd dispose];
    }
    NSLog(@"-------%@", x);
}];
  
- (RACSignal *)signInSignal
{
    return [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        RACDisposable *d = [[RACDisposable alloc] init];
        [[RACScheduler mainThreadScheduler] schedule:^{
            int a = 1, b = 1;
            while (!d.disposed) {
                [subscriber sendNext:@(a)];
                int oldB = b;
                b = a + b;
                a = oldB;
            }
        }];
        return d;
    }];   
}
```
一个简单的例子，输出斐波那契数列，在第五次循环的时候停止
如果没有[RACScheduler mainThreadScheduler]的话，循环会一直循环下去，不会停止，因为当前线程无法得到返回值d，因此不会被终止

##冷热信号
RACSignal通过createSignal创建了冷信号，而RACSubject则可以创建了热信号，RACSignal的冷信号也可以转化成热信号
RACSubject是RACSignal的子类，也实现了RACSubscriber协议
```
RACSignal *coolSignal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    [[RACScheduler mainThreadScheduler] afterDelay:1 schedule:^{
        [subscriber sendNext:@5];
    }];
    [[RACScheduler mainThreadScheduler] afterDelay:2 schedule:^{
        [subscriber sendNext:@6];
    }];
    return nil;
}];
 
RACSubject *hotSignal = [RACSubject subject];
[[RACScheduler mainThreadScheduler] afterDelay:3 schedule:^{
   [hotSignal subscribeNext:^(id x) {
       NSLog(@"aaa: %@", x);
   }];
   [coolSignal subscribe:hotSignal];
}];
[[RACScheduler mainThreadScheduler] afterDelay:3.5 schedule:^{
    [hotSignal subscribeNext:^(id x) {
        NSLog(@"bbb: %@", x);
    }];
}];
```

参考：
[ReactiveCocoa2 源码浅析](http://nathanli.cn/2015/08/27/reactivecocoa2-%E6%BA%90%E7%A0%81%E6%B5%85%E6%9E%90/)
[RACSignal的Subscription深入分析](http://tech.meituan.com/RACSignalSubscription.html)
[细说ReactiveCocoa的冷信号与热信号](http://tech.meituan.com/talk-about-reactivecocoas-cold-signal-and-hot-signal-part-1.html)
