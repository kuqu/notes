
# iOS多线程（二） - 多线程及Runloop

{docsify-updated}

## NSThread

OC提供的线程操作类。其他多线程方式已经不可以直接操作线程了。

__有三种使用方式：__

initWithTarget 和 detachNewThreadSelector 方式:
```
NSThread *thread = [[NSThread alloc] initWithBlock:^{
        NSLog(@"1---%@",[NSThread currentThread]);
    }];
[thread start];
```

继承方式：
```
// 继承NSThread 并重写 main 方法。
@interface RequestImageDataThread : NSThread

@end

@implementation RequestImageDataThread

- (void)main{
    NSLog(@"thread = %@", self.class.currentThread);
    NSData *data = [NSData dataWithContentsOfURL:[NSURL URLWithString:@"https://www.baidu.com/images/floloww.jpg"]];
    NSLog(@"thread finish");
    [self exit];
}
@end
```

分类方式：
```
NSObject *obj = [[NSObject alloc] init];

[obj performSelectorOnMainThread:@selector(show) withObject:nil waitUntilDone:YES];

[obj performSelector:@selector(run) onThread:[NSThread mainThread] withObject:nil waitUntilDone:YES modes:nil];
```

## NSOperation

属于比较重的操作方式，多使用继承的方式来使用。分为操作（NSOperation）和队列（NSOperationQueue），将操作初始化后放入对列则自动调用 [NSOperation start] 执行。

类关系图如下：
```
NSOperation
	- NSBlockOperation（通过block的方式使用）
	- NSInvocationOperation（通过target Selector的方式使用）
	- NSOperation自定义子类（通过继承的方式使用，主要使用方式）
	
NSOperationQueue 默认是并行对列
```

关键点1：
NSOperation 可单独使用，但是只能在当前线程执行。

```
NSOperation *operation = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"%@", [NSThread currentThread]);
}];

[operation start];
```

关键点2：
如何串行执行

```
NSOperationQueue *queue = [[NSOperationQueue alloc ]init];
queue.maxConcurrentOperationCount = 1;// 默认是-1表示由系统控制并发量
```

关键点3：
取消执行 - 对于已经执行的操作是无法取消的。

```
NSOperation *operation = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"%@", [NSThread currentThread]);
    while (1) {
        NSLog(@"aaaa");
        sleep(1);
    }
}];

NSOperationQueue *queue = [[NSOperationQueue alloc ]init];
queue.maxConcurrentOperationCount = 1;
[queue addOperation:operation];

dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(4 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
    [operation cancel];
    [queue cancelAllOperations];
});
// 这里的取消执行是没有用，对于未开始的操作可以取消，一开始的操作无法取消。
实现取消应在操作关键点检查isCancel属性，手动结束执行。
```

关键点4：
NSOperaation 的 addExecutionBlock 操作是在异步线程执行的。

```

NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"%@", [NSThread currentThread]);// 主线程
    NSLog(@"aaaa");
}];

[operation addExecutionBlock:^{
   NSLog(@"bbbb");
    NSLog(@"%@", [NSThread currentThread]);// 异步线程
}];

[operation start];
```

关键点5：
添加依赖关系
```
NSBlockOperation *operation1 = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"%@", [NSThread currentThread]);
    NSLog(@"aaaa");
}];

[operation1 addExecutionBlock:^{
    NSLog(@"bbbb");
    sleep(3);
    NSLog(@"%@", [NSThread currentThread]);
}];

NSBlockOperation *operation2 = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"%@", [NSThread currentThread]);
    NSLog(@"CCC");
}];

[operation1 addDependency:operation2];


NSOperationQueue *queue = [[NSOperationQueue alloc ]init];

[queue addOperation:operation1];
[queue addOperation:operation2];
/*
<NSThread: 0x10053a620>{number = 2, name = (null)}
CCC
bbbb
<NSThread: 0x10053a620>{number = 2, name = (null)}
aaaa
<NSThread: 0x100483690>{number = 3, name = (null)}
*/
```

## GCD

gcd包含操作方式（async&sync）和对列（queue）两个概念。而没有了线程的概念。线程和对列并不是相互对应的关系，但有一定的规律。考题中常常让你说出是某个queue是在哪个线程执行。

1. 同步异步执行
```
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"async & global on %@", [NSThread currentThread]);// 子线程
});


dispatch_sync(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"sync & global on %@", [NSThread currentThread]);// 主线程
});


dispatch_queue_t queue = dispatch_queue_create("", DISPATCH_QUEUE_SERIAL);

dispatch_async(queue, ^{
    NSLog(@"async  & serial on %@", [NSThread currentThread]);// 子线程
});

dispatch_sync(queue, ^{
    NSLog(@"sync & serial on %@", [NSThread currentThread]);// 主线程
});

// 从结果看，同步操作在当前线程执行，异步操作在子线程执行。
```
2. 延迟执行 dispatch_after

```
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            
});
```
3. 分组  dispatch_group
```

dispatch_group_t group = dispatch_group_create();

dispatch_group_async(group, dispatch_get_global_queue(0, 0), ^{
    NSLog(@"group 1");
});

dispatch_group_async(group, dispatch_get_global_queue(0, 0), ^{
    NSLog(@"group 2");
    sleep(2);
});


dispatch_group_notify(group, dispatch_get_main_queue(), ^{
   NSLog(@"group finish");
});
```

