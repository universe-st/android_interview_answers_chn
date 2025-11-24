# 介绍一下智能指针

在C++中，智能指针是一种封装了裸指针（raw pointer）的类模板，其核心作用是**自动管理动态内存**，避免因手动管理内存不当导致的内存泄漏、二次释放、悬垂指针等问题。智能指针通过重载`*`和`->`运算符，使其使用方式与普通指针类似，但在生命周期结束时（如超出作用域）会自动调用析构函数释放所管理的资源。


C++标准库（C++11及以后）提供了三种主要的智能指针：`std::unique_ptr`、`std::shared_ptr`和`std::weak_ptr`，此外`std::auto_ptr`已被C++11弃用（因设计缺陷），不再推荐使用。


### 1. `std::unique_ptr`：独占所有权的智能指针
`std::unique_ptr`是**独占型智能指针**，它确保同一时间内只有一个`unique_ptr`实例管理某块内存，即**所有权不可共享**。当`unique_ptr`被销毁时（如超出作用域），其管理的内存会自动释放。


#### 核心特性：
- **独占性**：不允许拷贝（copy），但允许移动（move）。即不能通过赋值或拷贝构造函数创建新的`unique_ptr`，但可以通过`std::move`转移所有权。
- **轻量高效**：无额外的引用计数开销，性能接近裸指针。
- **支持数组**：有针对数组的特化版本（`std::unique_ptr<T[]>`），会自动调用`delete[]`释放内存。


#### 基本用法：
```cpp
#include <memory>
#include <iostream>

int main() {
    // 创建unique_ptr（推荐使用make_unique，C++14引入）
    std::unique_ptr<int> ptr1 = std::make_unique<int>(42);
    std::cout << *ptr1 << std::endl;  // 输出：42

    // 转移所有权（ptr1不再拥有内存，变为nullptr）
    std::unique_ptr<int> ptr2 = std::move(ptr1);
    if (ptr1 == nullptr) {
        std::cout << "ptr1 is null" << std::endl;  // 输出：ptr1 is null
    }

    // 管理数组（自动调用delete[]）
    std::unique_ptr<int[]> arr_ptr = std::make_unique<int[]>(3);
    arr_ptr[0] = 1;
    arr_ptr[1] = 2;

    return 0;  // 函数结束时，ptr2和arr_ptr自动释放内存
}
```


#### 适用场景：
- 管理独占资源（如动态分配的单个对象、数组）。
- 作为函数的返回值（避免返回裸指针导致的内存管理问题）。
- 作为容器元素（因支持移动语义，可放入`std::vector`等容器）。


### 2. `std::shared_ptr`：共享所有权的智能指针
`std::shared_ptr`是**共享型智能指针**，允许多个`shared_ptr`实例共同管理同一块内存。它通过**引用计数（reference count）** 机制跟踪所有者数量：当最后一个`shared_ptr`被销毁时，内存才会被释放。


#### 核心特性：
- **共享性**：支持拷贝和赋值，每次拷贝或赋值都会使引用计数加1；每次析构或重置时，引用计数减1。
- **引用计数**：内部维护一个计数器，记录当前管理该内存的`shared_ptr`数量。当计数为0时，自动释放内存。
- **线程安全**：引用计数的增减操作是线程安全的，但对所管理对象的访问需要额外同步。


#### 基本用法：
```cpp
#include <memory>
#include <iostream>

int main() {
    // 创建shared_ptr（推荐使用make_shared，效率更高）
    std::shared_ptr<int> ptr1 = std::make_shared<int>(100);
    std::cout << "引用计数：" << ptr1.use_count() << std::endl;  // 输出：1

    // 拷贝 ptr1，引用计数加1
    std::shared_ptr<int> ptr2 = ptr1;
    std::cout << "引用计数：" << ptr1.use_count() << std::endl;  // 输出：2

    // 重置 ptr1（引用计数减1）
    ptr1.reset();
    std::cout << "引用计数：" << ptr2.use_count() << std::endl;  // 输出：1

    return 0;  // ptr2销毁，引用计数变为0，内存释放
}
```


