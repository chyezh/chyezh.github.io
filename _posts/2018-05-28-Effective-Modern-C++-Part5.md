---
layout: post
title:  "[Effective Modern C++] Part5"
date:   2018-05-28 08:10:36 +0800
categories: ["C++ Readings"]
tags: ["C++", "Effective Modern C++"]
---

- Item15: Use *constexpr* Whenever Possible
- Item16: Make *const* Member Functions Thread Safe
- Item17: Understand Special Member Function Generation
- Item18: Use *unique_ptr* for Exclusive-ownership Resource Management
- Item19: Use *std::shared_ptr* for Shared-ownership Resource Management
- Item20: Use *std::weak_ptr* for *std::shared_ptr* Like Pointers That Can dangle

<!-- more -->

# Item15: Use *constexpr* Whenever Possible

从概念上来说，*constexpr*标识了一个值时，表明这是一个可以在编译期间就知道的值。但*constexpr*对于函数的意义并不像对于值一样简单。

constexpr function的返回值既不一定是const的，也不一定是编译期决定的。这可以理解为constexpr function的一项*feature*，因为这对于实现来说是非常友好的。

## *constexpr* Object

对于*constexpr*修饰的obj，既是*const*的，也是编译期决定的（其实更多的是在*translation*决定的，包含编译和链接）。

编译期决定的值是具有特殊性的：可以位于read—only memory；可以用于*integral constant expression*，比如数组大小、枚举值等。

    int sz;         // Non-constexpr variable.
    constexpr auto arraySize1 = sz;     // error! sz's value not 
                                        // known at compilation.
    std::array<int, sz> data1;          // error!
    constexpr auto arraySize2 = 10;     // Fine, 10 is a const
                                        // expression
    std::array<int, arraySize2> data2;  // Fine.

注意*const*并不提供编译期确定的保障：

    const auto arraySize = sz;          // Fine. arraySize is const.

    std::array<int, arraySize> data;    // error!

即对于object，所有*constexpr*都是*const*的，但*const*的并不都是*constexpr*。

## *constexpr* Function

当*constexpr*用于function时，情况就变得复杂的多。

- *constexpr*函数在传入参数均为编译期可确定参数将产生*constexpr*，即编译期返回值。
- *constexpr*函数在传入参数不全为编译期可确定参数，其行为和普通函数一致。即在运行期返回。  
>

    constexpr int pow(int base, int exp) noexcept { // pow's a func
        ...                                         // never throws.
    }
    constexpr auto numConds = 5;
    std::array<int, pow(3,numConds)> results;       // Fine.

pow()函数在该调用环境下产生了编译期可确定的变量。但如果传入的base和exp不全是编译期常量，那上述代码将会报错，因为array的模板参数有相应的要求：

    auto base = readFromDB("base");             // Get these values
    auto exp = readFromDB("exponent");          // at runtime.
    std::array<int, pow(base, exp)> results;    // error! call pow
                                                // at runtime.

所以为了满足在编译期返回常量，对*constexpr function*有实现的约束，而C++11和C++14中的约束不同。

在C++11中：  
*constexpr function*必须只存在一条*return*语句，不能存在其他任何语句，所以为了符合产生了许多不符合直觉的实现：（使用递归和?:）
    
    constexpr int pow(int base, int exp) noexcept {
        return (exp == 0 ? 1: base*pow(base, exp - 1));
    }

在C++14中：  
这一约束被放宽，即：

    constexpr int pow(int base, int exp) noexcept {
        auto result = 1;
        for(int i = 0;i < exp; ++i) result *= base;
        return result;
    }

*constexpr function*被限制只能够获取和返回*literal types*，即其值可以在编译期确定的类型。

在C++11中：  
所有内置类型（除了*void*）和用户定义类型都可以时*literal types*，因为构造函数与其他成员函数都可以是constexpr。

    class Point {
    public:
        constexpr Point(double xVal = 0, double yVal = 0) noexcept
            : x(xVal), y(yVal){}
        constexpr double xValue() const noexcept { return x; }
        constexpr double yValue() const noexcept { return y; }
        
        void setX(double newX) noexcept { x = newX; }
        void setY(double newY) noexcept { y = newY; }
    private:
        double x, y;
    };

