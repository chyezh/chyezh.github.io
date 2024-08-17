---
layout: post
title:  "[Effective Modern C++] Part6"
date:   2018-06-11 17:30:13 +0800
categories: ["C++ Readings"]
tags: ["C++", "Effective Modern C++"]
---

- Item21: Prefer *std::make_unique* and *std::make_shared* to Direct Use of *new*
- Item22: When Using the *Pimpl* Idiom, Difine Special Member Functions in the Implementation File

<!-- more -->

# Item21: Prefer *std::make_unique* and *std::make_shared* to Direct Use of *new*

*std::make_shared*和*std::make_unique*是三个*make*函数之二，还有一个函数是*std::allocate_shared*。*make*函数的作用是接收参数包，完美转发参数包给对象的构造函数用于动态内存申请，并且返回指向该对象的智能指针。*std::allocate_shared*工作与*std::make_shared*类似，但额外接收一个空间适配器用于分配动态空间。

## *make* Perform Better When Writting Exception-Safe Code

使用*make*比不使用*make*在构造智能指针时不同：

    auto upw1(std::make_unique<Widget>());      // With make.
    std::unique_ptr<Widget> uwp2(new Widget);   // Without make.

    auto spw1(std::make_shared<Widget>());      // With make.
    std::shared_ptr<Widget> spw2(new Widget);   // Without make.

首先，使用new的方式需要重复对象的类型，make方式则不：源文件中重复类型导致编译成本提高，可能导致目标代码膨胀和代码不一致，同时也提高了拼写成本。第二个原因，*make*异常安全，而new不是，见下：

    void processWidget(std::shared_ptr<Widget> spw, int priority);

    int computePriority();

    processWidget(std::shared_ptr<Widget>(new Widget), computePriority());      // Potential resource leak.

当编译器将上述代码编译为目标代码：运行时，函数的调用必须要在函数参数评估完成后进行，所以在调用processWidget之前，下述操作必然已经发生：

- new Widget已经计算，Widget必然在堆上创建。
- shared_ptr<Widget>已经构造完成，并且指向new出的Widget。
- computePriority已经调用完毕。

然而编译器并没有被要求依次进行上述操作，于是有可能发生下面的情况："new Widget"的调用必须在shared_ptr<Widget>构造前进行；但是computePriority可以发生在两件事情的中间，之后或者之前:

- 进行"new Widget"
- 进行computePriority
- 进行shared_ptr<Widget>的构造

这样的目标代码显然不是异常安全的，一旦computePriority抛出异常，"new Widget"必然泄漏，因为shared_ptr<Widget>还没有构造出来去接管这个对象。但是使用*make*是异常安全的:

    processWidget(std::make_shared<Widget>(), computePriority());      // Exception safe.

运行时，不论两个函数那个先运行，都是异常安全的。当computePriority先行，并抛出异常，对象尚未构造；当computePriority后运行，新对象始终被一个智能指针所接管。因此*make*函数在异常安全方面比"new"的表现更好。

## *std::make_shared* is More Efficient

使用*std::make_shared*使得编译器能够产生更小更快的代码：

    std::shared_ptr<Widget> spw(new Widget);

该代码看上去进行了一次内存申请，其实进行了两次，因为*shared_ptr*额外需要一个控制块的申请；该申请会在构造函数中进行，所以直接使用"new"会进行两次内存申请，一个供给对象，一个供给控制块。

    auto spw = std::make_shared<Widget>();

如果使用*std::make_shared*，则只需要一次申请：这是因为*std::make_shared*直接申请了一块内存块，储存对象和控制块，提高了运行代码的速度。对*std::make_shared*的效率分析，同样适用与*std::allocate_shared*。

## Circumstances Where *make* Shouldnot be Used

- *make*函数不能用于声明自定义*deleter*
- *make*函数不能用于*braced—initializer*  

如下代码，调用的是非initializer_list版本的构造函数：

    auto upv = std::make_shared<std::vector<int>>(10, 20);
    auto spv = std::make_unique<std::vector<int>>(10, 20);

当使用*braced-initialzation*必须使用"new"，而不能使用*make*，除了直接调用*initializer*版本的构造函数：

    auto inilist = { 10, 20 };
    auto spv = std::make_shared<std::vector<int>>(inilist);

以上两个场景就是对*std::unique_ptr*的限制，而对于*std::shared_ptr*还有更多限制。

