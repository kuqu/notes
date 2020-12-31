
# iOS多线程（一） - Pthread与线程同步

{docsify-updated}

POSIX线程（英语：POSIX Threads，常被缩写为Pthreads）是POSIX的线程标准，定义了创建和操纵线程的一套API。

Pthreads API中大致共有100个函数调用，全都以"pthread_"开头，并可以分为四类：
* 线程管理，例如创建线程，等待(join)线程，查询线程状态等。
* 互斥锁（Mutex）：创建、摧毁、锁定、解锁、设置属性等操作
* 条件变量（Condition Variable）：创建、摧毁、等待、通知、设置与查询属性等操作
* 使用了互斥锁的线程间的同步管理

这么多的函数如果只是阅读理解，其实并不能完全掌握，需要再实践中不断巩固理解。对于面试而言阅读本文应该是有一定的帮助。

## 线程管理

相关函数：

```
pthread_create() 创建一个线程
pthread_exit() 终止当前线程
pthread_cancel() 中断另外一个线程的运行
pthread_join() 阻塞当前的线程，直到另外一个线程运行结束
pthread_detach() 不阻塞当前线程，分离子线程，线程运行结束后会自动释放所有资源
pthread_kill() 向线程发送一个信号


pthread_attr_init() 初始化线程的属性
pthread_attr_setdetachstate() 设置脱离状态的属性（决定这个线程在终止时是否可以被结合）
pthread_attr_getdetachstate() 获取脱离状态的属性
pthread_attr_destroy() 删除线程的属性

int pthread_equal(pthread_t tid1, pthread_t tid2) 比较两个pthread_t是否相等
pthread _t pthread_self(void);获得本线程的thread id

pthread_cleanup_push 设置线程结束时需要执行的清理函数

```


一个创建线程的例子：
```
// 1. 创建线程: 定义一个pthread_t类型变量
pthread_t thread;
// 2. 开启线程: 执行任务
pthread_create(&thread, NULL, run, NULL);
// 3. 设置子线程的状态设置为 detached，该线程运行结束后会自动释放所有资源
pthread_detach(thread);

void * run(void *param)    // 新线程调用方法，里边为需要执行的任务
{
    NSLog(@"%@", [NSThread currentThread]);

    return NULL;
}
```

## 互斥锁

对某个资源访问前后添加锁和解除锁，使得资源同一时间只有个线程可以访问。互斥锁的原始是将线程放到一个“等待队列”中，并同时操作线程的睡眠、唤醒状态呢。使得线程可控有序的访问资源。

互斥锁主要函数：
```
#include <pthread.h>

int pthread_mutex_init(pthread_mutex_t *restrict mutex,     
       const pthread_mutexattr_t *restrict attr);       /*初始化互斥量*/
int pthread_mutex_destroy(pthread_mutex_t *mutex);      /*销毁互斥量*/
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);

```

使用示例：
```
#include<stdio.h>
#include<string.h>
#include<stdlib.h>
#include<unistd.h>
#include<pthread.h>

#define LEN 100000
int num = 0;

void* thread_func(void* arg) {
    pthread_mutex_t* p_mutex = (pthread_mutex_t*)arg;
    for (int i = 0; i< LEN; ++i) {
        pthread_mutex_lock(p_mutex);
        num += 1;
        pthread_mutex_unlock(p_mutex);
    }
    
    return NULL;
}

int main() {
    pthread_mutex_t m_mutex;
    pthread_mutex_init(&m_mutex, NULL);

    pthread_t tid1, tid2;
    pthread_create(&tid1, NULL, (void*)thread_func, (void*)&m_mutex);
    pthread_create(&tid2, NULL, (void*)thread_func, (void*)&m_mutex);

    pthread_join(tid1, NULL);
    pthread_join(tid2, NULL);

    pthread_mutex_destroy(&m_mutex);

    printf("correct result=%d, result=%d.\n", 2*LEN, num);
    return 0;
}

```
#### 互斥锁产生死锁的两种情况：

同一线程调用两次加锁，造成死锁。第二次加锁时，由于互斥锁已经锁定，线程会等待互斥锁解锁，从而进入睡眠。而加锁的线程正是本线程，本线程已经进入睡眠，所以永远无法解锁。

