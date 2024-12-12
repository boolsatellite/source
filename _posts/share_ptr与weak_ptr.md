---
title: share_ptr和weak_ptr
date: 2024-1-15
tags:
- c++
---

# share_ptr和weak_ptr

## share_ptr

### 内存模型

shared_ptr 内部包含两个指针，一个指向对象，另一个指向控制块(control block)，控制块中包含一个引用计数(reference count), 一个弱计数(weak count)和其它一些数据。

![](/img/share_ptr内存模型.png)

当执行：```share_ptr<int> p1(new int(1));    share_ptr<int> p2 = p1``` 时对应的share_ptr结构中指针将指向同一个对象以及控制块

![](/img/share_ptr赋值.png)

std::shared_ptr使用引用计数，每一个shared_ptr的拷贝都指向相同的内存。再最后一个shared_ptr析构的时候，内存才会被释放。

share_ptr的线程安全问题： 

> **引用计数是线程安全的**，引用计数使用了原子类型，指向对象数据，如果发生修改，将不是线程安全的，若要数据安全，要在对象数据访问上增加锁机制保证对象的数据安全

### 基本用法，常用函数

只能通过复制构造或复制赋值其值给另一 `shared_ptr` ，将对象所有权与另一 `shared_ptr` 共享。用另一 `shared_ptr` 所占有的底层指针创建新的 `shared_ptr` 导致未定义行为。

