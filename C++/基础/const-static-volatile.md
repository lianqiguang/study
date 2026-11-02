# const / static / volatile

> 一句话结论：`const` 是只读契约，`static` 控制生命周期和链接，`volatile` 只告诉编译器「不要优化这次访问」；三者都不是线程同步工具。

## 为什么考 / 怎么考

这三个关键字几乎每场 C++ 面试都会出现，区别在于问法：`const` 考指针写法和 const 正确性，`static` 考初始化顺序和链接属性，`volatile` 专门用来抓「以为它能保证线程安全」的人。典型追问链：

1. `const int*`、`int* const`、`const int* const` 有什么区别？
2. `const` 成员函数里能修改成员吗？
3. 局部 `static` 变量什么时候初始化？线程安全吗？
4. 静态初始化顺序问题是什么？
5. `volatile` 能保证线程安全吗？

## 原理

### 1. const：只读契约

`const` 变量必须初始化，之后不能通过该名字修改。它修饰的是「通过这个接口不能改」，不保证底层对象永远不变（可能有别的非 const 路径）。

三种指针写法要分清：

```cpp
#include <iostream>

int main() {
    int a = 1, b = 2;

    const int* p1 = &a;        // 指向 const int：不能通过 p1 改 a
    int* const p2 = &a;        // const 指针：不能指向别处
    const int* const p3 = &a;  // 两者都不能改

    // *p1 = 3;   // 错误
    // p2 = &b;   // 错误
    std::cout << *p1 << *p2 << *p3 << '\n';
}
```

- `const` 成员函数：`this` 的类型变成 `const T*`，不能修改普通成员；`mutable` 成员例外。
- `const` 引用可以绑定临时对象，并延长其生命周期到引用作用域结束。
- 成员函数可以按 const 重载，编译器根据对象的 const 属性选择版本。

```cpp
#include <iostream>

class Counter {
public:
    int get() const { return value_; }
    void bump() const { ++calls_; }   // mutable 允许在 const 函数中修改
private:
    int value_ = 0;
    mutable int calls_ = 0;
};
```

### 2. const 与 constexpr

- `const`：运行期或编译期的只读，不保证能在编译期求值；
- `constexpr`：要求能在编译期求值（满足条件时），可用于数组大小、模板参数、`static_assert`。

```cpp
#include <cstdlib>
#include <iostream>

constexpr int square(int x) { return x * x; }
static_assert(square(4) == 16);   // 编译期求值

int main() {
    const int runtime = std::rand();
    constexpr int compile_time = square(5);
    std::cout << runtime << ' ' << compile_time << '\n';
}
```

### 3. static：生命周期与链接

`static` 在不同位置含义不同：

| 位置 | 含义 |
|---|---|
| 函数内局部变量 | 生命周期延长到程序结束，只初始化一次 |
| 全局/命名空间作用域 | 内部链接，只在当前翻译单元可见 |
| 类静态数据成员 | 所有对象共享一份，C++17 起可 `inline static` 就地定义 |
| 类静态成员函数 | 没有 `this`，只能访问静态成员 |

局部 `static` 的初始化在 C++11 起是线程安全的（编译器保证只执行一次）：

```cpp
#include <iostream>

int& counter() {
    static int value = 0;   // C++11 起初始化线程安全
    return value;
}

int main() {
    ++counter();
    std::cout << counter() << '\n'; // 1
}
```

类静态成员：

```cpp
#include <iostream>

struct Widget {
    static int count;
    Widget() { ++count; }
};

int Widget::count = 0;   // C++17 前需要类外定义

int main() {
    Widget a, b;
    std::cout << Widget::count << '\n'; // 2
}
```

### 4. 静态初始化顺序问题

不同翻译单元里的全局/静态对象，初始化顺序未定义。一个 TU 的全局对象在构造时使用另一个 TU 的全局对象，可能读到未初始化的值（static initialization order fiasco）。

常见解法是「首次使用时构造」：

```cpp
#include <iostream>

struct Config {
    int value = 42;
};

Config& config() {
    static Config instance;   // 第一次调用时构造
    return instance;
}

int main() {
    std::cout << config().value << '\n';
}
```

这也是 Meyers 单例的基础，细节见设计模式篇。

### 5. volatile：只针对「可观察的访问」

`volatile` 告诉编译器：这个对象的读写可能被程序之外的因素改变，不要缓存、不要优化掉、不要随意重排这些访问。典型用途是内存映射 I/O 和信号处理标志：

```cpp
#include <cstdint>

volatile std::uint32_t* reg =
    reinterpret_cast<volatile std::uint32_t*>(0x40000000);

void writeReg(std::uint32_t v) {
    *reg = v;   // 每次都必须真正写内存
}
```

但 `volatile` **不提供**原子性、内存序或线程间同步。两个线程用 `volatile int` 做计数器仍然有数据竞争。线程同步要用 `std::atomic`、`mutex`、条件变量。

### 6. 三者的关系

- `const` 解决「能不能改」；
- `static` 解决「存在多久、在哪个 TU 可见」；
- `volatile` 解决「编译器能不能优化这次访问」。

它们可以组合，例如 `static const int`、`volatile const`，但语义各自独立。

## 高频追问

**Q1：`const` 和 `constexpr` 有什么区别？**
A：`const` 只保证只读，可以是运行期初始化；`constexpr` 要求能在编译期求值（在满足常量表达式条件时），可用于编译期上下文。

**Q2：`const` 成员函数里能修改成员吗？**
A：普通成员不能；被 `mutable` 修饰的成员可以。`const` 成员函数承诺不改变对象的可观察状态，`mutable` 用于缓存、计数等不影响逻辑状态的成员。

**Q3：局部 `static` 变量什么时候初始化？线程安全吗？**
A：第一次执行到声明处时初始化，生命周期到程序结束。C++11 起标准保证初始化是线程安全的（只执行一次）。

**Q4：静态初始化顺序问题是什么？怎么解决？**
A：不同 TU 的全局/静态对象初始化顺序未定义，跨 TU 依赖可能读到未初始化对象。用「首次使用时构造」（函数内局部 static）或显式初始化函数解决。

**Q5：`volatile` 能保证线程安全吗？**
A：不能。它只阻止编译器优化访问，不提供原子性、可见性或顺序保证。线程间共享数据要用 `std::atomic` 或互斥量。

**Q6：`const int*`、`int* const`、`const int* const` 怎么记？**
A：从右往左读：`const int*` 是「指向 const int 的指针」；`int* const` 是「const 指针，指向 int」；`const int* const` 两者都 const。

**Q7：类静态成员在 C++17 前后有什么区别？**
A：C++17 前通常要在类外提供定义（`int Widget::count = 0;`）；C++17 起可以用 `inline static int count = 0;` 在类内定义，避免 ODR 问题。

## 易错点与加分点

- 把 `volatile` 当线程同步，是中级面试最常见的扣分点。
- 忘记 `const` 成员函数中 `mutable` 的用途。
- 混淆 `const` 和 `constexpr`，说「const 就是编译期常量」。
- 全局静态对象跨 TU 互相依赖，导致初始化顺序问题。
- 以为 `static` 全局变量只影响「存储位置」，忽略内部链接。
- 能指出 `volatile` 在 Java/C# 中的语义与 C++ 不同，C++ 中它不解决并发。

## 相关文章

- [类型转换](类型转换.md)
- [线程与锁](../多线程/线程与锁.md)
- [单例模式](../../设计模式/单例模式.md)
- [C++ 核心面试笔记索引](../README.md)