互斥锁环，造成死锁。两个或者更多线程互相等待对方解锁，并且产生了依赖环。此时造成死锁。例如线程1获得了锁1，线程2获得了锁2。这是线程1获得锁2，线程2获得锁1。由于锁1锁2都处于加锁状态，所以线程1线程2都进入睡眠状态，此时产生死锁。

#### 如何避免死锁：

死锁多数时候是由第二种情况产生的。在使用多个锁的产生死锁时主要是因为形成了环形结构，造成了首尾相接的情况。所以一种避免方式是按顺序访问锁，即所有线程都按 锁“1> 锁2 > 锁3”的顺序访问。

1. 程以相同顺序加锁
2. `read_mutex_trylock` 加锁，若失败就放弃上锁，同时释放已占有的锁
3. `hread_mutex_timedlock` 加锁，若超时就放弃上锁，同时释放已占有的锁

## 条件变量

条件变量提供了一种多个线程相互通信的方式。


相关函数：

```
#include <pthread.h>

int pthread_cond_destroy(pthread_cond_t *cond);
int pthread_cond_init(pthread_cond_t *restrict cond,
       const pthread_condattr_t *restrict attr);
       
// 超时等待
int pthread_cond_timedwait(pthread_cond_t *restrict cond,
       pthread_mutex_t *restrict mutex,
       const struct timespec *restrict abstime);
int pthread_cond_wait(pthread_cond_t *restrict cond,
       pthread_mutex_t *restrict mutex);
       
// 向所有线程发消息
int pthread_cond_broadcast(pthread_cond_t *cond);
// 向第一个等待的线程发消息
int pthread_cond_signal(pthread_cond_t *cond);

```

简单示例：
条件变量需要结合 mutex一起使用。关于`pthread_cond_wait`的使用需要特别理解一下。
1. pthread_mutex_lock  加锁
2. pthread_cond_wait 这条语句包含两个动作。a.解锁上一步加锁的mutex b.阻塞线程，等待其他线程的条件变量信号。
3. 当收到其他线程的信号，条件变量满足时继续执行本线程。首先加锁mutex，然后运行到pthread_mutex_unlock解锁mutex。

```
/*
 * 结合使用条件变量和互斥锁进行线程同步
*/

#include <pthread.h>
#include <stdio.h>

static pthread_cond_t  cond;
static pthread_mutex_t mutex;
static int             cond_value;
static int             quit;

void *thread_signal(void *arg)
{
    while (!quit)
    {
        pthread_mutex_lock(&mutex);
        cond_value++;                //改变条件，使条件满足
        pthread_cond_signal(&cond);  //给线程发信号 
        printf("signal send, cond_value: %d\n", cond_value);
        pthread_mutex_unlock(&mutex);
        sleep(1);
    }    
}

void *thread_wait(void *arg)
{
    while (!quit)
    {
        pthread_mutex_lock(&mutex);
        
        /*通过while (cond is true)来保证从pthread_cond_wait成功返回时，调用线程会重新检查条件*/
        while (cond_value == 0)
            pthread_cond_wait(&cond, &mutex);
            
        cond_value--;
        printf("signal recv, cond_value: %d\n", cond_value);
        
        pthread_mutex_unlock(&mutex);
        sleep(1);
    } 
}

int main()
{     
    pthread_t tid1;
    pthread_t tid2;
    
    pthread_cond_init(&cond, NULL);
    pthread_mutex_init(&mutex, NULL);
       
    pthread_create(&tid1, NULL, thread_signal, NULL);   
    pthread_create(&tid2, NULL, thread_wait, NULL);
    
    sleep(5);
    quit = 1;
    
    pthread_join(tid1, NULL);
    pthread_join(tid2, NULL);
    
    pthread_cond_destroy(&cond);
    pthread_mutex_destroy(&mutex);
      
    return 0;
}
```

## 线程同步

线程同步应该算是多线程中比较重要的。在这里做一个比较全面的总结。

#### 互斥锁 mutex
保证同一资源一定时间内只能由一个线程访问。加锁解锁成对出现，并且不能连续加锁两次。
```
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;//静态初始化互斥量
pthread_mutex_lock(&p_mutex);// 加锁
num += 1;
pthread_mutex_unlock(&p_mutex);// 解锁
pthread_mutex_destory(&p_mutex);
```

