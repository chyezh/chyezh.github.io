# Item1: Understand Template Type Dedution

函数模板的形式如下:

```c++
    tmeplate<typename T>
    void foo(ParamType param);
```

函数调用的形式如下:

```c++
    foo(expr);  //call foo with expresstion;
```

在编译期间，编译器会推断T和ParamType。ParParamType通常会包含一些修饰符，比如const、&等，如下：

    template<typename>
    void foo(const T& param);

T的类型推断是由expr和ParamType共同决定的。如下根据ParamTpye分成三种情况：

- ParamType是一个指针和引用（非万能引用）。
- ParamType是一个万能引用。
- ParamType不是引用和指针。

## case1: ParamType is a Reference or Pointer,but not a Universal Reference

在这种情况下,类型推断步骤如下：
- 如果expr是一个引用，则忽略引用部分。
- 依赖ParamType模式匹配expr的类型，得到T。

比如：

    template<typename T>
    void foo(T& param);
    
    int x = 27;
    const int cx = x;
    const int& crx = x;

    foo(x);     //T is int, ParamType is int&.
    foo(cx);    //T is const int, ParamType is const int&.
    foo(crx);   //T is const int, ParamType is const int&. reference-ness is ignored.

这种情况下，为了保证const变量的安全，const将成为T的一部分。如果param包含const修饰，如下：

    template<typename T>
    void foo(const T& param);

    foo(x);     //T is int, ParamType is const int&.
    foo(cx);    //T is int, ParamType is const int&.
    foo(crx);   //T is int, ParamType is const int&. reference-ness is ignored.

param为指针的表现与引用类似，如下：

    template<typename T>
    void foo(T* param);
    const int* cpx = &x;

    foo(&x);    //T is int, ParamType is int*.
    foo(&cx);   //T is const int, ParamType is const int*.
    foo(cpx);   //T is const int, ParamType is const int*.

## case2: ParamType is a Universal Reference

在这种情况下，推断原则如下：

- 如果expr是一个左值，T和ParamType都被推断为左值引用。
（1.只有这种情况，T才有可能被推断为引用；2.ParamType被推断为左值引用而非右值引用。）
- 如果expr是一个右值，直接适用通用情况。

比如：

    template<typename T>
    void foo(T&& param);    //ParamType is a Universal Reference.

    foo(x);     //T is int&, ParamType is int&.
    foo(cx);    //T is const int&, ParamType is const int&.
    foo(crx);   //T is const int&, ParamType is const int&.
    foo(27);    //27 is int and rvalue, T is int, ParamType is int&&.

当ParamType是万能引用时，expr左值和右值表现并不一样。

## case3： ParParamType is Neither a Pointer nor a Reference

按值传递,这意味着param将是传入的一个copy。原则如下：

- 如果expr是一个引用，则忽略引用。
- 如果忽略引用后，有const和volatile修饰，则亦忽略。

如下：
    
    template<typename T>
    void foo(T param);      //param is passing by value.

    foo(x);     //T is a int, ParamType is a int.    
    foo(cx);    //T is a int, ParamType is a int. const-ness is ignored.
    foo(crx);   //T is a int, ParamType is a int. reference-ness and const-ness is ignored.

const被忽略的原因在于：param是一个copy，被copy对象不可变不代表其复制不可变。
这是按值传递和引用或指针传递不同的地方。
考虑以下情况：

    const char* const ptr = ... //ptr is a const pointer pointing to const.
    foo(ptr);   //T is a const char*, ParamType is a const char*. 
                //top-level is ignored. but low-level should not be ignored.

由于按值传递的对象是指针。所以修饰指针本身的const（top-level）可以被忽略，
但是修饰指针所指向的obj的const不可以被忽略。

## Array Argments

值得注意数组和指针是不一样的，即时它们经常可以互相转换。
数组经常退化为指针（指向第一元素）：

    const char name[] = "abcd";     //name's type is const char[13].
    const char *ptr = name;         //array decays to pointer.

