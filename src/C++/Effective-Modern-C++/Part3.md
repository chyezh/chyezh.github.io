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
# Item27: Familiarize Yourself With Alternatives to Overloading on Universal Reference

Item26阐明了同时使用*perfect forwarding template*和*overloading function*是很危险的。所以当碰见这种情况的时候一般有以下几种方法：

- *Abandon overloading*：放弃重载，使用不同的函数名分别实现*perfect forwarding template*和其他函数的功能。
- *Pass by const T&*：放弃*perfect forwarding template*，使用*const T&*传参，缺点是不够高效，性能明显有损耗。
- *Pass by value*：放弃*perfect forwarding template*，使用按值传参，见Item41。

那当不可避免的同时使用*perfect forwarding template*和*overloading function*的时候，应该如何实现？

## Using *Tag Dispatch*

接着Item26的例子：  
第一种方法，为函数加上标签。函数的外层封装依旧不变，因为这是面向用户代码的，那么目标就是通过函数内部自动甄别传入参数的类型，来调用相应的重载，那么实现如下：

    class NameLog {
    public:
        ...
    template<typename T>
    void logAndAdd(T&& name) {
        logAndAddImpl(std::forward<T>(name), std::is_integaral<std::remove_reference_t<T>>());
    }
    private:
        std::multiset<std::string> names;
    };

*std::is_integral*是一个*type trait*，可以无视*cv-qualifier*来判断一个类型是否为整数，使用*std::remove_reference*来移除引用，最后调用*call*操作符产生一个*tag*，传个内层函数：

    template<typename T>
    void logAndAddImpl(T&& name, std::true_type) {
        auto now = std::chrono::system_clock::now();
        log(now, "logAndAdd");
        names.emplace(nameFromIdx(idx));
    }

    template<typename T>
    void logAndAddImpl(T&& name, std::false_type) {
        auto now = std::chrono::system_clock::now();
        log(now, "logAndAdd");
        names.emplace(std::forward<T>(name));
    }

*std::false*和*std::true_type*就是所谓的*tag*，可以帮助决定最终调用哪一个函数，注意没有为*tag*声明参数名：因为*tag*只是用于提示编译器最后调用哪个函数，在运行期间不起任何作用，最后有可能可以被编译器优化取代，这正是我们所期待的，这项设计被称为*tag dispatch*。

## Constraining template that take *universal references*

注意使用*tag dispatch*的基石是使用单个函数(非重载)作为用户端的API，但对于特殊函数，这是无法实现的，比如构造函数。Item26提到*perfect forwarding template*是十分贪婪的，经常能够完美匹配传入参数，而在匹配竞争中受到不合理的调用。这个时候，我们就希望限定*perfect forwarding template*在某些情况下起作用，而不是总是能够被匹配上。  

    std::enable_if<condition>::type

这项技术的关键就是*SFINAE*。在标准库中给出了*std::enable_if*，它在条件正确的情况下，会有类型成员*value*，而在条件错误的情况下，没有这个成员。

    class Person {
    public:
        template<typename T, 
                 typename = std::enable_if_t<!std::is_same_v<Person, std::decay_t<T>>>>
        explicit Person(T&& n)
            : name(std::forward<T>(n)) {}
        explicit Person(int idx)
            : name(nameFromIdx(idx)) {}
    private:
        std::string name;
    };

*std::enable_if_t*可以得到*std::enable_if*的类型成员*value*；*std::is_same_v*可以得到一个bool值反应两个模板参数是否同类型；*std::decay_t*可以得到类型T退化后的类型(删除*cv-qualifier*和引用，对数组和函数退化为指针)。当传入一个希望使用copy或者move构造函数的参数，*perfect forwarding template*参与匹配，但是*std::enable_if*没有类型成员value，匹配失败但是不报错(*SFINAE*)，最后这个匹配被踢出匹配队列，拷贝构造函数或者移动构造函数当选。