#### 递归锁 pthread_mutex(recursive)
递归锁相比于普通互斥锁的不同点是：同一个线程可以多次加锁，且加锁次数和解锁次数应该相同。给普通mutex设置一个递归属性即可创建递归锁。
```
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);// PTHREAD_MUTEX_NORMAL 普通互斥锁。
pthread_mutex_init(mutex, &attr);
```

#### 条件变量 condition variable
等待在发送信号之前。需要使用互斥锁保证独占性。
使用条件变量的一个经典的例子就是线程池(Thread Pool)了。
使方式参考本文前边部分。


#### 信号量 semaphore.h

sem_wait 等待信号；
sem_post 发送信号；
sem_init 创建信号；
sem_destroy 销毁信号

From：https://www.geeksforgeeks.org/use-posix-semaphores-c/

当信号量减一后为负数时，wait起作用，线程等待。当信号量增加后，线程继续执行。
```
#include <stdio.h> 
#include <pthread.h> 
#include <semaphore.h> 
#include <unistd.h> 
  
sem_t mutex; 
  
void* thread(void* arg) 
{ 
    //wait 
    sem_wait(&mutex); 
    printf("\nEntered..\n"); 
  
    //critical section 
    sleep(4); 
      
    //signal 
    printf("\nJust Exiting...\n"); 
    sem_post(&mutex); 
} 
  
  
int main() 
{ 
    sem_init(&mutex, 0, 1); 
    pthread_t t1,t2; 
    pthread_create(&t1,NULL,thread,NULL); 
    sleep(2); 
    pthread_create(&t2,NULL,thread,NULL); 
    pthread_join(t1,NULL); 
    pthread_join(t2,NULL); 
    sem_destroy(&mutex); 
    return 0; 
} 
```
#### 读写锁 pthread_rwlock

当读写锁被一个线程以读模式占用的时候，写操作的其他线程会被阻塞，读操作的其他线程还可以继续进行。
当读写锁被一个线程以写模式占用的时候，写操作的其他线程会被阻塞，读操作的其他线程也被阻塞。

总结：多读一写，读写互斥。

```
//
//  main.m
//  pthreadsDemo
//
//  Created by liushixiang on 2020/8/13.
//  Copyright © 2020 liushixiang. All rights reserved.
//

#import <Foundation/Foundation.h>

#include "pthread.h"
pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            pthread_rwlock_rdlock(&rwlock);
            NSLog(@"read 1");
            sleep(2);
            NSLog(@"read 1 end");
            
            pthread_rwlock_unlock(&rwlock);
        });
        
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            pthread_rwlock_rdlock(&rwlock);
            NSLog(@"read 2");
            sleep(1);
            NSLog(@"read 2 end");
            
            pthread_rwlock_unlock(&rwlock);
        });
        
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            pthread_rwlock_wrlock(&rwlock);
            NSLog(@"write");
            sleep(3);
            NSLog(@"write end");
            pthread_rwlock_unlock(&rwlock);
        });
        
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            pthread_rwlock_rdlock(&rwlock);
            NSLog(@"read 3");
            pthread_rwlock_unlock(&rwlock);
        });

        [[NSRunLoop currentRunLoop] run];
    }
    return 0;
}
```

#### 自旋锁 OSSpinLock (已经废弃)

没错，这就是你能想到额最简单锁实现了。一直循环检查变量。

自旋锁是计算机科学用于多线程同步的一种锁，线程反复检查锁变量是否可用。由于线程在这一过程中保持执行，因此是一种忙等待。一旦获取了自旋锁，线程会一直保持该锁，直至显式释放自旋锁。 自旋锁避免了进程上下文的调度开销，因此对于线程只会阻塞很短时间的场合是有效的。因此操作系统的实现在很多地方往往用自旋锁。

```
//  OSSpinLock自旋锁的初始化
OSSpinLock _lock = OS_SPINLOCK_INIT;

//  锁定
OSSpinLockLock(&_lock);

// 解锁
OSSpinLockUnlock(&_lock);
```

#### os_unfair_lock

