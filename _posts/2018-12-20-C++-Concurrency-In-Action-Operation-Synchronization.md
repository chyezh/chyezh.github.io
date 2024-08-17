---
layout: post
title:  "[C++ Concurrency In Action] Operation Synchronization"
date:   2018-12-20 22:01:03 +0800
categories: ["C++ Readings"]
tags: [C++]
---

<!-- more -->

# Operation Synchronization

# Waiting for events

## Why do not use shared data to synchronize operation

并发编程中，经常会碰见多个线程之间的操作同步*operation synchronization*。实现操作同步很容易想到利用一个共享数据进行同步，即线程A会通过改变一个*flag*发出同步信号，使得线程B通过检查*flag*实现同步：

```c++
    bool flag;
    std::mutex flag_mutex;

    void wait_flag() {
        std::unique_lock<std::mutex> lk(flag_mutex);
        while(!flag) {
            lk.unlock();
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
            lk.lock();
        }
        dosomething();
    }
```

线程B每隔100ms上一次锁并检查*flag*，如果*flag*为假，继续检查；如果*flag*为真，则执行*dosomething*。

但是这种设计有几大缺陷：

1. 检查时间间隔难以把握，如果时间太短，导致检查频繁，大量占用资源；如果时间太长，同步反应过慢，产生延迟。
2. 设计逻辑难看

## Condition variable

*std::condition_variable*提供了在一个线程唤醒其他线程的特性。*std::condition_variable*通过可以通过一个*std::mutex*来提供同步功能；*std::condition_variable_any*可以通过任何一个*mutex-like*对象进行同步，但需要消耗额外资源。

```c++
    std::mutex mut;
    std::queue<data_chunk> data_queue;
    std::condition_variable data_cond;
    // thread A
    void data_preparation_threadA() {
        while(more_data_to_prepare()) {
            data_chunk const data=prepare_data();
            std::lock_guard<std::mutex> lk(mut);
            data_queue.push(data);
            data_cond.notify_one();
        }
    }
    // thread B
    void data_processing_thread() {   
        while(true)
        {
            std::unique_lock<std::mutex> lk(mut) 
            data_cond.wait(lk,[]{return !data_queue.empty();});
            data_chunk data=data_queue.front();
            data_queue.pop();
            lk.unlock();        // unlock before processing
            process(data);
            if(is_last_chunk(data))
                break;
        }
    }
```

线程A作用是写入数据并在写入数据后唤醒线程B，线程B的作用是在线程A唤醒后执行数据处理。

线程A得到数据块后先申请队列的锁，上锁后将数据块*push*进入队列后，调用*notify_one*唤醒线程B，单次循环结束释放锁；

线程B进入循环，先要求申请队列的锁，然后调用*condition_variable*的*wait*，*wait*有两种形式：

```c++
    void wait( std::unique_lock<std::mutex>& lock );

    template< class Predicate >
    void wait( std::unique_lock<std::mutex>& lock, Predicate pred );
```

其中第二中实现相当于：

```c++
    while (!pred()) {
        wait(lock);
    }
```

每当调用*wait*时，wait先检验条件*pred*，如果*pred*为真则返回；否则调用*wait*第一种形式，释放锁并且阻塞当前线程直到被线程A唤醒，被唤醒后锁定并且继续检查*pred*，如果*pred*为真则返回。因为需要进行灵活的锁定控制，因此传入参数为*std::unique_lock*。

注意，在代码中调用一次*wait*时，*pred*可能会被调用任意次，不要使用一些带有边际效益的处理作为*pred*，因为存在*spurious wake*，即线程A并没有做唤醒操作，而线程B被唤醒了。

## Thread safe queue

```c++
    template<typename T>
    class threadsafe_queue {
    public:
        typename std::queue<T>::size_type size() const {
            std::lock_guard<std::mutex> lk(data_mutex);
            return data.size();
        }

        bool empty() const {
            std::lock_guard<std::mutex> lk(data_mutex);
            return data.empty();
        }

        void push(T new_val) {
            std::lock_guard<std::mutex> lk(data_mutex);
            data.push(new_val);
            data_condition.notify_one();
        }

        bool try_pop(T& val) {
            std::lock_guard<std::mutex> lk(data_mutex);
            if(data.empty())
                return false;
            val = data.front();
            data.pop();
            return true;
        }

        std::shared_ptr<T> try_pop() {
            std::lock_guard<std::mutex> lk(data_mutex);
            if(data.empty())
                return std::shared_ptr<T>();
            std::shared_ptr<T> ret = std::make_shared<T>(data.front());
            data.pop();
            return ret;
        }

        void wait_and_pop(T& val) {
            std::unique_lock<std::mutex> ulk(data_mutex);
            data_condition.wait(ulk, [this] { return !data.empty(); });
            val = data.front();
            data.pop();
        }

        std::shared_ptr<T> wait_and_pop() {
            std::unique_lock<std::mutex> ulk(data_mutex);
            data_condition.wait(ulk, [this] { return !data.empty(); });
            std::shared_ptr<T> ret(std::make_shared<T>(data.front()));
            data.pop();
            return ret;
        }
    private:
        std::queue<T> data;
        mutable std::mutex data_mutex;
        std::condition_variable data_condition;
    };
```

