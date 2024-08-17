---
layout: post
title:  "<C++ Concurrency In Action> Shared Data Protection"
date:   2018-12-01 13:52:23 +0800
categories: C++-Readings
---

# Share Date Protection

## Problem with sharing data between threads and race condition

当多个线程的共享数据是只读*readonly*的情况，是不会产生额外要分析的问题的。但当多个线程其中包含又对共享数据写权限的线程时候，就会产生额外的问题。

在编程过程中，往往有许多不变式*invariants*帮助我们理解代码的含义，但是这种不变式往往会在数值更新时被破坏，越是复杂的数据类型，越是在更新时进行多的操作的类型，这个破坏不变式的过程越容易在多线程编程中引发问题。

比如一个双向链表*doubly linked list*包含不定式：

- 节点A的后向指针指向节点B，则有节点B的前向指针指向节点A。

在某些链表操作就会短时间内破坏这个不定式，如删除节点N：

1. 找到节点N
2. 将节点N的前节点的后向指针指向节点N的后节点
3. 将节点N的后节点的前向指针指向节点N的前节点
4. 删除节点N

在步骤2-3之间，双向链表的一个重要的不定式被破坏。若在其他线程的逻辑中有用到该不定式的代码，该代码将会产生未定义后果。

这是在并发编程中常见的BUG原因：*race condition*，也称*data race*。

### Race condition and avoiding problematic race conditions

*race condition*通常指代那些会导致BUG的竞争条件，其带来的BUG通常是未定义行为。

*race condition*主要是各线程对共享数据的修改导致的。其带来的BUG对时间十分敏感，在DEBUG的模式下很容易被忽略。当修改操作被连续的计算机指令执行时，*race condition*的概率较低；随着计算机负荷的上升，有问题的竞争操作的发生概率也随之提高。

解决*race condition*的主要办法有以下几种：

1. 使用保护机制封装共享数据，保证只有一个线程能改变数据并且其他线程只能在该操作尚未发生或者已经结束的情况下访问数据，即*mutex*。
2. 改造共享数据结构的设计，使得该结构进行的修改操作不会破坏数据结构的不变性，即*lock-free programing*。
3. *software transactional memory*，STM

## Protecting shared data with mutexes

解决*race condition*的一种通用方法就是将访问共享数据的代码块设置为互斥，即*mutually exclusive*。这样一来当有线程访问共享数据，其他线程就必须等待访问结束后再进行访问。

这一过程使用*mutex*来实现，在访问共享数据前，*lock*一个和该数据关联的*mutex*；在访问结束后，*unlock*这个*mutex*。

标准线程库保证了当一个线程锁定一个*mutex*，其他试图锁定该*mutex*的线程都要等候至该*mutex*被解锁。

互斥器*mutex*听起来十分简单，但非银弹：
1. 构造好代码结构保护目标数据
2. 防止*race condition*在接口之间传递
3. *mutex*自身存在*deadlock*等问题

### Using mutexes in C++

C++标准库通过*std::mutex*实现*mutex*：
- *lock()*锁定*mutex*，当不能锁定时阻塞。
- *unlock()*解锁*mutex*。

并且有*RAII*风格的*std::lock_guard*来替代手动的*unlock*。

```c++
    class listForThread {
    public:
        void pushToList(int _val) {
            std::lock_guard<std::mutex> lguark(m_mutexForList);
            m_list.push_back(_val);
        }
        bool findInList(int _val) {
            std::lock_guard<std::mutex> lguark(m_mutexForList);
            return std::find(std::begin(m_list), std::end(m_list), _val) != m_list.end();
        }
    private:
        std::list<int> m_list;
        std::mutex m_mutexForList;
    };
```

通过上述代码，实现了*list*的*find*和*push*两种操作的互斥。

但是需要非常注意的是，即使所有操作都使用*std::mutex*实现操作互斥，但是函数接口仍然有可能会将*race condition*泄露。比如：

```c++
    class listForThread {
    public:
        typename std::list<int>::iterator beginOfList() {
            std::lock_guard<std::mutex> lguark(m_mutexForList);
            ...
            return m_list.begin();
        }
    }
```

该代码将内部数据的一个迭代器向外传递。外部代码则可以绕过类接口直接访问内部链表，导致*race condition*。因此函数接口涉及指针、引用和迭代器时，需要格外注意产生*race condition*的泄露。良好的互斥设计应该包含整个数据访问的过程，而不是仅仅是简单的函数开头与结尾。

