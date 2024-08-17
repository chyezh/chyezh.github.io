---
layout: post
title:  "[Effective Modern C++] Part1"
date:   2018-04-12 18:12:04 +0800
categories: ["C++ Readings"]
tags: ["C++", "Effective Modern C++"]
---

- Item1: Understand Template Type Dedution
- Item2: Understand Auto Type Dedution
- Item3: Understand *decltype*

<!-- more -->

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