注意成员函数中存在*const*成员函数、复制构造函数和复制赋值操作符，这类操作都会涉及到*std::mutex*的上锁，但是对*const std::mutex*上锁是不可能的，所以应该使用*mutable*修饰*std::mutex*。

# Waiting for one-off events

C++标准库使用*std::future*来标示一个一次性事件：

- *std::future*本身是用于提供一个链接异步操作的途径，可以通过：*std::async*|*std::packaged_task*|*std::promist*来构造
- *std::future*可以绑定一个异步操作返回的数据块，*std::future*的持有线程可以通过*wait*|*get*|*valid*等方式进行操作，如果异步操作尚未产生返回，则该方法会产生阻塞。

## std::async,returning values from background task

*std::async*通过接受*function object*产生一个*std::future*。*std::async*有一个控制选项：

- std::launch::async
- std::launch::deferred
- std::launch::async|std::launch::deferred or default.

选项一，总是启用新线程执行*callable object*（asynchronous execution）；  
选项二，在当前线程，第一次获取结果时执行*callable object*（lazy evaluation）；  
选项三，依赖实现。

如下代码实现并行加法：

```c++
    template<typename Iterator>
    struct add_block {
        auto operator()(const Iterator _start, const Iterator _stop) {	
            return std::accumulate(_start, _stop, 0);
        }
    };

    template<typename Iterator>
    auto parallel_add(const Iterator _begin, const Iterator _end, typename Iterator::value_type _init) {
        auto arraySize = std::distance(_begin, _end);
        if(!arraySize)
            return _init;
        size_t maxNumHardwareThread = std::thread::hardware_concurrency();
        size_t minAmountPerBlock = 25;
        size_t maxNumSoftwareThread = (arraySize + minAmountPerBlock - 1) / minAmountPerBlock;
        auto numThread = std::min(maxNumSoftwareThread == 0?1:maxNumSoftwareThread, maxNumHardwareThread);
        auto amountPerBlock = arraySize / numThread;
        std::vector<std::future<typename Iterator::value_type>> vecResultPerFuture;
        vecResultPerFuture.reserve(numThread - 1);
        auto _start = _begin;
        auto _stop = _start;
        for(int i = 0; i < numThread - 1; i++) {
            std::advance(_stop, amountPerBlock);
            vecResultPerFuture.emplace_back(std::async(std::launch::async,add_block<Iterator>(), _start, _stop));
            std::advance(_start, amountPerBlock);
        }
        std::vector<typename Iterator::value_type> vecResult;
        vecResult.resize(numThread);
        for(int i = 0; i < numThread - 1;i++) {
            vecResult[i] = vecResultPerFuture[i].get();
        }
        vecResult[numThread - 1] = std::accumulate(_start, _end, _init);
        return std::accumulate(std::begin(vecResult), std::end(vecResult), 0);
    }
```

## std::packaged_task，associating a task with a future

*std::packgaed_task*可以包装一个*callable object*来使得其可以异步调用，返回内容通过*std::future*来访问。可以用于构建线程池；以及其他任务管理。  
*std::packaged_task*的模板参数是一个函数类型（注意函数类型中存在隐式转换）。  
*std::packaged_task*自身也是一个*callable object*，可以进一步封装也可以直接调用。  
*std::packaged_task*是*movable*和*swapable*的。