有些类会重载它们的*operator new*和*operator delete*，那么全局的内存分配以及释放机制对这些类将不再适合。这个重载的实现往往只会申请一块对象大小的内存，只管理对象所需要的资源，而这样的实现是不适合std::shared_ptr的，因为申请的内存还要包含控制块的大小。所以*std::make_shared*使用类自定义的*operator new*和*operator delete*不是一个好主意。

*std::make_shared*将对象的控制块放在同一块内存中，当引用计数归零，对象被析构，但是内存块并没有被释放，因为控制块还没有被析构。控制块中包含了引用计数等等信息，在引用计数归零后，控制块不一定就被销毁，因为控制块还有其他的登记记录，即第二个计数*weak count*,这个计数中记录了多少个*std::weak_ptr*指向这个内存块。因为*std::weak_ptr*需要通过查询控制块的*reference count*来知道自己是否悬空，所以只要有*std::weak_ptr*指向控制块，那么控制块就不能被析构，控制块所在的内存块就不能被释放。

如果对象本身是十分巨大的，而且最后一个*std::shared_ptr*和最后一个*std::weak_ptr*的析构的时间差也很明显，那么对象析构以及内存释放之间就会存在滞后：

    class ReallyBigType {...};

    auto pBigObj = std::make_shared<ReallyBigType>();
    ... // Create std::weak_ptr and std::shared_ptr to obj.
    ... // final std::shared_ptr to obj and obj destroyed here.
        // but std::weak_ptr to it remain.
    ... // during this period, memory formerly occupied by large
        // obj remains allocate.
    ... // final std::weak_ptr to it desturyed here.
        // memory for control block and obj is freed.

但如果使用"new"，就不会存在这样的滞后，因为控制块和对象的内存块是分开的：

    std::shared_ptr<ReallyBigType> pBigObj(new ReallyBigType());
    ... // Create std::weak_ptr and std::shared_ptr to obj.
    ... // final std::shared_ptr to obj destroyed here.
        // obj is destoryed and memory for obj is freed.
        // but std::weak_ptr to it remain.
    ... // during this period, memory formerly occupied by large
        // obj remains allocate.
    ... // final std::weak_ptr to it desturyed here.
        // memory for control block is freed.

当然为了异常安全，在使用"new"直接生成智能指针，务必保证new出的裸指针马上被传给智能指针的构造函数，即该语句中不要再做其他事情来防止编译器产生顺序不合适的代码:

    void processWidget(std::shared_ptr<Widget> spw, int priority);

    void cusDel(Widget* ptr);

    processWidget(std::shared_ptr<Widget>(new Widget,cusDel), computePriority());     // Exception unsafe.

    std::shared_ptr<Widget> spw(new Widget, cusDel);
    processWidget(spw, computePriority())      // Safe. but not optimal.

将构造放在单独的一条语句中，可以避免异常不安全。即使构造函数抛出异常(比如控制块的申请出现异常)，可以保证cusDel用于析构对象，并且释放内存。

但是从性能影响上看，内存不安全版本更好，因为其传入构造的是一个右值，而异常安全版本是一个左值。右值构造使用的是move，而左值进行的是copy；而copy一个*std::shared_ptr*要求原子操作的引用增加，削弱性能表现。

    processWidget(std::move(spw), computePriority());   // both efficient and exception safe.

使用std::move可以兼顾性能与异常安全，但是原指针将被设置为空指针。

## Things to Remember

- 对比"new"，*make*函数减少代码膨胀，加强异常安全，*make_shared*和*allocate_shared*还有更好的性能表现和更小的代码。
- 不使用*make*的情形：需要使用自定义*deleter*或者*braced-initializer*。
- 对于*std::shared_ptr*，不使用*make*的情景还有：对象具有自定义的内存管理机制、(出于内存空间的考虑)对于非常大的类型同时*std::shared_ptr*和*std_weak_ptr*析构时间差很大的情况。

# Item22: When Using the *Pimpl* Idiom, Difine Special Member Functions in the Implementation File

## the *Pimpl* Idiom

*Pimpl*(pointer to implementation)是一门用于减少编译成本的技术：

    class Widget {          // In header "Widget.h"
    public:
        Widget();
        ...
    private:
        std::string name;
        Gadget g;           // User-defined type.
    }