考虑数组作为expr的情况：

    template<typename T>
    void foo(T param);      //by-value.

    foo(name);  //T is const char*, param is const char*.
                //array decays to pointer.

    template<typename T>
    void foo(T& param);      //by-reference.

    foo(name);  //T is const char[5], ParamType is const char(&)[5].

    template<typename T>
    void foo(T* param);      //by-pointer.

    foo(name);  //T is const char, ParamType is const char*.

通过该特性，就可以实现以下模板：

    //aquire the size of an array at compile time.
    template<typename T, size_t N>
    constexpr size_t ArraySize(T (&)[N]) noexcept {
        return N;
    }

## Function Argments

函数也可以退还成函数指针。模板类型推断时的函数与数组表现类似。
如下：

    void func(int,double);   //type is void(int,double).

    template<typename T>
    void foo(T param);       //by-value.

    foo(func);  //T is void(*)(int,double), ParamType is void(*)(int,double).

    template<typename T>
    void foo(T param);      //by-reference.

    foo(func);  //T is void(int,double), ParamType is void(&)(int,double).

    template<typename T>
    void foo(T* param);      //by-pointer.

    foo(func);  //T is void(int,double),ParamType is void(*)(int,double).


## Things to Remember

- 在模板类型推断中，argument的reference-ness被忽略。
- 对于universal reference parameter，lvalue arguments是具有特殊机制的。
- 对于by-value parameter，const和volatile是被忽略的。
- 对于array和funciton的arguments，除了reference parameter以外均退化为指针。

# Item2: Understand Auto Type Dedution

Auto的类型推断与item1的模板类型推断基本一致。在auto推断中，auto相当于
模板中的T，整个类型修饰符相当于PramaType，初始化的值相当于expr。(有一点不同)

如下：

    auto x = 27;

    template<typename T>
    void funcx(T param);
    
    funcx(27);      //T is int, so auto is int. ParamType is int.

    const auto cx = x;
    
    template<typename T>
    void funccx(const T param);

    funccx(x);      //T is int, so auto is int. ParamType is const int.
    

    const auto& crx = x;

    template<typename T>
    void funccrx(const T& param);

    funccrx(x);     //T is int, so auto is int. ParamType is const int&.

## Type Specifiers have effect on Type Deduction 

正如模板类型推断中的以PramaType分类讨论，auto类型推断可以将type specifier分成3类讨论，即：

- 类型修饰符是引用或者指针（不包含万能引用）
- 类型修饰符是万能引用
- 类型修饰符不是指针和引用

比如：

    auto x = 27；            //27 is int, x is int.
    const auto cx = x;     //x is int, cx is const int. 
    const auto& crx = x;    //x is int, crx is const int&.

    auto&& urx = x;         //x is int(lvalue), urx is int&.
    auto&& urx = cx;        //cx is const int(lvalue), urx is const int&.
    auto&& urx = 27;        //27 is rvalue, urx is int&&.


## Initializer is an Array or a function

指针和数组作为初始化auto类型推断和模板类型推断一致。

    const char name[] = "123456";   //name is const char[7].

    auto arr1 = name;       //by-value, arr1 is const char*.
    auto& arr2 = name;      //by-lreference, arr2 is const char(&)[7].

    void func(int,double);          //func is void(int,double).

    auto func1 = func;      //by-value, func1 is void(*)(int,double).
    auto& func2 = func;     //by-lreference, func2 is void(&)(int,double).

## Braced-Initializer and Parenthesis-Initializer

barced-initializer_list是auto和template类型推断的唯一不同之处。template是不能直接推断出initializer-list的。
考虑以下定义：

    auto x1 = 27;           //type is int, value is 27.
    auto x2(27);            //ditto.
    auto x3 = { 27, 42 };   //type is initializer_list<int>, value is {27}.
    auto x4{ 27 };          //type is int, value is 27.
    auto x5{ 27, 42 };      //error, need =.
    auto x6{ 1, 2, 3.0 };   //error, cannot deduce T for initializer_list.

注意直接初始化和赋值初始化对于{}initializer的区别。

    template<typename T>
    void foo(T param);
    
    foo({ 1, 2, 3 });       //error, connot deduce type for T.

