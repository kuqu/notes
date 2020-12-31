
# Runtime Note

{docsify-updated}

## 介绍

OC 运行时包括两部分的内容：类的结构体系和消息发送。
Runtime 为OC赋予了元编程能力，使其可以像脚本语言一样具有动态性。

涉及到的空开头文件：

```
// <NSObject>协议和NSObject类的定义
<objc/NSObject.h>

// OC类的一些操作
<objc/runtime.h>

// 消息发送的相关操作
<objc/message.h>
```

其实我们一直在使用runtime只不过是编译器帮我们实现的。包括下边这三种方式：

- OC代码经过编译后会生成对应的runtime代码。
- NSObject中定义的方法。
- 手动调用runtime api。

通过`clang -rewrite-objc main.m`编译以下内容

转换前
```
#import <Foundation/Foundation.h>
#import "Person.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...
        NSLog(@"Hello, World!");
        
        Person *person = [[Person alloc] init];
        [person publicMethod];
        
    }
    return 0;
}
```
转换后（main.cpp 最后部分）
```
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        NSLog((NSString *)&__NSConstantStringImpl__var_folders_qd_03prvzjd6b16s61vvfg2g2kr0000gn_T_main_7d494c_mi_0);

        Person *person = ((Person *(*)(id, SEL))(void *)objc_msgSend)((id)((Person *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("Person"), sel_registerName("alloc")), sel_registerName("init"));
        
        ((void (*)(id, SEL))(void *)objc_msgSend)((id)person, sel_registerName("publicMethod"));

    }
    return 0;
}
```

初步观察可以发现我们编写的OC代码被翻译成了一些C语言函数的调用，而`objc_msgSend`和`sel_registerName`正是runtime中的函数。