4. 栅栏 dispatch_barrier
```
// barrier block 将对列前后的操作分隔开来。可以与group一起使用。
dispatch_group_t group = dispatch_group_create();
dispatch_queue_t queue = dispatch_queue_create("", DISPATCH_QUEUE_CONCURRENT);

dispatch_group_async(group, queue, ^{
    NSLog(@"group 1");
});

dispatch_group_async(group, queue, ^{
    NSLog(@"group 2");
    sleep(2);
});

// 不能使用global queue。只能使用自定义并行对列。系统queue不是谁都可以拦截的。
dispatch_barrier_async(queue, ^{
    NSLog(@"====== 分隔 ======");
});

dispatch_group_async(group, queue, ^{
    NSLog(@"group 3");
});

dispatch_group_notify(group, dispatch_get_main_queue(), ^{
   NSLog(@"group finish");
});

```

5. 批量 dispatch_apply

```
// 属于会阻塞当前线程
dispatch_apply(10, dispatch_get_global_queue(0, 0), ^(size_t index) {
    NSLog(@"第%ld次添加", index);
    sleep(1);
});

NSLog(@"end");// 最后输出
```

6. 暂停与恢复 suspend resume

```
dispatch_async(queue, ^{
    NSLog(@"第1次添加");
    sleep(1);
});

dispatch_async(queue, ^{
    NSLog(@"第2次添加");
    sleep(1);
});

dispatch_async(queue, ^{
    NSLog(@"第3次添加");
    sleep(1);
});

dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{

    dispatch_suspend(queue);
    sleep(3);
    dispatch_resume(queue);
});
```

7. 信号量 semaphore

semaphore 实现两个线程交替执行。
```

dispatch_queue_t queue1 = dispatch_queue_create("com.k2hi.1", DISPATCH_QUEUE_SERIAL);
dispatch_queue_t queue2 = dispatch_queue_create("com.k2hi.2", DISPATCH_QUEUE_SERIAL);


dispatch_semaphore_t sem = dispatch_semaphore_create(1);
//
dispatch_async(queue1, ^{
    while (1) {
        dispatch_semaphore_wait(sem, DISPATCH_TIME_FOREVER);
        NSLog(@"A 报数");
        sleep(1);
        dispatch_semaphore_signal(sem);
    }
});

dispatch_async(queue2, ^{
    while (1) {
        dispatch_semaphore_wait(sem, DISPATCH_TIME_FOREVER);
        NSLog(@"B 报数");
        sleep(1);
        dispatch_semaphore_signal(sem);
    }
});
```
8. 单次执行 dispatch_once

```
static dispatch_once_t onceToken;
dispatch_once(&onceToken, ^{
    [NSObject alloc] init];
});
```

9. 使用dispatch_source实现定时器

如果dispatch_source_set_event_handler中没有强引用timer。则需要外界强引用timer，否则timer超出作用域后无法执行。

```
__block int seconds = 10;

dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, dispatch_get_main_queue());

dispatch_source_set_timer(timer, DISPATCH_TIME_NOW, 1 * NSEC_PER_SEC, 0 * NSEC_PER_SEC);

dispatch_source_set_event_handler(timer, ^{
    NSLog(@"倒计时%d", seconds--);

    if (seconds < 0) {
       dispatch_cancel(timer);
    }
});
dispatch_resume(timer);
```

## Runloop

从某种角度来说，Runloop是线程同步锁的升级版通用版。

Runloop是一种高级循环，普通的循环会一直占用cup，Runloop会在合适的时候调用线程休眠。并根据Input和Timer两种方式唤醒执行任务。Runloop 是跟线程一起工作的，主要任务是控制线程生命周期，让线程更有效的工作。可以理解为线程的控制器。在线程的高级使用中是必不可少的一部分。在其他语言系统中都有类似功能模块。

理解Runloop首先应该了解他的一些基本概念和结构。例如输入源、定时器、监控者、运行模式等。

官方结构图：
![runloop]({{site.baseurl}}/images/runloop.jpg)

Runloop启动后，循环不断的处理数据源锁触发的的事件。当全部任务处理完成后Runloop停止运行，相应的线程也会推出执行。


Runloop Sources
Runloop 执行事件是通过Source来添加的。分类如下：

#### Input sources
	- Port-Based Sources，系统底层的 Port 事件，例如 CFSocketRef ，在应用层基本用不到
	- Custom Input Sources，用户手动创建的 Source
	- Cocoa Perform Selector Sources， Cocoa 提供的 performSelector 系列方法，也是一种事件源
Timer sources
	- NSTimer

#### Runloop Observer

Observer 是Runloop给我提供的一个状态监控接口。让我们可以了解Runloop的运行状态。就像ICMP协议对IP/TCP的作用。

监控的主要状态变化如下：

The entrance to the run loop.
When the run loop is about to process a timer.
When the run loop is about to process an input source.
When the run loop is about to go to sleep.
When the run loop has woken up, but before it has processed the event that woke it up.
The exit from the run loop.

#### Runloop Mode

![runloop mode]({{site.baseurl}}/images/runloop-mode.png)

如图所示，Runloop Mode 实际上是 Source，Timer 和 Observer 的集合，不同的 Mode 把不同组的 Source，Timer 和 Observer 隔绝开来。Runloop可以拥有多个Mode，但是 Runloop 在某个时刻只能跑在一个 Mode 下，处理这一个 Mode 当中的 Source，Timer 和 Observer。

