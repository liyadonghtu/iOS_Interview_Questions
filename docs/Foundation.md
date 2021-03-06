# Foundation
## 1.nil、NIL、NSNULL、NULL 有什么区别？

- nil 我们给对象赋值时一般会使用object = nil，表示我想把这个对象释放掉；
或者对象由于某种原因，经过多次release，于是对象引用计数器为0了，系统将这块内存释放掉，这个时候这个对象为nil，我称它为“空对象”。（注意：我这里强调的是“空对象”，下面我会拿它和“值为空的对象”作对比！！！）所以对于这种空对象，所有关于retain的操作都会引起程序崩溃，例如字典添加键值或数组添加新原素等.
- NSNull和nil的区别在于，nil是一个空对象，已经完全从内存中消失了，而如果我们想表达“我们需要有这样一个容器，但这个容器里什么也没有”的观念时，我们就用到NSNull，我称它为“值为空的对象”。如果你查阅开发文档你会发现NSNull这个类是继承NSObject，并且只有一个“+ (NSNull *) null；”类方法。这就说明NSNull对象拥有一个有效的内存地址，所以在程序中对它的任何引用都是不会导致程序崩溃的。
- Nil和nil在使用上是没有严格限定的，也就是说凡是使用nil的地方都可以用Nil来代替，反之亦然。只不过从编程人员的规约中我们约定俗成地将nil表示一个空对象，Nil表示一个空类。

- NULL 我们知道Object-C来源于C、支持于C,当然也有别于C。而NULL就是典型C语言的语法，它表示一个空指针，参考代码如下：int *ponit = NULL;

## 2.如何实现一个线程安全的 NSMutableArray? 

NSMutableArray是线程不安全的，当有多个线程同时对数组进行操作的时候可能导致崩溃或数据错误

不能使用atomic 修饰属性
原因：atomic 的内存管理语义是原子性的，仅保证了属性的setter和getter方法是原子性的，是线程安全的，但是属性的其他方法，不能保证线程安全。同时atomic 效率低，编程时慎用。

- 线程锁：使用线程锁对数组读写时进行加锁

```
//可采用 NSLock 方式，简单来说就是操作前 lock 操作执行完 unlock。但注意，每个读写的地方都要保证用同一个 NSLock进行操作。
   NSLock *arrayLock = [[NSLock alloc] init];
    [...]
    [arrayLock lock]; // NSMutableArray isn't thread-safe
    [myMutableArray addObject:@"something"];
    [myMutableArray removeObjectAtIndex:5];
    [arrayLock unlock];

```

- 派发队列：在《Effective Objective-C 2.0..》书中第41条：多用派发队列，少用同步锁中指出：使用“串行同步队列”（serial synchronization queue），将读取操作及写入操作都安排在同一个队列里，即可保证数据同步。而通过并发队列，结合GCD的栅栏块（barrier）来不仅实现数据同步线程安全，还比串行同步队列方式更高效。
```
//利用 GCD 的 concurrent queue 来实现
   dispatch_queue_t concurrent_queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

   For read:

   (id)objectAtIndex:(NSUInteger)index {
   __block id obj;
   dispatch_sync(self.concurrent_queue, ^{
   obj = [self.searchResult objectAtIndex:index];
   });
   return obj;
   }
   For insert:

   (void)insertObject:(id)obj atIndex:(NSUInteger)index {
   dispatch_barrier_async(self.concurrent_queue, ^{
   [self.searchResult insertObject:obj atIndex:index];
   });
   }
   For remove:

   (void)removeObjectAtIndex:(NSUInteger)index {
   dispatch_barrier_async(self.concurrent_queue, ^{
   [self.searchResult removeObjectAtIndex:index];
   });
   }

```

## 3.atomic 修饰的属性是绝对安全的吗？为什么？

不是，所谓的安全只是局限于 Setter、Getter 的访问器方法而言的，你对它做 Release 的操作是不会受影响的。这个时候就容易崩溃了。

## 4.实现 isEqual 和 hash 方法时要注意什么？

- hash
	
	对关键属性的hash值进行位或运算作为hash值

