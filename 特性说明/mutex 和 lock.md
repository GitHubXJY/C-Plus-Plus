# mutex 和 lock

在多线程环境中，多个线程竞争同一个共享资源，为保证线程安全，引入锁的机制，保证任意时刻只有一个线程在访问共享资源。

C++多线程中的锁主要有：互斥锁、条件锁、自旋锁、读写锁、递归锁。

## 五种锁

**互斥锁**：互斥锁是用于控制多个线程对它们之间共享资源互斥使用的信号量，避免多个线程在同一时刻同时操作共享资源。在某一时刻，只有一个线程可以获取互斥锁，在释放互斥锁之前其他线程都不能获取该互斥锁。如果其他线程想要获取这个互斥锁，那么这个线程只能以阻塞方式进行等待。

**条件锁**：条件锁就是所谓的条件变量，某一个线程因为某个条件未满足时可以使用条件变量使该程序处于阻塞状态。一旦条件满足以“信号量”的方式唤醒一个因为该条件而被阻塞的线程(常和互斥锁配合使用)，唤醒后，需要检查变量，避免虚假唤醒。

**自旋锁**：互斥锁是一种`sleep-waiting`的锁，某一线程尝试获取一个已经被其它线程拥有的互斥锁时该线程进入阻塞队列等待，并释放CPU。而自旋锁是一种`busy-waiting`的锁，也就是说，如果其它线程正在使用自旋锁，而某一线程也去申请这个自旋锁，该肯定得不到这个自旋锁。与互斥锁相反的是，此时该线程会一直不断地循环检查锁是否可用（自旋锁请求），直到获取到这个自旋锁为止。这个过程会一致占用CPU请求这个自旋锁，直到获取这个锁为止。

**读写锁**：假设多个线程使用的共享资源是一个共享变量，那么多个线程想修改这个变量，应该和互斥锁一样，某一时刻只允许一个线程进行操作。如果多个线程只是想读这个变量，那么它们之间没有竞争关系，我们期望多个线程试图读取时可以立刻获得因为读取而加的锁，而不需要阻塞等待前一个线程。因此对资源的访问有两种模式：***写操作***：独占排它的访问；***读操作***：可以同时访问，分别对应读锁和写锁。因此：如果一个线程用读锁锁定了临界区，其他线程也可以用读锁来进入临界区。如果这时再用写锁加锁就会发生阻塞。写锁请求阻塞后，后面继续有读锁来请求时，这些后来的读锁都将会被阻塞，避免读锁长期占有资源，防止写锁饥饿。如果一个线程用写锁锁住了临界区，那么其它线程无论是读锁还是写锁都会发生阻塞。

**递归锁**：`std::recursive_mutex`与`std::mutex`一样，也是一种可以被上锁的对象，但是和`std::mutex`不同的是，`std::recursive_mutex`允许同一个线程对互斥量多次上锁（即递归上锁），来获得对互斥量对象的多层所有权，`std::recursive_mutex`释放互斥量时需要调用与该锁层次深度相同次数的`unlock()`，即`lock()`次数和`unlock()`次数相同。例如函数`F1`需要获取锁`mutex`，函数`F2`也需要获取锁`mutex`，同时函数`F1`中还会调用函数`F2`。如果使用`std::mutex`必然会造成死锁。但是使用`std::recursive_mutex`就可以解决这个问题。

## 1 Mutex

`Mutex`：互斥量、互斥锁。互斥锁是一个信号量，用于控制多个线程对其共享资源的互斥访问。`Mutex`系列的类有：

C++11

* `std::mutex`：最基本的`Mutex`类。
* `std::recursive_mutex`：递归`Mutex`类。
* `std::time_mutex`：定时`Mutex`类。
* `std::recursive_timed_mutex`：定时递归`Mutex`类。

C++14

* `std::shared_mutex`：共享（读写）的`Mutex`类。