苹果文档中提到的 Mode 有五个，分别是：

- NSDefaultRunLoopMode
- NSConnectionReplyMode
- NSModalPanelRunLoopMode
- NSEventTrackingRunLoopMode
- NSRunLoopCommonModes

iOS 中公开暴露出来的只有 `NSDefaultRunLoopMode` 和 `NSRunLoopCommonModes`。 NSRunLoopCommonModes 实际上是一个 Mode 的集合，默认包括 NSDefaultRunLoopMode 和 NSEventTrackingRunLoopMode。

## Runloop 使用

由于CFRunloop的代码开源的，且NSRunloop只是CFRunloop的简单封装。所以原理分析在CFRunloop的基础上进行阐释。

可以在这个地址得到CFCoreFoundation的源代码：https://github.com/opensource-apple/CF  

首先可以看下CFRunloop.h中的api，是不是和我们理解的概念想呼应。

可以发现相关概念都是由结构体指针表示：
```
typedef struct __CFRunLoop * CFRunLoopRef;// Runloop
typedef struct __CFRunLoopSource * CFRunLoopSourceRef;// Source
typedef struct __CFRunLoopObserver * CFRunLoopObserverRef;// Observer
typedef struct CF_BRIDGED_MUTABLE_TYPE(NSTimer) __CFRunLoopTimer * CFRunLoopTimerRef;// Timer
```

Observer 所对应的各个阶段：
```
/* Run Loop Observer Activities */
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),
    kCFRunLoopBeforeTimers = (1UL << 1),
    kCFRunLoopBeforeSources = (1UL << 2),
    kCFRunLoopBeforeWaiting = (1UL << 5),
    kCFRunLoopAfterWaiting = (1UL << 6),
    kCFRunLoopExit = (1UL << 7),
    kCFRunLoopAllActivities = 0x0FFFFFFFU
};
```

Runloop 启动、唤醒、停止：
```
CF_EXPORT void CFRunLoopRun(void);
CF_EXPORT SInt32 CFRunLoopRunInMode(CFStringRef mode, CFTimeInterval seconds, Boolean returnAfterSourceHandled);
CF_EXPORT Boolean CFRunLoopIsWaiting(CFRunLoopRef rl);
CF_EXPORT void CFRunLoopWakeUp(CFRunLoopRef rl);
CF_EXPORT void CFRunLoopStop(CFRunLoopRef rl);
```

数据源的管理：
```
CF_EXPORT Boolean CFRunLoopContainsSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFStringRef mode);
CF_EXPORT void CFRunLoopAddSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFStringRef mode);
CF_EXPORT void CFRunLoopRemoveSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFStringRef mode);

CF_EXPORT Boolean CFRunLoopContainsObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFStringRef mode);
CF_EXPORT void CFRunLoopAddObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFStringRef mode);
CF_EXPORT void CFRunLoopRemoveObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFStringRef mode);

CF_EXPORT Boolean CFRunLoopContainsTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFStringRef mode);
CF_EXPORT void CFRunLoopAddTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFStringRef mode);
CF_EXPORT void CFRunLoopRemoveTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFStringRef mode);
```

实现一个小Demo来测试Runtime的相关功能。

用线程表示一个士兵。士兵可以在DefaultMode下训练和成长，在FightingMode下战斗和成长。随着战斗的不断进行，士兵会在两种模式下切换。最终士兵会战死（比较残忍的设计）！死亡时士兵年龄即为本次游戏得分。主线程给士兵派发战斗任务，士兵切换到战斗模式执行战斗，战斗结束后回到训练模式。随着训练的进行士兵战斗力不断增强。


