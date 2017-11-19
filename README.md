# iOS面试题汇总
## 问答题
### 说说ARC和MRC的区别
ARC: 自动引用计数， MRC：手动引用计数  
在MRC时代，开发者通过retain, release, autorelease这些函数，手动的控制引用计数。  
而ARC将开发者从这个工作中解放出来，在编译期间，自动插入这些函数调用。  
ARC是一个编译器的特性。  
ARC引入了一些新的修饰关键字，如strong, weak。  
不管是ARC还是MRC，内存管理的方式并没有改变。
  

### 线程同步工具都有哪些？
主要有：Atomic Operations, Lock和Condition。  GCD中的group, barrier, semaphore也是用来在GCD中做同步的。
#### 1. Atomic Operations  
系统提供了一些原子性的数学运算和逻辑运算函数，声明在/usr/include/libkern/OSAtomic.h中。  原子操作的性能比锁要高。  
这些函数分为以下类别：  
Add: Adds two integer values together and stores the result in one of the specified variables.  
Increment: Increments the specified integer value by 1.  
Decrement: Decrements the specified integer value by 1.  
Logical OR: Performs a logical OR between the specified 32-bit value and a 32-bit mask.     
Logical AND: Performs a logical AND between the specified 32-bit value and a 32-bit mask.   
Logical XOR: Performs a logical XOR between the specified 32-bit value and a 32-bit mask.  
Compare and swap:    
Test and set:   
Test and clear:    

#### 2. Using Locks  
* Using a POSIX Mutex Lock   

```
pthread_mutex_t mutex;
void MyInitFunction()
{
    pthread_mutex_init(&mutex, NULL);
}
 
void MyLockingFunction()
{
    pthread_mutex_lock(&mutex);
    // Do work.
    pthread_mutex_unlock(&mutex);
}
```
* Using the NSLock Class  
基本的互斥锁，除了标准的加锁和解锁外，还提供了非阻塞的tryLock，设置超时的lockBeforeDate:  

```
BOOL moreToDo = YES;
NSLock *theLock = [[NSLock alloc] init];
...
while (moreToDo) {
    /* Do another increment of calculation */
    /* until there’s no more to do. */
    if ([theLock tryLock]) {
        /* Update display used by all threads. */
        [theLock unlock];
    }
}
```

* Using @synchronized  
一个使用互斥锁的便利方式，使用括号中的对象作为锁的token。性能是最差的。 
@synchronized 指令实现锁的优点就是我们不需要在代码中显式的创建锁对象，便可以实现锁的机制，但作为一种预防措施，@synchronized 块会隐式的添加一个异常处理例程来保护代码，该处理例程会在异常抛出的时候自动的释放互斥锁。所以如果不想让隐式的异常处理例程带来额外的开销，你可以考虑使用锁对象。     

```
- (void)myMethod:(id)anObj
{
    @synchronized(anObj)
    {
        // Everything between the braces is protected by the @synchronized directive.
    }
}
```
* NSRecursiveLock  
递归锁，可以被同一线程加锁多次，而不会导致死锁问题。递归锁会追踪被加锁多少次，每次成功的加锁都得匹配一次解锁。只有加锁和解锁的次数相同，锁才会被释放，其它的线程才可以加锁成功。
该锁经常使用在递归函数中。   

```
NSRecursiveLock *theLock = [[NSRecursiveLock alloc] init];
 
void MyRecursiveFunction(int value)
{
    [theLock lock];
    if (value != 0)
    {
        --value;
        MyRecursiveFunction(value);
    }
    [theLock unlock];
}
 
MyRecursiveFunction(5);
```
* NSConditionLock  
该类型的锁可以使用一个特定的值去加锁和解锁，一个典型的例子就是一个线程生产数据，另一个消费数据。  
lockWhenCondition: 1 是指当前condition为1时，才能加锁成功。  
unlockWithCondition: 1 是指解锁后，将condition置为1.

```
//生产者
id condLock = [[NSConditionLock alloc] initWithCondition:NO_DATA];
 
while(true)
{
    [condLock lock];
    /* Add data to the queue. */
    [condLock unlockWithCondition:HAS_DATA];
}
```