```c++
    std::mutex m;
    std::deque<std::packaged_task<void()> > tasks;
    bool gui_shutdown_message_received();
    void get_and_process_gui_message();
    void gui_thread()
    {
        while(!gui_shutdown_message_received())
        {
            get_and_process_gui_message();
            std::packaged_task<void()> task;
            {
                std::lock_guard<std::mutex> lk(m);
                if(tasks.empty())
                continue;
                task=std::move(tasks.front());
                tasks.pop_front();
            }
            task();
        }
    }
    std::thread gui_bg_thread(gui_thread);
    template<typename Func>
    std::future<void> post_task_for_gui_thread(Func f)
    {
        std::packaged_task<void()> task(f);
        std::future<void> res=task.get_future();
        std::lock_guard<std::mutex> lk(m);
        tasks.push_back(std::move(task));
        return res;
    }
```

来自书上的例子，*gui_thread*通过一个任务队列接收任务并且执行；*post_task_for_gui_thrad*通过向一个任务队列加入任务来发布任务，并且将该任务的*future*返回给外层，注意*get_future*对于一个任务只能调用一次。

## std::promise, make promises

*std::promise*用于设置一个异步接受的值，*movable*。

```c++
    void accumulate(std::vector<int>::iterator first,
                    std::vector<int>::iterator last,
                    std::promise<int> accumulate_promise)
    {
        int sum = std::accumulate(first, last, 0);
        accumulate_promise.set_value(sum);  // Notify future
    }
    
    void do_work(std::promise<void> barrier)
    {
        std::this_thread::sleep_for(std::chrono::seconds(1));
        barrier.set_value();
    }
    
    int main()
    {
        // Demonstrate using promise<int> to transmit a result between threads.
        std::vector<int> numbers = { 1, 2, 3, 4, 5, 6 };
        std::promise<int> accumulate_promise;
        std::future<int> accumulate_future = accumulate_promise.get_future();
        std::thread work_thread(accumulate, numbers.begin(), numbers.end(), std::move(accumulate_promise));
        accumulate_future.wait();  // wait for result
        std::cout << "result=" << accumulate_future.get() << '\n';
        work_thread.join();  // wait for thread completion
        std::promise<void> barrier;
        std::future<void> barrier_future = barrier.get_future();
        std::thread new_work_thread(do_work, std::move(barrier));
        barrier_future.wait();
        new_work_thread.join();
    }
```

来自cppreference的例子，在当前线程使用一个*std::future*获取一个*std::promise*的未来结果，让后将*promise*移动给新线程。新线程执行完成后设置*promise*的值，当前线程可以通过*future*获得该值。

*std::promise\<void>*用于不需要任何值的情况，可以单纯的用于阻塞或者定时或者标示某些代码块已经执行完成。

## Saving exception for the future

*std::future*除了返回值以外也可以返回异常，将异常抛到当前*std::future*所在线程进行处理，有以下几种情况会使得*std::future*产生异常。

- *std::async*和*std::packaged_task*执行的代码块抛出异常。
- *std::promise*设置异常。
- *std::packaged_task*和*std::promise*进行了非法操作。

使用get(),重新抛出异常，比如情况一:

```c++
    double sqrt_root(int i) { 
        if(i < 
        0)
            throw std::out_of_range("i<0");
        return sqrt(i);
    }

    void foo() {
        std::future<double> f = std::async(std::lauch::async, sqrt_root, -1);
        try {
            f.get();
        } catch(const std::out_of_range& e){
            std::cout << "out_of_range: " << e.what() << std::endl;
        }
    }
```

情况二：

```c++
    void some(std::promise<double> p) {
        try {
            p.set_value(some_func_may_throw());

        } catch(const std::exception& e) {
            p.set_exception(std::current_exception());
        }
    }

    void foo() {
        std::promise<double> p;
        std::future<double> f = p.get_future();
        std::thread t1(some, std::move(p));
        try {
            f.get();
        } catch(const std::exception& e) {
            std::cout << "out_of_range: " << e.what() << std::endl;
        }
    }
```

抛出*std::future_error*，情况三：

```c++
    void foo() {
        std::future<double> f;
        {
            std::promise<double> p;
            f = p.get_future();
        }
        try {
            f.get();
        } catch(cosnt std::future_error& e) {
            if(e.code() == std::future_errc::broken_promise)
			std::cout << "exception: " << e.what() << std::endl;
        }
    }
```

## Waiting from multiple threads

*std::future*用于在线程间（或者非线程间）一次性传递各种结果，并且对一个实例的操作没有任何同步行为。如果在多个线程对同一个*std::future*进行操作就会产生*data race*。  
因此*std::future*被设计对持有数据为*unique ownership*，一次绑定只能进行一次*get*操作，不应该在多个线程引用一个*std::future*实例。  