想要实现对T的template推断，可以如下声明：

    template<typename T>
    void foo(std::initializer_list<T> inilist);

    f({1, 2, 3});           //valid, T is int, ParamType is initializer_list<int>.

同时值得注意的是，C++14允许函数返回值与lambda表达式参数通过使用auto进行类型推断，但是这种情景下的类型推断是模板类型推断，
而不是auto类型推断，比如：

    auto createList() {
        return { 1, 2, 3 };     //error: cannot deduce return type for { 1, 2, 3 }.
    }

    std::vector<int> v;
    auto resetV = [&v](const auto& newvalue) {
        v = newvalue;           //C++ 14;
    }
    resetV({ 1, 2, 3 });        //error! cannot deduce parameter type for { 1, 2, 3 }.

## Things to Remember

- auto类型推断和template类型推断几乎一致，只是auto可以将braced-initializer推断为initializer_list，而template不接受这种推断。
- auto在函数返回值和lambda表达式参数使用时，代表的是template推断，而不是auto。

# Item3: Understand *decltype*

decltype获取一个表达式，返回一个表达式的类型。
大多数情况其结果如常规想象，但是偶尔也会出一些令人意想不到的结果。

## Typical Case

这些情况下，decltype给出表达式确实的类型。

    const int i =0;             //decltype(i) is const int.
    bool f(const widget& w);    //decltype(f) is bool(const widget&).
    sturct Point {              //decltype(Point::x) is int.
        int x, y;
    };
    widget w;                   //decltype(w) is widget.

    if(f(w))...                 //decltype(f(w)) is bool.

    template<typename T>
    class vector {
        public:
        T& operator[](std::size_t index);
    };

    vector<int> v;              //decltype(v) is vector<int>.

    if(v[0] == 0)...            //decltype(v[0]) is int&.

在这些情况下，decltype乖乖的推导出expr的类型，不进行任何添油加醋。

## Trailing Return Type with Decltype

在C++11中，函数返回值可以采用trailing的方式，而且可以使用decltype.

比如：

    template<typename Container, typename Index>
    auto access(Container& c, Index i) ->decltype(c[i]) {   
        ...
        return c[i];        //access c[i].
    }

    vetcor<int> veci{ 1, 2, 3, 4 };
    access(veci,2) = 5;     //change the veci[2].

由于我们不知道容器Container以及索引Index的准确类型及其内容物类型，
我们直接拼写出返回的内容物的类型，需要通过类型推断。
一种方法是通过Container的内置类型成员(typename)，另一种就是利用decltype.
由于c与i的声明位置，所以需要使用trailing的方式。

这种情况下，auto是仅有占位功能，真正进行推断的是decltye，
decltype分辨出c[i]的类型是T&，所以可以通过access读写c[i]。

## Auto Return Type Deduction

再看item2中提到的, 自C++14起，
允许为多语句lambda以及函数进行自动返回值推断。如下：

    template<typename Container, typename Index>
    auto access(Container& c, Index i) {   
        ...
        return c[i];        //omit the referenceness.
    }

    vetcor<int> veci{ 1, 2, 3, 4 };
    access(veci,2) = 5;     //error, expression is a rvalue.

这种情况下，执行的template的类型推断，所以access返回的是T类型变量，是一个右值。

值得注意，所以为了实现access的原本功能，decltype trailing return type就不能忽略。

## Use decltype(auto)

同时在C++14中，提供了更加优雅的使用decltype的方式，
使得decltype类型推断被使用，如下：

    template<typename Container, typename Index>
    decltype(auto) access(Container& c, Index i) {   
        ...
        return c[i];        //use the decltype rules.
    }

    vetcor<int> veci{ 1, 2, 3, 4 };
    access(veci,2) = 5;     //change the veci[2].

decltype(auto)不仅能用在函数返回推断上，也可以用在通常声明上，使得类型推断自行使用
decltype的规则。

    widget w;
    const widget& crw = w;
    auto w2 = crw;              //auto type deduction, w2 is widget.
    decltype(auto) w3 = crw     //decltype type deduction, w3 is const widget&.

## Decltype's Behaviour