因为构造函数是*constexpr*，如果在构造Point时，传入的xVal和yVal都是编译期常量的话，那么Point也将会是编译期常量。

    constexpr Point p1(-9, 27.7);      // Fine. "runs" constexpr                                        // ctor during compilation.
    constexpr double x = 28.8;
    constexpr Point p2(x, 5.3);         // Ditto.

同样的Point相应的*constexpr*成员函数也能够产生*constexpr*。

    constexpr Point midPoint(const Point& p1, const Point& p2) noexcept {
        return { (p1.xValue() + p2.xValue())/2, 
                 (p1.yValue() + p2.yValue())/2 };
    }

    constexpr auto mid = midPoint;      // Init constexpr object with
                                        // constexpr function.

这样的编程手段既完成了数据的抽象化，还通过*constexpr*将运行成本转移到编译期，提高程序的效率。同时还可以将这些数据直接创建在read-only-memory，使得这类数据可以用在*constant expression*中，用在数组长度、模板参数和枚举量这类参数上。

    p1.setX(1);             // error! p1 is const.

注意到两个set函数没有声明constexpr，因为在C++11中：所有constexpr成员函数默认const；函数返回值为void，非*literal type*。

但在C++14中，这两条也已经被忽略，可以进行这样的声明：

    constexpr void setX(double newX) noexcept {
        x = newX;
    }

    constexpr void setY(double newY) noexcept {
        y = newY;
    }

这可能有些违背直觉：

    p1.setX(1);             // error! p1 is const.

该函数的使用依旧是报错的，因为直接在非*constexpr*环境下执行该操作，是非法的，p1已经是一个*constexpr object*，对其进行任何改变都是非法的，但可以进行这样的操作:

    constexpr Point reflection(const Point& p) noexcept {
        Point result;                   // Create a non-const Point.
        result.setX(-p.xValue());
        result.setY(-p.yValue());
        return result;
    }

该函数内部声明了一个non-const Point，这个Point可以使用setX和setY，如果传入的p是编译期常量，那么result的一系列操作也可以在编译期完成，所以最后返回值可以是一个编译期常量。

    constexpr auto reflec = reflection(p1);    // Create a constexpr.
    std::array<int, static_cast<int>(reflec.xValue())> i;   // Fine.

## Clear Why Use *constexpr* Whenever Possible

- *constexpr object*和*constexpr function*比普通变量和函数拥有更大的适用范围，更强大的效率。
- 注意*constexpr*是函数接口的一部分，既该函数可以用于常量表达式*constant expression*。
- 如果移除*constexpr*有可能导致大量代码非法。所以设计*constexpr*务必小心和慎用。

## Things to Remember

- *constexpr object*是*const*的，而且在编译期初始化。
- *constexpr function*产生*constexpr*当传入编译期常量时。
- *constexpr*变量和函数拥有更广的适用范围和减少运行期销耗。
- *constexpr*是函数和对象接口的一部分。

# Item16: Make *const* Member Functions Thread Safe

# Item17: Understand Special Member Function Generation

对于C++类来说，有一些成员函数是特别的：  
在C++98中，包括默认构造函数(default constructor)、析构函数(destructor)、复制构造函数(copy constructor)、复制赋值操作符(copy assginment operator)。
这类函数在某些条件下，会由编译器自动生成，生成的函数隐含*public*和*inline*，而且总是非*virtual*的，除非基类函数的析构函数声明为*virtual*，则继承类的生成的析构函数为*virtual*。  

在C++11中，这类函数又多了两个成员：移动构造函数(move constructor)、移动赋值操作符(move assignment operator)。其编译器生成的作用(与copy类似)是对非静态类成员进行"memberwise moves"，对于基类部分调用相应的移动方法。  
这些move操作并不是真正的move，该move操作是依赖与copy实现的，因为类型本身不存在move这个概念。

    class Widget {
    public:
        Widget();           // default constructor.
        Widget(const Widget&);      // copy constructor.
        Widget(Widget&&);           // move constructor.
        Widget& operator=(const Widget&);   // copy assignment operator.
        Widget& operator=(Widget&&);    // move assignment operator.
    };e

move的自动生成与copy有一点不同：

两个copy操作是独立的，如果自行实现其中一个，另一个依旧可以由编译器实现；但move不独立，自行实现其中一个，另一个将不会实现。理由是一旦你自行实现move，就意味着你将不使用默认的"memberwise move"的语义，那么编译器没有理由实现一个错误语义的move操作。