```
//消费者
while (true)
{
    [condLock lockWhenCondition:HAS_DATA];
    /* Remove data from the queue. */
    [condLock unlockWithCondition:(isEmpty ? NO_DATA : HAS_DATA)];
 
    // Process the data locally.
}
```

#### 3. Using Conditions  
一种特殊类型的锁，主要是用来同步操作执行的顺序。  
A condition object acts as both a lock and a checkpoint in a given thread. The lock protects your code while it tests the condition and performs the task triggered by the condition. The checkpoint behavior requires that the condition be true before the thread proceeds with its task. While the condition is not true, the thread blocks. It remains blocked until another thread signals the condition object.  
The semantics for using an NSCondition object are as follows:  
1. Lock the condition object.  
2. Test a boolean predicate. (This predicate is a boolean flag or other variable in your code that indicates whether it is safe to perform the task protected by the condition.)  
3. If the boolean predicate is false, call the condition object’s wait or waitUntilDate: method to block the thread. Upon returning from these methods, go to step 2 to retest your boolean predicate. (Continue waiting and retesting the predicate until it is true.)  
4. If the boolean predicate is true, perform the task.  
5. Optionally update any predicates (or signal any conditions) affected by your task.  
6. When your task is done, unlock the condition object.  

Using a Cocoa condition：  

```
[cocoaCondition lock];
while (timeToDoWork <= 0)
    [cocoaCondition wait];
 
timeToDoWork--;
 
// Do real work here.
 
[cocoaCondition unlock];
```

Signaling a Cocoa condition:   

```
[cocoaCondition lock];
timeToDoWork++;
[cocoaCondition signal];
[cocoaCondition unlock];
```
一次signal调用只能唤醒一个线程，而broadcoast则能唤醒所有的线程。

### Lock与Condition的区别是什么？  
一个Lock只能由一个线程加锁成功，其它的线程必须等待，直到其它线程释放锁。
Condition相当于Lock + Condition，可以由多个线程加锁成功，加锁成功以后还需要检查condition是否满足，不满足的话需要一直wait。

### AutoreleasePool
每个线程都需要自动释放池，否则无法处理被调用了autorelease方法的对象。  
在main.m中，主线程已经包含了一个自动释放池的块。  
使用GCD, Operation这些多线程技术时，线程里面也会包含自动释放池的块。  
自动释放池可以有多个，形成一个栈的结构。  
可以自己创建自动释放池的块，在块开始的时候，会PUSH一个自动释放池对象，在块结束的时候，会从栈中pop。
自动释放池被pop的时候，会对它所包含的对象发送release消息。
每次runloop迭代结束的时候，主线程会向自动释放池中的对象发送release消息。