观察如下代码：

    int x = 0;      //decltype(x) is int.
    int y = 0;      //decltype((x)) is int&.

    decltype(true?x:0) i;   //true?x:0 is rvalue-expression, i is int. 
    decltype(true?x:y) i;   //true?x:y is lvalue-expression, i is int&.


产生了意想不到的结果。
因为decltype对name和expression的效果是不一样的。
当decltype作用于name时，产生的类型是T。
当decltype作用于lvalue-expression时，产生的是T&。

Trailing Return Type不容易出现类型的误写，
但decltype(auto)容易出现，如下：

    decltype(auto) foo() {
        int x = 0;
        return x;           //decltype（x） is int, so foo returns int.
    }

    decltype(auto) foo() {
        int x = 0;
        return (x);         //decltype((x)) is int&, so foo returns int&.
    }



## Things to Remember

- decltype大多数情况产生类型与表达式相应的类型，不会有任何修正。
- 对于lvalue-expression，decltype会产生T&.
- C++14支持decltype(auto)，表达使用decltype原则进行类型推断。

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

# Item8: Prefer *nullptr* to 0 and *NULL*

在C++中，字面值0是一个int，在contxet中，0有可能被解释为null pointer。
但是这是后置位的情形，0依旧是一个int而不是null pointer。

NLL这个宏依赖于实现，有可能是int(0),也有可能是long(0)，NULL对于指针也具有一样的问题。

在C++98中，对于指针和整数的参数重载具有存在陷阱：

    void f(int);
    void f(bool);
    void f(void*);

    f(0);       // Call f(int).

    f(NULL);    // Depend on implementation, might not complie. but  
                // never calls f(void*).

因为NULL的实现可以是0，也可以是0L，而0L转换给void*，int，bool是平等的，导致歧义，造成报错。

另一方面，nullptr的好处在于其不具有整数值，它能够转换为指向任意类型的null pointer。使用nullptr，就能避免上述的重载问题。

    f(nullptr)  // Call f(void*).

同时使用nullptr，也可以增强代码可读性。

    auto result = find(/*arg*/);

    if(result != 0)...

    if(result != nullptr)...

很显然下面的代码表面了result是一个指针。

当模板进入代码时，nullptr的作用更加明显：

    int f1(std::shared_ptr<Widget> spw);
    double f2(std::unique_ptr<Widget> upw);
    bool f3(Widget* pw);

    std::mutex f1m, f2m, f3m;

    using MuxGuard = std::lock_guard<std::mutex>;
    ...
    {
        MuxGuard g(f1m);
        auto result = f1(0);
    }
    ...
    {
        MuxGuard g(f2m);
        auto result = f2(NULL);
    }
    ...
    {
        MuxGuard g(f3m);
        auto result = f3(nullptr);
    }
以上代码具有高度的重复性，可以将其模板化：

    template<typename FuncType, typename MuxType, typename PtrType>
    decltype(auto) lockAndCall(FuncType func,
                                MuxType mutex,
                                PtrType ptr) {
        using MuxGuard = std::lock_guard<MuxType>;
        MuxGuard g(mutex);
        return func(ptr);
    }
    
    auto result1 = lockAndCall(f1, f1m, 0);         // error!
    auto result1 = lockAndCall(f2, f2m, NULL);      // error!
    auto result1 = lockAndCall(f3, f3m, nullptr);   // fine!
    
在第一个模板函数中，PrtType被推断为int，这就导致在模板内部func要接收一个int，
对于f1来说，相当于用一个int去初始化shared_ptr\<Widget>，这是错误的(因为0可以指代指针，但是int是不可以的)，对于第二个也是类似的情况。

而nullptr是没有这方面的问题的。传入nullptr时，PrtType被推断为std::nullptr_t。而nullptr_t是可以转化为Widget*和shared_ptr\<Widget>的。

## Things to Remember

- 使用nullptr代替NULL和0。
- 避免重载整数和指针类型。

# Item9: Prefer Alias Declarations to typedefs

别名是C++减少长类型名拼写的有效手段。  
在C++98中：


    typedef std::unique_str<std::unordered_map<std::string, std::string>> UPtrMapSS;

