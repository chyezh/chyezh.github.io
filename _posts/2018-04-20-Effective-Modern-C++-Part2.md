---
layout: post
title:  "[Effective Modern C++] Part2"
date:   2018-04-20 20:43:34 +0800
categories: ["C++ Readings"]
tags: ["C++", "Effective Modern C++"]
---

- Item4: Konw How to View Deduced Type
- Item5: Prefer auto to Explicit Type Declartions
- Item6: Use the Explicitly Type Initializer Idiom When auto Deduce Undesired Types
- Item7: Distinguish Between () and {} when creating objects

<!-- more -->

# Item4: Konw How to View Deduced Type

程序开发过程中，总共有3个阶段，会对推断类型感兴趣。

- during edit your code.
- during compilation.
- at runtime.

## During Edit Your Code

通常IDE能够直接告诉你当前类型推断是什么，但是这往往针对类型比较简单的情况下。
IDE能够显式这类信息是，其内部的compiler正在运行。如果这个compiler对当前情
况进行足够的语法解析。那么IDE就难以给出这类信息。
简单类型推断往往是快速准确的，但是涉及到比较复杂的推断，这种方式往往难以给出
有效的信息。

## During Compilation

在编译过程中，可以通过故意制造编译错误，查看错误报告来获得推断结果。

如下：

    template<typename T>
    class TD;               //TD means "Type Display".

这是一个未定义的模板，对该模板的实例化会导致编译时错误。

比如：

    int x;
    const int* y = &x;
    TD<decltype(x)> xType;
    TD<decltype(y)> yType;

编译器报告编译错误：

>error C2079: 'xType' uses undefined class 'TD<int>'

>error C2079: 'yType' uses undefined class 'TD<const int *>'

## At Runtime

typeid与type_info支持程序识别自身的变量类型。
如下:

    std::cout<< typeid(x).name() << std::endl;
    std::cout<< typeid(y).name() << std::endl;

于VS环境下可以获得：    
>int

>int const *

但是事情并没有这么简单，考虑以下例子：

    template<typename T>                //function to show T and param.
    void f(const T& param);

    std::vector<Widget> createVec();    //factory function.

    const auto vw = createVec();        

    if(!vw.empty()){
        f(&vw[0]);
    }

然后我们通过f来获得上面类型推断的结果：

    template<typename T>
    void f(const T& param){
        std::coud << "T = " << typeid(T).name() << '\n';
        std::cout << "param = " << typeid(param).name() << '\n';
    }

在VS中得到的结果如下：
>T = class Widget const *

>param = class Widget const *

很显然T和param推断出的类型应该是不一样的，因为param是const T&。
所以type_info::name并不可靠，
这是因为标准要求type_info::name对模板函数传入均使用by—value的方式。

按照Item1，vw[0]的类型是一个const Widget&，
&vw[0]的类型便是const Widget *,
按值传递类型推断出T和ParamType均为const Widget *。

>此外还可以使用boost库


## Things to Remember

- Deduced Types可以通过IDE，编译错误，以及type_info和Boost TypeIndex library查看.
- 结果可能并不是那么有效或者精确的，所以Item1的内容是至关重要的。

# Item5: Prefer auto to Explicit Type Declartions

auto不仅能够减少拼写，同时也可以防止手写类型带来的性能和正确性问题。
有时，auto带来的类型推断契合于当前算法，但从程序的角度来说，却是错的。
所以引导auto得到想要的类型是十分关键的。

## auto Make Code More Robust

比如：

    int x；  // do not initialize x.

这行代码为初始化x，所以x可能是未定义的，也有可能是值初始化的，具体看环境。

    template<typename It>
    void dwim(It b, It e) {
        for(;b != e; ++b) {
            typename std::iterator_traits<It>::value_type
            currValue = *b;         // Dereferce b and assgin it to currValue;
            ...
        }
    }

这段代码用到了traits，仅仅是声明一个变量便要用到traits等模板编程技巧，易错而且复杂。
再看使用auto的实现版本：

    auto x；  // Wrong！ do not initialize x.

    template<typename It>
    void dwim(It b, It e) {
        for(;b != e; ++b) {
            auto currValue = *b;         // Dereferce b and assgin it to currValue;
            ...
        }
    }