代替 OSSpinLock的。

```
#include <os/lock.h>
os_unfair_lock_t lock = &OS_UNFAIR_LOCK_INIT;

int main(int argc, const char * argv[]) {
    @autoreleasepool {
    
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            
            os_unfair_lock_lock(lock);
            NSLog(@"aaa");
            sleep(3);
            os_unfair_lock_unlock(lock);
        });
        
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            
            os_unfair_lock_lock(lock);
            NSLog(@"bbbb");
            
            os_unfair_lock_unlock(lock);
        });
        
        
        [[NSRunLoop currentRunLoop] run];
    }
    return 0;
}
```

#### 互斥锁 @synchronized()

为简化互斥锁使用，OC在语法层面实现的互斥锁。
```
@synchronized (<#token#>) {
    num ++;
}
```

#### 互斥锁 NSLock 
基于pthrea_mutext 的封装。

```
NSLock *lock = [[NSLock alloc] init];
[lock lock];
num ++;
[lock unlock];
```


#### 条件锁 NSCondition
NSCondition 的api 对应于 C语言的condition variable。使用方式也是相似的。
```
__block NSInteger age = 0;
NSCondition *condition = [[NSCondition alloc] init];

// 线程 1
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    [condition lock];
    while (age < 18) {
        NSLog(@"我还是个%ld岁的孩子。。。", age);
        [condition wait];
    }

    NSLog(@"我已经%ld岁了，该去工作了。", age);
    [condition unlock];
});

// 线程 2
dispatch_queue_t queue = dispatch_queue_create("", DISPATCH_QUEUE_SERIAL);
dispatch_async(queue, ^{

    for (;;) {
        [condition lock];
        age ++;
        [condition signal];
        [condition unlock];
        sleep(1);
    }
});

```

#### NSConditionLock

NSConditionLock 在NSCondition 的基础上再次封装抽象，将发送signal的语句合并进unlock方法，将条件判断合并进lock方法。使用起来更简便，同时灵活性及性能有所损失。

```
__block NSInteger age = 0;
NSConditionLock *conLock = [[NSConditionLock alloc] initWithCondition:0];

// 线程 1
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    [conLock lockWhenCondition:18];
    NSLog(@"我已经18岁了，该去工作了。");
    [conLock unlock];
});

// 线程 2
dispatch_queue_t queue = dispatch_queue_create("", DISPATCH_QUEUE_SERIAL);
dispatch_async(queue, ^{

    for (;;) {
        [conLock lock];
        age ++;
        NSLog(@"今年%ld岁了", age);
        [conLock unlockWithCondition:age];
        sleep(1);
    }
});
```


#### 递归锁 NSRecursiveLock

OC中的递归锁实现。与NSLock api完全相同，不同点是可多次加锁，即可重入。同一个线程多次调用不会造成死锁。递归锁内部维护Lock次数，来实现可重入。只有当次数为0时，才是真正的加锁或解锁。

#### NSThread 
带有`waitUntilDone `的几个函数具有一定的同步功能。更多时候NSThread是结合其他线程同步方式使用。

```
// NSObject扩展
- (void)performSelectorOnMainThread:(SEL)aSelector withObject:(nullable id)arg waitUntilDone:(BOOL)wait modes:(nullable NSArray<NSString *> *)array;
- (void)performSelectorOnMainThread:(SEL)aSelector withObject:(nullable id)arg waitUntilDone:(BOOL)wait;

- (void)performSelector:(SEL)aSelector onThread:(NSThread *)thr withObject:(nullable id)arg waitUntilDone:(BOOL)wait modes:(nullable NSArray<NSString *> *)array;
- (void)performSelector:(SEL)aSelector onThread:(NSThread *)thr withObject:(nullable id)arg waitUntilDone:(BOOL)wait;
```
#### GCD

gcd 在线程的基础上抽象出了队列的概念。程序员可以更专注于业务开发，屏蔽了底层线程的相关操作。相应的同步方式也更加高级和抽象。

##### dispatch_sync

同步方法算是最基本的同步方式了。

```
dispatch_async(queue, ^{
     NSLog(@"task 1");
});
NSLog(@"task 2");
```