而在C++11中：

    using UPtrMapSS = std::unique_str<std::unordered_map<std::string, std::string>>；

以上两例，无法看出typedef和using的区别与差距，也看不出using的优势。  
但是当我们想为某些类型，比如函数指针、数组等起别名时，语义的表达性就不一样了：

    typedef void (*fp)(int, const std::string&);     // typedef.

    using fp = void(*)(int, const std::string&);     // alias declaraiton.

显然using对于fp的表达更加直观，但这仍然不是alias declaration优于typedef的绝对理由。

## About Template Alias

using可以用于template(alias templates)，而typedef不可以。  
传统的将别名用于模板的做法是为typedef加上一层struct的封装：

    template<typename T>
    typedef std::list<T, MyAlloc<T>> MyAllocList;   // error! canot typedef.

    template<typename T>
    struct MyAllocList {
        typedef std::list<T, MyAlloc<T>> type;      // Add struct outside. 
    }

    MyAllocList<Widget>::type lw;           // Client code.

    template<typename T>
    using MyAllocList = std::list<T, MyAlloc<T>> ;   // Fine.

    MyAllocList<Widget> lw;                 // Client code.

using与typedef在用户代码上的差距就表现出来了。除此之外，还有一个using优于typedef的方面
，当我们在template中使用'typedef型'的模板别名时，::type是dependent type，编译器无法知道这是不是一个类型，就必须加typename来指示该名为类型成员：

    template<typename T>
    class Widget {
    private:
        typename MyAllocList<T>::type list;
    }


而using不需要外部封装，就没有这方面的问题：

    template<typename T>
    class Widget {
    private:
        MyAllocList<T> list;
    }

MyAllocList是一个类型别名，所以MyAllocList<T>必须是一个类型，而不是变量，所以MyAllocList<T>是non-dependent type，所以typename是可以忽略的。  
再看以下代码：

    class Wine { ... };

    template<>                      // A specialization of MyAllocList.
    class MyAllocList<Wine> {       // when T is wine.
    private:
        enum class WineType
        { White, Red, Rose };       // Here type is data member, not type member.
        WineType type;
        ...
    }

可以看到这里type是一个数据成员，而不是类型成员。如果Widget对T = Wine生成一个实例。那么::type
在Widget模板中就是一个数据名。这就是为何需要typename的原因。

C++11中提供了一系列类型处理的功能模板(base on TMP),<type_traits>

    std::remove_const<T>::type              // Yields T from const T.
    std::remove_reference<T>::type          // Yields T from T&.
    std::add_lvalue_reference<T>::type      // Yields T& from T.

这些模板的实现都是依赖于内嵌typedef的。

C++14给了更好的实现，依赖于using:

    std::remove_const_t<T>              // Yields T from const T.
    std::remove_reference_t<T>          // Yields T from T&.
    std::add_lvalue_reference_t<T>      // Yields T& from T.

    template<typename T>
    using remove_const_t = typename remove_const<T>::type;

    template<typename T>
    using remove_reference_t = typename remove_reference<T>::type;
    
    template<typename T>
    using add_lvalue_reference_t = typename add_lvalue_reference<T>::type;
    
## Things to Remember

- typedefs不支持模板化，但是using支持。
- alais template避免了嵌套和::type的后缀，注意typename对于dependent type的作用。
- C++14提供了traits的更好的实现。

# Item10: Prefer Scoped Enums to Unscoped Enums

## Scoped and Avoid Converting
在一般的原则下，一个block{}代表了一个scope。但是enum是例外的，enum中的变量的作用域在enum所在的域中。

    enum Color { black, white, red };       // black, white..has the same scope as Color.

    auto white = false;                     // error! white already declared in Color.

C++11中，提供了更加符合常理的enum：scoped-enum。

    enum class Color{ black, white, red };  // black, white...are scoped in Color.

    auto white = false;                     // Fine. 

    Color cc = white;                       // error, white is bool.

    Color cc = Color::white;                // Fine.

    auto cc = Color::white;                 // cc is Color.

通过scoped-enum来防止枚举变量的泄露。  

