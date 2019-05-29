# <p style="text-align: center;">c++ 多线程编程笔记</p>

## 1. c++中线程相关的内置类型
### 1.1）std::thread
  c++11以及更新的版本为了在语言层面支持多线程，在标准库添加了一个新的头文件\<thread\>，从而避免了过去利用各种平台相关的库（例如在linux下，必须
 使用pthread库）来创建线程，极大提高了程序可移植性。\<thread\>中定义了std::thread类，其构造函数接受普通函数、类成员函数，lambda
 函数或callable对象(任一重载了“（）”操作符的类的对象）作为新线程的执行函数，该执行函数的所有参数作为std::thread构造函数的后续参数传入. 
 当std::thread类型变量被创建后，新线程便开始执行，在很多情况下主线程需要执行线程变量的join(）函数来等待新线程结束,当join()返回后，新线
 程的执行也就结束了，相应资源已经被回收；在个别情况下，新线程可能需要一直和主线程并行执行直到程序退出或用户干预（例如，一个文档编辑器在同时打开多个文件时，
 每个编辑窗口可能运行于单独的线程，这些窗口会一直打开直到用户关闭），这时我们需要调用线程变量的detach()函数，该函数返回后，线程变量不再控制新线程，新
 线程会在它的线程函数执行完毕后有标准库负责销毁。 

 例如，以下示例用无参函数say_hello()创建了线程t1，用接受一个vector\<int\>作为参数的函数sum_vector()创建了线程t2, 用Wrapper类的成员函数run()创建了线程t3, 用
 bg_task对象创建了线程t4, 用lambda函数创建了线程t5,然后调用join(）函数等待这些线程结束：

```
#include <chrono>
#include <iostream>
#include <thread>
#include <vector>

void say_hello()
{
    std::cout << "Hello world" << std::endl;
}

void sum_vector(const vector<int>& nums)
{
    int sum = 0;
    for (int num: nums) {
        sum += num;
    }
    std::cout << "sum: " << sum << std::endl;
}

class Wrapper
{
public:
    void run(int n)
    { std::cout << "Thread in Wrapper::run(): " << n << std::endl; }
};

class bg_task {
public:
    void operator ()()
    {
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
        std::cout << "I've slept 10 ms in bg_task." << std::endl;
    }
}; 


int main(void)
{
    vector<int> grades {7, 8, 9, 10};
    Wrapper w;
    bg_task bt;

    std::thread t1(say_hello)
    std::thread t2(sum_vector, grades)
    std::thread t3(&Wrapper::run, w, 75);
    std::thread t4(bt);
    std::thread t5([](){ std::cout << "Greetings from lambda thread! " << std::endl; })

    t1.join();
    t2.join();
    t3.join();
    t4.join();
    t5.join();
}
```
需要注意的是，只有当线程还在运行并且之前没有调用过detach(),才能调用其相对应std::thread变量的join()和detach()方法，否则，则调用join()或detach()方法的结果未定义。为确保正确调用join()和detach(), 我们可以先调用std::hread的joinable()函数，如果返回真，则调用join()和detach()。

### 1.2) std::mutex
如果多个线程共享一个数据结构，则不可避免地面临race condition问题，例如：在线电影票销售系统，同一时间可能有许多人在购票，每售出一张票，剩余票数减1，
直至全部售罄，这里剩余票数成为了共享资源，同一时间只能由一个线程访问，否则很可能出现一票多卖的窘境。一个共享资源在同一时刻只能由一个线程访问，也
被称为互斥访问 (mutual exclusive access), 这也是同步原语mutex命名的含义。mutex一般有两个操作：lock()和unlock(), 线程访问共享数据前应先锁定mutex,访问
完毕后，释放mutex. mutex的实现可以保证，任何线程试图锁定一个已经被其它线程锁定的mutex时，它会等待直到该mutex被释放。

c++11在标准库添加了<mutex>头文件，其中定义了std::mutex类，只要创建该类的示例，我们就创建了一个互斥锁mutex，std::mutex同样具有lock()/unlock()成员函数。
例如,下面add_data()函数访问全局数据列表nums前先锁定std::mutex类型变量`data_lock`，访问完毕释放`data_lock`:

```
#include <list>
#include <mutex>

std::list<int> nums;
std::mutex data_lock;

void add_data(int num)
{
    data_lock.lock();
    nums.push_back(num);
    data_lock.unlock();
}
```
### 1.3) std::lock\_guard, std::unique\_guard, std::scoped\_guard
- <strong>std::lock_guard</strong><br>

上边add_data()代码的一个缺点是：如果`nums.push_back()`抛出异常，`add_data()`退出前将来不及释放`data_lock`。为此，<mutex>头文件定义了`lock_guard`模板类，以一种RAII(Resource 
Acquisition Is Initialization) 的方式确保不管add_data()是正常退出还是异常退出，data_lock,都能被释放：

```
void add_data(int num)
{
    std::lock_guard<std::lock_mutex> guard(data_lock);
    nums.push_back(num);
}
```

其实现原理：`std::lock_guard`类型变量guard初始化时以std::mutex作为参数，并寻求锁定该mutex (若它已被锁定，则`lock_guard`构造函数一直等待直到mutex被释放);
`add_data()`（正常或异常）退出后，变量guard生命期结束，其析构函数被自动调用并释放了 `data_lock`.

- <strong>std::unqiue_guard</strong><br>

除了	`lock_guard`外，\<thread\>中还定义了`unique_guard`模板类。`unique_guard`不但有`lock_guard`的所有功能（在构造函数中锁定mutex,在析构函数中释放mutex），还提供了unlock()/try_lock()/lock()接口，使得调用者不必等待`lock_guard`对象生命期结束，可以提前释放mutex, 或者再次获取mutex锁，如下例所示：

```
std::mutex data_mutex;
std::vector<int> nums;

void get_and_process_data(int i)
{ 
    std::unique_lock<std::mutex> lock(data_mutex);
    int data = nums[i];
    my_lock.unlock();        
    data = process(data);
    my_lock.lock();
    nums[i] = data;
}
```

`unique_guard`另一个用途是与条件变量配合使用，因为`std::variable_condition::wait()`的第一个参数要求是`std::unique_lock`类型，其原因为：std::variable_condition::wait()在将调用它的线程放入等待队列并进入睡眠状态前，需要释放mutex， 在等待线程被`std::variable_condition::notify_one()`唤醒后，需要重新获取mutex锁，这种灵活性只有`uniue_lock`能提供。

`unique_guard`还可以配合std::derfer_lock使用，其结果是在`unique_guard`对象被创建后，mutex并没有被锁定。利用这一特点，std::lock()可以同时锁定多个mutex，而不用担心死锁问题：

```
std::unique_lock<std::mutex> lk1(mutex1, std::defer_lock);
std::unique_lock<std::mutex> lk2(mutex2, std::defer_lock);
std::lock(lk1, lk2);
```

最后， `unique_guard`不能被copy，但可以被move, 这意味着被`unique_guard`锁定的mutex的所有权也会被转移， 例如, 下面示例中`get_lock()`获取了`some_mutex`锁，在其返回后，`some_mutex`锁所有权被转移给了`process_data()`。

```
std::unique_lock<std::mutex> get_lock()
{
    extern std::mutex some_mutex;
    std::unique_lock<std::mutex> lk(some_mutex);
    prepare_data();
    return lk;
}

void process_data()
{
    std::unique_lock<std::mutex> lk(get_lock());
    do_something();
}
```

既然，`unique_guard`即拥有`lock_guard`的全部功能，又提供如此多的额外功能，为什么我们还要使用`lock_guard`呢？天下没有免费的午餐，`unique_guard`众多额外功能是以性能为代价的，其执行速度要比`lock_guard`慢，因此，只要在满足需求的前提下，应优先使用`lock_guard`。<br>

- <strong>std::scoped_guard</strong>><br>

`std::scoped_guard`是c++17引入的新类型，可以作为`std::lock_guard`的替代品，除具有`std::lock_guard`全部功能外，因为`std::scoped_guard`构造函数接收不定个数的参数，可以一次锁定多个mutex，而且这种锁定是deadlock-free的：

```
std::scoped_lock<std::mutex, std::mutex> lock(mutex1, mutex2);
```
<br>


### 1.4) std::variable\_condition, std::variable\_condition\_any
std::mutex用于线程间保护共享数据，但有时多个线程间不仅需要互斥访问共享数据，还要同步协调其执行次序，例如著名的生产者-消费之问题：生产者生产产品放入公共队列，消费者从公共队列取走产品；公共队列作为共享资源，需要由mutex来保护；当队列已满，生产者不能继续放入产品，需要等待直到被告知队列有空余；当队列为空时，消费者不能从继续取走数据，需要等待直到被告知队列有产品。“等待直到某个事件发生或条件满足”正是同步原语条件变量（condition variable)要解决的问题。c++标准库提供了两种条件变量: ``std::condtion_variable` 和`std::condition_variable_any`，分别看一下：

<strong> std::condtion_variable</strong><br>
在生产者-消费者例子中，消费者从公共取取数据的实现如下：

```
std::queue<int> dataQ；
std::mutex data_mu;
std::conditional_variable cond_queue_not_empty;

void consume_data()
{
    while(true) { // Or some other exit condition 
        std::unique_lock<std::mutex> lck(data_mu);
        cond_queue_not_empty.wait(lck, [](){ return !dataQ.empty()}
        int data = dataQ.front();
        std::cout << "Read data from queue: " << data << std::endl;
        dataQ.pop();
    }
}
```

生产者向公共队列放入数据的实现如下：

```
void produce_data()
{
    while(true) { // Or some other exit condition 
        std::unique_lock<std::mutex> lck(data_mu);
        int data = GetData();
        dataQ.push(data);
        std::cout << "Put data into queue: " << data << std::endl;
        cond_queue_not_empty.notify_one();
    }
}
```

这里我们用到了std::condition_variable的wait()和 `notify_one()`函数。wait()第一个参数为std::unique_lock<*>类型，第二个参数为返回布尔值得函数或lambda函数，其逻辑为：调用wait()前，当前线程应已经锁定第一个参数所指向的unique_lock对象，接着执行第二个参数所代表的函数，若其返回值为真，则继续持有当前锁，并从wait()返回;否则，释放unique_lock, 把当前线程放入该条件变量的的等待队列。当调用条件变量的`notify_one()`时，会从该条件变量的等待队列中唤醒某个等待线程，被唤醒线程是因为调用同一条件变量的wait()函数进入等待状态的，被唤醒后，继续持有unique_lock锁，然后执行wait()的第二个参数所代表的函数，如果为真，则从wait()返回；否则，释放锁，继续进入等待队列。

std::condition_variable还提供了`notify_all()`函数，当有多个线程同时在等待该条件变量时，`notify_one()`只会唤醒一个等待线程，`notify_all()`会唤醒所有等待线程，被唤醒的线程会检查所等待的条件是否被满足，如仍不满足，则继续回到睡眠状态，否则从wait()返回。



## 2. 线程终止 

## 3. c++多线程示例
### 3.1) thread_guard
  有些情况下，新的线线程已被创建，但在调用线程join()函数前，异常发生，新的线程没有机会被释放。为避免这种情况，可以借鉴RAII思想，就像std::lock_guard作用于std::mutex一样，下面创建了一个thread_guard类：

```
#include <thread>

class thread_guard
{
public:
    explicit thread_guard(std::thread &t): t_(t) {}
    thread_guard(const thread_guard &) = delete;
    thread_guard operator=(const thread_guard &) = delete;
    ~thread_guard() { if t_.joinable() t.join; }

private:
    std::thread t_;
}；
```