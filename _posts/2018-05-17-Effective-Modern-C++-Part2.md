---
layout: post
title:  "<Effective Modern C++> Part2"
date:   2018-05-17 09:18:33 +0800
categories: C++-Readings
---

# Item11: Prefer deleted Functions to Private Undefined Ones

在C++中，有一类成员函数是特殊的，这些成员函数在某些情况下会被编译器默认实现：'member function5'，即：默认构造函数、复制构造函数、复制赋值操作符、移动构造函数、移动赋值操作符。

但是有时候这类函数是不被需要的，所以需要从实现中删除该实现。

## Declare Them private But Not Define Them or Use = delete.

在C++98中，要实现上面所言的特性，可以通过在private访问控制下声明函数，同时不实现他们，就可以达到目的。比如basic_ios的复制构造函数和复制赋值操作符：

    template<class charT, class traits = char_traits<charT>>
    class basic_ios : public ios_base {
    public:
        ...
        
    pirvate:
        basic_ios(const basic&);                // copy-constructor.
        basic_ios& operator=(const basic&);     // copy-assginment.
    }

C++11中提供了= delete来进行实现deleted function：

    template<class charT, class traits = char_traits<charT>>
    class basic_ios : public ios_base {
    public:
        ...
        basic_ios(const basic&) = delete;                // copy-constructor.
        basic_ios& operator=(const basic&) = delete;     // copy-assginment.
    }
    
使用= delete的好处在于：
- public的访问控制比private能够提供更好的纠错报告。
- = delete可以在编译期发现，而老方法有可能要在link期才能发现

## When Deleted Function is Not the Member Function

还有一个好处就是，= delete可以用在非成员函数上：

    bool isLucky(int number);       // Judge if a number is Lucky num.

    isLucky('a');
    isLucky(true);
    isLucky(3.5);           // A series of unexpected usage.

由于隐式转换带来的非期望的使用方式，可以使用= delete禁止：

    bool isLucky(char) = delete;         // Reject char.
    bool isLucky(bool) = delete;         // Reject bool.
    bool isLucky(double) = delete;       // Reject double and float.

注意double参数的重载同时禁止了float，因为c++对于float进行隐式转换总是优先转换为double而不是int。
牢记被delete的函数是会参加函数重载匹配的，而且优先级不会有改变。

还有一个点deleted function的优越性在于可以禁止一个模板的某些实现：

    template<typename T>
    void processPointer(T* ptr);        // Accept a pointer.

    template<>
    void processPointer<void>(void*) = delete;

    template<>
    void processPointer<char>(char*) = delete;

因为char\*可以代表c风格的字符串，void\*无法解除引用，需要将这两个实现禁止，以方便编译器报错。同样的，对于const指针也许也是不适用的：
    
    template<>
    void processPointer<const void>(const void*) = delete;

    template<>
    void processPointer<const char>(const char*) = delete;

甚至对于const volatile void*也是不适合的：

    template<>
    void processPointer<const volatile void>(const volatile void*) = delete;

    template<>
    void processPointer<const volatile char>(const volatile char*) = delete;

对于类内的模板函数也只能用delete，因为无法特化模板使之有不同的访问控制：

    class Widget {
    public:
        ...
        template<typename T>
        void processPointer(T* ptr) {}
        ...
    private:
        template<>
        void processPointer<void>(void*);       // error!
    }

    class Widget {
    public:
        ...
        template<typename T>
        void processPointer(T* ptr) {}
        ...
    }

    template<>
    void Widget::processPointer<void>(void*) = delete;

对于C++98的老方法，类外不能用，类内不一定能用，能用还可能在链接期才起作用，所以C++11的delete可以完全取代该方法。

## Things to Remember

- 总是使用= delete。
- 所有function都可以被delete，包含模板的特化实现、非成员函数...

# Item12: Declare Overriding Functions *override*

C++中涉及OOP的主要包含了类、继承、虚函数。虚函数的概念相当于在子类中覆盖(override)了基类函数的实现，而这一机制常常出现错误。

因为"overriding"和"overloading"是非常易于混淆的。

    class Base {
    public:
        virtual void doWork();      // Base class virtual func.
        ...
    };

    class Derived : public Base {
    public:
        virtual void doWork();      // Derived class virtual func.
        ...
    };

    std::unique_ptr<Base> uqb =         // Create a ptr of Base-typed
        std::make_unique<Derived>();    // pointing to Derived-obj.

    ...
    upb->doWork();                  // Call the doWork func.

## Requirement of Overriding

见上述代码，为了overriding机制生效，必须满足以下几个条件：

- 基类的函数必须声明为*virtual*
- 基类和子类的函数名必须一样(除了析构函数)
- 函数参数列表必须一样
- 函数constness必须一样
- 返回类型和异常修饰必须兼容

以上为C++98的内容，C++11新添加了：

- reference限定符必须相同 