### Structuring code for protecting shared data

使用*mutex*保护数据的代码设计十分复杂，不仅仅是使用*std::lock_guard*和注意函数是否有指针引用接口就完事的。某些隐藏性很强的接口也有可能会使得*race condition*泄露：

```c++
    class data {
    public:
        void some_func();
    prvate:
        int i;
    };

    class data_wraper {
    public:
        template<typename Func>
        void someFunc(Func f) {
            std::lock_guard<std::mutex> g(m_mutex);
            f(m_data);
        }
    private:
        data m_data;
        std::mutex m_mutex;
    };

    data* p;
    void maliciousFunc(data& _d) {
        p = &_d;
    }

    void foo() {
        data_wraper d;
        d.someFunc(maliciousFunc);
        p->somefunc();
    }
```

这种*race condition*泄露的问题的根本原因是没有做到：将所有涉及访问目标数据块的操作标示为互斥。因此使用*std::mutex*遵守以下原则以避免*race condition*外泄：

- 不要将被保护数据的指针和引用传出*mutex lock*的范围，不论是显式的还是隐式，比如传参、返回、传递给未知实现的函数。

### Spotting race conditions inherent in interfaces

即使严格遵守了上述原则，函数接口依旧有可能导致潜在的*race condition*，这是由于函数本身的操作决定的，不仅仅发生在互斥设计中，也会发生在*lock-free programing*中。

因为并发实现中，可能会使某些函数失去原本的意义，该函数所得到的结果是不可信的，比如*stack*：

```c++
    template<typename T,typename Container=std::deque<T> >
    class stack
    {
    public:
        explicit stack(const Container&);
        explicit stack(Container&& = Container());
        template <class Alloc> explicit stack(const Alloc&);
        template <class Alloc> stack(const Container&, const Alloc&);
        template <class Alloc> stack(Container&&, const Alloc&);
        template <class Alloc> stack(stack&&, const Alloc&);
        bool empty() const;
        size_t size() const;
        T& top();
        T const& top() const;
        void push(T const&);
        void push(T&&);
        void pop();
        void swap(stack&&);
    };
```

栈的操作只有5种，在并发编程中，函数*empty*和*size*将不再可信。因为其一旦完成了操作，则其他线程就可以访问改变*stack*，在该线程所得的信息将失去意义。假设*stack*的5种操作都加上了互斥保护，存在以下接口的组合使用：

```c++
    stack<int> s;
    if(!s.empty()) {
        const int i = s.top();  // 1
        s.pop();                // 2
        do_something(i);
    }
```

其他线程的操作可以插入上述每一行代码之间，导致未定义问题，甚至使程序崩溃。比如在代码1和2之间可以插入*pop*使得源代码的遍历功能失效。

想要将*pop*和*top*纳入互斥保护，就需要有新的实现接口来使得两个操作纳入一个互斥保护，但这又引入了新的异常安全的问题：

比如存在数据类型*stack\<vector\<int\>\>*。接口*topandpop*为了互斥保护，同时实现了*pop*和*top*的功能。但是*topandpop*为了返回被*pop*的对象，必须调用*std::vector*的构造函数，这一过程可能会抛出异常，但是抛出异常时，*stack*已经被改变。分离的*pop*和*top*操作不会有这样的问题，因为当*top*操作抛出异常时，*pop*操作还没有执行。为了解决*race condition*和*exception safety*纠缠的问题，需要设计好函数接口，有如下几种方法：

#### Pass a reference

为了使得构造函数不在接口内部被调用，可以使用引用参数作为返回接口：

    std::vector<int> result;
    some_stack.pop(result);

这样就要求用户代码必须先自行进行目标对象的构造，并且要求返回类型是可赋值类型。

#### Require a nothrow copy constructor or move constructor

许多类型的复制构造函数是*nothrow*的，即使复制构造函数会抛出异常，还有许多类型的移动构造函数也是*nothrow*的。可以使用编译期检查*std::is_nothrow_copy_constructible*和*std::is_nothrow_move_constructible*。

#### Return a pointer to the popped item

可以使用指针返回替代值返回，这样可以使得构造函数在改变*stack*之前被调用。但是这就涉及到了内存管理，而且对于简单类型而言，这样的开销甚至超出了简单的值返回。可以使用智能指针作为返回类型。

如下实现了引用参数返回和智能指针返回，解决了*race condition*和*exception safety*纠缠的问题：