但是可以发现仍然不能解决继承带来的匹配竞争，因为基类和继承类在*std::is_same_v*中得到的是false:

    class Person {
    public:
        template<typename T, 
                 typename = std::enable_if_t<!std::is_base_of_v<Person, std::decay_t<T>>>>
        explicit Person(T&& n)
            : name(std::forward<T>(n)) {}
        explicit Person(int idx)
            : name(nameFromIdx(idx)) {}
    private:
        std::string name;
    };   

*std::is_base_of_v*可以判断后一个类型是否为前一个类型的继承类，如果是用户定义类型同类型也被判断为true(内置类型为false)。最后在加上整数判断:

    class Person {
    public:
        template<typename T, 
                 typename = std::enable_if_t<!std::is_base_of_v<Person, std::decay_t<T>> &&
                 !std::is_integral_v<std::remove_reference_t<T>> >>
        explicit Person(T&& n)
            : name(std::forward<T>(n)) {}
        explicit Person(int idx)
            : name(nameFromIdx(idx)) {}
    private:
        std::string name;
    };       

## Trade-offs

所有方法中，采用*perfect forwarding template*的方法应该比其他方法更加高效，理由在Item26中说过。但是*perfect forwarding template*也有劣势。其一有些参数无法进行完美转发，见Item30。其二就是C++虐心的错误消息了：

    Person p(u"Konrad Zuse");   // const char16_t

u"Konrad Zuse"是不匹配string的构造函数的，这样会抛出错误消息。但是这个错误是在多层转发之后才能发现的，会导致错误信息可读性极差。

在这个例子中，我们知道参数是用于初始化*std::string*的，所以可以使用*static_assert*进行检查:

    class Person {
    public:
        template<typename T, 
                 typename = std::enable_if_t<!std::is_base_of_v<Person, std::decay_t<T>> &&
                 !std::is_integral_v<std::remove_reference_t<T>> >>
        explicit Person(T&& n)
            : name(std::forward<T>(n)) {
                static_assert(
                    std::is_constructible_v<std::string T>,
                    "Parameter n cannot be used to construct a std::string"
                );
            }
        explicit Person(int idx)
            : name(nameFromIdx(idx)) {}
    private:
        std::string name;
    };

*std::is_constructible*是用于检测initializer的类型是否可以用于构造另一个类型的。这能给我们一个特殊的错误提示，但是可惜的是*static_assert*的出现位置晚于构造，因为构造函数先构造对象后执行函数体。

## Things to Remember

- 解决*perfect forwarding template*和*overloading*的矛盾，可以有以下解决办法：放弃重载、用按常右值引用传参、用按值传参、使用*tag dispatch*。
- 可以使用*SFINAE*限制*perfect forwarding template*的应用场景。
- *perfect forwarding template*具有更高的效率，但是易错难用。

# Item28: Understand *Reference Collapsing*

*universal reference*当接收左值时，表现为左值引用，接收右值时，表现为右值引用。其背后的内在机制是*reference collapsing*，即引用折叠。

    template<typename T>
    void func(T&& param);

当param是一个左值时，T为左值引用；当param是一个右值时，T为非引用类型:

    Widget widgetFactory();
    Widget w;
    func(w);                    // T deduced to be Widget&.
    func(widgetFactory());      // T deduced to be Widget.

## *Reference Collapsing*

注意在C++中，定义实实在在的"引用的引用"是非法的：

    int x;
    auto& & rx = x;     // error.

但是在函数模板中，会发生*reference collapsing*引用折叠：

    template<typename T>
    void func(T&& param);

    func(w);        // T is Widget&, T&& is Widget& &&?

引用折叠的机制很简单，两个引用只要其中一个为左值引用，则折叠为左值引用；若两个引用均为右值引用，则折叠为右值引用：

    func(w);        // T is Widget&, T&& is Widget&(& && collapse to &).

再看看*std::forward*如何工作：

    Widget fparam;

    template<typename T>
    void f(T&& fparam) {
        someFunc(std::forward<T>(fparam));
    }

    template<typename T>
    T&& std::forward(remove_reference_t<T>& param) {
        return static_cast<T&&>(param);
    }

