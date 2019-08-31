# Item31: Avoid Default Capture Mode

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