# Item18: Use *unique_ptr* for Exclusive-ownership Resource Management

## Smart Pointers
裸指针是强大的，但同时也是令人厌烦的：

- 裸指针的声明没有指出其指向的是对象还是数组
- 裸指针的声明没有表明是否在完成使用后销毁其指向的对象，即声明没有指出该指针是否持有(*owns*)对象
- 即使需要销毁对象，裸指针也没有表明如何销毁该对象，是采用*delete*还是使用不同的销毁机制
- 使用*delete*时，不知道是使用*delete*还是*delete[]*
- 对指针指向的对象使用析构函数时，很难保证只进行一次析构。错过析构导致内存泄漏，多次析构导致未定义行为
- 无从知道一个指针是否为野指针，指针指向的对象被析构后，指针仍然指向该内存产生野指针

智能指针是一条解决上述问题的途径。智能指针是对裸指针的一次封装，保留指针特性的同时避免许多指针容易带来的错误。在C++11中，带来了4中智能指针：*std::auto_ptr std::unique_ptr std::shared_ptr std::weak_ptr*，用于管理对象的生命周期及资源，防止内存泄漏。

*std::auto_ptr*是一个有设计缺陷的智能指针，在移动成为语义的同时，*std::unique_ptr*可以完全替代*std::auto_ptr*，而且更加强大高效。*std::auto_ptr*已经被标准所抛弃。

## *std::unique_ptr*

- *std::unique_ptr*默认情况下和裸指针具有一样的大小，大多数操作，同样的高效。
- *std::unique_ptr*表现为*exclusive ownership*(专属所有)语义。
- 一个非空的*std::unique_ptr*总是独立持有它所指向的对象。
- 移动一个*std::unique_ptr*意味着转让所有权，即dst指针获得对象，src指针设置为空指针。
- *std::unique_ptr*是*move-only*类型，不支持copy语义，因为*std::unique_ptr*不允许共享对象。
- 当*std::unique_ptr*被销毁时，其指向的对象将在这时刻前进行销毁。

## Common Use for *std::unique_ptr*

*std::unique_ptr*可以用于工厂函数的返回类型。
比如我们拥有以下的层次结构：

    class Investment{...};
    class Stock : public Investment{...};
    class Bond : public Investment{...};
    class RealEstate : public Investment{...};

工厂函数通常在堆上创建一个对象通过指针返回给用户，而用户需要对该对象的资源管理负责，通过使用*std::unique_ptr*，保证用户不需要该对象时，对象随着*std::unique_ptr*的销毁而销毁，防止内存泄漏。

    template<typename... Targs>
    std::unique_ptr<Investment>
    makeInvestment(Targs... params);

用户代码：

    {
        ...
        auto pInvestment = makeInvestment(argments);
        ...
    }   // destroy *pInvestment.

同时*std::unique_ptr*也可以用于所有权交接的场景，比如工厂函数返回值移动至容器，容器按照序列移动给某对象的数据成员，最后这个对象会被销毁(带动*std::unique_ptr*销毁以及*std::unique_ptr*指向的对象销毁)。在这个交接过程中，如果出现非典型程序分支或者异常，使用裸指针带来非常高的资源管理成本，而使用*std::unique_ptr*就没有这个问题。

## Use Custom *deleters*

通常，对象的销毁通过*delete*实现，但*std::unique_ptr*可以在构造时配置自定义的*deleter*。比如在对象被销毁之前，需要将对象信息记录进日志：

    auto delInvmt = [](Investment* pInvestment) {
        makeLog(pInvestment);
        delete pInvestment;
    };

    template<typename... Targs>
    std::unique_ptr<Investment, decltype(delInvmt)>
    makeInvestment(Targs&&... params) {
        std::unique_ptr<Investment, decltype(delInvmt)>
            pInv(nullptr, delInvmt);
        if(...){ 
            pInv.reset(new Stock(std::forward<Targs>(params)...));
        } else if(...){
            pInv.reset(new Bond(std::forward<Targs>(params)...));
        } else if(...){
            pInv.reset(new RealEstate(std::forward<Targs>(params)...));
        }
        return pInv;
    }

以上程序有以下几个注意点：
- delInvmt是一个自定义的*deleter*。
- *std::unique_ptr*的第二个模板参数是*deleter*的类型。
- 将裸指针赋予*std::unique_ptr*是非法的。只可以通过初始化或者reset设置智能指针的内含指针。
- 使用完美转发传递参数。

