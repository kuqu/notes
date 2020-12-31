
# block 总结

{docsify-updated}

## 什么是block

带有自动变量瞬时值的匿名函数。

类似于其他语言中的闭包。 block 同OC对象一样具有isa 指针，可以retain、release、copy。其底层是由结构体实现的，结构体内部定义了截获的自动变量和block对应的C函数。

## block 有哪几种

根据所在内存中的位置分为三种类型：Global（全局）、Stack（栈）、Heap（堆）。

#### Global 

不截获外部变量的block，则放在内存的Data区域，属于全局block。

具体位置有两种表现形式：
  1. 写在全局变量位置的block。
  2. 写在函数内部，但是不截获自动变量。

#### Stack

由block语法新生成的block，并且截获了自动变量，则为栈上的block。

#### Heap

对栈上block进行copy操作，则产生堆上的block类型。

PS. ARC 下对block的赋值操作会进行copy。所以平时使用block大多数是在堆上的。


## 截获的自动变量可以修改吗，怎样修改，为什么能修改

1. 默认不可以修改。
2. 给变量添加 `__block` 的声明后则可以修改。或者添加 `static` ， 即成为局部静态变量则可以修改。
3. 自动变量类型发生了根本性变化，由原来的简单声明，变成了包含自动变量的结构体类型。而block所捕获的是结构体指针。所以能修改的根本原因是：block对自动变量的捕获由值传递变成了指针传递。

__追问：那为什么要改成结构体后再传指针，直接传自动变量的指针不行吗？__

不行。由于block具有可copy属性，这种复制对于自动变量来说是一种浅复制。需要保证复制前后block对自动变量的修改保持一致。所以要么栈上的自动变量指向堆上的自动变量，要么堆上的指向栈上的，而简单类型是无法修改指针指向的。所以采用了结构体的方式，结构体中forwording指针就是用于保证栈上和堆上变量的一致性。

（如果不采用修改指向的方式，就需要使用到一种同步机制。同步机制复杂性及效率都是不及修改指针指向的。修改指针指向也是很多语言高级特性的实现方式。）

PS.由于OC是对C的超集，所以结构体对C数组的截获是不支持的。


## Block 如何避免循环引用（考察重点）

三种方式：
1. `__weak`
2.  `__block`
3.  `__unsafe_unretained`



ARC下主要使用weak的方式。unsafe-unretained 也可以，只是不会设置为nil，会出现野指针。
MRC下主要使用__block的方式。效果同weak，不会对对象做retain操作。

`__block`在ARC和MRC下使用的区别。在ARC下回造成循环引用，要打破循环引用需要在Block执行中将截获的对象设置为nil才可以打破循环。必须在Block调用的情况下才有效，使用条件比较苛刻，ARC下建议使用weak的方式。

## 在Block内部再次获取对象强引用，为什么？

防止多线程中，捕获的对象销毁，造成不完成的操作。造成异常状态产生。


## 系统api中block 使用需要注意什么

系统的api 一般情况是不需要考虑循环引用的，但是当与系统api有了除block外的其他联系时，需要特别注意。例如通知的使用。

```
__weak __typeof(self) weakSelf = self;
_observer = [[NSNotificationCenter defaultCenter] addObserverForName:@"testKey"
                                                            object:nil
                                                             queue:nil
                                                        usingBlock:^(NSNotification *note) {
  __typeof__(self) strongSelf = weakSelf;
  [strongSelf dismissModalViewControllerAnimated:YES];
}];
self –> _observer –> block –> self 显然这也是一个循环引用。
```

系统api 为什么知道何时释放捕获的外部变量？

实际上它不知道何时释放外部变量，没有什么神奇的黑魔法。主要是分两种情况。
一种是单次执行的block，在block执行完成后，会释放block，block所捕获的自动变量也会随之释放。
第二种是重复执行的block，一般都提供了取消方法。让程序员来手动打破循环。

启示：我们设计自己的block接口的时候也可以考虑采用系统的这种方式。提供一个取消方法。这样使用的时候就不用时刻担心内存泄露了，self可以愉快随便写了。

## block 调用延时函数有什么注意的地方吗？

>在block内部如果调用了延时函数还使用弱指针会取不到该指针,因为已经被销毁了,需要在block内部再将弱指针重新强引用一下。

网上一直有这个说法，但是没找到这种说法的出处。个人认为这句总结的有些不妥。首先根据文意写出对应代码。

```
NSObject *obj = [[NSObject alloc] init];
__weak __typeof(obj) weakObj = obj;
        
void (^block1)(void) = ^{
            __strong __typeof(weakObj) obj = weakObj;
            dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), dispatch_get_global_queue(0, 0), ^{
                printf("obj = %s\n", obj.description.UTF8String);
            });
        };
block1();
```

这种情况下，gcd block会持有obj的强引用，block1会持有obj的弱引用。这样在gcd block 执行的时候obj并不会释放。那么这样看来这句话是没错的啊？

但是这里强调延时函数好像没啥道理，因为block 是可以异步执行的，就算不是延时函数，我们也可以在以后的一个时间点调用block。还是说这里的“延时函数”就是代表广义的“延后执行”。这是不妥之一。

block 是否需要强引用捕获的自动变量，需要根据需求来确定。不应该是不加区分的使用强引用。文中说“将弱指针重新强引用一下”是不妥之二。 比如需求就是在obj销毁后，就终止延时函数执行呢？正确的实现应该区分两种情况：

1. 定时block必须执行，直接使用原来的strong变量就可以。

```
NSObject *obj = [[NSObject alloc] init];
void (^block1)(void) = ^{
            dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), dispatch_get_global_queue(0, 0), ^{
                  printf("222 - %s\n", obj.description.UTF8String);
            });
        };
block1();
```

2. obj释放后要求定时block不必执行。
```
NSObject *obj = [[NSObject alloc] init];
__weak __typeof(obj) weakObj = obj;
        
block = ^(void) {
            dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), dispatch_get_global_queue(0, 0), ^{
                
                __strong __typeof(weakObj) obj = weakObj;
                if (obj) {
                  printf("222 - %s\n", obj.description.UTF8String);
                }
            });
        };
block();
```

## 面试题分析1


这段代码有没有内存泄露问题？如果去掉代码中的`__block` 或者`token = nil`会有什么变化吗？（原题不是这种提问方式，这里做了简化处理）
```
- (void)viewDidLoad {
    [super viewDidLoad];
    NSNotificationCenter *__weak center = [NSNotificationCenter defaultCenter];
    id __block token = [center addObserverForName:UIApplicationDidEnterBackgroundNotification
                                           object:nil
                                            queue:[NSOperationQueue mainQueue]
                                       usingBlock:^(NSNotification * _Nonnull note) {
        [self doSomething];
        [center removeObserver:token];
        token = nil;
    }];
}

- (void)doSomething {
    
}
```

1. 有内存泄露的，在block没有执行的情况下，无法打破token和block的环。
2. 如果token 不使用__block 声明，则block中token是捕获不到的，因为block创建在token生成之前。
3. 如果没有token = nil 这个语句，也是有内存泄露的，token -> block -> token




























