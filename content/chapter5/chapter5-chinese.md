#第五章 C++内存模型和原子类型操作

**本章主要内容**

>C++11内存模型详解<br>
>C++标准库提供的原子类型<br>
>如何操作各种原子类型<br>
>原子操作如何为线程提供同步功能<br>

C++11标准中，有一个十分重要特性，常被程序员们所忽略。它不是一个新的语法特性，也不是新的库工具，它就是新的多线程(感知)内存模型。没有内存模型明确的定义基本部件应该如何工作的话，之前介绍的那些工具就无法正常工作。当然，为什么大多数程序员都没有注意到它呢？当你使用互斥量去保护你的数据和条件变量，或者是“期望”上的信号事件，对于互斥量*为什么*能起到这样作用，大多数人不会去关心。只有当你试图去“接触硬件”，你才能详细的了解到内存模型是怎么起作用的。

C++是一个系统级别的编程语言。标准委员会的目标之一就是不需要比C++还要底层的高级语言。C++应该向程序员提供足够的灵活性，去做想要做的事情时，没有语言上的障碍；当需要的时候，可以让他们“接触硬件”。原子类型和操作就允许“接触硬件”，提供底层级别的同步操作，通常会将常规指令数缩减到1~2个CPU指令。

在本章，我们将讨论内存模型的基本知识，而后在了解一下原子类型和操作，最后了解与原子类型操作相关的各种同步。这个过程可能会比较复杂：除非你已经打算使用原子操作(比如，第7章的无锁数据结构)同步你的代码，要不你就没有必要了解过多的细节。

现在让我们轻松愉快的来看一下有关内存模型的基本知识。

##5.1 内存模型基础

这里从两方面来讲内存模型：一方面是基本结构，这个结构奠定了与内存相关的事物；另一方面就是并发。基本结构对于并发也是很重要的，特别是当你阅读到底层原子操作的时候，所以我将会从基本结构讲起。在C++中，它与所有的对象和内存位置有关。

###5.1.1 对象和内存位置

在一个C++程序中的所有数据都是由对象(objects)构成。这不是说你可以创建一个int的衍生类，或者在基本类型中存在成员函数，亦或在Smalltalk和Ruby语言下讨论程序时，就意味着“一切都是对象”。这仅仅是对C++数据构建块的一个声明。C++标准定义类对象为“存储区域”，但对象还是可以将自己的特性赋予其他对象，比如，类型和生命周期。

像这样的对象是简单基本类型，比如，int或float；当然，也有用户定义类的实例。一些对象(比如，数组，衍生类的实例，无静态数据成员的类的实现)拥有子对象，但是其他对象就没有。

无论对象是怎么样的一个类型，一个对象都会存储在一个或多个内存位置上。每一个内存位置不是一个标量类型的对象，就是一个标量类型的子对象，比如，unsigned short、my_class*或序列中的相邻位域。当你使用位域，就需要注意：虽然相邻位域中是不同的对象，但仍然视其为相同的内存位置。如图5.1所示，将一个struct分解为多个对象，并且展示了每个对象的内存位置。