包含有虚函数错误的代码往往是合法的，能够通过编译的，所以编译器往往不能提出警报：

    class Base {
    public:
        virtual void mf1() const;
        virtual void mf2(int x);
        virtual void mf3() &;
        void mf4() const;
    };

    class Derived : public Base {
    public:
        virtual void mf1();
        virtual void mf2(unsigned int x);
        virtual void mf3() &&;
        void mf4() const;
    };

以上代码不包含任何override机制的函数，都或多或少不满足要求。编译器可能不会提醒你，因为编译器也不明白你的需求是什么。

## *override* And *final*

为了解决这一问题，C++11提供了显式要求override的方法，提供了关键字*override*：

    class Derived : public Base {
    public:
        virtual void mf1() override;
        virtual void mf2(unsigned int x) override;
        virtual void mf3() && override;
        void mf4() const override;
    };

这样一来编译器必然报错，以上代码无法通过编译，因为编译器已经知道了你的需求是override，只有以下代码才能通过编译。

    class Base {
    public:
        virtual void mf1() const;
        virtual void mf2(int x);
        virtual void mf3() &;
        virtual void mf4() const;       // Add virtual
    };

    class Derived : public Base {
    public:
        virtual void mf1() const override;
        virtual void mf2(int x) override;
        virtual void mf3() & override;
        void mf4() const override;      // virtual is not necessary.
    };

C++11中的*final*和*override*是contextual keywords。意味着这两个关键字可只在某些情况下被看作关键字。比如*override*只有在成员函数尾巴才是关键字。

## Reference Qualifiers

reference qualifiers可以理解为修饰*this的qualifier，这点和member function的const类似：

    void doSomething(Widget& w);        // Accepts only lvalue.

    void doSomething(Widget&& w);       // Accepts only rvalue.

    class Widget {
    public:
        void doWork() &;        // Only when *this is lvalue.
        void doWork() &&;       // Only when *this is rvalue.
    ...
    };

    Widget makeWidget();
    Widget w;

    makeWidget().doWork();      // Use && version.
    w.doWork();                 // Use & version.

再看以下代码：

    class Widget {
    public:
        using DataType = std::vector<double>;
        ...
        DataType& data() { return values; }
    private:
        DataType values;
    };

    // Client code：
    Widget w;
    ...
    auto vals1 = w.data();      // Fine.copy the w.values to vals1.

    Widget makeWidget();

    auto vals2 = makeWidget().data();   // Copy too.

前一个用户代码，data()返回了一个value的左值引用，所以调用了copy-constructor，没有问题。第二个用户代码也是类似的操作，所以也没有问题。但是可以发现makeWidget()返回的是一个Widget的临时对象，是一个右值，在表达式结束后就被立刻销毁。所以使用拷贝整个values是浪费时间的，应该使用移动语义的方式节省时间，即调用vector的move-constructor:

    class Widget {
    public:
        using DataType = std::vector<double>;
        ...
        DataType& data() & { return values; }
        DataType&& data() && { return std::move(values); }
    private:
        DataType values;
    };

    // Client code：
    Widget w;
    ...
    auto vals1 = w.data();      // Fine.copy the w.values to vals1.
                                // Use copy-constructor.
    Widget makeWidget();

    auto vals2 = makeWidget().data();   // Move the values to vals2.
                                        // Use move-constructor.

还有一点要注意的是，一旦一个member function被reference qualify，所有重载函数都要reference qualify。因为不进行qualify，就意味这重载函数不论左右值*this都可以使用，这些重载函数会对reference qualified one产生竞争，容易导致多义。

## Things to Remember

- 使用*override*声明override function。
- 使用reference qualifier让左右值的对象有不同的行为。

# Item13: Prefer *const_iterator*s to *iterator*s

在C++11中完善了对*const_iterator*的支持：
- 对*const_iterator*的获取，即cbegin、cend等成员函数、以及C++14中的非成员函数。

- iterator仅做指示作用的算法，如insert、erase对const_iterator支持。   
> 

    std::vector<int> values; 
    ...
    auto it = std::find(values.cbegin(), values.cend(), 1983);
    values.insert(it, 1998);

这些对*const_iterator*的支持，也提升了泛型编程，以及C++标准化的水平：

    template<typename C, typename V>
    void findAndInsert(C& container,
                       const V& targetV, 
                       const V& insertV) {
        using std::cbegin;
        using std::cend;

        auto it = std::find(cbegin(container),      // Use non-member
            cend(container), targetV);              // version.
        
        container.insert(it,insertV);
    }

使用非成员函数的cbegin..，该模板函数不仅对STL标准容器提供了支持，还对那些符合标准容器的第三方容器以及原生数组提供支持。

再看看cbegin非成员函数的实现方法：

    template<typename C>
    decltype(auto) cbegin(const C& container) {
        return std::begin(container);
    }

令人奇怪的是函数最后返回的是begin而不是使用cbegin成员函数。解释如下：因为container是对传入容器的const引用，begin将会调用container的const版本的.begin()，最后返回的是*const_iterator*。这样既达到了目的，又实现了对没有cbegin成员函数的容器的适配，提高了泛型模板的覆盖面。

## Things to Remember