```
//
//  main.m
//  pthreadsDemo
//
//  Created by liushixiang on 2020/8/13.
//  Copyright © 2020 liushixiang. All rights reserved.
//

#import <Foundation/Foundation.h>
#include <pthread.h>

// 战争模拟小游戏《活到最后》

const CFStringRef kFightingMode = CFSTR("FightingMode");


BOOL Fighting = NO;//  战斗模式
BOOL Dead = NO;// 士兵死亡

int Age = 0;
int Strong = 10;// 战斗力


#pragma mark - 士兵线程

void train(){
    printf("💂士兵训练(%d)\n", ++Strong);
}

void grow(){
    Age++;
}

void fight(void *arg){
    
    printf("                    🤜🤜 战斗开始 🤛🤛\n");
    int random = arc4random() % 5;
    int enemy = random - (Strong % 5 + (Strong / 10) % 2 ) + Strong;// 随机战斗力的敌人
    
    if (Strong > enemy) {
        printf("                       💂胜利✌️✌️✌️( %d VS %d )\n", Strong, enemy);
        Strong++;
    }else{
        printf("                       💂失败😭😭😭( %d VS %d )\n", Strong, enemy);
        Dead = YES;
    }
    Fighting = NO;
}

void enterFight(void *arg){
    
    static CFRunLoopSourceRef source;
    if (source == NULL){
        CFRunLoopSourceContext context = {0,CFRunLoopGetCurrent(),NULL,NULL,NULL,NULL,NULL,NULL,NULL,fight};
        source = CFRunLoopSourceCreate(kCFAllocatorDefault, 0, &context);
        CFRunLoopAddSource(CFRunLoopGetCurrent(), source, kFightingMode);
    }
    
    CFRunLoopSourceSignal(source);
    CFRunLoopWakeUp(CFRunLoopGetCurrent());
    
    Fighting = YES;
    CFRunLoopStop(CFRunLoopGetCurrent());
}

/// 创建士兵
void* createFighter (void *arg){
    
    // 添加observer
    CFRunLoopObserverRef observer = CFRunLoopObserverCreateWithHandler(CFAllocatorGetDefault(), kCFRunLoopAllActivities, true, 0, ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {
        
        char *msg;
        switch (activity) {
            case kCFRunLoopEntry:
            {
                msg = "RunLoop进入";
            }
                break;
            case kCFRunLoopBeforeTimers:
                msg = "RunLoop将触发定时器";
                break;
            case kCFRunLoopBeforeSources:
                msg = "RunLoop将触发Source";
                break;
            case kCFRunLoopBeforeWaiting:
                msg = "RunLoop将睡眠";
                break;
            case kCFRunLoopAfterWaiting:
                msg = "RunLoop唤醒";
                break;
            case kCFRunLoopExit:
                msg = "RunLoop退出";
                break;
            default:
                msg = "Unknown";
        }
//        printf("(%s)\n",msg);
    });
    CFRunLoopAddObserver(CFRunLoopGetCurrent(), observer, kFightingMode);
    
    // 训练Timer
    {
        CFRunLoopTimerContext timerContext = {0, NULL, NULL, NULL, NULL};
        CFRunLoopTimerRef timer = CFRunLoopTimerCreate(kCFAllocatorDefault, 0.1, 3, 0, 0, train, &timerContext);
        CFRunLoopAddTimer(CFRunLoopGetCurrent(), timer, kCFRunLoopDefaultMode);
    }
    
    // 年龄Timer - 年龄在两种模式中都应该增加，所以加入CommonModes
    CFRunLoopAddCommonMode(CFRunLoopGetCurrent(), kFightingMode);
    {
        CFRunLoopTimerContext timerContext = {0, NULL, NULL, NULL, NULL};
        CFRunLoopTimerRef timer = CFRunLoopTimerCreate(kCFAllocatorDefault, 0.1, 3, 0, 0, grow, &timerContext);
        CFRunLoopAddTimer(CFRunLoopGetCurrent(), timer, kCFRunLoopCommonModes);
    }
    
    // 进入战斗状态Handler
    {
        CFRunLoopSourceContext context = {0,CFRunLoopGetCurrent(),NULL,NULL,NULL,NULL,NULL,NULL,NULL,enterFight};
        CFRunLoopSourceRef source = CFRunLoopSourceCreate(kCFAllocatorDefault, 0, &context);
        CFRunLoopAddSource(CFRunLoopGetCurrent(), source, kCFRunLoopDefaultMode);
        *((void **)arg) = source;
    }

    while (!Dead) {
        if (Fighting) {
            //进入战斗模式
            CFRunLoopRunInMode(kFightingMode, DBL_MAX, true);// true:执行完source0立即退出
        }else{
            // 进入训练模式
            CFRunLoopRunInMode(kCFRunLoopDefaultMode, DBL_MAX, false);
        }
    }
    printf("得分：%d\n", Age);
    CFRunLoopStop(CFRunLoopGetMain());
    return NULL;
}

#pragma mark - 主线程

/// 时局动荡，战争一触即发
void prefightHandler(CFRunLoopTimerRef timer, void *info){
    
    CFRunLoopSourceRef source = (CFRunLoopSourceRef)info;
    CFRunLoopSourceContext context = {0,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL};
    CFRunLoopSourceGetContext(source, &context);
    CFRunLoopRef runloop = (CFRunLoopRef)context.info;
    
    NSInteger flag = arc4random() % 5;
    if (flag < 2) {
        printf("🔥🔥🔥战斗爆发🔥🔥🔥\n");
        if (CFRunLoopSourceIsValid(source)) {
            CFRunLoopSourceSignal(source);
            CFRunLoopWakeUp(runloop);
        }
    }else{
        printf("💬💬💬和平谈判💬💬💬\n");
    }
}


int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        printf("=================== game start ===================\n");
        
        // 启动士兵线程
        CFRunLoopSourceRef source;
        pthread_t thread;
        pthread_create(&thread, NULL, createFighter, &source);// 传递指针的指针
        pthread_detach(thread);
        
        sleep(3);// 等待source异步生成，sleep不够高级，使用锁比较好，但会使demo复杂。
        
        // 给士兵添加随机战斗，使其在多个RunloopMode中切换
        CFRunLoopTimerContext timerContext = {0, source, NULL, NULL, NULL};
        CFRunLoopTimerRef timer = CFRunLoopTimerCreate(kCFAllocatorDefault, 0.1, 7, 0, 0,
                                                       prefightHandler, &timerContext);
        CFRunLoopAddTimer(CFRunLoopGetCurrent(), timer, kCFRunLoopDefaultMode);
        
        CFRunLoopRun();
        printf("=================== game over ===================\n\n");
    }
    return 0;
}

```

## Runloop 源码解析