auto类型推断是从initializer开始的，所以auto修饰变量必须初始化，避免了局部变量未初始化带来
的未定义行为。

auto类型推断大大减少了各类typename的声明。

同时auto还能推断出那些只有编译器才能表达的类型，比如lambda，见下：

    auto derefUPLess =                              // Compare *p1 and *p2
        [](const std::unique_ptr<Widget>& p1,
           const std::unique_ptr<Widget>& p2)
           { return *p1 < *p2; };

在C++14中，Lambda的参数也可以用auto表示（注意本质是模板）

    auto derefUPLess =                              // Compare *p1 and *p2
        [](auto& p1, auto& p2)
           { return *p1 < *p2; };

## What's a std::function Object

std::function是C++11泛化化函数指针的产物。函数指针只能指向同型函数，但是
std::function可以代表任何callable对象。

    bool(const std::unique_ptr<Widget>& p1,     // Signature for comparison function.
         const std::unique_ptr<Widget>& p2)
    
    std::function<(const std::unique_ptr<Widget>& p1,
           const std::unique_ptr<Widget>& p2)> func     // Create func.

lambda表达式是一个callable对象，所以也可以通过std::function refer to。

    
    std::function<(const std::unique_ptr<Widget>& p1,
           const std::unique_ptr<Widget>& p2)> 
        derefUPLess = [](const std::unique_ptr<Widget>& p1,
           const std::unique_ptr<Widget>& p2)
           { return *p1 < *p2; };

所以std::function是可以替代auto的。

但是可见，拼写复杂度auto远低于std::funtion。更重要的是，
auto储存lambda表达式的闭包所需的空间与闭包大小相同，
然而std::function相当于实例模板产生了一个function对象，其中一个固定空间的变量储存有闭包，
当这个空间不足以包含这个闭包，function的构造函数在heap上为闭包申请空间。
所以std::function往往比auto占用更多的内存。
同时由于函数调用等等原因，std::function总比auto要慢，还有可能造成内存耗尽的异常。
测试如下：

    clock_t beg, ed;
    beg = clock();
    for (int i = 0; i < 100000; ++i) {
        auto f = [](vector<int> v1, vector<int> v2) {return v1.size() > v2.size(); };
    }
    ed = clock();
    cout << "do not use auto:" << ed - beg << endl;
    beg = clock();
    for (int i = 0; i < 100000; ++i) {
        std::function<bool(vector<int> v1, vector<int> v2)> f =
            [](vector<int> v1, vector<int> v2) {return v1.size() > v2.size(); };
    }
    ed = clock();
    cout << "use auto:" << ed - beg << endl;

>do not use auto:1

>use auto:112

## auto Prevents "type shortcuts"

    std::vector<int> v;
    unsigned sz = v.size();

v.size()的返回值应该是std::vector<int>::size_type, 
在32位系统中，size_type和unsigned长度一致，均为32位。
但在64位系统中，则不，size_type为64位。
所以这行代码不具有移植性。
auto则没有任何问题。

    std::unordered_map<std::string, int> m;

    for(const std::pair<std::string,int>& p : m) {
        ...
    }

这部分代码也有错误，因为std::unordered_map的key部分是const的，
所以遍历类型应该是std::pair<const std::string,int>。
所以上述代码相当于通过拷贝每个pair产生一个std::pair<std::string,int>的临时对象，
然后p指向该对象。测试如下：

    std::unordered_map<std::string, int> m{ make_pair("123",1) };
    clock_t beg, ed;
    beg = clock();
    for (int i = 0; i < 100000; ++i) {
        for (const std::pair<std::string, int>& p : m);     // Copy element.
    }
    ed = clock();
    cout << "do not use auto:" << ed - beg << endl;
    beg = clock();
    for (int i = 0; i < 100000; ++i) {
        for (auto& p : m);
    }
    ed = clock();
    cout << "use auto:" << ed - beg << endl;

>do not use auto:464

>use auto:143

这不仅仅带来的是性能上的提升，更多的是程序的合理性与正确性。
对p取地址：前者带来的是对临时对象，后者是对m中元素。
临时对象将会过程结束后销毁，带来指针操作上的隐患。

显式的拼写类型经常会带来类型转换和类型不匹配，带来性能和可靠性上的损耗。

同时auto也降低了重构成本，比如一个函数的返回值为int，而后改成long，
显式声明需要修改所有位置，而auto自动更新。