### .Method, SEL, IMP都是什么含义？
```
struct objc_method {
    SEL method_name OBJC2_UNAVAILABLE;
    char *method_types  OBJC2_UNAVAILABLE;
    IMP method_imp OBJC2_UNAVAILABLE;
}

typedef struct objc_method *Method;

typedef struct method_list_t {
    uint32_t entsize_NEVER_USE;  // low 2 bits used for fixup markers
    uint32_t count;
    struct method_t first;
} method_list_t;

typedef struct objc_selector *SEL; //可以认为是一个C字符串

//定义了一个函数指针类型
typedef id _Nullable (*IMP)(id _Nonnull, SEL _Nonnull, ...);

```
### .实例，类，元类的关系图   
![实例，类，元类关系图](https://github.com/buptwsgprivate/iOSInterview/blob/master/Images/instance_class_metaclass.png)

### OC的消息转发过程
如果向一个对象发送一个不支持的消息，那么默认的实现是会调用NSObject类中的doesNotRecognizeSelector:方法，此方法会抛出异常，导致应用崩溃。
但是runtime在此之前，会给3次机会。  
**第1次，+(BOOL)resolveInstanceMethod:(SEL)name 或+ (BOOL)resolveClassMethod:(SEL)name**

```
@interface Person : NSObject

@property (nonatomic, copy) NSString* name;
@property (nonatomic, assign) NSUInteger age;

@end

@implementation Person
//如果需要传参直接在参数列表后面添加就好了
void dynamicAdditionMethodIMP(id self, SEL _cmd) {
    NSLog(@"dynamicAdditionMethodIMP");
}

+ (BOOL)resolveInstanceMethod:(SEL)name {
    NSLog(@"resolveInstanceMethod: %@", NSStringFromSelector(name));
    if (name == @selector(appendString:)) {
        class_addMethod([self class], name, (IMP)dynamicAdditionMethodIMP, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:name];
}

+ (BOOL)resolveClassMethod:(SEL)name {
    NSLog(@"resolveClassMethod %@", NSStringFromSelector(name));
    return [super resolveClassMethod:name];
}

@end
```

**第2次 - (id)forwardingTargetForSelector:(SEL)aSelector;**
可以override这个函数，返回其它的能处理这个消息的对象。  

```
@implementation TestClass
- (void)logString:(NSString *)str {
    NSLog(@"log string: %@", str);
}
@end

self.tc = [TestClass new];
//self并没有logString:这个方法，如果不处理转发，则会崩溃。
[self performSelector: @selector(logString:) withObject: @"Hello"];

- (id)forwardingTargetForSelector:(SEL)aSelector {
    if (aSelector == @selector(logString:)) {
        return self.tc;
    }
    
    return [super forwardingTargetForSelector: aSelector];
}   
```

**第3次 完整转发流程**

```
- (NSMethodSignature*)methodSignatureForSelector:(SEL)aSelector {
    if (aSelector == @selector(logString:)) {
        return [self.tc methodSignatureForSelector: aSelector];
    }
    return [super methodSignatureForSelector: aSelector];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    SEL aSelector = [anInvocation selector];
    if ([self.tc respondsToSelector: aSelector]) {
        [anInvocation invokeWithTarget: self.tc];
    }
    else {
        [super forwardInvocation: anInvocation];
    }
 }
```
完整的转发流程，代价比较高。

用图来总结：  
![消息转发](https://github.com/buptwsgprivate/iOSInterview/blob/master/Images/message_forwarding.png)

### weak如何实现
runtime对注册的类会进行布局，对于weak修饰的对象会放入一个hash表中。用weak指向的对象内存地址作为key，当此对象的引用计数为0的时候会dealloc，假如weak指向的对象内存地址是a，那么就会以a为键在这个weak表中搜索，找到所有以a为键的weak对象，从而设置为nil。

### 用于修饰属性的atomic关键字
atomic只能保证单步操作的原子性，因此，对于简单的赋值或者读取这类操作，还是可以保证该操作的完整性。
一旦涉及到多步骤的操作，还是需要lock等其它的同步机制来确保线程安全。

### 分类(Category)中定义了和原有方法重名的方法，会是什么效果？
将分类中的方法加入类中这一操作，是在运行期系统加载分类时完成的。运行期系统会把分类中所实现的每个方法都加入类的方法列表中。如果类中本来就有此方法，而分类又实现了一次，那么分类中的方法会覆盖原来那一份实现代码。实际上可能会发生很多次覆盖，比如某个分类中的方法覆盖了“主实现”中的相关方法，而另外一个分类中的方法又覆盖了这个分类中的方法。多次覆盖的结果以最后一个分类为准。

所以，最好的做法是为方法加前缀，避免发生覆盖。

### +initialize和+load都是什么？有什么区别？
load:     
      当一个类或是分类被加载进runtime的时候，会被调用load方法。  
      父类的load方法先被调用，子类的load方法之后被调用。   
      分类的load方法在类的load方法之后被调用。    
      
initialize: 当一个类，或是它的子类在接收到第一个消息之前，这个类都会收到initialize消息。  
            父类先收到消息，然后是子类。
            每个类只会收到一次，分类不会收到。    
            
顺序：先是load，然后是initialize。
         
### 讲一下NSObject协议中的isEqual:和hash
* 若想检测对象的等同性，需要同时提供isEqual:和hash两个方法。
* 相同的对象必须具有相同的哈希码，但是两个哈希码相同的对象却未必相同，这种情况发生时叫碰撞。
* hash值用于确定对象在哈希表中的位置。
* 不要盲目地爱个检测每条属性，而是应该依照具体需求来制定检测方案。
* 编写hash方法时，应该使用性能好而且哈希码碰撞几率低的算法。一个例子是：

```
- (NSUInteger)hash {
    NSUInteger firstNameHash = [_firstName hash];
    NSUInteger lastNameHash = [_lastName hash];
    NSUInteger ageHash = _age;
    return firstNameHash ^ lastNameHash ^ ageHash;
}
```

### Method Swizzle的实质是什么？
该特性的实质是，为类添加一个新的方法，然后将目标方法和新的方法的IMP进行互换，结果就是修改selector和IMP的对应关系。

### Block深入
Block本身是对象，下图描述了Block对象的内存而已：  
![Block对象内存布局](https://github.com/buptwsgprivate/iOSInterview/blob/master/Images/Block_Layout.png)

在内存布局中，最重要的就是invoke变量，这是一个函数指针，指向Block的实现代码，函数原型至少要接受一个void*型的参数，此参数代表块。

descriptor变量是指向结构体的指针，每个块里都包含此结构体，其中声明了块对象的总体大小，还声明了copy与dispose这两个辅助函数所对应的函数指针。辅助函数在拷贝及丢弃块对象时运行，其中会执行一些操作，比方说，前者要保留捕获的对象，而后者则将之释放。  

块还会把它所捕获的所有变量都拷贝一份。这些拷贝放在descriptor变量后面，捕获了多少个变量，就要占据多少内存空间。请注意，拷贝的并不是对象本身，而是指向这些对象的指针变量。invoke函数为何需要把块对象作为参数传进来呢？原因就在于，执行块时，要从内存中把这些捕获到的变量读出来。

* 全局Block  

```
    void (^globalBlock)(void) = ^{
        NSLog(@"This is a block");
    };
    NSLog(@"global block is kind of class: %@", [globalBlock class]);
```
输出：global block is kind of class: __NSGlobalBlock__

上面定义的globalBlock，因为没有捕捉任何状态（比如外围的变量等），运行时也无须有状态来参与。块所使用的整个内存区域，在编译期已经完全确定了，因此，全局块可以声明在全局内存里，而不需要在每次用到的时候于栈中创建。这种块实际上相当于单例。

* 堆上的Block

```
    void (^block)(void) = ^{
            NSLog(@"self = %@", self);
        };
    block();
    NSLog(@"block is kind of class: %@", [block class]);
```
输出：block is kind of class: __NSMallocBlock__  
上面定义的block，因为在内部捕捉了self，就从全局Block变成了堆上的Block。

* 栈上的Block   
Block在定义的时候，其所占的内存区域是分配在栈中的。  

```
CGFloat f = 1.1;
NSLog(@"%@", ^{NSLog(@"%lf",f);});
NSLog(@"%@",[^{NSLog(@"%lf",f);} copy]);
void(^deliveryBlock)(void) = ^{NSLog(@"%lf",f);};
NSLog(@"%@", deliveryBlock);
```
经打印得知，只有第一次打印输出类型为__NSStackBlock__, 其它的两次都变成了__NSMallocBlock__。所以栈上的Block，在ARC环境下，一经赋值，copy，参数传递，就都变成了堆上的Block。  


### 阴影的渲染为什么慢？Instruments在View渲染优化中的使用。
如果只是简单的设置了shadowColor, shadowOffset, shadowOpacity等属性，那么Core Animation就得自己去计算阴影的形状。这会导致在渲染的时候，从GPU切换到CPU去计算阴影的形状，计算完成之后再切换回GPU。这种现象叫离屏渲染，降低性能。
可以利用Simulator或Instruments去测试Core Animation的性能。如：  
Color Blended Layers  
Color Copied Images  
Color Misaligned Images
Color Off-screen Rendered

### KVC的原理
Key-Value Coding:   
通过key来访问一个对象的属性或是实例变量的值，而不是通过具体的访问方法。  
直接或者间接继承自NSObject的类，都拥有这个能力，是因为NSObject类中提供了缺省的实现。  
实现的原理就是，将key映射到真正的访问方法。  
例如Setter方法的寻找：  

* 找名字为set<Key>:或_set<Key>的方法，如果找到，调用之。  
* 如果第1步没有找到，并且类方法accessInstanceVariablesDirectly返回YES(默认)，那么会按`_<Key>, _is<Key>, <Key>, is<Key>`的顺序去寻找实例变量。如果找到了，就将值直接设置到实例变量上。  
* 如果最后没有找到，会去调用setValue:forUndefinedKey:。默认的实现是抛出异常，但是子类可以提供不同的行为。

### KVO背后的原理
原理：  
假设self.man为Person类的对象，有如下的注册观察者的代码：  

```
[self.man addObserver: self forKeyPath: @"name" options: kNilOptions context: NULL];
```
如果我们在这一行打上断点，当断点停下来的时候，在控制台里运行```po self.man->isa```，那么会打印NSKVONotifying_Person，可以发现对象的isa指针被指向了一个运行期创建的类。但是运行```po [self.man class]```，却还是显示的Person。
   
使用运行时的class_copyMethodList方法，获取并打印临时类的方法列表，会打印出以下：  
setName:  
class  
dealloc  
_isKVOA  

背后的实现原理：   
当类Person的对象被观察时，KVO在运行时动态创建了一个Person类的子类：NSKVONotifying_Person，并为这个新的子类重写了被观察属性keyPath的setter方法。子类的setter方法会去负责通知观察者对象属性的改变。被观察的实例对象的isa指针被修改为指向新创建的子类。   
子类中的setter方法的工作原理：

```
-(void)setName:(NSString *)newName{ 
    [self willChangeValueForKey:@"name"];    //KVO 在调用存取方法之前总调用 
    [super setValue:newName forKey:@"name"]; //调用父类的存取方法 
    [self didChangeValueForKey:@"name"];     //KVO 在调用存取方法之后总调用
}
```

集合类型的监听：
假设有一个属性是可变数组类型，通过常规的方法，只能监听到数组被赋值的时候。如果想监听到里面的值被添加，删除，那么就得依赖于手动的触发KVO了。

### 如何手动的触发KVO？
为了减少不必要的通知，或是为了将几个变化归为一组一起发出通知，可以覆写下面的类方法来告诉KVO，哪些key使用自动的触发。


```
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)theKey {
 
    BOOL automatic = NO;
    if ([theKey isEqualToString:@"balance"]) {
        automatic = NO;
    }
    else {
        automatic = [super automaticallyNotifiesObserversForKey:theKey];
    }
    return automatic;
}
```

非集合类型的属性，比较简单，只用指定变化的key。

```
///Testing the value for change before providing notification  
- (void)setBalance:(double)theBalance {
    if (theBalance != _balance) {
        [self willChangeValueForKey:@"balance"];
        _balance = theBalance;
        [self didChangeValueForKey:@"balance"];
    }
}
```

对于一个to-many类型的属性，不但要指定变化的key，还要指定变化的类型和所涉及到的对象的索引集。

```
//Implementation of manual observer notification in a to-many relationship  
- (void)removeTransactionsAtIndexes:(NSIndexSet *)indexes {
    [self willChange:NSKeyValueChangeRemoval
        valuesAtIndexes:indexes forKey:@"transactions"];
 
    // Remove the transaction objects at the specified indexes.
 
    [self didChange:NSKeyValueChangeRemoval
        valuesAtIndexes:indexes forKey:@"transactions"];
}
```

### 传值的几种方式？各自的使用场景是什么？
* 通知：  
使用Notification，可以在发送通知者和观察者之间解耦，观察者只需要知道通知的名字，以及通知中携带的参数即可。传值是通过userInfo来传一个字典。
* 代理：  
此种情况下，一般是A持有B，然后A遵从B制定的协议，实现代理方法。在代理方法中，可以自由的传值。  
* block:  
使用block，可以实现两种场景：回调；调用传入的block，来实现自身的逻辑。
   
* selector:  
例如UIGestureRecognizer, UIControl都有addTarget:action:之类的方法，支持添加多于一个的target, action，在需要时通过[target persormSelector]来回调。  

### 简述一个进程的内存分区情况  
1)代码区：存放函数二进制代码
2)数据区：系统运行时申请内存并初始化，系统退出时由系统释放。存放全局变量、静态变量、常量
3)堆区：通过malloc等函数或new等操作符动态申请得到，需程序员手动申请和释放
4)栈区：函数模块内申请，函数结束时由系统自动释放。存放局部变量、函数参数