Runloop 是什么？探究Runloop的来历。
```
CFRunLoopRef CFRunLoopGetMain(void) {
    CHECK_FOR_FORK();
    static CFRunLoopRef __main = NULL; // no retain needed
    if (!__main) __main = _CFRunLoopGet0(pthread_main_thread_np()); // no CAS needed
    return __main;
}

CFRunLoopRef CFRunLoopGetCurrent(void) {
    CHECK_FOR_FORK();
    CFRunLoopRef rl = (CFRunLoopRef)_CFGetTSD(__CFTSDKeyRunLoop);
    if (rl) return rl;
    return _CFRunLoopGet0(pthread_self());
}
// 无论是主线程Runloop还是子线程Runloop，在没有取到的时候都指向了同一个函数CFRunloopGet0. 因此这个函数会返回一个新生成的Runloop。

```

看一下这个CFRunloopGet0的实现方式。
```

static CFMutableDictionaryRef __CFRunLoops = NULL;

// should only be called by Foundation
// t==0 is a synonym for "main thread" that always works
CF_EXPORT CFRunLoopRef _CFRunLoopGet0(pthread_t t) {
    if (pthread_equal(t, kNilPthreadT)) {
	t = pthread_main_thread_np();
    }
    __CFLock(&loopsLock);
    // 使用全局字典__CFRunLoops保存线程和Runloop的映射。
    if (!__CFRunLoops) {
        __CFUnlock(&loopsLock);
	CFMutableDictionaryRef dict = CFDictionaryCreateMutable(kCFAllocatorSystemDefault, 0, NULL, &kCFTypeDictionaryValueCallBacks);
	// 主线程跟随全局字典一同创建。
	CFRunLoopRef mainLoop = __CFRunLoopCreate(pthread_main_thread_np());
	CFDictionarySetValue(dict, pthreadPointer(pthread_main_thread_np()), mainLoop);
	if (!OSAtomicCompareAndSwapPtrBarrier(NULL, dict, (void * volatile *)&__CFRunLoops)) {
	    CFRelease(dict);
	}
	CFRelease(mainLoop);
        __CFLock(&loopsLock);
    }
    CFRunLoopRef loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
    __CFUnlock(&loopsLock);
    if (!loop) {
	CFRunLoopRef newLoop = __CFRunLoopCreate(t);// Runloop 的最终创建函数，一会看一下内部实现。这里的创建并没有加锁，因为多个线程的创建并不互相影响。
        __CFLock(&loopsLock);
	loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
	if (!loop) {
	    CFDictionarySetValue(__CFRunLoops, pthreadPointer(t), newLoop);
	    loop = newLoop;
	}
        // don't release run loops inside the loopsLock, because CFRunLoopDeallocate may end up taking it
        __CFUnlock(&loopsLock);
	CFRelease(newLoop);
    }
    
    if (pthread_equal(t, pthread_self())) {
        _CFSetTSD(__CFTSDKeyRunLoop, (void *)loop, NULL);
        if (0 == _CFGetTSD(__CFTSDKeyRunLoopCntr)) {
            _CFSetTSD(__CFTSDKeyRunLoopCntr, (void *)(PTHREAD_DESTRUCTOR_ITERATIONS-1), (void (*)(void *))__CFFinalizeRunLoop);
        }
    }
    return loop;
}
```

Runloop的内存创建
```
static CFRunLoopRef __CFRunLoopCreate(pthread_t t) {
    CFRunLoopRef loop = NULL;
    CFRunLoopModeRef rlm;
    uint32_t size = sizeof(struct __CFRunLoop) - sizeof(CFRuntimeBase);
    loop = (CFRunLoopRef)_CFRuntimeCreateInstance(kCFAllocatorSystemDefault, CFRunLoopGetTypeID(), size, NULL);// 开辟普通CF对象内存空间
    if (NULL == loop) {
	return NULL;
    }
    (void)__CFRunLoopPushPerRunData(loop);
    __CFRunLoopLockInit(&loop->_lock);
    loop->_wakeUpPort = __CFPortAllocate();
    if (CFPORT_NULL == loop->_wakeUpPort) HALT;
    __CFRunLoopSetIgnoreWakeUps(loop);
    loop->_commonModes = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
    CFSetAddValue(loop->_commonModes, kCFRunLoopDefaultMode);// DefaultMode添加进CommonModes
    loop->_commonModeItems = NULL;
    loop->_currentMode = NULL;
    loop->_modes = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
    loop->_blocks_head = NULL;
    loop->_blocks_tail = NULL;
    loop->_counterpart = NULL;
    loop->_pthread = t;// Runloop 内部保留所在线程的指向。
#if DEPLOYMENT_TARGET_WINDOWS
    loop->_winthread = GetCurrentThreadId();
#else
    loop->_winthread = 0;
#endif
    rlm = __CFRunLoopFindMode(loop, kCFRunLoopDefaultMode, true);
    if (NULL != rlm) __CFRunLoopModeUnlock(rlm);
    return loop;
}
```

Runloop 如何运行的？
```
// 使用这种方式运行无需配合While循环，因为没有返回值。
void CFRunLoopRun(void) {	/* DOES CALLOUT */
    int32_t result;
    do {
        result = CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
        CHECK_FOR_FORK();
    } while (kCFRunLoopRunStopped != result && kCFRunLoopRunFinished != result);
}

// 这种方式运行可以配合While循环，就像上边《活到最后》demo中一样。可以添加自定义的运行条件，做到更细致的控制。
SInt32 CFRunLoopRunInMode(CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
    CHECK_FOR_FORK();
    return CFRunLoopRunSpecific(CFRunLoopGetCurrent(), modeName, seconds, returnAfterSourceHandled);
}
```

无论哪种方式最后都调用了一个内部细节函数：CFRunLoopRunSpecific

