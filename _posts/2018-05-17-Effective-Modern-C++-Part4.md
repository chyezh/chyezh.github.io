---
layout: post
title:  "[Effective Modern C++] Part4"
date:   2018-05-17 09:18:33 +0800
categories: ["C++ Readings"]
tags: ["C++", "Effective Modern C++"]
---

- Item11: Prefer deleted Functions to Private Undefined Ones
- Item12: Declare Overriding Functions *override*
- Item13: Prefer *const_iterator*s to *iterator*s
- Item14: Declare Functions *noexcept* if They Won't Emit Exceptions

<!-- more -->

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