### NSUserDefaults都可以存储哪些数据类型？
float, double, iteger, boolean, URL, NSData, NSString, NSNumber, NSDate, NSArray, NSDictionary.

### CALayer和UIView
* 为什么会有CALayer? 为了代码复用，在它之上，分别是NSView和UIView。  
* UIView持有一个CALayer的实例，并实现了CALayerDelegate协议，CALayer的内容需要重新加载时，通过-drawLayer:inContext:或-displayLayer:方法，要求UIView提供内容。CALayer之后去管理这个内容。当然CALayer也有自己的一些可视属性。  
* Core Animation是负责图形渲染和动画的框架。自身并不是一个绘制系统，它负责合成和操作一个应用的显示内容，并且利用GPU来实现硬件加速。  
* 很多属性，如frame, position，在UIView层面上修改，也会作用于CALayer上。从这一点儿来看，UIView象是一个CALayer的包装。  
* UIView还用来处理事件，参与职责链，一个APP并不能只用CALayer来去构建。  
* 普通的视图的layer都是CALayer的实例，但可以通过覆写+(Class)layerClass这个方法，改变layer的类型。    
* 树形结构：和UIView的树形结构类似，layer也有相应的树形结构。也可以在不添加视图的情况下，往layer上再添加子layer。  
* 动画：没有与UIView相关联的layer，修改属性值时，会带有隐式动画。UIView自带的那个layer，隐式动画被关闭了，需要通过UIView的动画block，或是Core Animation来做动画。
* 坐标系统：CALayer相比UIView多了一个anchorPoint属性，使用CGPoint结构表示，值域是[0 1]。这个点是各种图形变换的中心点，同时会决定layer的position的位置，缺省值是[0.5,0.5]。  