当*fparam*是一个左值时，T为Widget&，生成*std::forward<Widget&>*:

    Widget& && std::forward(Widget& param) {
        return static_cast<Widget& &&>(param);
    }

*reference collapsing*：

    Widget& std::forward(Widget& param) {
        return static_cast<Widget&>(param);
    }

函数返回一个左值引用，是一个左值(rvalue)。
当*fparam*是一个右值时，T为Widget，生成*std::forward<Widget>*:

    Widget&& std::forward(Widget& param) {
        return static_cast<Widget&&>(param);
    }

函数返回一个右值引用，是一个右值(消亡值xvalue)。

*reference collapsing*发生在4种语法环境中：其一就是上面的模板实例化；其二是*auto*生成类型时。因为auto的类型推断和template基本相同，所以*reference collapsing*也类似：

    auto&& w1 = w;  // auto is Widget&, auto&& is Widget&, reference collapsing happen.
    auto&& w2 = widgetFactory();    // auto is Widget, auto&& is Widget&&, no reference collapsing.

*universal reference*只是一个概念上的抽象，绕开繁杂的类型推断和引用折叠，所以*universal reference*不是新的引用，只是由以下两点原因作用而成的：

- 因为类型推断区分了*lvalue*和*rvalue*，前者推断为T&；后者推断为T。
- *reference collapsing*发生。

另外还有两种发生*reference collapsing*的场景：其三，使用*typedefs*和*alias declarations*；

    template<typename T>
    class Widget {
    public:
        typedef T&& RvalueRefToT;
        ...
    }

    Widget<int&> w;

    typedef int& && RvalueRefToT;   // reference collapsing happen.

    typedef int& RvalueRefToT;

其四，使用*decltype*：当涉及*decltype*的类型分析时，出现指向引用的引用，发生*reference collapsing*。

## Things to Remember

- *reference collapsing*在四种场景中发生：模板实例化，*auto*类型生成，*typedefs*和*alias declarations*，*decltype*
- *reference collapsing*发生时，其中一个引用是左值引用，则为左值引用；任意一个引用为右值引用，则为右值引用。
- *universal reference*的条件：类型推断区分了*lvalue*和*rvalue*，前者推断为T&；后者推断为T；*reference collapsing*。

# Item29: Assume That Move Operations Are Not Present, Not Cheap, And Not Used

*move semantics*是C++11的一块重要的拼图，它带来了更高效的拷贝操作，更准确的拷贝语义。但是*move semantics*并不像想象中的那么高效、普遍。

- 许多类型不支持移动操作
- 许多类型的移动操作并不如想象中的高效

首先许多类型是不支持*move semantics*的。C++标准库在C++11进行了一次彻底的翻新，许多类型都加入了更加高效的移动操作，但也有许多类型并没有。  
所有C++11的标准模板库中的容器都支持移动操作，但是并不是所有容器的移动操作都是高效的：可能因为这类容器根本无法支持高效的移动操作；也可能因为容器元素无法配合容器实现高效的移动。

比如*std::array*，其实是一个带有STL接口的原生数组。其他STL容器的元素大都是储存在堆上的，而*std::array*是存储在栈上的。所以*std::vector*的移动，本质上只需要改变标记元素位置用的若干个指针就可以，复杂度O(1)。而*std::array*的移动，需要依次调用每一个元素的移动，复杂度O(n):


    std::vector<Widget> vw1;
    ...
    auto vw2 = std::move(vw1);  // move vw1 into vw2. runs in constant time. only ptrs in vw1 and vw2 are modified.

    std::array<Widget, 10000> aw1;
    ...

    auto aw2 = std::move(aw1);  // move aw1 into aw2. runs in linear time. All elements in aw1 are move into aw2.

再看Widget的拷贝操作，如果Widget的移动比拷贝高效，那么上述容器的移动依旧是比拷贝要高效的，所以std::array确实需要支持移动语义。