```c++
    struct empty_stack: std::exception{
        const char* what() const noexcept;
    };

    template<typename T>
    class threadsafe_stack {
    public:
        threadsafe_stack() {}
        threadsafe_stack(const threadsafe_stack& _stack) {
            std::lock_guard<std::mutex> g(_stack.m);
            m_stack = _stack.m_stack;
        }
        threadsafe_stack& operator=(const threadsafe_stack&) = delete;

        void push(T _val) {
            std::lock_guard<std::mutex> g(m_mutex);
            m_stack.push(_val);
        }
        std::shared_ptr<T> pop() {
            std::lock_guard<std::mutex> g(m_mutex);
            if(m_stack.empty())
                throw empty_stack();
            std::shared_ptr<T> ret(std::make_shared<T>(m_stack.top()));
            m_stack.pop();
            return ret;
        }
        void pop(T& _val) {
            std::lock_guard<std::mutex> g(m_mutex);
            if(m_stack.empty())
                throw empty_stack();
            _val = m_stack.top();
            m_stack.pop();
        }
        bool empty() const {
            std::lock_guard<std::mutex> g(m_mutex);
            return m_stack.empty();
        }

    private:
        std::mutex m_mutex;
        std::stack<T> m_stack;
    };
```

实现去除了*copy assginment operator*，保留了*copy constructor*。同时在*copy constructor*也使用了互斥保护。

以上描述的都是因为*mutex*保护范围不够充分导致*race condition*的情况，但是如果*mutex*保护范围过大，会导致线程阻塞时间过长，不仅抹除并发所带来的性能提升，甚至导致其性能还不如单线程实现。

当程序需要多个*mutex*协作阻塞时，比如同一个类的不同实例，有可能出现多个线程都被*mutex*阻塞，等待其他线程，即死锁*deadlock*。

# Dead lock

## Dead lock problem and solution
当并发编程中，使用了多个*mutex*用于竞争资源的管理，就有可能使得多个线程同时出现无限期的阻塞，即*dead lock*。比如线程1占用了资源1并阻塞等待资源2，线程2占用了资源2并阻塞等待资源1，那线程1和2都会无限期阻塞下去。

为了解决*dead lock*问题，最直接的手段就是确定一个*mutex*的执行顺序，并严格按照其进行：如果总是将锁定*mutex A*放在锁定*mutex B*之前，那么就不会出死锁。在*mutexes*具有不同的用途时，这种方法能够工作的很好。

但是存在多个*mutex*用于一个用途的情况，最常见的就是一个类型的不同实例。比如存在有两个实例进行交换的友元操作*swap*:

```c++
    class obj;
    void swap(obj&,obj&);

    class X {
    private:
        obj data;
        std::mutex m;
    public:
        X(const obj& _data) : data(_data) {}
        friend void swap(X& lhs, X& rhs) {
            if(&lhs == &rhs)
                return;
            std::lock_guard<std::mutex> g1(lhs.m);
            std::lock_guard<std::mutex> g2(rhs.m);
            swap(lhs.data, rhs.data);
        }
    };
```

*swap*操作总是先锁定第一参数的*mutex*，后锁定第二参数的*mutex*。当有两个线程同时执行了对两个实例进行*swap*操作，但是参数顺序相反，就会出现死锁。为了解决该问题，标准库提供了*std::lock*解决该问题：

```c++
    class X {
    public:
        friend void swap(X& lhs, X& rhs) {
            if(&lhs == &rhs)
                return;
            std::lock(lhs.m, rhs.m);            // 1
            std::lock_guard<std::mutex> g1(lhs.m, std::adopt_lock);     // 2
            std::lock_guard<std::mutex> g2(rhs.m, std::adopt_lock);     // 3
            swap(lhs.data, rhs.data);
        }
    }
```

代码1用于锁定两个*mutex*并且防止死锁，代码2和3用于创建*RAII*形式的*mutex*管理，*std::adopt_lock*指明*mutex*已经锁定，*std::lock_guard*只需要接管*mutex*即可。这样就可以保证函数不论是正常退出还是异常退出时，*mutex*被正确解锁；*std::lock*也有可能抛出异常，但是其保证抛出异常后所有*mutex*处于*unlock*状态。

*std::lock*可以帮助解决多个*std::mutex*同时锁定时的*dead lock*问题，但是对于分离式的无能为力。多线程编程时遵循一些原则可以帮助实现*deadlock-free*的代码。