在C++14中，可以实现函数的返回值自动推断(Item3),可以实现更加优雅的表达：

    template<typename... Targs>
    auto makeInvestment(Targs&&... params) {
        auto delInvmt = [](Investment* pInvestment) {
            makeLog(pInvestment);
            delete pInvestment;
        };
        std::unique_ptr<Investment, decltype(delInvmt)>
            pInv(nullptr, delInvmt);
        ...
        return pInv;
    }

默认的*std::unique_ptr*使用*delete*作为*deleter*，这种情况下，*std::unique_ptr*和裸指针有同样的大小。当使用了自定义的*deleter*后，*std::unique_ptr*将会变大(两倍)。当使用*function*作为*deleter*时，*std::unique_ptr*将会多存储一个函数指针；而当使用*function object*时，膨胀将取决于对象；使用*stateless function object*时(E.g. 无捕获的lambda表达式)，不会有膨胀。所以相同的功能，使用*stateless function object*空间表现更好:

    auto delInvmt = [](Investment* pInvestment) {
        makeLog(pInvestment);
        delete pInvestment;
    };
    template<typename... Targs>
    std::unique_ptr<Investment, decltype(delInvmt1)>    // Return type has sizeof(Investment*).
    makeInvestment(Targs... args);
    
    void delInvmt2(Investment* pInvestment) {
        makeLog(pInvestment);
        delete pInvestment;       
    }
    template<typename... Targs>
    std::unique_ptr<Investment, void(*)(Investment*)>   // Return type has sizeof(Investment*)+sizeof(void(*)(Investment*)).
    makeInvestment(Targs... args);

*std::unique_ptr*还可以用于Pimpl(编译防火墙)的实现，见Item22。

## Tips of *std::unique_ptr*

*std::unique_ptr*拥有一个用于数组的特化类型：*std::unique_ptr<T[]>*,所以可以清楚的区分出一个*std::unique_ptr*是指向数组还是对象，防止API的混用：指向对象的*std::unique_ptr*没有下标访问操作(operator[])，数组则没有解除引用的操作(operator*和operator->)。

但通常容器类(*std::array* *std::vector*)是替代原生数组更好的方案。

*std::unique_ptr*表达了*exclusive ownership*的语义，这可能局限了它的使用范围。但C++11提供更加实用和高效的方法，即*std::unique_ptr*可以转化为*std::shared_ptr*：
    
    std::shared_ptr<Investment> sp = makeInvestment(...);   // Converts std::unique_ptr to std::shared_ptr

所以*std::unique_ptr*十分适合用于工厂函数的返回值，因为工厂函数不管其生产的对象是共用的还是独有的。这样的转换使得*std::unique_ptr*的使用更加灵活。

## Things to Remember

- *std::unique_ptr*是一个小型的、快速的、move-only的智能指针，用于*exclusive ownership*的对象的资源管理。
- 默认，*std::unique_ptr*使用*delete*作为*deleter*，*deleter*可以自定义；同时*stateful deleter*和*function pointer*会提高*std::unique_ptr*的大小，*stateless deleter*则不。
- *std::unique_ptr*可以转化为*std::shared_ptr*

# Item19: Use *shared_ptr* for Shared-ownership Resource Management

C++原始的手动生命周期管理(RAII)可以严格的控制变量的生命周期与资源管理。然而垃圾回收(garbage collection)机制是十分方便而且诱人的。*std::share_ptr*正是为了同时享受GC带来的方便与资源的可预测控制而设计的。

*std::share_ptr*表达的是*shared-ownership*语义，即多个指针共享同一个对象，协同处理对象的销毁。使用GC机制，使得用户端不再需要手动管理对象的生命周期及资源的释放，同时保证了销毁的确定性和可预测性。

## Reference Count

*std::share_ptr*使用*reference count*引用计数的方式，追踪管理对象的指针数量。当一个*std::share_ptr*被构造(除了移动构造)为指向某个对象，引用计数增加；当一个*std::share_ptr*被析构，引用计数减少；还有拷贝控制，也对引用计数产生影响。

移动操作使得旧*std::share_ptr*为空指针，不影响引用计数，所以移动操作比拷贝操作更加高效，包含构造、赋值的情形。