再看另一个例子*std::string*。*std::string*支持短字符串优化*small string optimization*(SSO)。如果*std::string*中的字符串足够小，那么字符串的存储将不会分配在堆上，而是分配在一块内置buffer上。所以对于小字符串的移动并不比拷贝高效。

即使有些类支持高效的移动操作，最后依旧可能被耗时的拷贝操作所替代。见Item14.有些容器操作要求强异常安全保证，只有当移动操作声明不抛出异常的情况下，才会使用*move*代替*copy*。

以下应用场景，*move semantics*不会有利于程序的效率:

- No move operations：类型不支持move。
- Move not faster：move并不比copy快。
- Move not usable：场景要求move操作不会抛出异常，然而move并没有声明*noexcept*。
- Source object is lvalue：除了个别例外，只有右值可以用作移动的来源。

## Things to Remember

- 总是假定*move operations*不存在、不效率、不可用。
- 在确认可以使用*move semantics*的情况下，无视上一条。

# Item30: Familiarize Yourself With *perfect forwarding* Failure Cases.

完美转发意味着不仅仅转发对象，而且转发对象的值类型(左值还是右值？)，对象的cv-限定(const or volatile)。而*universal reference*的类型推断包含了对象的该类信息，帮助我们转发这些信息。

    template<typename T>
    void fwd(T&& param) {
        f(std::forward<T>(param));      // forward it to f.
    }

    template<typename... Args>
    void fwd(T&& param...) {
        f(std::forward<T>(param)...);   // forward package to f.
    }

但是完美转发并不总是成功转发的。比如fwd传入参数不符合f的要求就会导致转发的失败。还有一些参数碍于某些语言特性，不能够通过完美转发：

- 模板无法推断当前类型
- 编译器推断类型不符合内层函数的参数要求

## *Braced Initializer*

比如f有如下形式：

    void f(const std::vector<int>& v);

对f传入*braced initializer*是能够通过编译的：

    f({1, 2, 3});       // fine.

但是如果进行完美转发，结果是失败

    fwd({1, 2, 3});     // failure.

前者，*braced initializer*能够对参数vector<int>进行初始化。但是后者先要通过模板的类型推断，但是模板是无法对*braced initialzer*进行推断的(见Item2)。

使用*braced initializer*就属于编译器无法推断模板类型。

Item2中提到，auto可以对*braced initializer*进行推断，所以以下代码可以将*braced initializer*推断为*std::initialzer_list*通过编译。

    auto il = {1, 2, 3};
    fwd(il);        // fine.

## 0 or NULL as null Pointer

Item8解释了0和宏NULL和空指针的关系。对于模板而言，类型推断总是把0和NULL推断为整型(通常为int)。所以使用0和NULL也是无法完美转发的，属于第二种错误：

    void f(void* p);
    fwd(0);             // failure.
    fwd(NULL);          // failure.
    fwd(nullptr);       // fine.

## Declaration-only Integral *static const* and *constexpr* Data Member 

一般来说，不需要定义*static const*或者*static constepxr*的值，只需要声明就行。因为编译器会进行*const propagation*，从而不需要为这类变量开拓空间：

    class Widget {
    public:
        static const std::size_t MinVals = 28;  // declare, not define.
    }

对于调用该变量的函数，编译器会使用常量直接去补充参数：

    void f(std::size_t val);

    f(Widget::MinVals);     // fine.

但是对于完美转发就不行了，因为没有定义该变量，所以一切有关于对该变量的地址操作都是非法的。而完美转发模板对该参数的推断会是一个引用。

    fwd(Widget::MinVals);   // T is const size_t&. error should not link.

所以该操作是无法通过链接的。

但是这是对于标准而言，而许多编译器都会通过上述代码，但是这毕竟是不符合标准的，为了上述代码的可移植性，应当加入对该变量的定义:

    constexpr std::size_t Widget::MinVals;      // in cpp.

注意定义不应该重复初始化，初始化应该只存在于一个地方。

## Overload Function Names and Template Names

假如有以下接收函数指针为参数的函数：

    void f(int(*pf)(int));

