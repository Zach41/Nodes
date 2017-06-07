# Sharing Data Notes

## 1. 用std::lock防止死锁
考虑`swap`这一操作，假设我们有如下的代码：
```C++
class some_big_object
void swap(some_big_object& lhs, some_big_objects& rhs);

class X {
private:
    some_big_object some_object;
    std::mutex m;
public:
    X(some_big_object const& sd): some_object(sd) {}
    friend void swap(X& lhs, X& rhs) {
        /* code */
    }
}
```

对于两个Ｘ对象，如果我们简单的在`swap`函数里面顺序获取`lhs`和`rhs`的锁，由于获取两个锁的操作不是原子性的，那么就有可能造成死锁（另外一个线程先获得了`rhs`的锁再尝试获取`lhs` 锁）。为了解决这一个问题，可以引入`std::lock`来一次性获得两个对象的锁。

```C++
...
friend void swap(X& lhs, X& rhs) {
    if (&lhs == &rhs)
        return;
    std::lock(lhs.m, rhs.m);
    std::lock_guard<std::mutex> lock_guard(lhs.m, std::adopt_lock);
    std::lock_guard<std::mutex> lock_guard(rhs.m, std::adopt_lock);
    swap(lhs.some_object, rhs.some_object);
}
```

`std::adopt_lock`参数用来表明锁已经获得，不需要再次尝试获取。

## 2. hierachical mutex规h获得锁的顺序
防止死锁的一个教条便是总是以同样的顺序来获取锁。为了规范程序的行为，我们可以设计一个锁结构来强制执行这一规则，而代价就是牺牲一些runtime的操作。

```C++
class hierachical_mutex {
    std::mutex internal_mutex;
    unsigned long const hierachy_value;
    unsigned long previous_hierachy_value;
    static thread_local unsigned long this_thread_hierachy_value;

    void check_for_hierachy_violation() {
        if (this_thread_hierachy_value <= hierachy_value) {
            throw std::logic_error("mutext hierarchy violated");
        }
    }

    void update_hierarchy_value() {
        previous_hierachy_value = this_thread_hierachy_value;
        this_thread_hierachy_value = hierachy_value;
    }

public:
    explicit hierachical_mutex(unsigned long value):
        hierachy_value(value),
        previous_hierachy_value(0) {}
    void lock() {
        check_for_hierachy_violation();
        internal_mutex.lock();
        update_hierarchy_value();
    }

    void unlock() {
        this_thread_hierachy_value = previous_hierachy_value;
        internal_mutex.unlock();
    }

    bool try_lock() {
        check_for_hierachy_violation();
        if (!internal_mutex.try_lock())
            return false;
        update_hierarchy_value();
        return true;
    }
};

thread_lock unsigned long hierachical_mutex::this_thread_hierachy_value(ULONG_MAX);
```

由于`this_thread_hierachy_value`被设置成了线程本地变量，每一个线程都有各自的一个拷贝，同时它被初始化成`ULONG_MAX`，那么这样一来在一个线程中，第一个锁的优先级可以取任意小于`ULONG_MAX`的值，但是第二个锁的值必须小于之前获得的锁，这就在运行时保证了锁获取的顺序。

一个使用的例子：
```C++
hierachical_mutex high_mutex(10000);
hierachical_mutex low_mutex(5000);

int do_low_stuff();

int low_func() {
    std::lock_guard<hierachical_mutex> lk(low_mutex);
    return do_low_stuff();
}

void high_stuff(int some_param);

void high_func() {
    std::lock_guard<hierachical_mutex> lk(high_mutex);
    high_stuff(low_func());             // fine
}

void thread_a() {
    high_func();
}

hierachical_mutex other_mutex(100);
void do_other_stuff();

void other_stuff() {
    high_func();
    do_other_stuff();
}

void thread_b() {
    std::lock_guard<hierachical_mutex> lk(other_mutex);
    other_stuff();          // will throw an exception
}