- isEqual
	
	==运算符判断是否是同一对象, 因为同一对象必然完全相同

	判断是否是同一类型, 这样不仅可以提高判等的效率, 还可以避免隐式类型转换带来的潜在风险

	判断对象是否是nil, 做参数有效性检查

	各个属性分别使用默认判等方法进行判断

	返回所有属性判等的与结果

## 5.id 和 instanceType 有什么区别？

- 相同点
	
	instancetype 和 id 都是万能指针，指向对象。

- 不同点：

	1.id 在编译的时候不能判断对象的真实类型，instancetype 在编译的时候可以判断对象的真实类型。

	2.id 可以用来定义变量，可以作为返回值类型，可以作为形参类型；instancetype 只能作为返回值类型。


## 6.self和super的区别

- self调用自己方法，super调用父类方法

- self是类，super是预编译指令

- [self class] 和 [super class] 输出是一样的

- self和super底层实现原理

	1.当使用 self 调用方法时，会从当前类的方法列表中开始找，如果没有，就从父类中再找；

	而当使用 super 时，则从父类的方法列表中开始找，然后调用父类的这个方法

	2.当使用 self 调用时，会使用 objc_msgSend 函数：

	``` c
	id objc_msgSend(id theReceiver, SEL theSelector, ...)
	```

	第一个参数是消息接收者，第二个参数是调用的具体类方法的 selector，后面是 selector 方法的可变参数。以 [self setName:] 为例，编译器会替换成调用 objc_msgSend 的函数调用，其中 theReceiver 是 self，theSelector 是 @selector(setName:)，这个 selector 是从当前 self 的 class 的方法列表开始找的 setName，当找到后把对应的 selector 传递过去。

	3.当使用 super 调用时，会使用 objc_msgSendSuper 函数：

	``` c
	id objc_msgSendSuper(struct objc_super *super, SEL op, ...)
	```

	第一个参数是个objc_super的结构体，第二个参数还是类似上面的类方法的selector

	``` c
	struct objc_super {
  		id receiver;
  		Class superClass;
	};
	```

## 7.@synthesize和@dynamic分别有什么作用？

- @property有两个对应的词，一个是 @synthesize，一个是 @dynamic。如果 @synthesize和 @dynamic都没写，那么默认的就是@syntheszie var = _var;

- @synthesize 的语义是如果你没有手动实现 setter 方法和 getter 方法，那么编译器会自动为你加上这两个方法。

- @dynamic 告诉编译器：属性的 setter 与 getter 方法由用户自己实现，不自动生成。（当然对于 readonly 的属性只需提供 getter 即可）。假如一个属性被声明为 @dynamic var，然后你没有提供 @setter方法和 @getter 方法，编译的时候没问题，但是当程序运行到 instance.var = someVar，由于缺 setter 方法会导致程序崩溃；或者当运行到 someVar = var 时，由于缺 getter 方法同样会导致崩溃。编译时没问题，运行时才执行相应的方法，这就是所谓的动态绑定。

## 8.typeof 和 __typeof，__typeof__ 的区别?

- __typeof __() 和 __typeof() 是 C语言 的编译器特定扩展，因为标准 C 不包含这样的运算符。 标准 C 要求编译器用双下划线前缀语言扩展（这也是为什么你不应该为自己的函数，变量等做这些）

- typeof() 与前两者完全相同的，只不过去掉了下划线，同时现代的编译器也可以理解。

所以这三个意思是相同的，但没有一个是标准C，不同的编译器会按需选择符合标准的写法。

## 9.类族

系统框架中有许多类簇，大部分collection类都是类族。例如NSArray与其可变版本NSMutableArray。这样看来实际上有两个抽象基类，一个用于不可变数组，一个用于可变数组。尽管具备公共接口的类有两个，但任然可以合起来算一个类族。不可变的类定义了对所有数组都通用的方法，而可变类则定义了那些只适用于可变数组的方法。两个类共同属于同一个类族，这意味着二者在实现各自类型的数组时可以共用实现代码，此外还能把可变数组复制成不可变数组，反之亦然。

## 10.struct和class的区别

- 类： 引用类型（位于栈上面的指针（引用）和位于堆上的实体对象）

- 结构：值类型（实例直接位于栈中）