`std::shared_ptr` 可以用于不完整类型T 。然而，参数为裸指针的构造函数（ template<class Y> shared_ptr(Y * ) ）和 template<class Y> void reset(Y*) 成员函数只可以用指向完整类型的指针调用（注意 [std::unique_ptr](https://www.apiref.com/cpp-zh/cpp/memory/unique_ptr.html) 可以从指向不完整类型的裸指针构造）。

* 通过构造函数，```make_shared()``` ，```reset()```方法来初始化```shared_ptr```

```c++
std::shared_ptr<int> p1(new int(1));
std::shared_ptr<int> p2 = std::make_shared<int>(100);
std::shared_ptr<int> p3;
p3.reset(new int(1));
```

不能将一个原始指针直接赋值给一个智能指针```std::shared_ptr<int> p = new int(10)```这种写法将无法编译，shared_ptr不能通过“直接将原始这种赋值”来初始化，需要通过构造函数或辅助方法来初始化，因为```template< class Y >
explicit shared_ptr( Y* ptr );```参数类型是指针的构造函数被explict修饰(指定构造函数或转换函数 (C++11 起)或推导指引显式，**即它不能用于隐式转换和复制初始化**

* 智能指针可以通过重载的bool类型操作符来判断

提供了```explicit operator bool() const noexcept;```检查 ```*this``` 是否存储非空指针，即是否有 ```get() != nullptr```

```c++
void report(std::shared_ptr<int> ptr) {
    if(ptr) {
        std::cout << *ptr <<std::endl;
    } else {
        std::cout << "*ptr is not a valid pointer \n";
    }
}
```

* std::shared_ptr<T>::use_count，返回引用计数

返回管理当前对象的不同 `shared_ptr` 实例（包含 this ）数量。若无管理对象，则返回 0。**多线程环境下， use_count 返回的值是近似的**

```c++
void fun(std::shared_ptr<int> sp) {
    std::cout << "fun: sp.use_count() == " << sp.use_count() << '\n'; 
}
int main()  { 
    auto sp1 = std::make_shared<int>(5);
    std::cout << "sp1.use_count() == " << sp1.use_count() << '\n'; 
    //sp1.use_count() == 1
    fun(sp1);
    //fun: sp.use_count() == 2
}
```

* reset方法

对于无参数reset，```void reset() noexcept```释放被管理对象的所有权，若存在。调用后， *this 不管理对象。等价于 shared_ptr().swap( *this)

对于存在一个参数的reset，```template< class Y >void reset( Y* ptr );```以 `ptr` 所指向的对象替换被管理对象，以 delete 表达式为删除器。合法的 delete 表达式必须可用，即 delete ptr 必须为良式，拥有良好定义行为且不抛任何异常。等价于 shared_ptr<T>(ptr).swap(*this); 。

* 获取原始指针get方法

当需要获取原始指针时，可以通过get方法来返回原始指针

```c++
std::shared_ptr<int> ptr(new int{1});
int *p = ptr.get();            //获取原始指针
```

谨慎使用`p.get()`的返回值：

> 不要保存ptr.get()的返回值 ，无论是保存为裸指针还是shared_ptr都是错误的 。保存为裸指针不知什么时候就会变成空悬指针，保存为shared_ptr则产生了独立指针
> 
> 不要delete ptr.get()的返回值 ，会导致对一块内存delete两次的错误

```c++
int main() {
    int * p = new int(10);
    std::shared_ptr<int> p1(p);    
    std::shared_ptr<int> p2(p1.get());    //错误将会double free
}
```

**如上代码是错误的**，`p1.get()`返回指针类型，故调用`template< class Y > explicit shared_ptr( Y* ptr )`，对于此构造函数，只是构造 `shared_ptr` ，管理 `ptr` 所指向的对象，此构造过程并不会产生共享对象，`shared_ptr( const shared_ptr& r ) noexcept;`和`operator=`会产生共享对象

**不要将this指针作为shared_ptr返回出来**，原理同上

```cpp
class A {
public:
    shared_ptr<A> GetSelf() {
        return shared_ptr<A>(this); // 不要这么做
    }
    ~A(){ cout << "Destructor A" << endl; }
};
int main(){
    shared_ptr<A> sp1(new A);
    shared_ptr<A> sp2 = sp1->GetSelf();
    return 0;
}
```

**正确返回this的shared_ptr的做法是：让目标类通过std::enable_shared_from_this类，然后使用基类的 成员函数shared_from_this()来返回this的shared_ptr**

```c++
class A: public std::enable_shared_from_this<A>
{
public:
shared_ptr<A> GetSelf() {
    return shared_from_this(); 
}
~A(){cout << "Destructor A" << endl;}
};
int main() {
    shared_ptr<A> sp1(new A);
    shared_ptr<A> sp2 = sp1->GetSelf(); // ok
    return 0;
}
```

* 不要在函数实参中创建 `share_ptr`

如：``` function(std::shared_ptr<int> (new int(10)) , g() )```因为C++的函数参数的计算顺序在不同的编译器不同的约定下可能是不一样的，一般是从右到左，但也 可能从左到右，所以，可能的过程是先```new int```，然后调用```g()```，如果恰好```g()```发生异常，而```shared_ptr```还没有创建， 则```int```内存泄漏了，正确的写法应该是先创建智能指针

```c++
shared_ptr<int> p(new int);
function(p, g());
```

* 避免循环引用

循环引用导致ap和bp的引用计数为2，在离开作用域之后，ap和bp的引用计数减为1，并不回减为0，导致两个指针都不会被析构，产生内存泄漏，解决的办法是把A和B任何一个成员变量改为weak_ptr

```c++
class A;
class B;
class A
{
public:
    std::shared_ptr<B> bptr;
    ~A()
    {
        cout << "A is deleted" << endl;
    }
};
class B
{
public:
    std::shared_ptr<A> aptr;
    ~B()
    {
        cout << "B is deleted" << endl;
    }
};
int main()
{
    {
        std::shared_ptr<A> ap(new A);
        std::shared_ptr<B> bp(new B);
        ap->bptr = bp;
        bp->aptr = ap;
    }
    cout << "main leave" << endl; // 循环引用导致ap bp退出了作用域都没有析构
    return 0;
}
```

## weak_ptr

`weak_ptr 是一种不控制对象生命周期的智能指针, 它指向一个 `Shared_ptr` 管理的对象. 进行该对象的内存管理的是那个强引用的`shared_ptr`，` weak_ptr`只是提供了对管理对象的一个访问手段。

### 基本用法：

* 通过use_count()方法获取当前观察资源的引用计数

返回共享被管理对象所有权的 `shared_ptr` 实例数量，或 0 ，若被管理对象已被删除，即 `*this` 为空。

```c++
int main()
{
    shared_ptr<int> sp(new int(10));
    weak_ptr<int> wp = sp;
    cout << wp.use_count() <<endl;      //1
    shared_ptr<int> sp1 = sp;
    cout <<wp.use_count() << endl;      //2
}
```

* 通过expired()方法判断所观察资源是否已经释放

`bool expired() const noexcept;`等价于 `use_count() == 0` 。可能仍未对被管理对象调用析构函数，但此对象的析构已经临近（或可能已发生），若被管理对象已被删除则为 true ，否则为 false 。若被管理对象在线程间共享，则此函数内在地不可靠，通常 false 结果可能在能用之前就变得过时。 true 结果可靠。

```c++
shared_ptr<int> sp(new int(10));
weak_ptr<int> wp(sp);
if(wp.expired())
    cout << "weak_ptr无效,资源已释放";
else
    cout << "weak_ptr有效";
```

* 通过lock方法获取监视的shared_ptr

`std::shared_ptr<T> lock() const noexcept`，创建新的 `std::shared_ptr` 对象，它共享被管理对象的所有权。若无被管理对象，即 `*this` 为空，则返回亦为`nullptr`的 `shared_ptr`，等效地返回 `expired() ? shared_ptr<T>() : shared_ptr<T>(*this)` ，**原子地执行**

在多线程环境下：

线程一：

```c++
std::weak_ptr<int> gw;
gw = sp;
void f() {
    auto spt = gw.lock();
    if ( gw.expired() ) {
        cout << " gw Invalid , resource released";
    } else {
        cout << " gw Vaild , *spt = " << *spt;
    }
}
```

线程二：

```c++
std::shared_ptr<int> sp = std::make_shared<int>(10);
```

在线程二中资源有可能释放，为了保证线程一不出错，要使用weak_ptr观察，要**先上锁后检查**，根据加锁情况分类

* weak_ptr解决循环引用

```c++
class A;
class B;
class A {
public:
    std::weak_ptr<B> bptr; // 修改为weak_ptr
    ~A()
    { cout << "A is deleted" << endl;}
};
class B {
public:
    std::shared_ptr<A> aptr;
    ~B()
    { cout << "B is deleted" << endl; }
};
int main() {
    {
        std::shared_ptr<A> ap(new A);
        std::shared_ptr<B> bp(new B);
        ap->bptr = bp;
        bp->aptr = ap;
    }
    cout << "main leave" << endl;
    return 0;
}
```

这样在对B的成员赋值时，即执行`bp->aptr=ap;`时，由于`aptr`是`weak_ptr`，它并不会增加引用计数，所以ap的引用计数仍然会是1，在离开作用域之后，ap的引用计数为减为0，A指针会被析构，析构后其内部的`bptr`的引用计数会被减为1，然后在离开作用域后`bp`引用计数又从1减为0，B对象也被析构，不会发生内存泄漏

https://www.cnblogs.com/Solstice/archive/2013/01/28/2879366.html    多线程环境下share_ptr写时加锁原因
