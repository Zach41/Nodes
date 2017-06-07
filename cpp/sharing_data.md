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
