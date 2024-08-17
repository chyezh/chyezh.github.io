---
layout: post
title:  "[Effective Modern C++] Part7"
date:   2018-07-19 14:32:05 +0800
categories: ["C++ Readings"]
tags: ["C++", "Effective Modern C++"]
---

- Item23: Understand *std::move* and *std::forward*
- Item24: Distinguish Universal References From Rvalue References
- Item25: Use *std::move* on rvalue References, *std::forward* on Universal References
- Item26: Avoid Overloading on Universal References

<!-- more -->

# Item23: Understand *std::move* and *std::forward*

C++11的一大特性就是引入了右值引用(*rvalue reference*)，移动语义(Move Semantics)与完美转发(Perfect Forwarding)就是由右值引用联结起来的：

- 移动语义(*move semantics*)：让编译器进行“移动”而不是“拷贝”操作来减少运行成本；同时也可以实现move-only类型。
- 完美转发(*perfect forwarding*)：使得函数模板能够“完美”的转发参数给内层函数。

但是注意移动语义并不移动、完美转发也并不完美、"type&&"也不代表右值。

## *std::move* and *std::forward*

C++11提供了*std::move*和*std::forward*，用于指示*move semantics*和*perfect forwarding*。  
但是*std::move*不移动任何东西、*std::forward*也不转发任何东西，甚至在运行时刻不做任何事情，不产生额外的代码。*std::move*和*std::forward*只是一个用于强制转换的函数模板。
- *std::move*无条件地将参数转换为右值
- *std::forward*只在特殊情况下，进行强制转换

## How *std::move* Works

*std::move*的C++11实现：

    template<typename T>
    stuct remove_reference {
        using type = T;
    };

    template<typename T>
    stuct remove_reference<T&> {
        using type = T;
    };

    template<typename T>
    stuct remove_reference<T&&> {
        using type = T;
    };

    template<typename T>
    typename remove_reference<T>::type&& move(T&& param) {
        using Type = typename remove_reference<T>::type&&;
        return static_cast<Type>(param);
    }

从实现可以看出，不论传入参数是右值还是左值(注意万能引用)，*std::move*通过*remove_reference*移除引用，然后强制转换得到右值引用作为返回值返回；重要的一点在于当右值引用作为函数值返回时，返回值是一个右值。  
C++14可以有更简便的实现：

    template<typename T>
    decltype(auto) move(T&& param) {
        using Type = remove_reference_t<T>&&;
        return static_cast<Type>(param);
    }

*std::move*其实是指示当前变量希望被进行*move*操作。  
这是一个支持从*std::string*构造的类，传入参数采用值传递，见Item41。

    class Annotation {
    public:
        explicit Annotation(std::string text): val(text);
        ...
    private:
        std::string val;
    }

当然传入参数应当是不变的：

    class Annotation {
    public:
        explicit Annotation(const std::string text): val(text) {};
        ...
    private:
        std::string val;
    }