Widget是一个拥有std::string, Gadget类型成员变量的类，定义在"Widget.h"中；任何需要调用Widget的源代码都需要包含这个头文件，在包含这个头文件的同时也就包含了声明Gadget的头文件，如果"Gadget.h"是一个经常改变内容的头文件，那么就大大增加了编译成本。

    class Widget {          // In header "Widget.h".
    public:
        Widget();
        ~Widget();
        ...
    private：
        struct Impl;        // Declare implementation struct.
        Impl *pImpl;        // and pointer to it.
    }

使用Pimpl，将数据成员封装进一个声明了的结构体(不定义)中，再用指针指向这个结构体。因为"Widget.h"中没有使用这些类型，所以就不用包含这些头文件，对这些头文件做修改不影响包含"Widget.h"的源代码，提高了编译效率。

但注意Impl是一个只声明而未定义的类型(*incomplete type*)，只有少数对它的行为是合法的，比如声明指向它的指针。以上代码只完成了声明，之后需要进行定义：

    #include "Widget.h"     // In impl. file "Widget.cpp".
    #include "Gadget.h"
    #include <string>

    struct Widget::Impl {   // Define Impl.
        std::string name;
        Gadget g;
    };

    Widget::Widget() : pImpl(new Impl)
    {}

    Widget::~Widget() {
        delete pImpl;
    }

通过*Pimpl*，分离了头文件"Widget.h"对于"Gadget.h"的依赖。于是当Gadget.h的内容发生变化时，只需要重新编译"Widget.cpp"，而不需要重新编译其它包含"Widget.h"的用户源代码，提高了编译期间的效率。

## Use *std::unique_ptr* instead of raw pointer

在*Pimpl*中，使用智能指针*std::unique_ptr*替代裸指针会产生一个编译错误*cannot delete an incomplete type*：

    class Widget {          // In header "Widget.h".
    public:
        Widget();
        ...
    private：
        struct Impl;        // Declare implementation struct.
        std::unique_ptr<Impl> pImpl;        // and *std::unique_ptr* to it.
    }

然后定义：

    #include "Widget.h"     // In impl. file "Widget.cpp".
    #include "Gadget.h"
    #include <string>

    struct Widget::Impl {   // Define Impl.
        std::string name;
        Gadget g;
    };

    Widget::Widget() : pImpl(std::make_unique<Impl>()){}    // Create std::unique_ptr.

因为*std::unique_ptr*可以自动管理资源，所以我们理应可以不再自行定义析构函数，而交给编译器自行实现，这是没有问题的，问题在于在哪实现。我们无法对一个*incomplete type*的变量进行如*delete*和*sizeof*之类的操作，编译错误意味着生成析构函数参与编译的位置处Impl还没有被定义。

对编译错误信息层层查看可以发现：编译器默认生成的析构函数时内联的，内联位置Impl还没有被定义，而析构函数调用*std::unique_ptr*的析构，*std::unique_ptr*的析构又会对Impl进行默认的*delete*，最后static_assert进行对象是否为*imcomplete type*的判断时出现了fail。

解决这个问题只要对析构函数进行显示的定义即可：

    class Widget {          // In header "Widget.h".
    public:
        Widget();
        ~Widget();          // Declare destructor.
        ...
    private：
        struct Impl;        // Declare implementation struct.
        std::unique_ptr<Impl> pImpl;        // and *std::unique_ptr* to it.
    }

然后定义：

    #include "Widget.h"     // In impl. file "Widget.cpp".
    #include "Gadget.h"
    #include <string>

    struct Widget::Impl {   // Define Impl.
        std::string name;
        Gadget g;
    };

    Widget::Widget() : pImpl(std::make_unique<Impl>()){}    // Create std::unique_ptr.

    Widget::~Widget() = default;    // Default destructor.

因为显式定义了析构函数，所以move操作都不会被编译器生成；所以如果需要支持move操作，需要显式定义：

    class Widget {          // In header "Widget.h".
    public:
        Widget();
        ~Widget();          // Declare destructor.
        
        Widget(Widget&& rhs) = default;     // Wrong.
        Widget& operator=(Widget&& rhs) = default;  // Wrong.
        ...
    private：
        struct Impl;        // Declare implementation struct.
        std::unique_ptr<Impl> pImpl;        // and *std::unique_ptr* to it.
    }