当然可以忽略指针：

    void f(int(pf)(int));

有以下两个过程函数:

    int processVal(int val);        // ver.1
    int processVal(int val, int priority);  // ver.2

调用f:

    f(processVal);      // fine. use ver.1.

    fwd(processVal);    // error! which processVal？

要注意到完美转发模板终究只是一个模板，它不会顾及自己内部，只会先进行参数推断，再生成实例，才有内部实现。

解决这个问题，可以先将重载函数绑定给一个固定函数类型的指针上或者进行强制转化，再进行传参：

    int(*pf)(int) = processVal;
    fwd(pf);

    fwd(static_cast<int(*)(int)>(processVal));

## *Bitfields*

比如有以下位域定义：

    struct IPv4Header {
        std::uint32_t   version:4,
                        IHL:4,
                        DSCP:6,
                        ECN:2,
                        totalLength:16;
    }；

    void f(std::size_t sz);

    IPv4Header h;

    f(h.totalLength);       // fine. implicit conversion happen
    fwd(h.totalLength);     // error.

这个问题类似于*braced initializer*，模板无法对位域进行类型推断，同时没有任何引用和指针可以指向位域。所以解决方案就是将位域拷贝出来并做强制转换：

    auto length = static_cast<std::uint16_t>(h.totalLength);
    fwd(length);

## Things to Remember

- 完美转发通常在两种场景下失败：无法推断类型和“错误”推断类型。
- 常见的失败有：*braced initializer*、*0 or NULL as null pointer*、*declaration-only const static data member*、*template and overloaded funciton names*、*bitfields*# Item31: Avoid Default Capture Mode

*lambda*表达式并没有给C++带来什么新的东西，但是极大的简化了工作，快速生成临时对象(比如*trivial predicates*)，对于STL的使用意义重大。关于*lambda*主要有以下几点概念:

- *lambda expression*：单纯的*lambda*表达式。
- *closure*：由*lambda*生成的运行时对象，包含有由捕捉模式规定的数据块。
- *closure class*：*closure*对应的类，由编译器自动生成。

## Capture Mode

- identifier：拷贝捕获
- identifier...	：拷贝捕获包
- identifier initializer  (C++14)：初始化拷贝捕获
- &identifier：引用捕获
- &identifier...：引用捕获包
- &identifier initializer	(C++14)：初始化引用捕获
- this：引用捕获母对象
- *this	(C++17)：拷贝捕获母对象

如果默认引用捕获，则特化捕获不能再是引用；如果默认捕获是拷贝，则特化捕获必须是引用或者*this，每个捕获只能出现一次。


## Avoid Default Reference Capture

C++11为*lambda*提供了两种捕捉模式：by-value和by-reference。默认的*by-reference*有可能导致悬空引用。这是因为一旦引用对象的生命比*lambda*要短，就会导致*lambda*内部的引用悬空：

    using FilterContainer = std::vector<std::function<bool(int)>>;      // a container class for callable object.

    FilterContainer filters     // container.
    filters.emplace_back(
        [](int value){ return value % 5 == 0; }
    );

如果*lambda*中的常数使用一个捕获数，捕获模式采用默认引用捕获：

    void addDivsorFilter() {
        auto calc1 = computeVal1();
        auto calc2 = computeVal2();

        auto divisor = computeDivisor(calc1, calc2);

        filters.emplace_back(
            [&](int value){ return value % divisor == 0; }      // divisor may be dangle!
        );
    }

默认采用引用捕获的一大缺点就是上述bug难以发现，因为没有明确divisor是一个引用：

    filters.emplace_back(
        [&divisor](int value){ return value % divisor == 0; }      // divisor may be dangle!
    );  

虽然这样的代码依旧是错误的，但是至少对divisor是一个引用有足够的提示。当然如果*lambda*执行时间得当如下述代码依旧是安全的：

    template<typename C>
    void workWithContainer(const C& container) {
        auto calc1 = computeVal1();
        auto calc2 = computeVal2();
        auto divisor = computeDivisor(calc1, calc2);
        if(std::all_of(
            std::begin(container), std::end(container), [&](const typename C::value_type& value){ return value % divisor == 0; })
        ) {
            ...
        }
    }