```
SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
    CHECK_FOR_FORK();
    if (__CFRunLoopIsDeallocating(rl)) return kCFRunLoopRunFinished;
    __CFRunLoopLock(rl);

    // 首先判断当前模式是否存在以及当前模式中是否有要执行的Items（Source 和 Timer）。没有则Runloop直接return退出。
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(rl, modeName, false);
    if (NULL == currentMode || __CFRunLoopModeIsEmpty(rl, currentMode, rl->_currentMode)) {
        Boolean did = false;
        if (currentMode) __CFRunLoopModeUnlock(currentMode);
        __CFRunLoopUnlock(rl);
        return did ? kCFRunLoopRunHandledSource : kCFRunLoopRunFinished;
    }

    volatile _per_run_data *previousPerRun = __CFRunLoopPushPerRunData(rl);
    CFRunLoopModeRef previousMode = rl->_currentMode;
    rl->_currentMode = currentMode;
    int32_t result = kCFRunLoopRunFinished;
    
    // Observer 进入通知
    if (currentMode->_observerMask & kCFRunLoopEntry ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
    // 结合理论知识和上下两句通知的发出，__CFRunLoopRun()应该就是线程一直循环执行不退出的函数。就是Runloop真正loop的地方。
    result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
    // Observer 退出通知
    if (currentMode->_observerMask & kCFRunLoopExit ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);
    
    __CFRunLoopModeUnlock(currentMode);
    __CFRunLoopPopPerRunData(rl, previousPerRun);
    rl->_currentMode = previousMode;
    __CFRunLoopUnlock(rl);
    return result;
}
```

Runloop睡眠唤醒的逻辑分析