引用计数对*std::share_ptr*的性能有一定的影响：

- *std::share_ptr*比裸指针大一倍，因为增加了一个指向引用计数的指针。
- 为了多个指针能够访问引用计数，引用计数被动态创建在堆上。因为指向的对象无法储存这个计数。Item21中会解释使用*std::make_shared*来避免动态分配对性能的影响。
- 增加和减少引用计数的操作必须为原子操作。因为不同线程中，对引用计数的读和写有可能同时发生。因为原子操作比非原子操作更慢，所以*std::share_ptr*一般比裸指针性能要差。

## *std::share_ptr*'s *Control Block*

*std::share_ptr*默认采用*delete*作为*deleter*，但也支持自定义的*deleter*。但是与*std::unique_ptr*的设计不同，*deleter*的类型是*std::unique_ptr*类型的一部分，而*std::shared_ptr*不同：

    auto logingDel = [](Widget* pw) {
        makeLogEntry(pw);
        delete pw;  
    };

    std::unique_ptr<Widget, decltype(logingDel)> upw(new Widget, loggingDel);       // deleter type is part of ptr type.

    std::shared_ptr<Widget> spw(new Widget, logingDel);     // deleter type is not part of ptr type.

*std::shared_ptr*的设计更加灵活，考虑到两个*std::shared_ptr*可以拥有各自的自定义*deleter*:

    auto customDeleter1 = [](Widget* pw){...};
    auto customDeleter2 = [](Widget* pw){...};

    std::shared_ptr<Widget> pw1(new Widget, customDeleter1);
    std::shared_ptr<Widget> pw2(new Widget, customDeleter2);

因为两个*std::shared_ptr*具有相同的类型，那么它们就可以放进同一个容器：

    std::vector<std::shared_ptr<Widget>> vpw{ pw1, pw2 };

同样的，就可以对具有不同的*deleter*但指向对象相同的两个*std::shared_ptr*进行赋值操作。

另一个不同点在于，使用自定义*deleter*并不会影响*std::shared_ptr*的大小，它始终包含两个指针。这意味这不论*deleter*有多大，都不影响*std::shared_ptr*的大小，因为*deleter*是被动态分配的(可能就在堆上，取决于allocator)。

其实*std::shared_ptr*包含有两个指针，其中一个指向目标对象，另一个指向的不仅仅是引用计数，而是一块控制块(control block)。这个控制块中包含了Reference count、Weak count、Other Data(e.g. custom deleter if specified, allocator if specified...)。Control Block的具体实现可能涉及虚函数和继承。

控制块在第一个*std::shared_ptr*构造时创建。一般来说，构造*std::shared_ptr*指向一个对象，构造函数无法获知这个对象是否已经被其他指针指向，所以对控制块有以下约定：

- *std::make_shared*总是创建控制块。
- 当一个*std::shared_ptr*通过*std::unique_ptr*来构造时，总是创建控制块。
- 当一个*std::shared_ptr*通过一个裸指针来构造时，总是创建控制块。

当希望构造*std::shared_ptr*不会产生新的控制块时，使用*std::shared_ptr*或者*std::weak_ptr*作为构造函数的参数。使用*std::shared_ptr*的原则就是一个对象只有一个control block，一个对象对应多个control block将会反复销毁一个对象，导致未定义行为。

因此最好不要尝试使用裸指针初始化*std::shared_ptr*，使用*std::make_shared*替代(见Item21)，当使用自定义*deleter*时，无法使用*std::make_shared*，使用new直接替代，以防止产生裸指针。

一个十分容易使用裸指针去初始化*std::shared_ptr*的场景就是使用this指针初始化智能指针：

    std::vector<std::shared_ptr<Widget>> processedWidgets;  // processedWidgets keep track of Widgets.

    class Widget {
    public:
        ...
        void process() {
            ...
            processedWidgets.emplace_back(this);    // Lead to 'raw pointer constructor'.
        }
        ...
    }

这段代码可以通过编译，并且将this指针传入*std::shared_ptr*的构造函数，导致但对象多控制块，最终导致未定义行为。为了解决这个问题，引入*std::enable_shared_from_this*：

    class Widget : public std::enable_shared_from_this<Widget> {
    public:
        ...
        void process() {
            ...
            processedWidgets.emplace_back(shared_from_this());
        }
        ...
    };