但如果*lambda*表达式如果希望被复用，经过copy到其他环境下使用，就会发生错误。
同时C++14支持了lambda的参数推断，所以参数列表可以简化：

    if(std::all_of(
        std::begin(container), std::end(container), [&](const auto& value){ return value % divisor == 0; })
    )

## Avoid Default Value Capture 

对之前的*lambda*的容器类使用默认值捕获，就避免了引用悬空的问题：

    filters.emplace_back(
        [=](int value){ return value % divisor == 0; }      // divisor may be dangle!
    );  

但是默认值捕获，可能会导致指针悬空：

    filters.emplace_back(
        [=](int value){ return value % *pdivisor == 0; }      // pdivisor may be dangle!
    );

当然不要使用裸指针，使用智能指针可以避免这个问题，但是我们不总是能够避免使用裸指针，最突出的情况就是this指针。假设有如下情况：

    class Widget {
    public:
        ...
        void addFilter() const;

    private:
        int divisor;
    };

    void Widget::addFilter() const {
        filters.emplace_back(
            [=](int value){ return value % divisor == 0; }      // pass compile, but not safe. 
        );
    }

我们无法捕获类的数据成员，只能捕获local object，所以以下代码是会报错的：

    filters.emplace_back(
        [divisor](int value){ return value % divisor == 0; }      // cannot capture divisor. divisor is a member.
    );

由于this是一个局部变量，使用默认捕获相当于捕获了引用捕获了this，编译器会使用this->divisor替代divisor：

    filters.emplace_back(
        [this](int value){ return value % divisor == 0; }      // not safe.
    );

那么如果对象Widget已经销毁，捕获的this悬空，再次调用容器中的*lambda*将会不安全。这种错误十分危险：

    void doSomeWork() {
        auto pw = std::make_unique<Widget>();
        pw->addFilter();
        ...                 // destory Widget; now filters holds dangling pointer.
    }

该问题的解决方案就是进行拷贝：

    void Widget::addFilter() const {
        auto divisorCopy = divisor;
        filters.emplace_back(
            [divisorCopy](int value){ return value % divisorCopy == 0; }     // copy the divisor.
        );
    }

C++17提供了拷贝捕获整个对象，这里不赘述。使用默认捕获十分危险，因为很容易导致一些难以发现的错误。

## Use Init Capture and Take Notice of Static 

C++14提供了初始化捕获：

    void Widget::addFilter() const {
        filters.emplace_back(
            [divisorCopy = divisor](int value){ return value % divisorCopy == 0; }     // copy the divisor.
        );
    }

*lambda*只能捕获automatic storage duration的变量，即，static变量lambda不会捕获：

    template<typename C>
    void workWithContainer(const C& container) {
        static auto calc1 = computeVal1();
        static auto calc2 = computeVal2();
        static auto divisor = computeDivisor(calc1, calc2);
        if(std::all_of(
            std::begin(container), std::end(container), 
            [=](const auto& value){ return value % divisor == 0; })
            // capture nothing.
        ) {
            ...
        }
        divisor++;
    }

## Things to Remember

- 默认引用捕获可能导致悬空引用。
- 默认拷贝捕获可能导致悬空指针(特别是this)，而且错误的指示*lambda*完全*self-contained*。

# Item32: Use Init Capture to Move Objects Into Closures

C++11的*lambda*中无法移动一个对象进入*closure*，C++14中提供了一个新的机制，使得移动对象进入闭包变为可能，即*init capture*。*init capture*可以表达除了默认捕捉以外的所有形式的捕捉。

## *Init Capture*

*init capture*具有两个部分：一是闭包中的变量名，二是初始化闭包中变量的表达式。

    class Widget {
    public:
        ...
        bool isValidated() const;
        bool isProcessed() const;
        bool isArchived() const;
        ...
    private:
        ...
    };

    auto pw = std::make_unique<Widget>();

    auto func = [pw = std::move(pw)]{
        return pw->isValidated() && pw->isArchived();
    }