很显然auto也并不完美，auto类型推断依赖于initializer，
initializer expression可能并不是我们想要的类型。
同时auto也带来了可读性上的问题。

## Things to Remember

- auto使得变量必须初始化，免疫了类型不匹配带来的转换，进一步防止转换带来的可靠性和性能的问题，
方便于重构代码，减小拼写成本。
- auto的使用也容易陷入一些陷阱，见Item2和Item6。

# Item6: Use the Explicitly Type Initializer Idiom When auto Deduce Undesired Types

## Proxy class

auto具有许多优点，但是某些情况下，auto会产生一个非期望的结果。如下：

    vector<bool> features(const Widget& w);     // A function return a vector<bool>.

bool的vector是一个特化类型，其中bool的存储是bit级别的。比如bit5指代了Widget优先级。

    Widget w;
    ...
    bool highPriority = features(w)[5];        // Get the Priority of w.
    ...
    process(w,highPriority);            // Do something with w.

这段代码没有任何问题。但如果使用auto去声明highPriority。

    auto highPriority = features(w)[5];        // Is auto giving the right type.

    process(w,highPriority);            // Undefined behaviour.

该代码的运行结果是不确定的。

因为vector<bool>是vector的一个特化类型。出于储存空间的考虑，bool只是概念性的存在于vector容器中，
operator[]返回的并不是bool reference to element，而是vector<bool>::reference。这是一个嵌套在vector<bool>中的类。

>The std::vector<bool> specialization defines std::vector<bool>::reference as a publicly-accessible nested class.

bool在vector<bool>中的存在方式是一个个的bit位。所以operator[]返回的是一个行为类似于bool的对象，该对象于bool之间存在隐式转换。
所以

    bool highPriority = features(w)[5];
    auto highPriority = features(w)[5]; 

前者触发了隐式转换，而后者并没有。后者的值取决于std::vector<bool>::reference的实现。

比如reference中包含了一个指示bit位置的指针。由于features返回了一个vector<bool>的对象，该对象是临时的调用了operator[]，最终highPriority被初始化为reference。然而待语句结束后，vector<bool>的临时对象已经消失，highPriority中的指针悬空，造成了未定义行为(比如而后发生了bool的隐式转换)。

    process(w,highPriority);            // Undefined behaviour. highPriority implicit 
                                           convert to bool with dangling pointer.

proxy class: 代理类，是为了模仿和补强某些类型存在的。比如std::vector<bool>::reference对于bool和智能指针对于raw指针。
有些代理类是暴露给用户的，比如智能指针，有些是隐藏的，比如std::vector<bool>::reference。

某些C++库中，使用一种expression templates的技术。见下表达式:

    Matrix sum = m1 + m2 + m3 + m4;

可以直接实现运算符重载operator+返回一个Matrix对象，这样一来每个operator+都会产生一个临时变量，

但是使用expression templates技术可以提高效率。operator+不再返回一个Matrix对象，而是返回一个proxy class比如Sum<Matrix，Matrix>，这是一个可以隐式转换为Matrix对象的类，同时还允许Sum从表达式初始化，means：

    Sum<Sum<Sum<Matrix，Matrix>，Matrix>，Matrix>

这样一来，减少了临时变量的拷贝和生成，这项使用显然是对用户隐藏的。

auto与invisable proxy class的相性不好，因为大多数invisible proxy class都是设计为短寿命的。

所以应该避免以下情况:

    auto tmp = expression of invisable proxy class type;

## How to Recognize the Proxy Class Type

一是通过文档，二是通过源码。熟悉类的设计，可以大幅降低这方面的错误。

- explicitly typed initializer idiom

auto并不是无法用在proxy class上的，如下：

    auto highPriority = static_cast<bool>(features(w)[5]);


feature(w)[5]返回了一个std::vector<bool>::operator[],然后应用强制转换成bool。
由于是在同一个语句中进行的，所以不会出现上述所言的悬空指针的情况，便不会出现undefined behaviour。
最后auto进行类型推断即可。

同时显式类型初始化语句也可以用于强调转换，使得某些隐式转换不被忽略。比如：

    double calcEpsilon();       // Return tolerance value.
    auto ep = static_cast<float>(calcEpsilon());


## Things to Remember