```
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode) {
    uint64_t startTSR = mach_absolute_time();


    if (__CFRunLoopIsStopped(rl)) {
        __CFRunLoopUnsetStopped(rl);
	return kCFRunLoopRunStopped;
    } else if (rlm->_stopped) {
	rlm->_stopped = false;
	return kCFRunLoopRunStopped;
    }
    
    mach_port_name_t dispatchPort = MACH_PORT_NULL;
    Boolean libdispatchQSafe = pthread_main_np() && ((HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && NULL == previousMode) || (!HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && 0 == _CFGetTSD(__CFTSDKeyIsInGCDMainQ)));
    if (libdispatchQSafe && (CFRunLoopGetMain() == rl) && CFSetContainsValue(rl->_commonModes, rlm->_name)) dispatchPort = _dispatch_get_main_queue_port_4CF();
    
#if USE_DISPATCH_SOURCE_FOR_TIMERS
    mach_port_name_t modeQueuePort = MACH_PORT_NULL;
    if (rlm->_queue) {
        modeQueuePort = _dispatch_runloop_root_queue_get_port_4CF(rlm->_queue);
        if (!modeQueuePort) {
            CRASH("Unable to get port for run loop mode queue (%d)", -1);
        }
    }
#endif
    
    dispatch_source_t timeout_timer = NULL;
    struct __timeout_context *timeout_context = (struct __timeout_context *)malloc(sizeof(*timeout_context));
    if (seconds <= 0.0) { // instant timeout
        seconds = 0.0;
        timeout_context->termTSR = 0ULL;
    } else if (seconds <= TIMER_INTERVAL_LIMIT) {
    // 可以发现Runloop运行的时间限制是由dispatch_source实现的
	dispatch_queue_t queue = pthread_main_np() ? __CFDispatchQueueGetGenericMatchingMain() : __CFDispatchQueueGetGenericBackground();
	timeout_timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
        dispatch_retain(timeout_timer);
	timeout_context->ds = timeout_timer;
	timeout_context->rl = (CFRunLoopRef)CFRetain(rl);
	timeout_context->termTSR = startTSR + __CFTimeIntervalToTSR(seconds);
	dispatch_set_context(timeout_timer, timeout_context); // source gets ownership of context
	dispatch_source_set_event_handler_f(timeout_timer, __CFRunLoopTimeout);
        dispatch_source_set_cancel_handler_f(timeout_timer, __CFRunLoopTimeoutCancel);
        uint64_t ns_at = (uint64_t)((__CFTSRToTimeInterval(startTSR) + seconds) * 1000000000ULL);
        dispatch_source_set_timer(timeout_timer, dispatch_time(1, ns_at), DISPATCH_TIME_FOREVER, 1000ULL);
        dispatch_resume(timeout_timer);
    } else { // infinite timeout
        seconds = 9999999999.0;
        timeout_context->termTSR = UINT64_MAX;
    }

    Boolean didDispatchPortLastTime = true;
    int32_t retVal = 0;

    // Runloop 睡眠-唤醒 循环开始 
    do {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        voucher_mach_msg_state_t voucherState = VOUCHER_MACH_MSG_STATE_UNCHANGED;
        voucher_t voucherCopy = NULL;
#endif
        uint8_t msg_buffer[3 * 1024];
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        mach_msg_header_t *msg = NULL;
        mach_port_t livePort = MACH_PORT_NULL;
#elif DEPLOYMENT_TARGET_WINDOWS
        HANDLE livePort = NULL;
        Boolean windowsMessageReceived = false;
#endif
	__CFPortSet waitSet = rlm->_portSet;

        __CFRunLoopUnsetIgnoreWakeUps(rl);

        // 发送通知：将要处理TimerSource
        if (rlm->_observerMask & kCFRunLoopBeforeTimers) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);

        // 发送通知：将要处理Input Source
        if (rlm->_observerMask & kCFRunLoopBeforeSources) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);

        // 可以发现两个通知的发送是仅接着的，属于异步的通知，Source 的通知并没有等待Timer的真正处理。


        // 执行 CFRunLoopPerformBlock 添加的block；
	__CFRunLoopDoBlocks(rl, rlm);


        //  加入 Input Source 0 到待执行
        Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
        if (sourceHandledThisLoop) {
            __CFRunLoopDoBlocks(rl, rlm);//  执行source 0 
        }

        
        Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR);
        if (MACH_PORT_NULL != dispatchPort && !didDispatchPortLastTime) {
            msg = (mach_msg_header_t *)msg_buffer;
            if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0, &voucherState, NULL)) {
                goto handle_msg;// Runloop 跳转：执行 mach port 任务；不让Runloop进入睡眠模式
            }
        }

        didDispatchPortLastTime = false;


    // 发送通知：将要进入睡眠状态
	if (!poll && (rlm->_observerMask & kCFRunLoopBeforeWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
	__CFRunLoopSetSleeping(rl);
	// do not do any user callouts after this point (after notifying of sleeping)

        // Must push the local-to-this-activation ports in on every loop
        // iteration, as this mode could be run re-entrantly and we don't
        // want these ports to get serviced.

        __CFPortSetInsert(dispatchPort, waitSet);
	__CFRunLoopModeUnlock(rlm);
	__CFRunLoopUnlock(rl);

        CFAbsoluteTime sleepStart = poll ? 0.0 : CFAbsoluteTimeGetCurrent();

#if USE_DISPATCH_SOURCE_FOR_TIMERS
        do {
            if (kCFUseCollectableAllocator) {
                // objc_clear_stack(0);
                // <rdar://problem/16393959>
                memset(msg_buffer, 0, sizeof(msg_buffer));
            }
            msg = (mach_msg_header_t *)msg_buffer;
            // 进入睡眠状态：在唤醒之前这个函数不会返回，唤醒后词函数返回
            __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy);
            
            if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {
                // Drain the internal queue. If one of the callout blocks sets the timerFired flag, break out and service the timer.
                while (_dispatch_runloop_root_queue_perform_4CF(rlm->_queue));
                if (rlm->_timerFired) {
                    // Leave livePort as the queue port, and service timers below
                    rlm->_timerFired = false;
                    break;
                } else {
                    if (msg && msg != (mach_msg_header_t *)msg_buffer) free(msg);
                }
            } else {
                // Go ahead and leave the inner loop.
                break;
            }
        } while (1);
#else
        if (kCFUseCollectableAllocator) {
            // objc_clear_stack(0);
            // <rdar://problem/16393959>
            memset(msg_buffer, 0, sizeof(msg_buffer));
        }
        msg = (mach_msg_header_t *)msg_buffer;
        __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy);
#endif
        
        __CFRunLoopLock(rl);
        __CFRunLoopModeLock(rlm);

        rl->_sleepTime += (poll ? 0.0 : (CFAbsoluteTimeGetCurrent() - sleepStart));

        // Must remove the local-to-this-activation ports in on every loop
        // iteration, as this mode could be run re-entrantly and we don't
        // want these ports to get serviced. Also, we don't want them left
        // in there if this function returns.

        __CFPortSetRemove(dispatchPort, waitSet);
        __CFRunLoopSetIgnoreWakeUps(rl);

        // user callouts now OK again
	__CFRunLoopUnsetSleeping(rl);

    // 发送通知：Runloop 已经唤醒
	if (!poll && (rlm->_observerMask & kCFRunLoopAfterWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting);

        handle_msg:;// 跳转标志
        __CFRunLoopSetIgnoreWakeUps(rl);

        if (MACH_PORT_NULL == livePort) {
            CFRUNLOOP_WAKEUP_FOR_NOTHING();
            // handle nothing
        } else if (livePort == rl->_wakeUpPort) {
            CFRUNLOOP_WAKEUP_FOR_WAKEUP();
            // do nothing on Mac OS
        }
#if USE_DISPATCH_SOURCE_FOR_TIMERS
        else if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {
            // 执行定时器Timer任务，第一种方式
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) {
                // Re-arm the next timer, because we apparently fired early
                __CFArmNextTimerInMode(rlm, rl);
            }
        }
#endif
#if USE_MK_TIMER_TOO
        else if (rlm->_timerPort != MACH_PORT_NULL && livePort == rlm->_timerPort) {
            // 执行定时器Timer任务，第二种方式
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            // On Windows, we have observed an issue where the timer port is set before the time which we requested it to be set. For example, we set the fire time to be TSR 167646765860, but it is actually observed firing at TSR 167646764145, which is 1715 ticks early. The result is that, when __CFRunLoopDoTimers checks to see if any of the run loop timers should be firing, it appears to be 'too early' for the next timer, and no timers are handled.
            // In this case, the timer port has been automatically reset (since it was returned from MsgWaitForMultipleObjectsEx), and if we do not re-arm it, then no timers will ever be serviced again unless something adjusts the timer list (e.g. adding or removing timers). The fix for the issue is to reset the timer here if CFRunLoopDoTimers did not handle a timer itself. 9308754
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) {
                // Re-arm the next timer
                __CFArmNextTimerInMode(rlm, rl);
            }
        }
#endif
        else if (livePort == dispatchPort) {
            // 执行Dispatch 任务
            CFRUNLOOP_WAKEUP_FOR_DISPATCH();
            __CFRunLoopModeUnlock(rlm);
            __CFRunLoopUnlock(rl);
            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)6, NULL);

            __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)0, NULL);
            __CFRunLoopLock(rl);
            __CFRunLoopModeLock(rlm);
            sourceHandledThisLoop = true;
            didDispatchPortLastTime = true;
        } else {
            // 执行 Source1 任务
            CFRUNLOOP_WAKEUP_FOR_SOURCE();
            
            // If we received a voucher from this mach_msg, then put a copy of the new voucher into TSD. CFMachPortBoost will look in the TSD for the voucher. By using the value in the TSD we tie the CFMachPortBoost to this received mach_msg explicitly without a chance for anything in between the two pieces of code to set the voucher again.
            voucher_t previousVoucher = _CFSetTSD(__CFTSDKeyMachMessageHasVoucher, (void *)voucherCopy, os_release);

            // Despite the name, this works for windows handles as well
            CFRunLoopSourceRef rls = __CFRunLoopModeFindSourceForMachPort(rl, rlm, livePort);
            if (rls) {

		mach_msg_header_t *reply = NULL;
		sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls, msg, msg->msgh_size, &reply) || sourceHandledThisLoop;
		if (NULL != reply) {
		    (void)mach_msg(reply, MACH_SEND_MSG, reply->msgh_size, 0, MACH_PORT_NULL, 0, MACH_PORT_NULL);
		    CFAllocatorDeallocate(kCFAllocatorSystemDefault, reply);
		}

	    }
            
            // Restore the previous voucher
            _CFSetTSD(__CFTSDKeyMachMessageHasVoucher, previousVoucher, os_release);
            
        } 
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        if (msg && msg != (mach_msg_header_t *)msg_buffer) free(msg);
#endif
        
	__CFRunLoopDoBlocks(rl, rlm);
        

	if (sourceHandledThisLoop && stopAfterHandle) {
	    retVal = kCFRunLoopRunHandledSource;
        } else if (timeout_context->termTSR < mach_absolute_time()) {
            retVal = kCFRunLoopRunTimedOut;
	} else if (__CFRunLoopIsStopped(rl)) {
            __CFRunLoopUnsetStopped(rl);
	    retVal = kCFRunLoopRunStopped;
	} else if (rlm->_stopped) {
	    rlm->_stopped = false;
	    retVal = kCFRunLoopRunStopped;
	} else if (__CFRunLoopModeIsEmpty(rl, rlm, previousMode)) {
	    retVal = kCFRunLoopRunFinished;
	}
        
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        voucher_mach_msg_revert(voucherState);
        os_release(voucherCopy);
#endif

    // 循环结束。判断是否具有退出条件
    } while (0 == retVal);

    if (timeout_timer) {
        dispatch_source_cancel(timeout_timer);
        dispatch_release(timeout_timer);
    } else {
        free(timeout_context);
    }

    return retVal;
}

```