通过初始化捕捉就可以把变量移动到闭包中去。通过初始化捕捉还可以实现表达式的捕捉：

    auto func = [pw = std::make_unique<Widget>()]{
        return pw->isValidated() && pw->isArchived();
    }

原本*lambda*是无法捕捉一个表达式的，通过初始化捕捉可以捕捉一个表达式的值，又称初始化捕捉为*generalized lambda capture*。

## How to Achieve move-capture in C++11

*lambda*并不是一个全新的功能，*lambda*能做的任何事，使用*callable class*都可以实现:

    class IsValandArch {
    public:
        using DataType = std::unique_ptr<Widget>;
        explicit IsValAndArch(DataType&& ptr):
            pw(std::move(ptr)){}
        bool operator()() const {
            return pw->isValidated() && pw->isArchived();
        }
    private:
        DataType pw;
    };

    auto func = IsValAndArch(std::make_unique<Widget>());

同时还可以利用*std::bind*和*lambda*的参数：

    std::vector<double> data;

    auto func = [data = std::move(data)]{...};      // C++14 style.

    auto func = std::bind(
        [](const std::vector<double>& data){...}, 
        std::move(data))                            // C++11 style.

称第二种对象为*bind object*，注意到bind使用*const reference*做类型声明。这是因为在C++14的形式中，data是拷贝捕获，而*lambda*默认const，所以捕获参数不能被修改，在C++11的版本中，变量data模拟了被捕获变量，也应该是const的。

    auto func = [data = std::move(data)]() mutable {...};
    
    auto func = std::bind(
        [](std::vector<double>& data) mutable {...}, 
        std::move(data))

以上两个*mutable lambda*是类似实现。待捕捉对象先通过*bind*,*bind object*中包含*lambda*的一个*closure*。所以闭包的生命周期和*bind object*一样。

## Things to Remember

- 使用C++14中的*init cpature*移动捕捉对象或者捕获表达式。
- C++11可以通过*callable class*或者*std::bind*实现。

# Item33: Use *decltype* on *auto&&* Parameters to Forward Them

C++14带来了*generic lambda*：

    auto f = [](auto x){ return normalize(x); };

其闭包相当于：

    class SomeCompilerGeneratedClassName {
    public:
        template<typename T>
        auto operator()(T x) const {
            return normalize(x);
        }
    }

*lambda*起到的作用相当于将参数x转发给内部函数normalize，如果需要完美转发，结构应该如下：

    auto f = [](auto&& x){ return normalize(std::forward<?>(x)); };

问题转化为如何填入*std::forward*的模板参数。因为auto和模板的类型推断基本一致，所以x的类型就包含了传入参数的*value category*。通过Item3可知道，使用*decltype*帮助推断传入的x的*value category*。

如果x绑定一个左值，那么x就是decltype(x)就会产生左值引用类型；如果x绑定一个右值，decltype(x)就会产生一个右值引用类型，这与模板直接传入值类型不同，hui发生*reference collapsing*,再看*std::forward*。

    template<typename T>
    T&& forward(std::remove_reference_t<T>& param) {
        return static_cast<T&&>(param);
    }

如果传入右值，使用decltype(x)作为*std::forward*的模板参数，那么T = Widget&&，并发生*reference collapsing*，和直接传入值类型(T = Widget)一样：

    Widget&& forward(Widget& param) {
        return static_cast<Widget&&>(param);
    }

这样通过decltype(x)作为*std::forward*的模板参数也可以实现完美转发：

    auto f = [](auto&& x){ 
        return normalize(std::forward<decltype(x)>(x));
    };

对于传入参数包：

    auto f = [](auto&&... xs){ 
        return normalize(std::forward<decltype(xs)>(xs)...); 
    };    

## Things to Remember

- 使用auto&&(*universal reference*)和*decltype*实现*lambda*的完美转发。