## Further guideline for avoiding deadlock

*deadlock*不仅仅发生在*lock*的情况下，只要线程之间有可能出现存在彼此等待对方的情况就会存在死锁的情况，利用*join*就能创造出死锁：

```c++
    void g(std::thread& t) {
        std::this_thread::sleep_for(2s);
        t.join();
    }

    int main() {
        std::this_thread::sleep_for(2s);
        std::thread t1;
        std::thread t2(g, std::ref(t1));
        t1 = std::thread(g, std::ref(t2));
        while(t1.joinable()||t2.joinable())
            ;
    }
```

主线程创建了两个子线程，两个线程都在等待对方的join(),从而出现*deadlock*。

### Avoid nested locks

避免嵌套的*lock*，可以完全避免由锁定导致的*deadlock*。当使用多个*lock*时，使用*std::lock*来避免死锁。

### Avoid calling user-supplied code while holding a lock

在存在*lock*的情况下，避免使用用户提供的代码。这和避免*lock*嵌套是一个道理，因为用户代码可能会涉及*lock*操作。但是有时这也是无法避免的，比如泛型编程和函数式编程，永远不可能知道用户提供的类型和操作是怎样的。

### Acquire locks in a fixed order

如果的确需要进行多个*lock*操作，并且无法应用*std::lock*，就应当遵循“在所有线程中按固定顺序进行*lock*相关操作”的原则。

有时这种要求的实现是很简单的，比如之前的*thread_safe stack*。因为*stack*会调用用户代码，所以只要用户代码不会进行改变*stack*的结构就能保证锁定的稳定顺序。

但有时就很难保证，具有隐藏风险导致锁定顺序被破坏，比如之前因为用户代码的不同就会导致*swap*死锁。这种情况可以使用*std::lock*解决，但是有些情况就不行。比如使用一系列的*mutex*分别保护双向链表的每个节点，这样可以实现多个线程同时访问一个链表。所有操作总是先锁定靠前节点再锁定靠后节点，可以避免死锁的情况，但是这就不能实现反向遍历链表。

### Use a lock hierarchy

使用有层次的*mutex*，即使得*mutex*自身具有限制顺序的能力，比如不能够在拥有低层的*mutex*的情况锁定高层的*mutex*。

```c++
    hierarchical_mutex high_level_mutex(10000);
    hierarchical_mutex low_level_mutex(5000);
    int do_low_level_stuff();
    int low_level_func() {
        std::lock_guard<hierarchical_mutex> lk(low_level_mutex);
        return do_low_level_stuff();
    }

    void high_level_stuff(int some_param);
    void high_level_func() {
        std::lock_guard<hierarchical_mutex> lk(high_level_mutex);
        high_level_stuff(low_level_func());
    }

    void thread_a(){
        high_level_func();
    }

    hierarchical_mutex other_mutex(100);
    void do_other_stuff();
    void other_stuff() {
        high_level_func();
        do_other_stuff();
    }

    void thread_b() {
        std::lock_guard<hierarchical_mutex> lk(other_mutex);
        other_stuff();
    }
```

*thread_a*可以正常工作，而*thread_b*就会触发*hierarchical__mutex*的异常。

*hierarchical_mutex*的实现也比较简单，相当于对*std::mutex*座一层封装，应当保持*std::mutex*应有的接口以保证和标准库的兼容：

```c++
    class hierarchical_mutex {
    public:
        hierarchical_mutex() = delete;
        hierarchical_mutex(const hierarchical_mutex&) = delete;
        explicit hierarchical_mutex(unsigned int _val) 
            :hierarchyValue(_val), previousHierarchyValue(0) {}
        ~hierarchical_mutex() {	}

        void lock() {
            checkForHierarchyRule();
            internalMutex.lock();
            updateHierarchyValue();
        }
        void unlock() {
            internalMutex.unlock();
            thisThreadHierarchyValue = previousHierarchyValue;
        }
        bool try_lock() {
            checkForHierarchyRule();
            if(!internalMutex.try_lock())
                return false;
            updateHierarchyValue();
            return true;
        }
    private:
        std::mutex internalMutex;
        unsigned int hierarchyValue;
        unsigned int previousHierarchyValue;
        static thread_local unsigned int thisThreadHierarchyValue;
        void checkForHierarchyRule() {
            if(hierarchyValue >= thisThreadHierarchyValue) {
                throw std::logic_error("mutex hierarchy violated");
            }
        }
        void updateHierarchyValue() {
            previousHierarchyValue = thisThreadHierarchyValue;
            thisThreadHierarchyValue = hierarchyValue;
        }
    };

    thread_local unsigned int hierarchical_mutex::thisThreadHierarchyValue(UINT_MAX);
```