## Runloop 实现的功能

实现的功能只有自己实际探究才有意义。只要能理解Runloop的原理，下面这些应用也能猜出个大概。由于还没有实际探究过，不做详细说明，只列出一个列表，方便以后研究。
（多数是来自“深入理解Runloop”复制的）

#### 自动释放池AutoreleasePool

App启动后，苹果在主线程 RunLoop 里注册了两个 Observer，其回调都是 _wrapRunLoopWithAutoreleasePoolHandler()。

第一个 Observer 监视的事件是 Entry(即将进入Loop)，其回调内会调用 _objc_autoreleasePoolPush() 创建自动释放池。其 order 是-2147483647，优先级最高，保证创建释放池发生在其他所有回调之前。

第二个 Observer 监视了两个事件： BeforeWaiting(准备进入休眠) 时调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池；Exit(即将退出Loop) 时调用 _objc_autoreleasePoolPop() 来释放自动释放池。这个 Observer 的 order 是 2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。

在主线程执行的代码，通常是写在诸如事件回调、Timer回调内的。这些回调会被 RunLoop 创建好的 AutoreleasePool 环绕着，所以不会出现内存泄漏，开发者也不必显示创建 Pool 了。

#### 事件相应

苹果注册了一个 Source1 (基于 mach port 的) 用来接收系统事件，其回调函数为 __IOHIDEventSystemClientQueueCallback()。

当一个硬件事件(触摸/锁屏/摇晃等)发生后，首先由 IOKit.framework 生成一个 IOHIDEvent 事件并由 SpringBoard 接收。这个过程的详细情况可以参考这里。SpringBoard 只接收按键(锁屏/静音等)，触摸，加速，接近传感器等几种 Event，随后用 mach port 转发给需要的App进程。随后苹果注册的那个 Source1 就会触发回调，并调用 _UIApplicationHandleEventQueue() 进行应用内部的分发。

_UIApplicationHandleEventQueue() 会把 IOHIDEvent 处理并包装成 UIEvent 进行处理或分发，其中包括识别 UIGesture/处理屏幕旋转/发送给 UIWindow 等。通常事件比如 UIButton 点击、touchesBegin/Move/End/Cancel 事件都是在这个回调中完成的。


#### 手势识别
#### 界面更新
#### 定时器
#### PerfomSelecter
#### GCD
#### 网络请求