*std::enable_shared_from_this*是一个基类模板，它提供一个*shared_from_this*的方法，能够返回包装了this的*std::shared_ptr*，从而避免裸指针this初始化*std::shared_ptr*。

这样的设计样式称作*The Curiously Recurring Template Pattern(CRTP)*，即奇异递归模板样式。即通过使基类的模板参数包含了继承类型的信息，使得基类成员函数能够实现普通OO不能实现的功能。

在C++17中，*std::enable_shared_from_this*中包含了一个*std::weak_ptr*用于追踪和记录控制块。在使用*shared_from_this*获得指针之前，需要确保控制块已经存在。所以*std::weak_ptr*必须在调用*shared_from_this*之前记录控制块。

## About *std::shared_ptr*

*std::shared_ptr*涉及到动态分配控制块、可能大的*deleter*和空间分配器、虚函数机制、原子操作，所以在性能上和裸指针有着较大的差距，因为不存在没有完美的解决资源管理的方案。

在普通场景中，*std::shared_ptr*使用默认的*deleter*和*allocator*以及使用*std::make_shared*，控制块只有3个字的大小，性能依旧良好；常用操作比如解除引用的开销不比裸指针大；涉及引用计数改变的操作可能包含了1到2个原子操作，可能比非原子操作消耗更大，但是对于单个计算机指令，依旧为单指令；还有虚函数机制只发生一次，即对象销毁的时候。

通过以上代价实现了资源管理的自动化，所以使用*std::shared_ptr*在大多数场景下是合适的；同时在不需要*shared—ownership*的情况下，使用*std::unique_ptr*会有更好的表现，而且*std::unique_ptr*转化为*std::shared_ptr*也十分方便，但牢记反向转换是不允许的，即使计数值为1.

同时，*std::shared_ptr*不具备代替原生数组的能力，因为*std::shared_ptr*所有API都是面向对象实现的，不存在*std::shared_ptr<T[]>*的特化。注意通过设置包含*delete[]*的可调用对象作为*deleter*，将*std::shared_ptr*指向原生数组可以通过编译，但这并不是一个好主意：一方面，*std::shared_ptr*不提供下标访问，只能通过指针算术来实现访问；另一方面，*std::shared_ptr*保持继承类-基类指针转化，对于数组来说这样的操作是未知的。使用容器类替代数组是更好的方案。

## Things to Remember

- *std::shared_ptr*提供了GC机制，用于管理资源和变量的生命周期。
- *std::shared_ptr*比*std::unique_ptr*更大，包含了控制块，要求原子性的引用计数操作。
- 默认析构操作采用*delete*，支持自定义*deleter*。*deleter*的类型不影响*std::shared_ptr*的类型。
- 尽量避免使用裸指针初始化*std::shared_ptr*。

# Item20: Use *std::weak_ptr* for *std::shared_ptr* Like Pointers That Can Dangle

实现一个行为和*std::shared_ptr*相似，但不参加*shared—ownership*的智能指针在某些场景是很方便的，即这个指针不会影响引用计数。这种指针存在一个*std::shared_ptr*不存在的问题，就是指针悬空。一个真正的智能指针需要跟踪指针是否悬空，*std::weak_ptr*就是为了这个问题而实现的。

## About *std::weak_ptr*

*std::weak_ptr*的API很奇特，甚至不像一个指针：不能解除引用、不能进行指针运算、不能比较，*std::weak_ptr*更向是*std::shared_ptr*的一个增强。

*std::weak_ptr*往往通过*std::shared_ptr*构造：

    auto spw = std::make_shared<Widget>();  // Create a share_ptr pointing to a Widget. ref_count set 1.
    ...

    std::weak_ptr<Widget> wpw(spw);     // Create a weak_ptr pointting to the same Widget. ref_count is 1.

    spw = nullptr;  // ref_count is 0. Widget is destroyed.

    if(wpw.expired()) ...   // wpw is dangled.

我们可能希望检测一个*std::weak_ptr*是否为空，然后解除引用，但是*std::weak_ptr*没有解除引用操作。即使这里存在解除引用操作，但在检查与引用的操作间隔，另一个线程可能恰好销毁了对象，导致访问未定义行为。