为了满足多个线程获取同一个消息，*std::shared_future*被设计为*shared ownership*。*std::future*只能被*move*，数据可以被传递并且只能被*get*一次。而*std::shared_future*可以被*copy*，多个*future*可以关联同一块数据。

注意为了多个线程访问一块数据不出现*data race*，应该在不同线程*copy*一个*std::shared_future*而不是引用它，因为对于一个实例的不同成员函数并没有同步，一个线程只应该保留一个*std::shared_future*。

```c++
    std::promise<std::string> p;
    std::shared_future<std::string> sf(p.get_future());

    std::promise<int> p;
    std::future<int> f(p.get_future());
    std::shared_future<int> sf(std::move(f));
```

使用auto来推断*std::shared_future*类型：

```c++
    std::promise< std::map< SomeIndexType, SomeDataType, SomeComparator,
    SomeAllocator>::iterator> p;
    auto sf=p.get_future().share()
```

# Time limit

## std::chrono

*std::chrono*是C++11提供的时间库，其中有三个重要概念：

- *clock*：表示时钟。 
- *duration*：表示时间间隔。
- *time_point*：表示时间点。

## clocks

一个时钟至少要表达下列4个信息：

- *now*，现在的时间点。
- *type of value*，表达时间的类型。
- *tick period*，计数表达的间隔。
- *is_steady*，是否可以看作稳定时钟。

C++11标准库提供了3种时钟：

- *std::chrono::system_clock*
- *std::chrono::steady_clock*
- *std::chrono::high_resolution_clock*

### std::chrono::system_clock

- 方法*now*会返回当前操作系统时钟的时间点。
- 时钟的起始时刻不定，一般采用*Unix Time*。
- 是唯一可以与C标准库*std::time_t*进行转换的C++标准库时钟。
- 一般不稳定，因为系统时钟是可以随时更改的。

一般用于获取系统时钟，日期，不适合做测量计时。

### std::chrono::steady_clock

- 方法*now*会返回时钟的时间点。
- 时钟的起始时刻不定，和系统时钟无关。
- 总是稳定。

一般用于测量计时。

### std::chrono::high_resolution_clock

- 方法*now*会返回当前操作系统时钟的时间点。
- 提供当前系统最高精度的时钟，有可能会是*std::chrono::system_clock*和*std::chrono::steady_clock*别名。

## duration

*std::chrono::duration*模板用于表示时间段：第一个模板参数为表达时间段的类型，第二个模板参数为模板*std::ratio\<>*用于各种类型的*duration*的转换以及表达*tick period*（分数/秒）。

*std::chrono::duration*支持各种时间计算，以及不同时间类型的转换*duration_cast*。

以及常用的特化：

```c++
    std::chrono::nanoseconds
    std::chrono::microseconds
    std::chrono::milliseconds
    std::chrono::seconds
    std::chrono::minutes
    std::chrono::hours
```

比如：

```c++
    auto a_hour = hours(1);

	std::cout << "\n" << a_hour.count() << " hour is "
		<< duration_cast<minutes>(a_hour).count() << " minutes\nis "
		<< duration_cast<seconds>(a_hour).count() << " seconds\nis "
		<< duration_cast<milliseconds>(a_hour).count() << " milliseconds\n";

    auto some_seconds = seconds(30);
	std::cout << "\n" << a_hour.count() << " hour minus " << some_seconds.count()
		<< " seconds leaves " << (a_hour - some_seconds).count() << " seconds\n";
```


## time point

模板*std::chrono::time_point*用于表达时间点：第一个模板参数*Clock*用于表达时钟类型，第二个模板参数*Duration*使用*std::chrono::duration*的某个特化来表达从*epoch*开始的时间段（默认使用*Clock::duration*）。

*time_point*通过某个时间点*epoch*为基准加上某时间段来表达某个时间点，调用*time_since_epoch*来返回这个时间间隔。

*std::chrono::time_point*支持各种时间计算，以及不同时间类型的转换*duration_cast*。

比如：

```c++
	using namespace std::chrono;

	time_point<system_clock> t1;
	time_point<system_clock> t2 = system_clock::now();

	// show system time now and epoch
	auto epoch_time = system_clock::to_time_t(t1);
	auto now_time = system_clock::to_time_t(t2);
	std::cout << "now " << std::ctime(&now_time);
	std::cout << "epoch " << std::ctime(&epoch_time);

	// time arithmetic
	time_point<system_clock> t3 = t2 - hours(24);

	// use time_since_epoch.
	std::cout << "hours since epoch: "
		<< duration_cast<std::chrono::hours>(
			t2.time_since_epoch()).count()
		<< '\n';
	std::cout << "yesterday, hours since epoch: "
		<< duration_cast<std::chrono::hours>(
			t3.time_since_epoch()).count()
		<< '\n';
```