*thisThreadHierarchyValue*是一个静态线程存储周期的类成员变量，用于记录线程层次，每个线程都各自拥有自己的记录，并初始化为最大值，为所有在该线程被使用的*mutex*共享：

每当有新的*hierarchical mutex*在该线程要求*lock*时，将会将*mutex*的层次和线程层次进行对比，若该线程层次已经高于*mutex*的层次，则正常进行*lock*操作；否则抛出异常进行异常处理。

*try_lock*操作同理。

当一个*mutex*进行*unlock*操作时，线程层次将退回该*mutex*进行*lock*之前的值。

应当配合*std::lock_guard*或者*std::lock*来保证*hierarchical mutex*的正常使用。

当然这种实现也可以作为多线程设计的一种参考，即使不用于运行期检查。

### Extending these guidelines beyond locks

*deadlock*不一定要求出现*lock*，任何同步构建的过程都有可能造成等待死循环，从而导致*deadlock*。所以应当将上述建议扩充，比如：避免在持有*lock*的情况下，进行等待其他线程的行为；尽量只在开启子线程的线程中进行该线程的*join*。

使用*std::lock_guard*和*std::lock*代替直接操作*std::mutex*。

## Flexible locking with std::unique_lock

标准库提供了比*std::lock_guard*更加灵活的*std::unique_lock*:它关联一个*std::mutex*而不像*std::lock_guard*完全持有。

*std::unique_lock*提供了和*std::mutex*一样的一套*lock*操作，

*std::unique_lock*提供了*std::defer_lock*，*std::defer_lock*使得*std::unique_lock*的构造函数不进行*lock*操作，可以延后手动进行*std::lock*操作。

但提供额外灵活性的同时*std::unique_lock*会要求更多的存储空间用于存储状态，以及稍慢的运行效率：

```c++
    class some_big_object;
    void swap(some_big_object& lhs,some_big_object& rhs);

    class X {
    private:
        some_big_object some_detail;
        std::mutex m;
    public:
        X(some_big_object const& sd):some_detail(sd){}
        friend void swap(X& lhs, X& rhs) {
            if(&lhs==&rhs)
                return;
            std::unique_lock<std::mutex> lock_a(lhs.m,std::defer_lock);
            std::unique_lock<std::mutex> lock_b(rhs.m,std::defer_lock);
            std::lock(lock_a,lock_b);
            swap(lhs.some_detail,rhs.some_detail);
        }
    }
```

*std::unique_lock*还提供移动构造函数和移动赋值操作符来传递*std::mutex*的所有权，是典型的可移动不可拷贝对象。

*std::lock_guard*提供了最简单*std::mutex*锁定管理的RAII实现，意味着其无法穿过其作用域发挥锁定作用，而*std::unique_lock*可以通过转移持有权来使得锁定穿越其生命周期：

```c++
    std::unique_lock<std::mutex> get_lock() {
        extern std::mutex some_mutex;
        std::unique_lock<std::mutex> lk(some_mutex);
        prepare_data();
        return lk;
    }
    void process_data() {
        std::unique_lock<std::mutex> lk(get_lock());
        do_something();
    }
```

通过转移*std::mutex*的持有权，其锁定就可以连续的发生在两个函数的生命周期内，从process_data开始，穿过get_lock，直到process_data结束。

同时*std::unique_lock*也提供了提前解锁或者释放*std::mutex*的能力，这意味锁定可以在*std::unique_lock*的生命周期内结束。因为长时间的锁定会造成严重的性能问题，应当以“必要时锁定”为设计目标，*std::lock_guard*不具备这样的能力。

# Alternative facilities for protecting shared data
## Lock at an appropriate granularity

一个良好的多线程数据的互斥设计应该至少有以下两方面：

1. 锁的数据保护范围应该恰好只保护需要该锁保护的数据。（大小）
2. 锁应当只保护那些需要其保护的操作。（时间）

>Not only is it important to choose a sufficiently coarse lock granularity to ensure the required data is protected, but it’s also important to ensure
that a lock is held only for the operations that actually require it.