因此在这里需要一个原子操作，使得检查和解除引用一气呵成。这可以通过使用*std::weak_ptr*构造一个*std::shared_ptr*实现：

    std::shared_ptr<Widget> spw1 = wpw.lock();  // If wpw dangles, spw1 is nullptr.

    auto spw2 = wpw.lock();

    std::shared_ptr<Widget> spw3(wpw);  // If wpw dangles，throw exception(std::bad_weak_ptr).

使用lock()来实现时，若wpw悬空，则*std::shared_ptr*为空指针；使用*std::shared_ptr*的构造函数实现时，若wpw悬空，则抛出异常*std::bad_weak_ptr*。

## How can *std::weak_ptr* be Useful

见如下函数：

    std::unique_ptr<const Widget> loadWidget(WidgetID id);

如果loadWidget是一个成本很高的call，而且同一个Id可能反复使用的。一个很可靠的优化方式就是缓存其返回值；但对每一个Widget进行阻塞缓存同样影响了性能，所以还可以进一步优化：销毁不再使用的Widget。

这种情形下，使用*std::unique_ptr*作为返回值不再合适，因为调用者希望获得缓存指针的同时，还可以自行决定缓存的生命周期。这个缓存指针需要报告指针是否悬空，因为使用者一旦完成了使用，就会销毁对象，见如下实现：

    std::shared_ptr<const Widget> fastLoadWidget(WidgetID id) {
        static std::unordered_map<WidgetID, std::weak_ptr<const　Wdiget>> cache;
        auto objPtr = cache[id].lock();
        if(!objPtr) {
            objPtr = loadWidget(id);
            cache[id] = objPtr;
        }
        return objPtr;
    }

在该实现中，，cache是一个hash表，其中存储了id对应*std::weak_ptr*:

- 当使用新的id读取时，表中没有id对应的key，构造一个空的*std::weak_ptr*，
lock()返回一个空*std::shared_ptr*初始化objPtr，调用loadWidget()获得Widget对象，并将刚构造的*std::weak_ptr*指向对象，最后返回objPtr；
- 当使用已经存在的id读取时，lock()返回一个指向对应对象的*std::shared_ptr*初始化objPtr并返回，大大提高了读取性能。

值得注意的是，*std::weak_ptr*依赖于*std::shared_ptr*，所以返回值必然为*std::shared_ptr*。当客户端使用完id对应的最后一个*std::shared_ptr*，对象被销毁，再次使用该id时，需要重新调用loadWidget，因此依旧有重构空间。

再看另一个设计样式，观察者样式：发布者(subject)是一个会改变状态的对象，观测者(observer)是一个会接收改变通知并刷新通知的对象。通常发布者中包含指向观测者的指针保证能够在状态改变时，更改通知；但发布者不关心观测者的资源问题，只关心观测者是否还存在，以防止访问一个已经不存在的观测者。*std::weak_ptr*十分契合这样的需求，即发布者可以包含一个元素为指向发布者的*std::weak_ptr*的容器。

最后一个例子：如果存在A,B,C三个对象，A和C共享B(即AC中含有一个*std::shared_ptr*指向B)；B中也要含有一个指针指向A，那这个指针有如下几种选择：

- 裸指针：如果A被析构了，而C存在，B依旧存在，但是该裸指针无从知道A是否已经析构，对该指针的解除引用将导致未定义行为。
- *std::shared_ptr*：在这个设计下，AB分别持有对方，即存在循环指针(A to B to A to B)，导致A和B都无法被析构(引用计数至少为1)，必然导致内存泄漏，
- *std::weak_ptr*：可以防止以上的问题，如果A被析构，B中的指针将会悬空，B可以检测到；A，B也可成功的依次析构，因为*std::weak_ptr*不影响引用计数。

这种应用场景其实并不普遍。比如，一些严格层次的数据结构(树...)。子节点通常只被父节点持有，因此父节点中的指向子节点指针可以使用*std::unique_ptr*，而子节点中指向父节点的指针可以直接使用裸指针(因为父节点的生命周期总是比子节点更加长)。

*std::weak_ptr*和*std::shared_ptr*性能相似，都使用一样的控制块，涉及原子操作。值得注意的是:*std::weak_ptr*不影响*shared ownership*的引用计数，但是影响*weak count*，详见Item21。

## Things to Remember

- 使用*std::weak_ptr*，当指针表现为可悬空的情况下。
- *std::weak_ptr*常用在缓存、观察者列表、防止*shared_ptr*循环导致无法析构的错误。