- "invisable" proxy class types 可以造成auto类型推断出“某种意义上不对”的类型。
- explicitly typed initializer idiom可以防止上述错误的发生。
# Item7: Distinguish Between () and {} when creating objects

在modern C++中，初始化从语法上分类在大致有以下三种：parentheses、equals、braces。

    int x(0);       // Initializer is an int parentheses.
    int y = 0;      // Initializer follows '='.
    int z{ 0 };     // Initializer is in braces.

还可以见到：

    int z = { 0 }；      // Initializer uses '=' and braces.

这种情况在C++中和只用braces是一样的。

## Initialization is not Assignment

首先必须意识到，initialization并不是assignment。对于built-in类型，没有什么问题；但对于定义型的类型
区别赋值和初始化在于调用的函数的不同。

    Widget w1;          // Call default constructor.
    Widget w2 = w1;     // Not an assignment; calls copy constructor.
    w1 = w2;            // Assignment, calls copy assignment(operator=).

## Uniform Initialization
在C++98时期，没有语法支持一些初始化，比如STL容器内一系列值的初始化。
C++11为了解决这类问题，引入了一个uniform initialization，至少在概念上可以运用于所有初始化场景的初始化语法。在概念上可以称为"uniform initialization",在句法上可以称为"braced initialization"。
比如：

    std::vector<int> v{ 1, 2, 3 };      // Initialize the vector with a particular set                                       // of value.

Braced-initialization还可以用于类成员非static对象的默认初始化。

    class ex {
        ...
    private:
        int x{ 0 };     // Fine, braced-initialization.
        int y = 0;      // Fine, copy-initialization.
        int z(0);       // Wrong!
    };

另一方面，不可复制对象(uncopyable)也可以用braced-initialization。

    std::atomic<int> ai1{ 0 };      // Fine.
    std::atomic<int> ai2(0);        // Fine.
    std::atomic<int> ai3 = 0;       // Wrong! Connot copy-initialization.

从上面的例子可以看出braced-initialization在所有情况的初始化下都是适用的。

值得注意一点：braced-initialization是不允许进行隐式缩窄转换(implicit narrowing convertion)。
(但是在clang中可以啊。。。)

    double x, y;
    int sum1{ x + y};       // Error! Requiring a narrowing convertion. 
    int sum2(x + y);        // Fine.
    int sum3 = x + y;       // Fine.

还有一点有价值的内容是关于默认构造函数的。

    Widget w1(10);          // Calls the constructor with one argument 10.
    Widget w2();            // Declare a function that return type is Widget, Do not                             // Calls the default constructor.
    Widget w3{};            // Calls the default constructor.

可以看出braced-initialization防止了一些语法习惯带来的意外。

## Some Superising Behaviour About Braced-initialization

由于braced-initialization和std::initialize_list之间纠缠的关系，braced-initialization会出现一些出乎意料的情况。比如Item2中所言的auto问题（在C++17中有更改）。还有就是涉及到构造函数的调用问题：

    class Widget{
    public:
        Widget(int i, bool b);
        Widget(int i, double b);
        ...
    };

    Widget w1(10, true);        // Calls first ctor.
    Widget w2{10, true};        // Calls first ctor.
    Widget w3(10, 5.0);         // Calls second ctor.
    Widget w4{10, 5.0};         // Calls secong ctor.

如果说构造函数中没有涉及到任何initializer_list的parameter，那么{}和()的对于ctor的调用行为是一致的。
但是如果涉及了initializer_list的parameter，情况就不一样了：

    class Widget{
    public:
        Widget(int i, bool b);
        Widget(int i, double b);
        Widget(std::initializer_list<long double> il);  // Initializer_list parameter.
        ...
    };

    Widget w1(10, true);        // Calls first ctor.
    Widget w2{10, true};        // Calls third ctor. 
                                // 10 and true will convert to long double.
    Widget w3(10, 5.0);         // Calls second ctor.
    Widget w4{10, 5.0};         // Calls third ctor. 
                                10 and 5.0 will convert to long double.

在这种情况下，w2和w4会优先调用新的构造函数，即使该构造函数显然没有其他函数更加匹配传入参数的形式。
甚至拷贝和移动构造函数也会被Initializer_list-parameter-ctor劫持。