##### 分组（dispatch_group）
分组可以分成两个阶段：分组并行阶段，结尾阶段。并行和结尾各有两种实现方式，组合起来可以产生四种使用方式。

方式一
```
dispatch_group_t group = dispatch_group_create();

dispatch_group_async(group, dispatch_get_global_queue(0, 0), ^{
    sleep(2);
    NSLog(@"下载图片1。。。");
});

dispatch_group_async(group, dispatch_get_global_queue(0, 0), ^{
    sleep(1);
    NSLog(@"下载图片2。。。");
});

dispatch_group_notify(group, dispatch_get_main_queue(), ^{
   NSLog(@"展示图片");
});
```

方式二
```
dispatch_group_t group = dispatch_group_create();

// 下载1
dispatch_group_enter(group);
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    sleep(2);
    NSLog(@"下载图片1。。。");
    dispatch_group_leave(group);
});

// 下载2
dispatch_group_enter(group);
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    sleep(1);
    NSLog(@"下载图片2。。。");
    dispatch_group_leave(group);
});

dispatch_queue_t queue = dispatch_queue_create("finish.thread", DISPATCH_QUEUE_SERIAL);
dispatch_async(queue, ^{
    dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
    NSLog(@"展示图片");
});
```

##### 分组栅栏 group_barrier

```
// 火车道口
dispatch_queue_t queue = dispatch_queue_create("", DISPATCH_QUEUE_CONCURRENT);
dispatch_async(queue, ^{
    NSLog(@"第一辆汽车");
});
dispatch_async(queue, ^{
    NSLog(@"第二辆汽车");
});
dispatch_barrier_async(queue, ^{
    NSLog(@"==========火车1来了=========");
});
dispatch_async(queue, ^{
    NSLog(@"第三辆汽车");
});

dispatch_async(queue, ^{
    NSLog(@"第四辆汽车");
});
```
##### gcd信号量

当信号大于等于零的时候可以通过。
相当于乘船过河。船可以多，人来了直接过河。人也可以多，但是需要等船来了才能走。来一个船只能走一个人。

```
dispatch_semaphore_t sem =  dispatch_semaphore_create(0);

dispatch_async(dispatch_get_global_queue(0, 0), ^{
   dispatch_semaphore_wait(sem, DISPATCH_TIME_FOREVER);
    NSLog(@"乘客1");
});
dispatch_async(dispatch_get_global_queue(0, 0), ^{
   dispatch_semaphore_wait(sem, DISPATCH_TIME_FOREVER);
    NSLog(@"乘客2");
});

// 检票
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"==开始检票==");
    sleep(1);
    NSLog(@"==检票成功==");
    dispatch_semaphore_signal(sem);

    sleep(1);
    NSLog(@"==检票成功==");
    dispatch_semaphore_signal(sem);
});
```

#### NSOpreation

NSOpreation 通过设置最大并发数和添加依赖关系实现同步操作。

最大并发数
```
NSOperationQueue *queue = [[NSOperationQueue alloc]init];
queue.maxConcurrentOperationCount = 1;
```

添加依赖关系
```
- (void)downloadImage
{

    NSOperationQueue *queue = [[NSOperationQueue alloc]init];

    //下载图片1
    NSBlockOperation *op1 = [NSBlockOperation blockOperationWithBlock:^{
        NSURL *url = ...;
        NSData *data = [NSData dataWithContentsOfURL:url];
        self.image1 = [UIImage imageWithData:data];
    }];

    //下载图片2
    NSBlockOperation *op2 = [NSBlockOperation blockOperationWithBlock:^{
        NSURL *url = ...;
        NSData *data = [NSData dataWithContentsOfURL:url];
        self.image2 = [UIImage imageWithData:data];
    }];

    //合成图片
    NSBlockOperation *combine = [NSBlockOperation blockOperationWithBlock:^{
				UIImage *image = image1 + image2;
        [[NSOperationQueue mainQueue]addOperationWithBlock:^{
            self.imageView.image = image;
            NSLog(@"刷新UI---%@",[NSThread currentThread]);
        }];

    }];


    [combine addDependency:op1];
    [combine addDependency:op2];

    [queue addOperation:op1];
    [queue addOperation:op2];
    [queue addOperation:combine];
}
```