#### 注意事项：
- **避免循环引用**：若两个`shared_ptr`相互引用（如A持有B的`shared_ptr`，B持有A的`shared_ptr`），会导致引用计数永远不为0，内存泄漏。此时需配合`std::weak_ptr`解决。
- **自定义删除器**：默认使用`delete`释放内存，若管理的资源需要特殊释放方式（如文件句柄、网络连接），可指定自定义删除器：
  ```cpp
  // 自定义删除器（例如释放数组）
  std::shared_ptr<int> arr_ptr(new int[3], [](int* p) { delete[] p; });
  ```


#### 适用场景：
- 多所有者共享同一资源（如多个对象需要访问同一个动态分配的对象）。
- 无法确定哪个所有者最后释放资源的场景。


### 3. `std::weak_ptr`：弱引用智能指针
`std::weak_ptr`是一种**弱引用**智能指针，它不拥有对资源的所有权，仅作为`shared_ptr`的“观察者”。它不会增加`shared_ptr`的引用计数，因此无法直接访问资源，必须通过`lock()`方法转换为`shared_ptr`后才能使用。


#### 核心特性：
- **不影响引用计数**：创建`weak_ptr`时，`shared_ptr`的引用计数不变。
- **解决循环引用**：通过弱引用打破`shared_ptr`之间的循环依赖。
- **检查有效性**：可通过`expired()`方法判断所观察的资源是否已释放。


#### 基本用法：
```cpp
#include <memory>
#include <iostream>

int main() {
    std::shared_ptr<int> shared = std::make_shared<int>(200);
    std::weak_ptr<int> weak = shared;  // 弱引用shared管理的资源

    std::cout << "shared引用计数：" << shared.use_count() << std::endl;  // 输出：1
    std::cout << "weak是否有效：" << !weak.expired() << std::endl;        // 输出：1（true）

    // 通过lock()获取shared_ptr（若资源有效）
    if (auto temp = weak.lock()) {
        std::cout << "资源值：" << *temp << std::endl;  // 输出：200
    }

    // 释放shared管理的资源
    shared.reset();
    std::cout << "weak是否有效：" << !weak.expired() << std::endl;  // 输出：0（false）

    return 0;
}
```


#### 解决循环引用示例：
```cpp
#include <memory>

class B;  // 前向声明

class A {
public:
    std::shared_ptr<B> b_ptr;  // A持有B的shared_ptr
    ~A() { std::cout << "A被销毁" << std::endl; }
};

class B {
public:
    std::weak_ptr<A> a_ptr;    // B持有A的weak_ptr（而非shared_ptr）
    ~B() { std::cout << "B被销毁" << std::endl; }
};

int main() {
    {
        std::shared_ptr<A> a = std::make_shared<A>();
        std::shared_ptr<B> b = std::make_shared<B>();
        a->b_ptr = b;  // A引用B
        b->a_ptr = a;  // B弱引用A（不增加a的引用计数）
    }  // 离开作用域，a和b的引用计数均为0，A和B正常销毁

    return 0;
}
```
输出：
```
A被销毁
B被销毁
```


#### 适用场景：
- 打破`shared_ptr`的循环引用。
- 缓存场景（观察资源是否存在，避免缓存导致资源无法释放）。
- 观察者模式（观察者持有被观察者的弱引用，避免被观察者无法销毁）。


### 智能指针使用的最佳实践
1. **优先使用`make_unique`和`make_shared`**：这两个函数能避免裸指针暴露，且`make_shared`能一次性分配对象和引用计数的内存，效率更高。
2. **避免混合使用智能指针和裸指针**：不要将裸指针交给多个智能指针管理，也不要用智能指针管理栈上的对象。
3. **不使用`get()`返回的指针手动释放资源**：`get()`仅用于临时访问裸指针，若手动`delete`会导致二次释放。
4. **管理this指针时使用`enable_shared_from_this`**：若需要在类内部返回自身的`shared_ptr`，应让类继承`std::enable_shared_from_this`，并通过`shared_from_this()`获取，而非直接用`this`初始化。


### 总结
C++智能指针通过自动化内存管理，极大降低了内存泄漏的风险：
- `unique_ptr`：独占资源，轻量高效，适合大多数场景。
- `shared_ptr`：共享资源，通过引用计数管理，需注意循环引用。
- `weak_ptr`：配合`shared_ptr`使用，解决循环引用，不拥有资源所有权。

在现代C++开发中，应尽量使用智能指针替代裸指针，以编写更安全、可靠的代码。