即，如果多个线程都在等待同一个资源，如果一个线程在其不需要该资源的情况下持有该资源的互斥锁，就会增加整个系统的时间开销。因此在持有锁的情况下：不要进行高时间消耗的行为，比如I/O。

> In particular, don’t do any really time-consuming activities like file I/O while
holding a lock.

*std::unique_lock*的灵活性使得能够在必要的时候释放锁，进行不相关的耗时操作：

```c++
    void get_and_process_data()
    {
        std::unique_lock<std::mutex> my_lock(the_mutex);
        some_class data_to_process=get_next_data_chunk();
        my_lock.unlock();
        result_type result=process(data_to_process);    // 1
        my_lock.lock();
        write_result(data_to_process,result);
    }
```

因为在代码1时，不用锁相关的资源，可以先释放锁，让其他线程可以访问该资源。

> In general, a lock should be held for only the
minimum possible time needed to perform the required operations

为了多线程的效率，应该是锁尽可能的持续最短的时间，比如进行两个int型的比较，由于*int*的拷贝成本很低，可以将两者先拷贝再比较拷贝副本的的大小，避免在比较过程中持有两者的锁：

```c++
    class Y
    {
    private:
        int some_detail;
        mutable std::mutex m;
        int get_detail() const {
            std::lock_guard<std::mutex> lock_a(m);
            return some_detail;
        }
    public:
        Y(int sd):some_detail(sd){}
        friend bool operator==(Y const& lhs, Y const& rhs) {
            if(&lhs==&rhs)
                return true;
            int const lhs_value=lhs.get_detail();
            int const rhs_value=rhs.get_detail();
            return lhs_value==rhs_value;
        }
    };
```

值得注意的是，通过以上的代码虽然减少了锁的持有时间，同时避免了死锁的出现，但同时也改变了这个操作的语义。

- 原语义，锁定两个*int*后，比较锁定之后的值并返回比较结果并释放锁。
- 后语义，锁定一个*int*，取副本，释放锁；锁定另一个*int*，取副本，释放锁；比较两个副本的值并返回结果。

这样两种语义下，其他线程可进行的操作是不一样的；比如前者无法在该操作中插入*swap*这样的操作，然而后者可以在两次取副本之间进行这样的操作:

>if you don’t hold the required locks for the entire duration of an operation, you’re exposing yourself to
race conditions

互斥设计就是要找到一个合适的*granularity*，但是这种设计可能是不存在的，因为不同的线程需要的数据保护等级不同，仅仅靠基本的*std::mutex*难以实现。C++标准库提供了一些替代*std::mutex*和*lock*的机制。

## Protecting shared data during initialization

有一些初始化是十分消耗资源的，所以实现总是希望在真正需要的时候进行这样的初始化，比如打开数据库，申请大块内存等等操作。

*lazy initialization*是一种常见的单线程代码：在使用资源前检查资源是否初始化，如若未初始化则初始化后使用：

```c++
    std::shared_ptr<resource> resource_ptr;
    void foo() {
        if(!resource_ptr)
            resource_ptr.reset(std::make_shared<resource>());
        resource_ptr->do();
    }
```

如果资源类型*resource*自身能够保证线程安全，那上述代码转化成多线程代码只需要保护初始化即可：

```c++
    std::shared_ptr<resource> resource_ptr;
    std::mutex resource_mutex;
    void foo() {
        std::unique_lock<std::mutex> lk(resource_mutex);
        if(!resource_ptr)
            resource_ptr.reset(std::make_shared<resource>());
        lk.unlock;          // because resource itself is safe for concurrency.
        resource_ptr->do();
    }
```

因为资源类型本身保证并发安全，所以在初始化之后就可以解开锁。

但上述代码有一个致命的缺陷，因为所有线程执行*foo*，都必然会先上锁，后检查资源是否初始化完成，这显然在初始化完毕后是不必要的，违反了多线程设计的原则：

>Not only is it important to choose a sufficiently coarse lock granularity to ensure the required data is protected, but it’s also important to ensure
that a lock is held only for the operations that actually require it.

为了解决这个问题，有许多“更好”的办法被应用，比如*infamous*的*double checked locking*模式：

```c++
    void UB_with_double_checked_locking() {
        if(!resource_ptr) {
            std::unique_lock<std::mutex> lk(resource_mutex);
            if(resource_ptr) {
                resource_ptr.reset(std::make_shared<resource>());
            }
        }
        resource_ptr->do();
    }
```