移动赋值函数需要销毁当前对象持有的Impl；移动构造函数默认生成的异常处理中包含了对Impl的销毁，因此上述代码不能够通过编译，处理方法同析构函数：

    class Widget {          // In header "Widget.h".
    public:
        Widget();
        ~Widget();          // Declare destructor.
        
        Widget(Widget&& rhs);
        Widget& operator=(Widget&& rhs);
        ...
    private：
        struct Impl;        // Declare implementation struct.
        std::unique_ptr<Impl> pImpl;        // and *std::unique_ptr* to it.
    }

然后定义：

    #include "Widget.h"     // In impl. file "Widget.cpp".
    #include "Gadget.h"
    #include <string>

    struct Widget::Impl {   // Define Impl.
        std::string name;
        Gadget g;
    };

    Widget::Widget() : pImpl(std::make_unique<Impl>()){}    // Create std::unique_ptr.

    Widget::~Widget() = default;    // Default destructor.
    Widget::Widget(Widget&& rhs) = default;
    Widget& Widget::operator=(Widget&& rhs) = default;

如果Impl中的成员支持copy操作，那么Widget也可以支持copy操作：1、因为编译器无法自行实现包含*move-only*类型成员的类的copy操作，2、即使编译器能够实现，也只能进行浅层拷贝，无法进行深层拷贝；所以需要手动实现copy操作：

    class Widget {          // In header "Widget.h".
    public:
        Widget();
        ~Widget();          // Declare destructor.
        
        Widget(Widget&& rhs);
        Widget& operator=(Widget&& rhs);
        Widget(const Widget& rhs);
        Widget& operator=(const Widget& rhs);
        ...
    private：
        struct Impl;        // Declare implementation struct.
        std::unique_ptr<Impl> pImpl;        // and *std::unique_ptr* to it.
    }

然后定义：

    #include "Widget.h"     // In impl. file "Widget.cpp".
    #include "Gadget.h"
    #include <string>

    struct Widget::Impl {   // Define Impl.
        std::string name;
        Gadget g;
    };

    Widget::Widget() : pImpl(std::make_unique<Impl>()){}    // Create std::unique_ptr.

    Widget::~Widget() = default;    // Default destructor.
    Widget::Widget(Widget&& rhs) = default;
    Widget& Widget::operator=(Widget&& rhs) = default;

    Widget::Widget(const Widget& rhs) : pImpl(nullptr) {
        if(rhs.pImpl) pImpl = std::make_unique<Impl>(*rhs.pImpl);
    }
    Widget& Widget::operator=(const Widget& rhs) {
        if(!rhs.pImpl) pImpl.reset();
        else if(!pImpl) pImpl = std::make_unique<Impl>(*rhs.pImpl);
        else *pImpl = *rhs.pImpl;
        return *this;
    }

copy操作的定义中需要考虑传入参数的情况、现对象的情况、指针为空的情况。总之，得益于编译器对Impl自动实现的一系列copy操作，使得函数的实现变得十分简单。

## When using *std::shared_ptr*

*Pimpl*的实现一般使用*std::unique_ptr*因为其指针逻辑很显然是独占的，但也可以考虑一下使用*std::shared_ptr*的情况：

    class Widget {          // In header "Widget.h".
    public:
        Widget();
        ...
    private：
        struct Impl;        // Declare implementation struct.
        std::shared_ptr<Impl> pImpl;        // and *std::unique_ptr* to it.
    }

然后定义：

    #include "Widget.h"     // In impl. file "Widget.cpp".
    #include "Gadget.h"
    #include <string>

    struct Widget::Impl {   // Define Impl.
        std::string name;
        Gadget g;
    };

    Widget::Widget() : pImpl(std::make_shared<Impl>()){}    // Create std::shared_ptr.

可以发现使用*std::shared_ptr*不再有上述繁杂的特殊函数定义，之所以如此：  
*std::shared_ptr*和*std::unique_ptr*的*deleter*的实现方式不同，*deleter*的类型是*std::unique_ptr*的一部分，所以*deleter*再编译期就可以连接上*std::unique_ptr*，这样具有更好的运行期空间时间效率；而*std::shared_ptr*不同，*deleter*其实是*std::shared_ptr*实例的一部分，会有更大的数据结构和运行期成本，但是*deleter*不必要在编译期就被连接上。

## Things to Remember

- *Pimpl*是一项通过减少编译头文件间(类用户代码和类实现)依赖来降低编译成本的技术。
- 对于使用*std::unique_ptr*的*Pimpl*，需要手动定义类特殊函数来支持实现，尽量使用编译器的默认实现。
- 对于*std::shared_ptr*没有上述要求。