参考：
[](https://halfrost.com/objc_runtime_isa_class/)
[Runtime源码](https://opensource.apple.com/tarballs/objc4/)

## 探究类的本质

通过runtime源码可以找到类、对象相关结构体定义如下

```
// 类对象结构体
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags

    class_rw_t *data() const {
        return bits.data();
    }
    void setData(class_rw_t *newData) {
        bits.setData(newData);
    }
    ...// 相关函数
 }

// 实例对象结构体
struct objc_object {
private:
    isa_t isa;// 联合体内部包含指向类结构体的cls指针

public:

    // ISA() assumes this is NOT a tagged pointer object
    Class ISA();

    // rawISA() assumes this is NOT a tagged pointer object or a non pointer ISA
    Class rawISA();

    // getIsa() allows this to be a tagged pointer object
    Class getIsa();

    ...// 相关函数
 }
```
objc_class 继承自 objc_object, objc_object 和 objc_class 具有ISA 指针。所以Class 也可以认为是一个对象。
superclass 指向父类结构体。cache 则为方法缓存。bits 表示二进制数据，也是objc_class 中最后一个数据段，所以其中应该包括大多数类相关的数据。
比如属性、实例变量、方法列表、继承的协议等。bits 在稍后做分析。


在面向对象的语言中，类是对象创建的模板。这在OC中是通过isa指针实现。objc_object 中的 isa_t isa 就是对象指向类的指针。

isa_t 是一个联合体，结构如下。其中cls为指向类结构体指针，若为objc_class则cls指向元类。
struct 中包含了内存优化相关的标志位。
```
union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;
#if defined(ISA_BITFIELD)
    struct {
        //ISA_BITFIELD;  // defined in isa.h

        uintptr_t nonpointer        : 1;  // 表示 isa_t 的类型，0 表示 raw isa，也就是没有结构体的部分，访问对象的 isa 会直接返回一个指向 cls 的指针，也就是在 iPhone 迁移到 64 位系统之前时 isa 的类型。1 表示当前 isa 不是指针，但是其中也有 cls 的信息，只是其中关于类的指针都是保存在 shiftcls 中。
        uintptr_t has_assoc         : 1;  // 对象含有或者曾经含有关联引用，没有关联引用的可以更快地释放内存
        uintptr_t has_cxx_dtor      : 1;  // 表示当前对象有 C++ 或者 ObjC 的析构器(destructor)，如果没有析构器就会快速释放内存。
        uintptr_t shiftcls          : 44; // MACH_VM_MAX_ADDRESS 0x7fffffe00000
        uintptr_t magic             : 6;  // 用于调试器判断当前对象是真的对象还是没有初始化的空间
        uintptr_t weakly_referenced : 1;  // 对象被指向或者曾经指向一个 ARC 的弱变量，没有弱引用的对象可以更快释放
        uintptr_t deallocating      : 1;  // 对象正在释放内存
        uintptr_t has_sidetable_rc  : 1;  // 对象的引用计数太大了，存不下
        uintptr_t extra_rc          : 8;  // 对象的引用计数超过 1，会存在这个这个里面，如果引用计数为 10，extra_rc 的值就为 9
    };
#endif
};
```

到目前为止，我们找到类实现背后的结构体objc_class, 实例对象背后的结构体objc_object，并且obcj_class继承自objc_object。在两个结构体中都有一个isa_t的联合体成员。isa_t中包含了一个指向类或者元类的Class 指针。

但是类中的属性、成员变量、方法、协议却没有发现。在objc_class中还有一个成员变量：`class_data_bits_t bits;`。那是不是bits就是这些内容的容器呢？

class_data_bits_t 的结构如下：

```
struct class_data_bits_t {

	...

    uintptr_t bits;// 存储容器

    // bits 中主要储存了 class_rw_t * 类型的地址。
    class_rw_t* data() const {
        return (class_rw_t *)(bits & FAST_DATA_MASK);
    }

    void setData(class_rw_t *newData)
    {
        ASSERT(!data()  ||  (newData->flags & (RW_REALIZING | RW_FUTURE)));
        // Set during realization or construction only. No locking needed.
        // Use a store-release fence because there may be concurrent
        // readers of data and data's contents.
        uintptr_t newBits = (bits & ~FAST_DATA_MASK) | (uintptr_t)newData;
        atomic_thread_fence(memory_order_release);
        bits = newBits;
    }

    ...
 }

```

可以发现 class_data_bits_t 中最终要的是包含一个 class_rw_t 的结构体指针。继续看 class_rw_t 的内部结构。


```

struct class_rw_t {

	const class_ro_t *ro() const {}

	// 方法集合
	const method_array_t methods() const {}
	// 属性集合
    const property_array_t properties() const {}
    // 协议集合
    const protocol_array_t protocols() const {}
}

/*

class method_array_t : 
    public list_array_tt<method_t, method_list_t> 
{
}
*/

```

可以发现在 class_rw_t 中包含了类的基本数据内容。现在基本了解了类的构成方式，但还有很多细节需要研究。

类内容的集合这里使用的是 C++ 的类来实现的。这三个类都继承自同一个类 list_array_tt，而这个类使用了C++的模板方式 `template <typename Element, typename List>`

这里还有两个问题需要讨论：

1. class_ro_t的是什么作用？
2. 实例变量存到哪里了？


```

struct class_ro_t {

    const char * name;

    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;
    property_list_t *baseProperties;
 }
```

可以发现 class_ro_t 中还有一套类内容集合，并且多了ivars。这正是类实例变量的存储位置。通过ro（readonly）可以猜测这个结构体是不可修改的。

在类初始化的初期，class_data_bits_t 中 bits 是指向 class_ro_t 的，这个时候是只读的。之后创建了 class_rw_t 并在内部指向了之前的 class_ro_t 。

在类的扩展中不能添加实例变量，应该就是 ivars 只在 class_ro_t 存在有关。（其实不是这个原因）


method_list_t、protocol_list_t、ivar_list_t、property_list_t 等容器是采用 struct 和 template 实现的。C++ 模板还不太熟悉，知道这里是个容器就差不多了。
```

struct method_list_t : entsize_list_tt<method_t, method_list_t, 0x3> {
}

template <typename Element, typename List, uint32_t FlagMask>
struct entsize_list_tt {
}
```

下边在来看一下类中属性和方法等的结构体类型。

```

// 相关结构体指针

typedef struct method_t *Method;
typedef struct ivar_t *Ivar;
typedef struct category_t *Category;
typedef struct property_t *objc_property_t;

// 类的方法 *Method
// Method 其实就是 SEL 和 IMP 的一层包装。将两者做了一层映射。在向类发送消息时会遍历method_t的列表通过SEL找到对应的IMP。
struct method_t {
    SEL name;
    const char *types;
    MethodListIMP imp;

    struct SortBySELAddress :
        public std::binary_function<const method_t&,
                                    const method_t&, bool>
    {
        bool operator() (const method_t& lhs,
                         const method_t& rhs)
        { return lhs.name < rhs.name; }
    };
};

// 实例变量 *Ivar
struct ivar_t {
    int32_t *offset;// 保存的就是对应的全局变量的地址。
    const char *name;
    const char *type;
    // alignment is sometimes -1; use alignment() instead
    uint32_t alignment_raw;
    uint32_t size;

    uint32_t alignment() const {
        if (alignment_raw == ~(uint32_t)0) return 1U << WORD_SHIFT;
        return 1 << alignment_raw;
    }
};

// 属性 - 属性实际只有一个名字。对属性的访问都是调用了get set 方法。
struct property_t {
    const char *name;
    const char *attributes;
};


```

到这里类相关的内部结构分析差不多完成了，但是还有一个内容并没有涉及到，那就是类的分类。


如下，可以发现分类的结构和我们的直觉是一样的，包含了两类方法、协议和属性。
```
// 扩展 *Category
struct category_t {
    const char *name;// 扩展的名字
    classref_t cls;// 扩展的类

    struct method_list_t *instanceMethods;// 实例方法
    struct method_list_t *classMethods;// 类方法
    struct protocol_list_t *protocols;// 协议
    struct property_list_t *instanceProperties;// 属性

    // Fields below this point are not always present on disk.
    struct property_list_t *_classProperties;

    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }

    property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
    
    protocol_list_t *protocolsForMeta(bool isMeta) {
        if (isMeta) return nullptr;
        else return protocols;
    }
};

```

下面通过clang重写的方式探究编译期做了哪些工作。


原始OC代码
```

// 原类
@interface Person : NSObject
@property (nonatomic) NSInteger age;
@property (nonatomic) NSString *name;

- (void)publicMethod;
@end

@implementation Person

- (void)publicMethod{
    NSLog(@"共有方法");
}


- (void)privateMethod{
    NSLog(@"私有方法");
}

@end

// 分类
#import "Person+Sing.h"

@implementation Person (Sing)
- (void)singSong{
}
@end
```

重写后的代码

```

// Person 类，在OC中定义的一个类，经过编译后变成了一个初始化后的 class_t 类型变量。
extern "C" __declspec(dllexport) struct _class_t OBJC_CLASS_$_Person __attribute__ ((used, section ("__DATA,__objc_data"))) = {
	0, // &OBJC_METACLASS_$_Person,
	0, // &OBJC_CLASS_$_NSObject,
	0, // (void *)&_objc_empty_cache,
	0, // unused, was (void *)&_objc_empty_vtable,
	&_OBJC_CLASS_RO_$_Person,
};

// Person 的元类，元类同类一样被实例化了。
extern "C" __declspec(dllexport) struct _class_t OBJC_METACLASS_$_Person __attribute__ ((used, section ("__DATA,__objc_data"))) = {
	0, // &OBJC_METACLASS_$_NSObject,
	0, // &OBJC_METACLASS_$_NSObject,
	0, // (void *)&_objc_empty_cache,
	0, // unused, was (void *)&_objc_empty_vtable,
	&_OBJC_METACLASS_RO_$_Person,
};


// 使用一个函数设置类的继承关系
static void OBJC_CLASS_SETUP_$_Person(void ) {
	OBJC_METACLASS_$_Person.isa = &OBJC_METACLASS_$_NSObject;
	OBJC_METACLASS_$_Person.superclass = &OBJC_METACLASS_$_NSObject;
	OBJC_METACLASS_$_Person.cache = &_objc_empty_cache;
	OBJC_CLASS_$_Person.isa = &OBJC_METACLASS_$_Person;
	OBJC_CLASS_$_Person.superclass = &OBJC_CLASS_$_NSObject;
	OBJC_CLASS_$_Person.cache = &_objc_empty_cache;
}

// 类中的class_ro_t，保存了类的属性、方法、变量。
static struct _class_ro_t _OBJC_CLASS_RO_$_Person __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	0, __OFFSETOFIVAR__(struct Person, _age), sizeof(struct Person_IMPL), 
	(unsigned int)0, 
	0, 
	"Person",
	(const struct _method_list_t *)&_OBJC_$_INSTANCE_METHODS_Person,
	0, 
	(const struct _ivar_list_t *)&_OBJC_$_INSTANCE_VARIABLES_Person,
	0, 
	(const struct _prop_list_t *)&_OBJC_$_PROP_LIST_Person,
};


// 实例变量
static struct /*_ivar_list_t*/ {
	unsigned int entsize;  // sizeof(struct _prop_t)
	unsigned int count;
	struct _ivar_t ivar_list[2];
} _OBJC_$_INSTANCE_VARIABLES_Person __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_ivar_t),
	2,
	{{(unsigned long int *)&OBJC_IVAR_$_Person$_age, "_age", "q", 3, 8},
	 {(unsigned long int *)&OBJC_IVAR_$_Person$_name, "_name", "@\"NSString\"", 3, 8}}
};


// 方法列表
static struct /*_method_list_t*/ {
	unsigned int entsize;  // sizeof(struct _objc_method)
	unsigned int method_count;
	struct _objc_method method_list[10];
} _OBJC_$_INSTANCE_METHODS_Person __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_objc_method),
	10,
	{{(struct objc_selector *)"publicMethod", "v16@0:8", (void *)_I_Person_publicMethod},
	{(struct objc_selector *)"privateMethod", "v16@0:8", (void *)_I_Person_privateMethod},
	{(struct objc_selector *)"age", "q16@0:8", (void *)_I_Person_age},
	{(struct objc_selector *)"setAge:", "v24@0:8q16", (void *)_I_Person_setAge_},
	{(struct objc_selector *)"name", "@16@0:8", (void *)_I_Person_name},
	{(struct objc_selector *)"setName:", "v24@0:8@16", (void *)_I_Person_setName_},
	{(struct objc_selector *)"age", "q16@0:8", (void *)_I_Person_age},
	{(struct objc_selector *)"setAge:", "v24@0:8q16", (void *)_I_Person_setAge_},
	{(struct objc_selector *)"name", "@16@0:8", (void *)_I_Person_name},
	{(struct objc_selector *)"setName:", "v24@0:8@16", (void *)_I_Person_setName_}}
};


//  属性列表
static struct /*_prop_list_t*/ {
	unsigned int entsize;  // sizeof(struct _prop_t)
	unsigned int count_of_properties;
	struct _prop_t prop_list[2];
} _OBJC_$_PROP_LIST_Person __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_prop_t),
	2,
	{{"age","Tq,N,V_age"},
	{"name","T@\"NSString\",N,V_name"}}
};


// 实例变量的存储位置。实例变量采用了全局变量的方式保存。ivar_t 中保留了全局变量的地址。
extern "C" unsigned long int OBJC_IVAR_$_Person$_age __attribute__ ((used, section ("__DATA,__objc_ivar"))) = __OFFSETOFIVAR__(struct Person, _age);
extern "C" unsigned long int OBJC_IVAR_$_Person$_name __attribute__ ((used, section ("__DATA,__objc_ivar"))) = __OFFSETOFIVAR__(struct Person, _name);

```

可以发现类经过编译后变成了普通C语言代码，类转变成了结构体和函数的形式，类中的具体信息通过字面量的形式出现。runtime初始化的时候加载运行以上代码就可以还原类了。
（时间有限，类在运行期的加载过程暂时不分析了，）

下边看一下分类重写后的代码。

```

// 指定分类cls = Person
static void OBJC_CATEGORY_SETUP_$_Person_$_Sing(void ) {
	_OBJC_$_CATEGORY_Person_$_Sing.cls = &OBJC_CLASS_$_Person;
}

// 定义 _category_t 类型变量，表示一个分类
static struct _category_t _OBJC_$_CATEGORY_Person_$_Sing __attribute__ ((used, section ("__DATA,__objc_const"))) = 
{
	"Person",
	0, // &OBJC_CLASS_$_Person,
	(const struct _method_list_t *)&_OBJC_$_CATEGORY_INSTANCE_METHODS_Person_$_Sing,
	0,
	0,
	0,
};

// 分类中的方法列表
static struct /*_method_list_t*/ {
	unsigned int entsize;  // sizeof(struct _objc_method)
	unsigned int method_count;
	struct _objc_method method_list[1];
} _OBJC_$_CATEGORY_INSTANCE_METHODS_Person_$_Sing __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_objc_method),
	1,
	{{(struct objc_selector *)"singSong", "v16@0:8", (void *)_I_Person_Sing_singSong}}
};

extern "C" __declspec(dllimport) struct _class_t OBJC_CLASS_$_Person;

```
可以发现扩展编译后跟类是类似的，只是少了实例变量。（因为没有设置属性、协议，所以这里只有方法了）

在运行时，runtime会将扩展中的内容复制到类的相应位置。具体分析参考下面链接。

https://juejin.im/post/6844903937552678926


最终可以画出类的结构体系图：

![objc_class]({{site.baseurl}}/images/objc_class.jpeg)


#### 类的继承体系

鉴于网上的经典类关系图不容易记忆，画了一个方便记忆的。

![objc_class]({{site.baseurl}}/images/class_link_class.png)

继承体系可以根据runtime源码来分析，暂时不做了，网上挺多的。

## 方法是如何调用的

方法的调用分为两个阶段：消息发送和消息转发。

#### 1. 给类发送消息

对象消息发送的流程是首先在类（objc_class）的Method list寻找对应于SEL的实现（IMP）,如果找到则执行相应的C函数。如果没有找到，则通过superclass去父类寻找，直到根父类。如果还是没有找到则进入第二个阶段：消息转发。
找到后会对消息做缓存，方便以后再次调用。


#### 2. 消息动态转发


消息动态转发这三个阶段的中文名字实在难记，就记做：到函数、到类、到Invocation吧。


```

#pragma mark - 第一阶段(转发到函数)

// resolve 对象方法
+ (BOOL)resolveInstanceMethod:(SEL)sel{
    return NO;// skip 1

    if (sel == @selector(run)) {
        class_addMethod(self, @selector(run), (IMP)runc, "V@:");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}

// resolve 类方法
+ (BOOL)resolveClassMethod:(SEL)sel{
    
    return NO;// skip 1
    if (sel == @selector(classRun)) {
        class_addMethod(class_getSuperclass(self), sel, (IMP)runc, "V@:");
        return YES;
    }
    
    return [class_getSuperclass(self) resolveClassMethod:sel];
}

#pragma mark - 第二阶段(转发到类)

- (id)forwardingTargetForSelector:(SEL)aSelector{
    
    return nil;// skip 2
    if (aSelector == @selector(run)) {
        
        return [NSObject new];
    }
    
    return [super forwardingTargetForSelector:aSelector];
}


+ (id)forwardingTargetForSelector:(SEL)aSelector{
    
    return nil;// skip 2
    if (aSelector == @selector(classRun)) {
        
        return [NSObject class];
    }
    return nil;
}

#pragma mark - 第三阶段(NSInvocation转发)

#pragma mark 对象转发到 NSInvocation

- (void)forwardInvocation:(NSInvocation *)anInvocation{
    
//    [anInvocation invoke];
//    [anInvocation invokeWithTarget:self];
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector{
    
    if (aSelector == @selector(run)) {
        NSMethodSignature *sig = [super methodSignatureForSelector:aSelector];
        if (!sig) {
            sig = [NSMethodSignature signatureWithObjCTypes:"v@:"];
        }
        return sig;
    }
    return [super methodSignatureForSelector:aSelector];
}

#pragma mark 类转发到 NSInvocation


+ (void)forwardInvocation:(NSInvocation *)anInvocation{
    
    printf("%s", sel_getName(anInvocation.selector));
}

+ (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector{
    
    if (aSelector == @selector(classRun)) {
        NSMethodSignature *sig = [super methodSignatureForSelector:aSelector];
        if (!sig) {
            sig = [NSMethodSignature signatureWithObjCTypes:"v@:"];
        }
        return sig;
    }
    return [super methodSignatureForSelector:aSelector];
}
```


## 实战Demo

1. 让分类可以自动添加属性。


2. 给类实现 mix 功能。(好像就是原先的匿名协议啊，把方法都写在NSObject上就行吗？)

在使用Flutter的时候发现Dart语言具有Mix功能，使用起来比较方便。多数情况下这个需求使用组合也是可以实现的。但有的时候组合表意并没有Mix来的直接顺畅。
比如：张三继承自父亲的多数能力，但同时也有你母亲的某些能力，这个时候使用组合的话就显得很不顺畅，如果张三也能使用母亲的某些功能就简单的多了。而直接使用母亲能力的效果就叫MIX吧。

需要研究的问题？
（OC 如何实现多继承、多继承的优点和缺点、MIX的优点和缺点）协议到底适不适合这样做。？MIX是做成类还是做成独立的MIX组件。

时间来不及了，先不实现了。

## 面试题解析

define instancetype id 区别

objective 1 和 2 的主要区别？

扩展为什么不能添加实例变量？

内存中一个对象占用多大空间

OC模拟实现多继承。

健壮的实例变量

是么是ABI稳定？