### CALayer的三个树状结构
不是指三个树状层级，而是指三个layer对象的集合。  

* model layer tree     
  这个集合里面的对象，存储了动画的目标值。平常我们都是和这里的对象打交道。  
* presentation tree   
  这个集合里面的对象，存储的是动画的执行过程中的即时值。  
* render tree   
  这个集合里面的对象，执行实际的动画，是核心动画的私有的。

### CoreGraphics和CoreAnimation的区别和内容
从架构上来说，Core Graphics位于Core Animation的下方，如图所示：  
![架构](https://github.com/buptwsgprivate/iOSInterview/blob/master/Images/ca_architecture_2x.png)  

Core Graphics也叫Quartz 2D, 是一个先进的，二维绘图引擎，可以工作于iOS, tvOS, macOS。
三个核心概念：  
上下文： 主要用于描述图形写入哪里  
路径：是在图层上绘制的内容  
状态：  用于保存配置变换的值，填充和alpha值等。  

个人的理解：UIView会去利用Core Graphics去绘制，绘制好的内容交给Core Animation去做渲染。  

### 说说UIScrollView的原理
scroll view的大小固定，但它的content view可以是任意的尺寸，contentSize属性表明了内容视图的大小。它上面添加了好几种手势，用来滚动内容视图。
感觉这个题用来考新手的吧。  

### UIWebView中注入JS时，获取到的JSContext为什么总是变化？
```
@implementation NSObject (magic)
- (void) webView: (id) unused didCreateJavaScriptContext: (JSContext*) ctx forFrame: (id) frame
{
    // ...
}
@end
```
### 数据库的数据迁移场景及实现

### 用户感觉卡顿后, 如何系统分析卡顿的原因？

### 什么是长连接？有没有优化方案？

### 多线程下载文件实现方案？要能够支持暂停，重新开始。

### GPU和CPU是如何协同工作的？

### 网络优化方案都有哪些？

### 电量优化方案都有哪些？

## 算法相关
### 反转二叉树，非递归

### M个红球和N个黑球排序，有多少种排法？