同时enum具有和整型之间的隐式转换，而enum class则没有。

    enum Color { black, white, red };
    std::vector<std::size_t> primeFactors(std::size_t x);

    Color cc =red;

    if(cc < 14.5) {
        auto factors = primeFactors(cc);        // Implicitly convert happen.
    }

    enum class Color { black, white, red };

    Color cc = Color::red;

    if(cc < 14.5) {                             // error! cannot convert.
        auto factors = primeFactors(cc);        // error! cannot convert.
    }

    if(static_cast<double>(cc) < 14.5) {        // Fine.
        auto factors = primeFactors(static_cast<std::size_t>(cc));  // Fine.    
    }

## Forward enum Declaration.

注意enum是一个编译期确定的量，scoped-enum的另一个优越性就是可以进行前置声明。

    
    enum Color;         // error！ cannot forward-declaration.

    enum class Color;   // Fine.

这是不完全的，因为enum在C++11中也可以通过一些额外动作使得其可以进行前置声明。前置声明的好处在于可以减少编译。比如存在以下头文件：

    // file locstring.h
    #include <string>
    enum localized_string_id
    {
    /* very long list of ids */
    };

    std::istream& operator>>(std::istream& is, localized_string_id& id);
    std::string get_localized_string(localized_string_id id);

可以看到下方的两个函数是依赖于enum的。如果localized_string_id中的变量是频繁改变的。那所有包含了该头文件的组件都要重新编译，这要付出很高的成本。

如果使用前置声明，就可以直接在头文件中保留前置声明部分，并且为每一个编译单元实现各自的enum（注意enum是编译期确定的），一些不依赖于新加入的enumerator的单元就可以不用再次进行编译了。

## Underlying-Type

之所以在C++98中没有前置声明，是因为enum实现类似于一个打包宏定义。

    enum Color { black, white, red};

    #define black 0;
    #define white 1;
    #define red 2;

所以enum其实是有一个底层实现的intergal type的，可以是int，char...具体依赖于编译器自己实现的。

unscoped-enum是不确定的，但是scoped-enum是确定的，默认为int，但是在c++11后，enum也可以进行强类型的声明。

    enum class status;      // Underlying type is int.

    enum status;            // Unknown.

    enum class status :uint8_t;     // Underlying type is uint8_t.

    enum status :uint8_t;           // Underlying type is uint8_t.

## Where Unscoped-enum is Better Than Scoped-enum

虽然scoped-enum具有许多优点：防止隐式转换，防止命名空间的污染，具有前置声明之类的。但是有一个地方enum比scoped-enum更加适用-tuple：

    using UserInfo = std::tuple<string,     // Name. 
                                string,     // Email.
                                size_t>     // Reputation.

    UserInfo uInfo;     // Object of UserInfo.
    auto val = std::get<1>(uInfo);      // Get the field 1-email.

就和注释中所言，字段1代表了uInfo的email，但是1代表email总是不直观的。

    enum UserInfoField { uiName, uiEmail, uiReputation };
    auto val = std::get<uiEmail>(uInfo);        // Get the field uiemail.

这就利用了enum的隐式转换，使用scoped-enum显然就要费事的多。

或许可以通过外加的包装实现更简单的语法，但是注意field-1是一个template parameter，这意味着值需要在编译期间确定，enum具有这样的能力（或者宏），所以这层包装就需要用到constexpr function:

    template<typename E>
    constexpr auto
    toUType(E enumerator) noexcept {
        return static_cast<typename 
            std::underlyting_type<E>::type>(enumerator);
    }

    enum class UserInfoField { uiName, uiEmail, uiReputation };

    auto val = std::get<toUType(uiEmail)>(uInfo);

即使加上封装，还是不如enum来的简单，但是这又避免了污染命名空间。

## Things to Remember

- scoped-enum不会污染命名空间，而且只能通过cast转换为其他类型。
- scoped-enum和unscoped-enum都具有指定underlying-type的方法，不同的是，scoped-enum具有默认的int，而unscoped-enum没有。
- scoped-enum总是可以前置声明，而unscoped只有在指定underlying-type时才可以，注意enum工作在编译期。