## _until and _for

标准线程库提供了一系列带有*_until*和*_for*后缀的方法。包括*sleep*、*condition_variable*、*future*以及前面没有提到的*timed_mutex*。  

- *_until*表达直到某个时间点*std::chrono::time_point*。
- *_for*表达等待一个时间段*std::chrono::duration*。

这类函数返回会是一个*bool*或者一个状态值，具体见库。

# Using synchronization of operations to simplify code

C++标准库提供了许多同步操作，通过这些同步操作可以简化多线程的代码并且提高可读性。以下是一些适合于多线程的编程范式。

## Functional programing with future

C++11中加入了许多函数式编程（*functional programing*）需要的特性，包括*lambda*、自动类型推断*auto*、更好用的合并*std::bind*。因为严格的函数没有副作用，只需要关心输入和输出，可以极大避免*race condition*。  
*std::future*的特性，使得多线程编程中使用函数范式更加简单，可以更加简单的将单线程函数式代码改造成多线程函数式代码：  

比如一个快排程序：

```c++
    template<typename T>
    std::list<T> sequential_qsort(std::list<T> input) {
        if(input.empty())
            return input;
        std::list<T> result;
        result.splice(result.begin(), input, input.begin());
        const T& pivot = *result.begin();
        auto divide_point = std::partition(input.begin(), input.end(), [&pivot](const T& t) { return t < pivot; });

        std::list<T> lower_group;
        lower_group.splice(lower_group.end(), input, input.begin(), divide_point);

        auto lower(sequential_qsort(std::move(lower_group)));
        auto higher(sequential_qsort(std::move(input)));

        result.splice(result.end(), higher);
        result.splice(result.begin(), lower);
        return result;
    }
```

函数的接口是FP形式的，但是为了避免过多的复制操作，内部操作并不是严格的函数式。为了将其改造成多线程代码，可以开辟出一个新线程负责划分后的一半数据的快排：

```c++
    template<typename T>
    std::list<T> parallel_qsort(std::list<T> input) {
        if(input.empty())
            return input;
        std::list<T> result;
        result.splice(result.begin(), input, input.begin());
        const T& pivot = *result.begin();
        auto divide_point = std::partition(input.begin(), input.end(), [&pivot](const T& t) { return t < pivot; });

        std::list<T> lower_group;
        lower_group.splice(lower_group.end(), input, input.begin(), divide_point);

        bool flag = false;
        std::future<std::list<T>> lower_future;
        std::list<T> lower;
        if(lower_group.size() > 1000) {
            lower_future = std::async(std::launch::async,
                                    &parallel_qsort<T>,
                                    std::move(lower_group));
            flag = true;
        }
        else
            lower = parallel_qsort(std::move(input));


        auto higher(parallel_qsort(std::move(input)));

        result.splice(result.end(), higher);
        if(flag)
            result.splice(result.begin(), lower_future.get());
        else
            result.splice(result.begin(), lower);
        return result;
    }
```

这并不是并发快排的最佳实现，比如*std::partition*依旧是一个单线程操作。这样每一次递归调用快排就会开辟一个新的线程负责一半数据的快排，另一半在当前线程进行快排。但显然通过递归，线程数量会快速上升，导致*massive oversubcription*。（注意*std::async*的默认模式是取决于实现的）。另一方面使用封装的*spawn_task*会比*std::async*更好：

```c++
    template<typename F, typename A>
    std::future<std::result_of_t<F(A&&)>>
    spawn_task(F&& f, A&& a) {
        using result_type = std::result_of_t<F(A&&)>;
        std::packaged_task<result_type(A&&)> task(std::move(f));
        std::future<result_type> res(task.get_future());
        std::thread t(std::move(task), std::move(a));
        t.detach();
        return res;
    }
```

*spawn_task*使用*std::packaged_task*和*std::thread*封装了一个功能和*std::async*类似的类，虽然不能自动防止*massive oveersubscription*，但是更加适合扩展（比如加入线程池）。

## Synchronizing operations with message passing

CSP（*Communicating Sequential Processes*），即线程之间没有任何*shared data*，线程之间只有通过信道交换信息。