然后我们希望能够支持从string中move而不是copy字符串:

    class Annotation {
    public:
        explicit Annotation(const std::string text): val(std::move(text) {};
        ...
    private:
        std::string val;
    }

这看起来没有问题，通过*std::move*强制转化为右值，调用*std::string*的移动构造函数，但其实不然：

问题的关键就在于text是一个const变量：

    class string {
    public:
        ...
        string(const string& rhs);
        string(string&& rhs);
        ...
    }

从*std::string*的构造函数可以看出，移动构造函数不能接收*const std::string&&*的右值，反而复制构造函数能够接受这样的右值，所以最终进行的是copy而不是move。从该例子可以看出：

- 如果希望使用*move operation*，就不要将变量声明为const。const的*move*最终匹配上的是*copy*。
- *std::move*只是进行了强制转换，指示该对象适合进行*move*，并不代表最终的操作是*move*。

## How *std::forward* Works

*std::move*是无条件的强制转换，而*std::forward*进行的是有条件的强制转换：

    template<typename T>
    constexpr T&& forward(typename remove_reference<T>::type& param) {
        // forward an lvalue as either an lvalue or an rvalue
        return static_cast<T&&>(param);
    }

    tempalte<typename T>
    constexpr T&& forward(typename remove_reference<T>::type&& param) {
        // forward an rvalue as an rvalue
        static_assert(!is_lvalue_reference<T>::value, "bad forward call");
        return static_cast<T&&>(param);
    }

关于参数的完美转发，即有如下函数：

    template<class T>
    void wrapper(T&& arg) 
    {
        // arg is always lvalue
        foo(std::forward<T>(arg)); // Forward as lvalue or as rvalue, depending on T
    }

参数arg的类型可以称作*forwarding reference*，这是*std::forward*实现完美转发的前提。
通过一层调用后arg对于内层函数总是一个左值，所以参数的完美转发使用前一个*std::forward*定义，但是模板参数T包含了原传入参数的信息实现完美转发：

- 当传入参数为rvalue，T总是为非引用类型，返回的将会是T&&，是一个rvalue。
- 当传入参数为lvalue，T总是为左值引用类型(包含const等限定)，返回的将会是T&，是一个lvalue。  

如下代码：

    void process(const Widget& lval);   // copy
    void process(Widget&& rval);        // move

    template<typename T>
    void UseProcess(T&& param) {
        process(std::forward<T>(param));
    }

    Widget w;
    UseProcess(w);          // use copy version.
    UseProcess(std::move(w));   // use move version.

## Things to Remember

- *std::move*只是将对象无条件强制转换为右值，并没有*move*。
- *std::forward*只是有条件的强制转换对象，并没有*forward*。
- *std::move*和*std::forward*在运行期没有作用，只是对编译的一种指示。

# Item24: Distinguish Universal References From Rvalue References

在C++中，"&&"是具有迷惑性的：

    void f(Widget&& param);
    Widget&& var1 = Widget();
    template<typename T>
    void f(std::vector<T>&& param);
    
    auto&& var2 = var1;
    template<typename T>
    void f(T&& param);      

前3个声明，"&&"指代的是右值引用，而后两个并不是，它指代的是左值或者是右值，即万能引用(*universal reference* or *forwarding reference*)。它不仅可以绑定rvalue，也可以绑定lvalue；不仅可以绑定const，也可以绑定non-const；不仅可以绑定volatile，也可以绑定non-volatile。

## *Universal Reference* and Template Deduce

*Universal Reference*主要出现在两种情况，都属于类型推断：  
作为函数模板参数时：

    template<typename T>
    void f(T&& param);

作为auto推断时:

    auto&& var2 = var1;

*universal reference*的推断结果由initializer决定：当initializer是一个左值时，*universal reference*就是左值引用；当initializer是右值时，*universal reference*就是右值引用。

    Widget w;
    f(w);       // lvalue passed to f; param's type is Widget&.
    f(std::move(w));    // rvalue passed to f; param's type is Widget&&.

万能引用一定是与类型推断相关的，但是存在类型推断并不一定满足万能应用，引用的声明形式也是很重要的：

    template<typename T>
    void f(std::vector<T>&& param);     // rvalue reference.

    std::vector<int> v;
    f(v);        // error! cannot bind lvalue to rvalue reference.

该声明不满足T&&的形式，同时还有：

    template<typename T>
    void f(const T&& param);        // rvalue reference.

万能引用适配const和non-const的情况，上述参数声明也不是万能引用。

    template<class T, class Allocator = allocator<T>>
    class vector {
        ...
        void push_back(T&& x);

        template<class... Args>
        void emplace_back(Args&&... args);
    }

该形式，push_back看上去满足万能引用的形式，但是没有用到类型推断。当定义一个vector：

    std::vector<Widget> v;

类型推断出T为Widget，而push_back对应就有了实现:

    void push_bakc(Widget&& x);     // rvalue reference.

再emplace_back，仍然需要类型推断：

    template<class... Args>
    void emplace_back(Args&&... args);      // universal reference.

所以一个满足参数万能引用的函数模板有如下形式：

    template<typename T>
    void foo(T&& x);        // x is a universal reference.

## *Universal Reference* and Auto Deduce

auto类型推断也可以实现*universal reference*。因为auto的推断结果和模板的推断基本上一样，所以auto的*universal reference*有如下形式：

    auto&& ref = exp;

auto的万能引用不如模板的普遍，但是十分实用，特别是在C++14引入了lambda表达式的auto参数：

    auto FuncInvocation = 
        [](auto&& func, auto&&... params) {
            ...
            // Invoke func on params.
            std::forward<decltype(func)>(func)(std::forward<decltype(params)>(params)...);
            ...
        }

func是*universal reference*，params是*universal reference*的一个包；在函数内进行完美转发。

万能引用的背后其实是引用折叠(*reference collapsing*)机制在起作用，Item28。通过区分*rvalue reference*和*universal reference*使得代码更加具有抽象意义，减少定义模糊。

## Things to Remember

- *universal reference*有两种表达形式，分别在模板推断和auto推断中。
- 类型推断是*universal reference*的前提，不存在类型推断*type&&*就是*rvalue reference*。
- 当*universal reference*绑定左值时，结果为左值引用；当*universal reference*绑定右值时，结果为右值引用。

# Item25: Use *std::move* on rvalue References, *std::forward* on Universal References

*rvalue references*总是和可以*move*的对象绑定在一起：

    class Widget {
    public:
        Widget(Widget&& rhs);   // move constructor. rhs definitely  
                                //refers to an object eligible to moveing.
        ...
    };

所以总是可以对rhs引用的对象进行移动，所以移动构造函数就应该进行这样的实现，对可移动的数据成员进行移动：

    class Widget {
    public:
        Widget(Widget&& rhs) : 
            name(std::move(rhs.name)), pImpl(std::move(rhs.pImpl)){}
        ...
    private:
        std::string name
        std::shared_ptr<Impl> pImpl;
    };

*universal references*(*forwarding references*)指代其绑定的对象有可能可以进行*move*：

    class Widget {
    public:
        template<typename T>
        void setName(T&& newName) {     // newName might be moving-able.
            name = std::forward<T>(newName);
        }
    };

## Why do so

对*rvalue reference*使用*std::forward*是可以的，因为*std::forward*可以实现*std::move*，但是这样的代码可读性差，易出错；对*universal reference*使用*std::mvoe*也是可以的，但是后果很严重，因为有可能篡改一些*lvalue*。

比如以下代码，可以通过编译：

    class Widget {
    public:
        template<typename T>
        void setName(T&& newName) {
            name = std::move(newName);
        }
    };
    
    std::string getWidgetName();
    Widget w;
    auto n = getWidgetName();       // n is lvalue.
    w.setName(n);                   // Move n to w.name.
                                    // Now n is unknown.

也有认为可以通过重载函数替代模板和万能引用来实现：

    class Widget {
    public:
        void setName(const std::string& newName) {
            name = newName;                 // copy version.
        }
        void setName(std::string&& newName) {
            name = std::move(newName);      // move version.
        }
    };

这样的代码好像可以替代模板与万能引用，但其实有许多缺点：  
使用两个函数替代一个函数带来更多的维护成本；同时原版本的实现更加有效率：  
假设有如下代码：

    w.setName("some string");

"some string"是一个字符串字面值，前一个版本可以生成函数：

    void setName(char& newName[12]) {
        name = newName;
    }

这个函数可以直接将字符串传给name的赋值操作符，而后一个版本需要先构造newName，然后将newName传给移动赋值操作符，这就比原版本多出了需要更多的操作，更低的效率。但是致命的还不在这两条原因。  

如果函数的参数数量增加，甚至用上了可变参数，重载的方法将无法使用，因为为了覆盖所有情况需要重载2^n个函数，所以只能使用*universal reference*的方法实现。

## When Use reference More Than One Times

有时传入的*rvalue reference*或者*universal reference*可能需要多次使用，*std::move*和*std::forward*应该只用在最后一次调用：

    template<typename T>
    void setSignText(T&& text) {
        sign.setText(text);
        auto now = std::chrono::system_clock::now();
        signHistory.add(now, std::forward<T>(text));
    }

原因很简单，在最后一次使用之前我们需要保证引用中的内容不能被移走。*std::move*在某些情况下，应该被*std::move_if_noexcept*替代。

## Return Value and *RVO*

如果函数是以值返回，如果想要返回的对象是绑定在*rvalue reference*或者*universal reference*，应该使用*std::move*和*std::forward*包装返回的引用：

    Matrix operator(Matrix&& lhs, const Matrix& rhs) {
        lhs += rhs;
        return std::move(lhs);      // move lhs into return value.
    }

通过这样，如果Matrix是可移动的，可以把参数右值lhs移动到返回的临时对象中去，对比不用的*std::move*：

    Matrix operator(Matrix&& lhs, const Matrix& rhs) {
        lhs += rhs;
        return lhs;     // copy lhs to return value.
    }

如果不用*std::move*，那么进行的就是copy，而lhs是一个建议进行移动的对象，增加了拷贝成本。

如果Matrix不支持移动，那么使用*std::move*的版本也能够成功匹配上复制操作，如果未来Matrix支持了移动操作，那么该函数的代码也不需要重新修改，增加了代码的可维护性。

同样的，对于*universal reference*：

    template<typename T>
    Fraction reduceAndCopy(T&& frac) {
        frac.reduce();
        return std::forward<T>(frac);   // move rvalue into return  
                                        // value. copy lvalue into return value.
    }

注意以上条件限于*reference*，而不是*local variable*，而且是按值返回：

    Widget makeWidget {
        ...
        Widget w;
        return w;       // RVO.
    }

    Widget makeWidget {
        ...
        Widget w;
        return std::move(w);        // move.
    }

对于*local variable*，不要在return中使用*std::move*和*std::forward*，因为这阻止了编译器的优化(*Return Value Optimization*).

RVO：编译器在用于返回的局部变量类型与返回类型一致的情况下可以省略*return value*的copy：这个局部变量包括return语句中创建的临时对象。有时RVO特指对临时对象的返回，而NRVO指代对*named value*的返回。

如果使用*std::move*阻止了RVO，反而多了移动操作，因为std::move(w)其实是一个指向*local variable*的引用，不满足*RVO*的条件。

但同时RVO是一项标准建议的优化，并不是标准(尽管大多数编译器都支持这门优化)。当没有RVO的时候，编译器会被要求使用*move*而不是*copy*进行返回，所以在return中使用*std::move*是徒劳无功的。

同样的理由可以用于返回*by-value parameter*的情形，因为parameter是不会进行RVO的，但是编译器会主动使用*move*而不是*copy*返回这个值，所以没有必要使用*std::move*：

    Widget makeWidget(Widget w) {
        ...
        return w;       // by-value parameter of same type of return.
    }

编译器会自动进行如下的实现:

    Widget makeWidget(Widget w) {
        ...
        return std::move(w);        // treat w as rvalue.
    }

对于返回*local variable*使用*std::move*，并不能优化代码，反而禁止了编译器的优化。只有某些情况下对*skocal variable*使用*std::move*才有意义(比如传入某些函数，而你已经不再需要这个变量了)。所以不要对return的变量使用*std::move*。

## Things to Remember

- 在最后一次使用引用的时候，对*rvalue reference*使用*std::move*,对*universal reference*使用*std::forward*。
- 对于返回*rvalue reference*和*universal reference*，执行第一条规则。
- 对于返回局部变量或者按值传递参数，不要使用*std::move*或者*std::forward*，这禁止了RVO。

# Item26: Avoid Overloading on Universal References

## Use *perfect forwarding template*

*perfect forwarding template*具有超广的适应性和良好的性能，见如下实现：

    class NameLog {
    public:
        ...
        void logAndAdd(const std::string& name) {
            auto now = std::chrono::system_clock::now();
            log(now, "logAndAdd");
            names.emplace(name);
        }
    private:
        std::multiset<std::string> names;
    };

函数logAndAdd的参数为*const std::string&*，可以绑定以下对象：

    std::string myname("Darla");

    NameLog l;

    l.logAndAdd(myname);          // pass lvalue std::string.
    l.logAndAdd(std::string("David"));      // pass rvalue std::string.
    l.logAndAdd("shanshan");                // pass string literal.

第一个传值将myname(lvalue)绑定给name，最后在函数内部作为emplace的函数参数，调用*std::string*的复制构造函数，基本没有可优化的余地；第二个传值将一个rvalue绑定给name，最后在函数内部作为emplace的函数参数，调用*std::string*的复制构造函数，这里显然可以调用*std::string*的移动构造函数来节省时间；第三个传值多了一个临时对象的创建，见Item25。使用*perfect forwarding template*，可以完美优化传值：

    template<typename T>
    void logAndAdd(T&& name) {
        auto now = std::chrono::system_clock::now();
        log(now, "logAndAdd");
        names.emplace(std::forward<T>(name));
    }

第一个传值不变；第二个传值可以使用移动构造函数，第三个传值可以调用*std::string*的以字符串为参数的构造函数，避免临时对象的构造，完美提升代码效率。

## Do Not overloading *perfect forwarding template*

为*perfect forwarding template*重载函数是十分危险的，比如为logAndAdd重载一个以index为参数的函数：

    void logAndAdd(int idx) {
        auto now = std::chrono::system_clock::now();
        log(now, "logAndAdd");
        names.emplace(nameFromIdx(idx));
    }

    l.logAndAdd(22);        // call the non-template version.

    l.logAndAdd(22U);       // call the template version. error.

可以看到只有完美匹配上非模板的版本，才能正确的实现意图。所以重载*perfect forwarding template*函数是非常危险的，因为这个模板很容易实现一些非计划的重载，

由于构造函数经常是重载函数，使用*perfect forwarding constructor*也是非常危险的：

    class Person {
    public:
        template<typename T>
        explicit Person(T&& n)
            : name(std::forward<T>(n));
        explicit Person(int idx)
            : name(nameFromIdx(idx));
        Person(const Person& rhs);
        Person(Person&& rhs);
    private:
        std::string name;
    };

因为*perfect forwarding template*会和其他构造函数产生竞争。

    Person p("Nancy");

    auto cloneOfP(p);       // create new Person from p, template consturctor.
 
因为*perfect forwarding template*生成了比复制构造函数匹配优先级更高的函数:

    explicit Person(Person& n)
        : name(std::forward<Person&>(n));       // have high priority.

只有这个调用是使用复制构造函数：

    const Person cp("const Nancy");

    auto cloneOfP(cp)       // use copy ctor.


同理，继承类的拷贝和移动构造函数不使用基类的拷贝和移动构造函数，而是使用基类的*perfect forwarding constructor*,因为传入的参数并不是完美契合基类的拷贝和移动构造函数。

    class SpecialPerson : public Person {
    public:
        SpecialPerson(const SpecialPerson& rhs)
            : Person(rhs) {
            ...
        }

        SpecialPerson(SpecialPerson&& rhs)
            : Person(std::move(rhs)) {
            ...
        }
    };

所以有可能的化，避免重载一切*perfect forwarding template*，这不是一个良好的设计，容易带来出人意料的后果。但如果不可避免的要重载，解决方案见Item27。

## Things to Remember

- 重载*perfect forwarding template*会使得*perfect forwarding template*在某些出人意料的情况下被调用。
- *perfect forwarding constructor*不是一个良好的设计，会与其他的构造函数产生不可预料的竞争与替代，这种情况在继承类调用基类构造函数时也会发生。