### 1.1 std::mutex

C++11起提供`std::mutex`类型，`metux`类是一个同步原语，用于保护共享数据不被多个线程同时访问。`mutex`提供独占的、非递归的所有权语义。`std::mutex`不可复制也不可移动。

对于`std::mutex`类型的对象，任意时刻最多允许一个线程对其进行锁定。在某一时刻，只有一个线程可以获取互斥锁，在释放互斥锁之前其他线程都不能获取该互斥锁。如果其他线程想要获取这个互斥锁，那么这个线程只能以阻塞方式进行等待。

* 调用线程从成功调用`lock()`或`try_lock()`直到调用`unlock()`为止都拥有`mutex`。
* 当一个线程拥有`mutex`时，所有其它调用线程将阻塞（对于调用`lock()`）获取`mutex`的所有权或收到`false`的返回值（对于调用`try_lock()`）。
* 调用线程在调用`lock()`或`try_lock()`之前不得拥有`mutex`。

成员函数：

* `lock()`：调用该函数的线程尝试锁定互斥锁，如果锁定不成功（即其它线程已经锁定且未释放），则当前线程阻塞在`lock()`这里不停的尝试去锁定。如果锁定成功，进行操作，操作完成后调用`unlock()`释放锁，否则会产生死锁。
* `try_lock()`：尝试锁定互斥锁，如果互斥锁不可用则返回`false`，且锁定不成功当前线程不阻塞。
* `unlock()`：解锁，释放对互斥量的所有权。

通过构造`std::mutex`类型的实例创建互斥量，调用成员函数`lock()`来锁定它，调用`unlock()`来解锁。不过一般不推荐这种做法，因为一个拥有`mutex`的线程被销毁或被终止，程序的行为是未定义的。标准库提供`std::lock_guard`、`std::unique_lock`、`std::scoped_lock`(C++17起)类模板以更加`exception-safe`的方式管理`std::mutex`。

### 1.2 std::recursive_mutex

`std::recursive_mutex`和`std::mutex`一样，都是可以被上锁的对象，但`std::recursive_mutex`允许同一个线程对互斥量多次上锁（即递归上锁），来获得互斥量对象的多层所有权，`std::recursive_mutex`释放互斥量时需要调用和该锁层次深度相同次数的`unlock()`，即`lock()`的次数要和`unlock()`相同。除此之外，`std::recursive_mutex`的特性和 `std::mutex`大致相同。

### 1.3 std::time_mutex

`std::time_mutex`比`std::mutex`多两个成员函数：`try_lock_for()`和`try_lock_until()`。

`try_lock_for()`函数接受一个时间范围，表示在这一段时间范围之内线程如果没有获得锁则被阻塞住（与`std::mutex`的`try_lock()`不同，`try_lock`如果被调用时没有获得锁则直接返回`false`），如果在此期间其他线程释放了锁，则该线程可以获得对互斥量的锁，如果超时（即在指定时间内还是没有获得锁），则返回`false`。

`try_lock_until()`函数则接受一个时间点作为参数，在指定时间点未到来之前线程如果没有获得锁则被阻塞住，如果在此期间其他线程释放了锁，则该线程可以获得对互斥量的锁，如果超时（即在指定时间内还是没有获得锁），则返回`false`。

## 2 lock

### 2.1 std::lock_guard

```C++
template<class Mutex>
class lock_guard;

std::mutex mutex_;
std::lock_guard<std::mutex> lock_(mutex_);
```