因为外部指针检查和内部的资源初始化不同步，有可能内部资源的初始化尚未完成，但指针检查已经指示资源已经初始化完成（空间分配完成），所以其他线程可能会在初始化未完成时执行*do*，造成*data race*，是一种未定义行为。

C++标准库提供了*std::once_flag*和*std::call_once*来处理这种情况，使用*std::once*一般可以比使用*lock*占用更少的消耗，特别是初始化完成后：

```c++
    std::shared_ptr<resource> resource_ptr;
    std::once_flag resource_flag;

    void init_resource() {
        resource_ptr.reset(std::make_shared<resource>());
    }

    void foo() {
        std::call_once(resource_flag, init_resource);
        resource_ptr->do();
    }
```

*std::call_once*保证同一个*std::once_flag*对象指示的可调用对象在多个线程中只会执行一次：

- 可调用对象的参数由被选中执行的线程的*std::call_once*传入的参数决定。
- 异常引起的函数退出不会归入“执行一次”的范畴。
- 如果某线程被选中执行的函数因为异常退出，并会返回给调用者。其他线程中的函数会被选中继续执行，知道成功执行一次。

*std::call_once*也可以用于成员变量的初始化：

```c++
    class X {
    public:
        void send(const data_packet& data) {
            std::call_once(connection_init_flag, &X::open_connection, this);
            connection.send(data);
        }
        void receive(data_packet& data) {
            std::call_once(connection_init_flag, &X::open_connection, this);
            connection.receive(data);
        }
    private:
        connection_info connection_details;
        connection_handle connection;
        std::once_flag connection_init_flag;
        void open_connection() {
            connection = connection_manager.open(connection_details);
        }
    };
```

另一个可能出现*data race*的初始化场景就是静态局部变量，因为静态局部变量在运行至其声明初进行初始化，不同的线程会出现*data race*，通过*std::call_once*可以避免：

```c++
    class my_class;
    std::once_flag my_class_flag;
    my_class& get_my_class_instance() {
        static my_class instance;
        return instance;
    }

    void foo() {
        std::call_once(my_class_flag, get_my_class_instance);
        ...
    }
```

## Protecting rarely updated data structures

“仅在初始化保护”其实是“保护稀少更新数据”的一种特殊情况。比如一样数据长时间只更新一次或者很少更新，平常表现基本等同于*read—only*数据。比如：

DNS储存着“域名-IP”的表，这种数据长时间不会进行更新。因此除去“写操作”以外，其他所有“读操作”可以同时访问数据并且不会出现*race condition*。那么就希望：

- “写操作”：只允许当前线程对数据进行访问，即*exclusive*。
- “读操作”：允许其他线程一起对数据进行访问，即*shared*。

如果使用*std::mutex*，因为*std::mutex*只有一种锁定状态，即“exclusive”。需要其他类型的*mutex*替代。

C++17提供了*std::shared_mutex*用于该控制（C++ concurrency in action举的是boost库的内容，因为那时*std::shared_mutex*未被纳入标准库），其具有两种锁定状态*shared*和*exclusive*：

```c++
    #include <map>
    #include <string>
    #include <mutex>
    #include <shared_mutex>

    class dns_entry;
    class dns_cache {
    private:
        std::map<std::string, dns_entry> entries;
        mutable std::shared_mutex entry_mutex;
    public:
        dns_entry find_entry(const std::string& domain) const {
            std::shared_lock<std::shared_mutex> slk(entry_mutex);
            auto it = entries.find(domain);
            return (it == entries.end())?dns_entry():it->second;
        }  // read operation

        void update_or_add_entry(const std::string& domain, const dns_entry& dns_details) {
            std::lock_guard<std::shared_mutex> lk(entry_mutex);
            entries[domain] = dns_details;
        }
    };
```

## Recursive locking

对已经锁定的*std::mutex*再次进行锁定是UB，但有些情况下期待能够递归锁定一个互斥器。C++标准库提供了*std::recursive_mutex*。一个线程可以反复给*std::recursive_mutex*多次上锁，但同时在释放锁时也是进行同样次数的解锁。比如一个类的成员函数彼此发生调用，使用*std::mutex*会导致UB，使用*std::recursive_mutex*可以避免。

多数情况下，*recursive_mutex*都是可以用其他*mutex*配合良好的设计替代，尽量不要使用*recursive_mutex*造成设计混乱。