？？？此处貌似和程序验证不符合，未查明原因(MSVC和clang均不符合)

    class Widget{
    public:
        Widget(int i, bool b);
        Widget(int i, double b);
        Widget(std::initializer_list<long double> il);  // Initializer_list parameter.
        operator float() const;         // Convert to float.
        ...
    };

    Widget w5(w4);              // Calls copy-ctor.
    Widget w6{ w4 };              // Calls Initializer_list-parameter-ctor.
                                // w4 converts to float through operator float()
                                // then converts to long double.
    Widget w7(std::move(w4));       // Calls move-ctor.
    Widget w8{ std::move(w4) };     // Calls Initializer_list-parameter-ctor.
                                    // w4 converts to float through operator float()
                                    // then converts to long double.


## Initializer_list and Constructor.

当一个non-aggregate class类型braced-initialization，重载方案选择遵守以下两条规则：

- 如果有initializer-list ctor，则候选函数只有initializer-list ctor，并且{}列表至少有一个元素
则整个参数列表作为initializer_list传入ctor。
- 如果没有initializer-list ctor匹配(包含转换)，其他所有构造函数都可以成为候选函数，参数列表分开传入ctor。

如果{}为空并且类有默认构造函数，则跳过前一条原则。
如果是copy-list-initialization，如果选拔出的ctor是explicit的，则ill-formed。

    class Widget{
    public:
        Widget(int i, bool b);
        Widget(int i, double b);
        Widget(std::initializer_list<bool> il);  // Initializer_list parameter.
        ...
    };

    Widget w{ 1, 5.0 };         // error! Requires narrowing conversion.

如上即使这里拥有完美配对的Widget(int i, double b)，但是该构造函数不在候选之中，
反而是使用了initializer_list ctor，同时缩窄转换带来错误。

如果初始化列表中的元素与initializer_list的元素不存在转换，则符合原则二：

    class Widget{
    public:
        Widget(int i, bool b);
        Widget(int i, double b);
        Widget(std::initializer_list<string> il);  // Initializer_list parameter.
        ...
    };

    Widget w{ 1, 5.0 };         // Fine. Use non-initializer_list constructor.

由于1和5.0无法隐式转换为string，所以候选才加入了前两个构造函数。

接着如果是空的初始化列表并且有默认构造函数，则跳过原则一：

    class Widget{
    public:
        Widget()；           // Default constructor.
        Widget(std::initializer_list<string> il);  // Initializer_list parameter.
        ...
    };

    Widget w1;          // Call the default ctor.
    Widget w2{};        // Call the default ctor.
    Widget w3();        // Define a function that return Widget.
    Widget w4({});      // Call the initializer_list one.
    Widget w5{{}};      // ditto.

在设计类的构造函数时应该尽量避免和vector类似的情况：

    std::vector<int> v1{ 10, 20 };  // Initialize v1 with two elements in the list.

    std::vector<int> v2(10, 20);    // Initialize v2 with ten elements (20).
  
设计类时应当尽量使用户不论使用{}还是()都获得一样的结果。比如，原本类中不含有initializer_list ctor，但是后来添加了，如果像vector一样的设计，会导致原本用户的代码调用不同的构造函数。这与普通的重载不一样，因为initializer_list ctor总是trump其他构造函数，导致更大的问题。


## Choose Braces or Parentheses

作为类用户，选择Braces和Parentheses有两种方法，以其中一个为主，不到万不得已时使用另一个，并且持之以恒。
两种方法各有所长。

作为模板类设计，这个选择便十分关键了，见下：

    template<typename T, typename... Ts>
    void doSome(Ts&&... params) {
        create local object from params.
    }

可以有以下两种实现：

    T localobj(std::forward(params)...);        // Using paren.
    T localobj{ std::forward(params)... };      // Using brace.

对于vector就会产生不同的效果：

    doSome<std::vector<int>>(10, 20);

前者生成10个元素，初始化为20；后者生成2个元素，初始化为10、20。
这个问题，在标准库设计中也存在，std::make_shared和std::make_unique。

## Things to Remember

- braced-initialization可以应用到更广泛的情景，避免缩窄转换(clang好像并不会)，避免某个混淆语句。
- 注意构造函数重载中，initializer_list constructor的特殊。
- std::vector< numeric type>构造在选用{}和()的不同。
- 关注模板中使用{}和()进行实现的不同。