`std::lock_guard`类型是一个互斥锁包装器，它提供一种方便的`RAII`样式机制，用于在对象的作用域块的持续时间内拥有互斥锁。`lock_guard`是不可复制的，要锁定的互斥体的类型（`template <class Mutex>`）必须满足 [BasicLockable](https://en.cppreference.com/w/cpp/named_req/BasicLockable) 要求。

`std::lock_guard`类只有构造函数和析构函数。当创建`lock_guard`类型对象时，其构造函数自动调用传入对象的`mutex::lock()`函数，尝试获取为其提供的互斥量的所有权。当对象超出作用域时，`lock_guard`被destroy，自动调用析构函数，其中自动调用传入对象的`mutex::unlock()`函数，解锁互斥量（释放互斥锁）。`std::lock_guard`对象并不负责管理`std::mutex`对象的生命周期，`std::lock_guard`对象只是简化了mutex对象的上锁和解锁操作

`std::mutex`和`std::lock_guard`搭配使用，但常见的错误是不给`lock_guard`变量命名，例如`std::lock_guard(mtx);`（默认构造一个传入`mtx`的`lock_guard`变量），它构造了一个立即销毁的纯右值对象，没有有效的构造一个保存互斥锁的对象。（`std::lock_guard`和`std::unique_lock`同理）

```C++
explicit lock_guard(mutex_type& m);
lock_guard(mutex_type& m, std::adopt_lock_t t);
```

实际上，`std::lock_guard`有两个构造函数，第2个构造函数有两个参数，其中第二个参数类型为`std::adopt_lock()`，是一个结构体对象，起标记作用，表示这个互斥量已经被`lock`。这个构造函数假定：当前线程已经对传入对象进行锁定，所以不需要在构造函数中再调用它的`lock()`函数，即传入的互斥量已经被锁定。

### 2.2 std::unique_lock

```C++
template<class Mutex>
class unique_lock;
```

`std::unique_lock`类是通用互斥锁的所有权包装器，是对通用`mutex`的封装，以独占所有权的方式(unique owership)管理mutex对象。允许延迟锁定、时间受限的锁定尝试、递归锁定、锁所有权转移以及和`condition_variable`一起使用。`std::unique_lock`类是可移动的，但不可复制。它满足 [MoveConstructible](https://en.cppreference.com/w/cpp/named_req/MoveConstructible) 和 [MoveAssignable](https://en.cppreference.com/w/cpp/named_req/MoveAssignable) 的要求，但不满足 [CopyConstructible](https://en.cppreference.com/w/cpp/named_req/CopyConstructible) 或 [CopyAssignable](https://en.cppreference.com/w/cpp/named_req/CopyAssignable) 的要求。

`std::unique_lock`类满足BasicLockable要求，模板参数`Mutex`满足BasicLocable要求。如果`Mutex`满足Lockable要求，`std::unique_lock`也满足Lockable要求。如果`Mutex`满足 [TimedLockable](https://en.cppreference.com/w/cpp/named_req/TimedLockable) 要求，则`std::unique_lock`也满足TimedLockable要求。

成员函数：

* `(constructor)`：构造一个std::unique_lock，可选择锁定传入的互斥量。
* `(destructor)`:析构对象，同时解锁关联的互斥量（如果拥有的话）。
* `operator=`：释放管理的互斥量的所有权（如果已拥有），并获取另一个互斥量的所有权。

* `lock`：锁定管理的互斥量。
* `try_lock`：尝试在不阻塞的情况下锁定管理的互斥量。
* `try_lock_for`：尝试锁定管理的TimedLockable互斥量，如果该互斥量在指定的持续时间内不可用则返回。
* `try_lock_until`：尝试锁定管理的TimedLockable互斥体，如果互斥体在达到指定时间点之前不可用则返回。
* `unlock`：解锁管理的互斥量。

* `swap`：与另一个std::unique_lock交换状态。
* `release`：返回管理的互斥量的指针，并释放对互斥量的所有权，但不改变互斥量的锁状态。
* `mutex`：返回指向管理的互斥量的指针。
* `owns_lock`：测试锁是否拥有（即已锁定）其管理的互斥量。

和`std::lock_guard`类似，`std::unique_lock`也有两个构造函数，第2个构造函数有两个参数，其中第二个参数类型为`std::adopt_lock()`，这个构造函数假定：当前线程已经对传入对象进行锁定，所以不需要在构造函数中再调用它的`lock()`函数，即传入的互斥量已经被锁定。

`std::unique_lock`类是可移动的，因此可以使用`std::move()`转移所有权。但不可复制。

```C++
std::mutex mutex_;

std::unique_lock<std::mutex> uLock_(mutex_);
// 把 mutex_和 uLock_绑定在了一起，即 uLock_拥有 mutex_的所有权。
// uLock_可以把自己对 mutex_的所有权转移，但是不能复制。

std::unique_lock<std::mutex> uLock2_(std::move(uLock_));
//现在 uLock2_拥有 mutex_的所有权。
```

`std::unique_lock`与`std::lock_guard`都能实现自动加锁和解锁，但是前者更加灵活，能实现更多的功能。`std::unique_lock`可以进行临时解锁和再上锁，如在构造对象之后调用成员函数`unlock()`就可以进行解锁， 再调用成员函数`lock()`进行上锁，而不必等到析构时自动解锁。`std::lock_guard`是不支持手动释放的。

## 3 多个mutex

假如程序中有两个互斥量`M1`和`M2`，两个线程`T1`和`T2`，两个线程都目的都是获取两个锁的所有权。假设此时`T1`获得`M1`的所有权，`T2`获得`M2`的所有权，`T1`在等待`M2`并占据`M1`，`T2`在等待`M1`并占据`M2`。这种情况就会产生死锁（`ABBA`死锁）。因此需要一种安全的机制对多个互斥量进行上锁。

### 3.1 std::lock()模板函数

```C++
template<class Lockable Lock1, class Lock2, class... LockN>
void lock(Lock1& lock1, Lock2& lock2,LockN&... lockN);
```

该函数一次锁定多个互斥量，用于处理多个互斥量。使用死锁避免算法锁定给定的对象，如果有一个没锁住，就会把已经锁住的释放掉，然后等待，直到所有互斥量都可以同时锁住，才继续执行。（要么互斥量都锁住，要么都没锁住，防止死锁）

`std::scoped_lock`为此函数提供了RAII包装器，通常优于对`std::lock`的裸调用。

## 3.2 scoped_lock

```C++
template<class... MutexTypes>
class scoped_lock;

std::mutex mutex_;
std::lock_scoped<std::mutex> lock_(mutex_);
```

`scoped_lock`类型是一个互斥锁包装器，它提供一种方便的`RAII`样式机制，用于在作用域块的持续时间内拥有**零个或多个**互斥锁。在C++17中添加，其工作原理与`std::lock_guard`和`std::unique_lock`一样：其构造函数会进行上锁操作，并且析构函数会对互斥量进行解锁操作。`std::scoped_lock`特别之处是，可以指定多个互斥量。可以和`condition_variable`一起使用控制线程同步。`scoped_lock`是不可复制的。

当创建`scoped_lock`对象时，其构造函数尝试获取传入的互斥锁的所有权，当`scoped_lock`对象超出作用域范围时，`scoped_lock`的析构函数释放互斥锁。如果给出多个互斥量，则使用死锁避免算法，就像`std::lock`一样。

模板参数`MutexTypes`是要锁定的互斥体类型，该类型必须满足 [Lockable](https://en.cppreference.com/w/cpp/named_req/Lockable) 要求，除非 `sizeof...(MutexTypes)==1`，在这种情况下，唯一的类型必须满足[BasicLockable](https://en.cppreference.com/w/cpp/named_req/BasicLockable) 要求。

`std::scoped_lock`使用`std::lock`函数，其会调用一个特殊的算法对所提供的互斥量调用t`ry_lock`函数，这是为了避免死锁。因此，在加锁与解锁的顺序相同的情况下，使用`std::scoped_lock`或对同一组锁调用`std::lock`都是非常安全的。