- 尽量使用*const_iterator*
- 在泛型编程中尽量使用non-member function形式的begin、end、rbegin...

# Item14: Declare Functions *noexcept* if They Won't Emit Exceptions

C++98中的exception specifications带来糟糕的维护成本，在C++11中给出了更加简单直接的异常标识，就是直接指出代码是否会抛出异常，由此带来了关键字*noexcept*。代码是否会抛出异常关乎到用户代码的调用效率和异常安全。

    int foo() throw();          // C++98 style. less optimizable.
    int foo() noexcept;         // C++11 style. most optimizable.

同时一个函数是否会抛出异常对于编译器优化也十分关键。比如foo抛出了异常，违反了无异常声明。noexcpt可能会在程序终止前进行栈展开，而throw版本必须进行栈展开。这就给予编译器更灵活的空间去优化。

## In Standard Library

*noexcept*在C++标准库的优化也十分重要。

比如vector的push_back函数在需要reserve的情况下，需要将旧空间中的元素转移至新空间中。

在C++98中，一致使用copy。这样的好处是异常安全，因为即使抛出异常，原空间中的所有元素在完成copy之前都不会有改变；同时这样带来的开销也是巨大的。

在C++11中，将使用"move if you can, but copy if you must"的策略：move的方式可以带来更好的效率，但是move是异常不安全的，因为在移动过程中抛出异常而原本空间中的元素已经被改变，如果反向还原也有可能会抛出新的异常。

所以在C++11中，如果元素的move操作被标识不抛出异常，那这类函数(eg. deque::insert)就将采用move而非copy的方式转移元素。这样可以提高许多效率。

另一个例子是swap函数，swap函数的异常性质是由用户的swap函数的异常决定的。比如对于数组的swap：

    template<class T, size_t N>
    void swap(T (&a)[N], T (&b)[N])
        noexcept(noexcept(swap(*a, *b)));

这里使用到了*conditionally noexcept* 

## Use *noexcept* With Caution

许多函数其实是异常中立（*exception neutral*）的，即函数自己并不会抛出异常，但是其调用的函数会。最好不要为了使用noexcept去对外刻意隐藏这些异常，比如catch了所有异常、或者转换为错误码之类的方式，无疑会增加代码的复杂度，以及维护成本，换取的性能可能反而被这些处理所淹没。

对于一部分函数*noexcept*是十分重要的，所以这类函数默认为*noexcept*:

- memory deallocation function(i.e., operator delete\operator deletep[])
- destructor

在C++11中，这已经上升到了语言规则的层次，所有memory deallocation function和destructor，不论是编译自动生成还是用户定义的，都应该默认为*noexcept*（并不是必须，只是非常非常应该）。

只有一种情况，destructor非默认*noexcept*，即当有数据成员（包含基类）的destructor会抛出异常（e.g. "noexcept(false)"）。这种类型在使用标准库算法与容器时抛出异常都将是未定义行为。

## *wide contracts* and *narrow contracts*

有些库会将函数区分为*wide contracts*和*narrow contracts*。*wide contracts*函数不对传入参数添加约束，不必照顾程序的状态。这类函数永远不会出现未定义行为，比如vector::size，我们申请了一块内存并将它强制转换为vector，但这个情况下size()的输出是合理的，符合定义的，但是该程序的行为确实没有定义保障。

而*narrow contracts*函数会对传入参数增加限制，并且需要照顾到程序状态。如果传入参数违反了限制，那么程序的结果是未定义的：

    void f(cosnt std::string& s) noexcept;      // Prediction: s.size()>=32.

如果传入string的长度小于32，那么该函数将会是未定义的。但是保证该前提是用户代码的义务，而不是f()的义务，所以f()不会进行参数检查，也不会抛出任何异常，所以声明*noexcept*的理由是充分的。

但是如果f()的实现进行了参数检查（防御式编程），因为处理异常往往比处理未定义行为要简单的多。所以f()将抛出异常，但是由于*noexcept*将会导致程序直接终止。

所以往往这种库设计时只会为*wide contracts*函数保留*noexcept*。

## Compiler Offer no Help About Inconsistencies Between Implementations and Exception Specifications

    void setup();            // Functions defined elsewhere.
    void cleanup();

    void doWork() noexcept {
        setup();
        ...
        cleanup();
    }

doWork的异常声明和实现是矛盾的，因为setup和cleanup均没有声明*noexcept*。但是setup和cleanup却有可能在文档中说明了不会抛出任何异常，可能由于其他原因（比如库设计时间过去久远...）而没有声明*noexcept*，。所以doWork声明*noexcept*是完全合理的，编译器可能不会提出警告。

## Things to Remember

- *noexcept*是函数接口的一部分，和*const*...一样，函数调用时可能会依赖于它。
- 编译器对*noexcept*函数的优化更强。
- *noexcept*对于swap和move操作等十分重要，对memory deallocation函数和destructor具有特殊的机制。
- 绝大多数函数都是异常中立的。
- 编译器对于函数的实现和异常声明上的矛盾有可能不会进行检查与警报。

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
