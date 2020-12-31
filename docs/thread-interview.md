
# iOS多线程（三） - 问题分析

{docsify-updated}

## NSRunloop 三种启动方式区别?
```
// 第一种方式，runloop会一直运行下去，在此期间会处理来自输入源的数据，并且会在NSDefaultRunLoopMode模式下重复调用runMode:beforeDate:方法；
- (void)run;  

// 第二种方式，可以设置超时时间，在超时时间到达之前，runloop会一直运行，在此期间runloop会处理来自输入源的数据，并且也会在NSDefaultRunLoopMode模式下重复调用runMode:beforeDate:方法；
- (void)runUntilDate:(NSDate *)limitDate；

// 第三种方式，runloop会运行一次，超时时间到达或者第一个input source被处理，则runloop就会退出
- (void)runMode:(NSString *)mode beforeDate:(NSDate *)limitDate;
```

## 输出下边代码的执行顺序
```
NSLog(@"1");
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"2");
    [self performSelector:@selector(test) withObject:nil afterDelay:10];
    NSLog(@"3");
});
NSLog(@"4");
- (void)test
{
    NSLog(@"5");
}
```
**答案是1423，test方法并不会执行。**
原因是如果是带afterDelay的延时函数，会在内部创建一个 NSTimer，然后添加到当前线程的RunLoop中。也就是如果当前线程没有开启RunLoop，该方法会失效。
那么我们改成:

```
dispatch_async(dispatch_get_global_queue(0, 0), ^{
        NSLog(@"2");
        [[NSRunLoop currentRunLoop] run];
        [self performSelector:@selector(test) withObject:nil afterDelay:10];
        NSLog(@"3");
    });
```
**然而test方法依然不执行。**
原因是如果RunLoop的mode中一个item都没有，RunLoop会退出。即在调用RunLoop的run方法后，由于其mode中没有添加任何item去维持RunLoop的时间循环，RunLoop随即还是会退出。
所以我们自己启动RunLoop，一定要在添加item后

```
dispatch_async(dispatch_get_global_queue(0, 0), ^{
        NSLog(@"2");
        [self performSelector:@selector(test) withObject:nil afterDelay:10];
        [[NSRunLoop currentRunLoop] run];
        NSLog(@"3");
    });
```
## 怎样保证子线程数据回来更新UI的时候不打断用户的滑动操作？

这个问题应该这样问比较好：数据返回后用户在滑动TableView，如何更好的刷新数据，让用户操作更顺畅一些。

答案是放在Runloop的defaultMode下进行刷新。

```
[self performSelectorOnMainThread:@selector(reloadData) withObject:nil waitUntilDone:NO modes:@[NSDefaultRunLoopMode]];
```
## 如何切换Runloop模式

1. 可以使用CFRunloopStop()函数来停止，然后重新Run一个新的模式。
2. Runloop是可以重复运行的。重复运行时就进入了新的模式，当退出时则会回到之前的模式。Runloop是可以嵌套调用的。


## NSTimer不精准的解决方法


开辟子线程中进行NSTimer的操作，再在主线程中修改UI界面显示操作结果，这么做相对于将NSTimer添加在主线程RunLoop来说是会提高精确度，但是如果说子线程的RunLoop也比较繁忙的话，那一样会带来误差；

## 你用过NSOperationQueue么？如果用过或者了解的话，你为什么要使用NSOperationQueue，实现了什么？请描述它和GCD的区别和类似的地方（提示：可以从两者的实现机制和适用范围来描述)。

GCD是纯C语言的API，NSOPerationQueue是基于GCD的OC版本的封装；
GCD只支持FIFO的队列，NSOPerationQueue可以很方便的调整执行顺序、设置最大并发数量；
NSOPerationQueue可以轻松在Operation间设置依赖关系，而GCD需要写很多的代码才能实现；
NSOperationQueue支持KVO，可以检测operation是否正在执行（isExecuted）、是否结束（isFinished）、是否取消（isCanceled）;


任务之间不太相互依赖：GCD
任务之间有依赖或者要监听任务的执行情况：NSOperationQueue

## 代码分析1

```
- (void)interview1{
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    
    dispatch_sync(queue, ^{
        NSLog(@"1---%@",[NSThread currentThread]);
        [self performSelector:@selector(test1) withObject:nil afterDelay:.0f];
        NSLog(@"3---%@",[NSThread currentThread]);
    });
}

- (void)test1{
    NSLog(@"2---%@",[NSThread currentThread]);
}

// ***************打印结果***************
2019-12-30 17:47:01.936609+0800 MultithreadingDemo[39150:4282068] 1---<NSThread: 0x6000009660c0>{number = 1, name = main}
2019-12-30 17:47:01.936724+0800 MultithreadingDemo[39150:4282068] 3---<NSThread: 0x6000009660c0>{number = 1, name = main}
2019-12-30 17:47:01.936904+0800 MultithreadingDemo[39150:4282068] 2---<NSThread: 0x6000009660c0>{number = 1, name = main}
```