![](https://raw.githubusercontent.com/xiaoweiChen/Cpp_Concurrency_In_Action/master/images/chapter5/5-1.png)

图5.1 分解一个struct，展示不同对象的内存位置

首先，完整的struct是一个有多个子对象(每一个成员变量)组成的对象。位域bf1和bf2共享同一个内存位置(int是4字节、32位类型)，并且`std::string`类型的对象s由内部多个内存位置组成，但是其他的每个成员都拥有自己的内存位置。注意，位域宽度为0的bf3是如何与bf4分离，并拥有各自的内存位置的。(译者注：图中bf3是一个错误展示，在C++和C中规定，宽度为0的一个未命名位域强制下一位域对齐到其下一type边界，其中type是该成员的类型。这里使用命名变量为0的位域，可能只是想展示其与bf4是如何分离的。有关位域的更多可以参考[MSDN](https://msdn.microsoft.com/zh-cn/library/ewwyfdbe.aspx)、[wiki](http://en.wikipedia.org/wiki/Bit_field)的页面)。

有四条比较重要的事情需要我们记着：<br>
1. 每一个变量都是一个对象，包括作为其成员变量的其他对象。<br>
2. 每个对象都占有至少一个内存位置。<br>
3. 如int和char的基本类型都有确定的内存位置(无论类型大小如何，即使他们是相邻的，或是数组的一部分)。<br>
4. 相邻位域是相同位域内的一部分。<br>

我确定你会好奇，这些在并发中有什么作用，那么下面就让我们来看一看。

###5.1.2 对象、内存位置和并发

这部分对于C++的多线程应用来说是至关重要的：所有东西都在内存位置上。当两个线程访问不同(*separate*)的内存位置，就不会存在任何问题，一切都工作顺利。另一种情况，当两个线程访问同一(*same*)个内存位置，你就需要小心了。如果没有线程更新内存位置上的数据，那还好；制度数据不需要保护或同步。当有线程对内存位置上的数据进行修改，那就有可能会产生条件竞争，如第3章所述的那样。

为了避免条件竞争，在两个线程间就需要有执行顺序的规定。第一种方式，如第3章所述那样，使用互斥量来确定访问的顺序；当同一互斥量在两个线程同时访问前被锁住，那么在同一时间内就只有一个线程能够访问到对应的内存位置，所以后一个访问必须在前一个访问之后。另一种方式是使用原子操作(*atmic operations*)同步机制(详见5.2节中对于原子操作的定义)，去决定两个线程的访问顺序。使用原子操作来规定顺序在5.3节中会有介绍。当多余两个线程访问同一个内存地址时，对每个访问这都需要定义一个顺序。

如果不去规定两个不同线程对同一内存地址访问的顺序，那么访问都不是原子的；并且，当两个线程都是作者的时候，就会产生数据竞争和未定义行为。

一下的声明由为重要：未定义的行为是C++中最阴暗的角落。根据语言的标准，一旦应用中有任何未定义的行为，就很难预料会发生什么事情；完整应用中的行为现在是未定义的，并且其可能做任何事情。我就知道一个未定义行为的特定实例，让某人的显示器起火的案例。虽然这种事情应该不会发生在你身上，但是数据竞争绝对是一个严重的错误，并且需要不惜一切代价避免它

另一个重点是：当程序中的对同一内存地址中的数据访问存在竞争，你可以使用原子操作来避免未定义行为。当然，这不会阻止竞争自身——原子操作并没有指定那个访问首先去访问——但是原子操作把程序拉回了定义行为的区域内。

在我们了解原子操作前，还有一个有关对象和内存地址的概念需要重点去了解：修改顺序。

###5.1.3 修改顺序

每一个在C++程序中的对象，都有(由程序中的所有线程对象)确定好的修改顺序(*modification order*)，在的初始化开始阶段确定。在大多数情况下，这个顺序不同于执行中的顺序，但是在给定的执行程序中，系统中的所有线程都需要遵守这顺序。如果对象不是一个原子类型(将在5.2节详述)，你有责任确保有足够的同步操作，来确定每个线程都遵守了变量的修改顺序。当不同线程在不同序列中访问同一个值时，你可能就会遇到数据竞争或未定义行为(详见5.1.2节)。如果你使用原子操作，编译器就有责任去替你确定必要的同步操作。

这一要求意味着：投机执行是不允许的，因为一旦线程按修改顺序访问一个特殊的输入，之后的读操作，必须由线程返回较新的值，并且之后的写操作必须发生在修改顺序之后。同样的，对象读取操作允许在同一线程上，要不返回一个已写入的值，要不在对象的修改顺序后(也就是在读取后)写入另一个值。虽然，所有线程都需要遵守程序中每个独立对象的修改顺序，但是它们没有必要遵守在独立对象上的相对操作顺序。在5.3.3节中会有更多关于不同线程间操作顺序的内容。

所以，什么是原子操作？如何让它来规定顺序？接下来的一节中，会为你揭晓答案。

##5.2 C++中的原子操作和原子类型

原子操作是一类不可分割的操作。当这样操作在任意线程中进行一半的时候，你是不能查看的；它的状态要不就是完成，要不就是未完成。如果从一个对象中读取一个值的加载操作是原子的话，并且对对象的所有修改也都是原子的话，那么加载操作要不就会检索对象初始化的值，要不就将值存在某一次修改中。

另一方面，非原子操作可能会被视为由一个线程完成一半的操作。如果这种是一个存储操作，那么可被其他线程观察到的值可能，既不是存储前的值，也不是已存储的值。如果非原子操作是一个加载操作，那么它可能会去检索对象的部分成员，或是有另一个线程修改了对象的值，并且检索之后的对象；因此，检索出来的值可能既不是第一个值，也不是第二个值，可能是某种两者结合的值。这就是一个简单的条件竞争(如第3章所描述)，但是这种级别的竞争会构成数据竞争(详见5.1节)，并且会有未定义行为的发生。

在C++中，大多数情况下，你需要一个原子类型去执行一个原子操作，所以我们来看一下原子类型。

###5.2.1 标准原子类型

标准原子类型(*atomic types*)可以在<atomic>头文件中找到。所有在这种类型上的操作都是原子的，虽然可以使用互斥量去达到原子操作的效果，但只有在这些类型上的操作是原子的(语言明确定义)。实际上，标准原子类型自身可能会效仿：它们(大多数)都有一个is_lock_free()成员函数，这个函数允许用户决定是否直接对一个给定类型使用原子指令(x.is_lock_free()返回true)，或对编译器或运行库使用内部锁(x.is_lock_free()返回false)。

只用`std::atomic_flag`类型不提供is_lock_free()成员函数。这个类型是一个简单的布尔标志，并且在这种类型上的操作都需要是无锁的(*lock-free*)；当你有一个简单无锁的布尔标志是，你可以使用其实现一个简单的锁，并且实现其他基础的原子类型。当你觉得“真的很简单”的时候，就说明：`std::atomic_flag`对象初始化很明确，之后可做查询和设置(使用test_and_set()成员函数)，或清除(使用clear()成员函数)。这就是：无赋值，无拷贝结构，没有测试和清除，没有其他任何操作。

剩下的原子类型都可以通过特化`std::atomic<>`类型模板而访问到，并且拥有更多的功能，但可能不都是无锁的(如之前解释的那样)。在最流行的平台上，会期望原子变量都是无锁的内置类型(例如`std::atomic<int>`和`std::atomic<void*>`)，但这不是必须的。你在后面将会看到，每个特化接口所反映出的类型特点；位操作(如&=)就没有为普通指针所定义，所以它也就不能为原子指针多定义。

除了直接使用`std::atomic<>`类型模板外，你可以使用在表5.1中所示的原子类型集。由于历史原因，原子类型已经添加入C++标准中，这些备选类型名可能参考相应的`std::atomic<>`特化类型，或是特化的基类。在同一程序中混合使用备选名与`std::atomic<>`特化类名，会使代码不可移植。

表5.1 标准原子类型的备选名和与其相关的`std::atomic<>`特化类

| 原子类型 | 相关特化类 |
| ------------ | -------------- |
| atomic_bool | std::atomic&lt;bool> |
| atomic_char | std::atomic&lt;char> |
| atomic_schar | std::atomic&lt;signed char> |
| atomic_uchar | std::atomic&lt;unsigned char> |
| atomic_int | std::atomic&lt;int> |
| atomic_uint | std::atomic&lt;unsigned> |
| atomic_short | std::atomic&lt;short> |
| atomic_ushort | std::atomic&lt;unsigned short> |
| atomic_long | std::atomic&lt;long> |
| atomic_ulong | std::atomic&lt;unsigned long> |
| atomic_llong | std::atomic&lt;long long> |
| atomic_ullong | std::atomic&lt;unsigned long long> |
| atomic_char16_t | std::atomic&lt;char16_t> |
| atomic_char32_t | std::atomic&lt;char32_t> |
| atomic_wchar_t | std::atomic&lt;wchar_t> |

C++标准库不仅提供基本原子类型，还定义了与原子类型对应的非原子类型，就如同标准库中的`std::size_t`。在表5.2中展示这些类型。

表5.2 标准原子类型定义(typedefs)和对应的内置类型定义(typedefs)

| 原子类型定义 | 标准库中相关类型定义 |
| ------------ | -------------- |
| atomic_int_least8_t | int_least8_t |
| atomic_uint_least8_t | uint_least8_t |
| atomic_int_least16_t | int_least16_t |
| atomic_uint_least16_t | uint_least16_t |
| atomic_int_least32_t | int_least32_t |
| atomic_uint_least32_t | uint_least32_t |
| atomic_int_least64_t | int_least64_t |
| atomic_uint_least64_t | uint_least64_t |
| atomic_int_fast8_t | int_fast8_t |
| atomic_uint_fast8_t | uint_fast8_t |
| atomic_int_fast16_t | int_fast16_t |
| atomic_uint_fast16_t | uint_fast16_t |
| atomic_int_fast32_t | int_fast32_t |
| atomic_uint_fast32_t | uint_fast32_t |
| atomic_int_fast64_t | int_fast64_t |
| atomic_uint_fast64_t | uint_fast64_t |
| atomic_intptr_t | intptr_t |
| atomic_uintptr_t | uintptr_t |
| atomic_size_t | size_t |
| atomic_ptrdiff_t | ptrdiff_t |
| atomic_intmax_t | intmax_t |
| atomic_uintmax_t | uintmax_t |

好多种类型！不过，它们有一个相当简单的模式；对于标准typedef T，相关的原子类型就在原来的类型名前加上atomic_的前缀：atomic_T。除了singed类型的缩写是s，unsigned的缩写是u，和long long的缩写是llong之外，这种方式也同样适用于内置类型。对于`std::atomic<T>`模板，使用对应的T类型去特化模板的方式，要好于使用别名的方式。

通常，标准原子类型时不能拷贝和赋值，他们没有拷贝构造函数和拷贝赋值操作。但是，因为可以隐式转化成对应的内置类型，所以这些类型支持赋值，也就可以使用load()和store()成员函数，exchange()、compare_exchange_weak()和compare_exchange_strong()。它们都支持复合赋值符：+=, -=, *=, |= 等等。并且使用整型和指针的特化类型还支持 ++ 和 --。当然，这些操作也有功能相同的成员函数对应：fetch_add(), fetch_or() 等等。返回值通过赋值操作返回，并且成员函数不是对值进行存储(在有赋值符操作的情况下)，就是对值进行操作(在命名函数中)。这就能避免赋值操作符返回引用。为了获取存储在引用的的值，代码需要执行单独读操作，从而允许另一个线程在赋值和读进行的同时修改这个值，这也就为条件竞争打开了大门。

`std::atomic<>`类模板不仅仅一套特化的类型，其作为一个原发模板也可以使用用户定义类型创建对应的原子变量。因为它是一个通用类模板，很多成员函数的操作在这种情况下有所限制：load(),store()(赋值和转换为用户类型), exchange(), compare_exchange_weak()和compare_exchange_strong()。

每种函数类型的操作都有一个可选内存排序参数，这个参数可以用来指定所需存储的顺序。在5.3节，会对存储顺序选项进行详述。现在，只需要知道操作分为三类：

1. Store操作，可选如下顺序：memory_order_relaxed, memory_order_release, memory_order_seq_cst。<br>
2. Load操作，可选如下顺序：memory_order_relaxed, memory_order_consume, memory_order_acquire, memory_order_seq_cst。<br>
3. Read-modify-write(读-改-写)操作，可选如下顺序：memory_order_relaxed, memory_order_consume, memory_order_acquire, memory_order_release, memory_order_acq_rel, memory_order_seq_cst。<br>
所有操作的默认顺序都是memory_order_seq_cst。

现在，让我们来看一下你能对每个标准原子类型进行的操作，从`std::atomic_flag`开始吧。

###5.2.2 std::atomic_flag的相关操作

`std::atomic_flag`是最简单的标准原子类型，它表示了一个布尔标志。这个类型的对象可以在两个状态间切换：设置和清除。它就是那么的简答，只作为一个构建块存在。我从未期待这个类型被使用，除非在十分特别的情况下。正因如此，它将作为讨论其他原子类型的起点，因为它会展示一些原子类型使用的通用策略。

`std::atomic_flag`类型的对象必须被ATOMIC_FLAG_INIT初始化。初始化标志位是“清除”状态。这里没得选择；这个标志总是初始化为“清除”：

```c++
std::atomic_flag f = ATOMIC_FLAG_INIT;
```

这适用于任何对象的声明，并且可在任意范围内。它是唯一需要以如此特殊的方式初始化的原子类型，但它也是唯一保证无锁的类型。如果`std::atomic_flag`是静态存储的，那么就的保证其是静态初始化的，也就意味着没有初始化顺序问题；它常被对标志的第一次操作初始化。

当你的对象标志已初始化，那么你只能做三件事情：销毁，清除或设置(查询之前的值)。这些事情对应的析构函数分别是，clear()成员函数，和test_and_set()成员函数。clear()和test_and_set()成员函数可以有一个指定好的内存顺序。clear()是一个存储操作，所以不能有memory_order_acquire或memory_order_acq_rel语义，但是test_and_set()是一个“读-改-写”操作，所有可以应用于任何内存顺序标签。每一个原子操作，默认的内存顺序都是memory_order_seq_cst。例如：

```c++
f.clear(std::memory_order_release);  // 1
bool x=f.test_and_set();  // 2
```

这里，调用clear()①明确要求，使用释放语义清除标志，当调用test_and_set()②使用默认内存顺序设置表示，并且检索旧值。

你不能由一个对象，拷贝构造另一个`std::atomic_flag`对象；并且，你不能将一个对象赋予另一个`std::atomic_flag`对象。着并不是`std::atomic_flag`特有的，却是所有原子类型共有的。一个原子类型的所有操作都定义为原子的，因为赋值和拷贝调用了两个对象，所以就破坏了操作的原子性。在这样的情况下，拷贝构造和拷贝赋值都会将第一个对象的值进行读取，然后再写入另外一个。对于两个独立的对象，这里就有两个独立的操作了，合并这两个操作必定是不原子的。因此，这些操作就不被允许。

有限的特性集使得`std::atomic_flag`非常适合于作自旋互斥锁。初始化标志是“清除”，并且互斥量处于解锁状态。为了锁上互斥量，循环运行test_and_set()直到旧值为false，就意味着这个线程已经被设置为true了。解锁互斥量是一件很简单的事情，将标志清除即可。实现如下面的程序清单所示：

清单5.1 使用`std::atomic_flag`实现自旋互斥锁
```c++
class spinlock_mutex
{
  std::atomic_flag flag;
public:
  spinlock_mutex():
    flag(ATOMIC_FLAG_INIT)
  {}
  void lock()
  {
    while(flag.test_and_set(std::memory_order_acquire));
  }
  void unlock()
  {
    flag.clear(std::memory_order_release);
  }
};
```

这样的互斥量是最最基本的，但是它已经足够`std::lock_guard<>`使用了(详见第3章)。其本质就是在lock()中等待，当期望其有任何形式的竞争，那么这就是一个糟糕的选择，但是它可以确保互斥。当我们看到内存顺序语义时，你将会看到它们是如何对一个互斥锁保证必要的强制顺序的。这个例子将在5.3.6节中展示。

`std::atomic_flag`局限性太强了，它甚至不能像普通的布尔标志那样使用，因为它没有非修改查询操作。所以你最好使用`std::atomic<bool>`，接下来让我们看看应该如何使用它。

###5.2.3 std::atomic<bool>的相关操作

最基本的原子整型类型就是`std::atomic<bool>`。如你所料，它有着比`std::atomic_flag`更加齐全的布尔标志特性。虽然它还是不可拷贝构造和拷贝赋值，你可以使用一个非原子的bool类型构造它，所以它可以被初始化为true或false，并且你也可以从一个非原子bool变量赋值给`std::atomic<bool>`的实例：

```c++
std::atomic<bool> b(true);
b=false;
```

另一件需要注意的事情时，非原子bool类型的赋值操作不同于通常的操作(转换成对应类型的引用，再赋给对应的对象)：它返回一个bool值来代替指定对象。这是在原子类型中，另一种常见的模式：赋值操作通过返回值(返回相关的非原子类型)完成，而非返回引用。如果一个原子变量的引用被返回了，任何依赖与这个赋值结果的代码都需要显式加载这个值，潜在的问题是，结果可能会被另外的线程所修改。通过使用返回非原子值进行赋值的方式，你可以避免这些多余的加载过程，并且得到的值就是实际存储的值。

虽然有内存顺序语义指定，但是使用store()去写入(true或false)还是好于，使用`std::atomic_flag`中有限制性的clear()函数。同样的，test_and_set()函数也可以被更加通用的exchange()成员函数所替换，exchange()成员函数允许你使用你新选的值替换已存储的值，并且自动的检索原始值。`std::atomic<bool>`也支持对值的普通(不可修改)查找，其会将对象隐式的转换为一个普通的bool值，或显示的调用load()来完成。如你预期，store()是一个存储操作，而load()是一个加载操作。exchange()是一个“读-改-写”操作：

```c++
std::atomic<bool> b;
bool x=b.load(std::memory_order_acquire);
b.store(true);
x=b.exchange(false, std::memory_order_acq_rel);
```

`std::atomic<bool>`提供的exchange()，不仅仅是一个“读-改-写”的操作；它还介绍了一种，当当前值与预期值一致时，存储新值的操作。

**存储一个新值(或旧值)取决于当前值**

这是一种新型操作，叫做“比较/交换”，它的形式表现为compare_exchange_weak()和compare_exchange_strong()成员函数。“比较/交换”操作是原子类型编程的基石；它比较原子变量的当前值和提供的预期值，当两值相等时，存储所需要的值。当两值不等，期望值就会被更新为原子变量中的值。“比较/交换”函数值是一个bool变量，当返回true时执行存储操作，当false则更新期望值。

对于compare_exchange_weak()函数，当原始值与期望值一致时，存储也可能会不成功；在这个例子中变量的值不会发生改变，并且compare_exchange_weak()的返回是false。这可能发生在缺少独立“比较-交换”指令的机器上，当处理器不能保证这个操作能够自动的完成——可能是因为线程的操作将指令队列从中间关闭，并且另一个线程安排的指令将会被操作系统所替换(这里线程数多于处理器数量)。这被称为“伪失败”(*spurious failure*)，因为造成这种情况的原因是时间，而不是变量值。

因为compare_exchange_weak()可以“伪失败”，所以这里通常使用一个循环：

```c++
bool expected=false;
extern atomic<bool> b; // 设置些什么
while(!b.compare_exchange_weak(expected,true) && !expected);
```

在这个例子中，循环中expected的值始终是false，表示compare_exchange_weak()会莫名的失败。

另一方面，如果实际值与期望值不符，compare_exchange_strong()就能保证值返回false。这就能消除对循环的需要，就可以知道是否成功的改变了一个变量，或已让另一个线程完成。

如果你想要改变变量值，且无论初始值是什么(可能是根据当前值更新了的值)，更新后的期望值将会变更有用；经历每次循环的时候，期望值都会重新加载，所以当没有其他线程同时修改期望时，在循环中对compare_exchange_weak()或compare_exchange_strong()的调用都会在下一次(第二次)成功。如果值的计算很容易存储，那么使用compare_exchange_weak()能更好的避免一个双重循环的执行，即使compare_exchange_weak()可能会“伪失败”(因此compare_exchange_strong()包含一个循环)。另一方面，如果值计算的存储本身是耗时的，那么当期望值不变时，使用compare_exchange_strong()可以避免对值的重复计算。对于`std::atomic<boo>`这些都不重要——毕竟只可能有两种值——但是对于更大一些的原子类型就有较大的影响了。

当 “比较/交换”函数很少操作两个拥有内存顺序的参数。这就允许内存顺序语义在成功和失败的例子中有所不同；其可能是对memory_order_acq_rel语义的一次成功调用，而对memory_order_relaxed语义的一次失败的调动。一次失败的“比较/交换”将不会进行存储，所以它将不能拥有memeory_order_release或memory_order_acq_rel语义。因此，这里不能保证提供的这些值能作为失败的顺序。你也不可以提供严格的失败内存顺序；如果你需要memory_order_aquire或memory_order_seq_cst作为失败语序，你必须要像指定它们是成功语序时那样做。

如果你没有指定失败的语序，那就假设和成功的顺序是一样的，除了release部分的顺序：memory_order_release编程memory_order_relaxed，并且memoyr_order_acq_rel变成memory_order_acquire。如果你都不指定，他们默认顺序将为memory_order_seq_cst，这个顺序提供去排序无论顺序是成功，还是失败。下面对compare_exchange_weak()的调用等价于：

```c++
std::atomic<bool> b;
bool expected;
b.compare_exchange_weak(expected,true,
  memory_order_acq_rel,memory_order_acquire);
b.compare_exchange_weak(expected,true,memory_order_acq_rel);
```

我在5.3节中会详解对于不同内存顺序选择的结果。

`std::atomic<bool>`和`std::atomic_flag`的不同是，`std::atomic<bool>`不是无锁的；其实现需要一个内置的互斥量，为了保证操作的原子性。当处于特殊情况时，你可以使用is_lock_free()成员函数，去检查`std::atomic<bool>`上的操作是否无锁。这就是另一个所有原子类型都拥有的特征，除了`std::atomic_flag`之外。

第二简单的原子类型就是特化原子指针——`std::atomic<T*>`，接下来就看看它是如何工作的吧。

###5.2.4 std::atomic<T*>:指针运算

原子指针类型，可以使用内置或自定义类型T，通过特化`std::atomic<T*>`进行定义，就如同使用bool类型定义`std::atomic<bool>`类型一样。虽然接口几乎一致，但是它的操作是对于相关的指针类型的值，而非bool值本身。就像`std::atomic<bool>`，虽然它既不能拷贝构造，也不能拷贝赋值，但是他可以通过合适的类型指针进行构造和赋值。如同必须的成员函数is_lock_free()一样，`std::atomic<T*>`也有load(), store(), exchange(), compare_exchange_weak()和compare_exchage_strong()成员函数，与`std::atomic<bool>`的语义相同，获取与返回的类型都是T*，而不是bool。

`std::atomic<T*>`提供新的操作都是指针运算运算。基本操作有fetch_add()和fetch_sub()提供，它们在存储地址上做原子加法和减法，为+=, -=, ++和--提供简易的封装。操作对于内置类型的操作，如你所预期：如果x是`std::atomic<Foo*>`类型的数组的首地址，然后x+=3让其偏移到第四个元素的地址，并且返回一个普通的`Foo*`类型值，这个指针值是指向数组中第四个元素。fetch_add()和fetch_sub()的返回值略有不同(所以x.ftech_add(3)让x指向第四个元素，并且函数返回指向第一个元素的地址)。这种操作也被称为“交换-相加”，并且这是一个原子的“读-改-写”操作，如同exchange()和compare_exchange_weak()/compare_exchange_strong()一样。正像其他操作那样，返回值是一个普通的`T*`值，而非是`std::atomic<T*>`对象的引用，所以调用代码可以基于之前的值进行操作：

```c++
class Foo{};
Foo some_array[5];
std::atomic<Foo*> p(some_array);
Foo* x=p.fetch_add(2);  // p加2，并返回原始值
assert(x==some_array);
assert(p.load()==&some_array[2]);
x=(p-=1);  // p减1，并返回原始值
assert(x==&some_array[1]);
assert(p.load()==&some_array[1]);
```

函数也接受内存顺序语义作为给定的函数参数：

```c++
p.fetch_add(3,std::memory_order_release);
```

因为fetch_add()和fetch_sub()都是“读-改-写”操作，它们可以拥有任意的内存顺序标签，以及加入到一个释放序列中。指定的语序时不可能是操作符的形式，因为没办法提供必要的信息：这些形式都具有memory_order_seq_cst语义。

剩下的基本原子类型基本上都差不多：它们都是整型原子类型，并且都拥有同样的接口(除了相关的内置类型不一样)。下面我们就看看这一类类型。

###5.2.5 标准的原子整型的相关操作

###5.2.6 std::atomic<>主要类的模板

###5.2.7 原子操作的释放函数