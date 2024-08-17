---
layout: post
title:  "[Effective Modern C++] Part3"
date:   2018-04-29 17:42:06 +0800
categories: ["C++ Readings"]
tags: ["C++", "Effective Modern C++"]
---

- Item8: Prefer *nullptr* to 0 and *NULL*
- Item9: Prefer Alias Declarations to typedefs
- Item10: Prefer Scoped Enums to Unscoped Enums

<!-- more -->

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