同步执行是在主线程中执行的，由于主线程有Runloop，所以test1会执行，由于使用Runloop唤醒，所以慢一些。

## 代码分析2

```
- (void)interview2{
    NSThread *thread = [[NSThread alloc] initWithBlock:^{
       NSLog(@"1---%@",[NSThread currentThread]);
    }];
    [thread start];
    
    [self performSelector:@selector(test2) onThread:thread withObject:nil waitUntilDone:YES];
}

- (void)test2{
    NSLog(@"2---%@",[NSThread currentThread]);
}

// ***************运行结果(闪退)***************
2019-12-31 08:36:07.132133+0800 MultithreadingDemo[40268:4493885] 1---<NSThread: 0x6000010d9880>{number = 6, name = (null)}
2019-12-31 08:36:07.432190+0800 MultithreadingDemo[40268:4493455] *** Terminating app due to uncaught exception 'NSDestinationInvalidException', reason: '*** -[Interview performSelector:onThread:withObject:waitUntilDone:modes:]: target thread exited while waiting for the perform'
```
执行test2时，thread已经执行结束，不可用了。


## 代码分析 3

```
- (void)interview3{
    NSLog(@"执行任务1--%@",[NSThread currentThread]);
    dispatch_queue_t queue = dispatch_get_main_queue();
    dispatch_sync(queue, ^{
        NSLog(@"执行任务2--%@",[NSThread currentThread]);
    });
    
    NSLog(@"执行任务3--%@",[NSThread currentThread]);
}

```
执行1后，主线程卡死。

## 代码分析 4

```
- (void)interview5{
    NSLog(@"执行任务1--%@",[NSThread currentThread]);
    
    dispatch_queue_t queue = dispatch_queue_create("myqueu", DISPATCH_QUEUE_SERIAL);
    dispatch_async(queue, ^{
        NSLog(@"执行任务2--%@",[NSThread currentThread]);
        
        dispatch_sync(queue, ^{
            NSLog(@"执行任务3--%@",[NSThread currentThread]);
        });
    
        NSLog(@"执行任务4--%@",[NSThread currentThread]);
    });
    
    NSLog(@"执行任务5--%@",[NSThread currentThread]);

// ***************打印结果(打印1、5、2后卡死)***************    
2019-12-31 10:45:29.071774+0800 MultithreadingDemo[41379:4551961] 执行任务1--<NSThread: 0x6000038460c0>{number = 1, name = main}
2019-12-31 10:45:29.071923+0800 MultithreadingDemo[41379:4551961] 执行任务5--<NSThread: 0x6000038460c0>{number = 1, name = main}
2019-12-31 10:45:29.071932+0800 MultithreadingDemo[41379:4552048] 执行任务2--<NSThread: 0x600003824f40>{number = 6, name = (null)}
}

```
任务3 是在当前线程同步执行，产生死锁。

## 代码分析 5

```
- (void)interview7{
    NSLog(@"执行任务1--%@",[NSThread currentThread]);
    
    dispatch_queue_t queue = dispatch_queue_create("myqueu", DISPATCH_QUEUE_CONCURRENT);
    
    dispatch_async(queue, ^{
        NSLog(@"执行任务2--%@",[NSThread currentThread]);
        
        dispatch_sync(queue, ^{
            NSLog(@"执行任务3--%@",[NSThread currentThread]);
        });
        
        NSLog(@"执行任务4--%@",[NSThread currentThread]);
    });
    
    NSLog(@"执行任务5--%@",[NSThread currentThread]);
}

// ***************打印结果***************
2019-12-31 11:14:27.690008+0800 MultithreadingDemo[41445:4562142] 执行任务1--<NSThread: 0x6000011badc0>{number = 1, name = main}
2019-12-31 11:14:27.690102+0800 MultithreadingDemo[41445:4562142] 执行任务5--<NSThread: 0x6000011badc0>{number = 1, name = main}
2019-12-31 11:14:27.690122+0800 MultithreadingDemo[41445:4562301] 执行任务2--<NSThread: 0x6000011f5900>{number = 3, name = (null)}
2019-12-31 11:14:27.690202+0800 MultithreadingDemo[41445:4562301] 执行任务3--<NSThread: 0x6000011f5900>{number = 3, name = (null)}
2019-12-31 11:14:27.690285+0800 MultithreadingDemo[41445:4562301] 执行任务4--<NSThread: 0x6000011f5900>{number = 3, name = (null)}

```
任务3 在并发对列中同步执行，并不会产生死锁。

















