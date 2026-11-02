# new/delete 与 malloc/free

> 一句话结论：`new`/`delete` 是运算符表达式，负责「分配内存 + 构造/析构对象」，失败抛 `bad_alloc`；`malloc`/`free` 只管理原始内存，不调用构造析构，失败返回空指针。两者不能混用。

## 为什么考 / 怎么考

这是 C++ 和 C 的分界线，也是内存错误的源头。中级面试常从一个「为什么 `delete` 不需要传大小」的问题切入，一路问到 `operator new` 重载、placement new、数组 cookie。典型追问链：

1. `new` 和 `malloc` 有什么区别？
2. `delete` 怎么知道要释放多少内存？
3. placement new 有什么用？
4. `new[]` 为什么必须配 `delete[]`？
5. 可以重载 `operator new` 吗？

## 原理

### 1. new/delete 表达式做了什么

`new T(args)` 大致分两步：

1. 调用 `operator new(sizeof(T))` 分配原始内存；
2. 在这块内存上调用 `T` 的构造函数。

`delete p` 的顺序相反：先调用析构函数，再调用 `operator delete(p)` 释放内存。

- 分配失败时，默认抛 `std::bad_alloc`；
- 使用 `new (std::nothrow) T` 时失败返回 `nullptr`；
- 数组形式 `new T[n]` 调用 `operator new[]`，`delete[]` 对每个元素调用析构，再释放内存。

### 2. malloc/free 只管理内存

`malloc(size)` 返回 `void*`，内存内容未初始化，不调用任何构造函数；失败返回 `nullptr`，需要手动检查。`free(p)` 只释放内存，不调用析构函数。`realloc` 可能搬移内存，原指针失效。

```cpp
#include <cstdlib>

struct T {
    T() {}
    ~T() {}
};

int main() {
    void* raw = std::malloc(sizeof(T)); // 只有内存，没有 T 对象
    std::free(raw);

    T* obj = new T;                     // 分配 + 构造
    delete obj;                         // 析构 + 释放
}
```

### 3. 核心区别

| 维度 | `new` / `delete` | `malloc` / `free` |
|---|---|---|
| 是否调用构造/析构 | 是 | 否 |
| 返回类型 | `T*`（类型安全） | `void*`（需转换） |
| 失败行为 | 抛 `bad_alloc`（或用 nothrow 返回空） | 返回 `nullptr` |
| 大小 | 编译器推导 | 手动传字节数 |
| 能否重载 | 可以重载 `operator new/delete` | 不能 |
| 数组 | `new[]` / `delete[]` | 自己算总字节数 |

### 4. 数组与 delete[]

`new[]` 和 `delete[]` 必须配对。数组形式可能在内存块头部存放元素个数（cookie，实现相关），`delete[]` 依靠它决定调用多少次析构函数。用 `delete` 释放 `new[]` 的结果是未定义行为，反之亦然。

```cpp
#include <iostream>

struct T {
    T() { std::cout << "ctor\n"; }
    ~T() { std::cout << "dtor\n"; }
};

int main() {
    T* arr = new T[3];   // 3 次构造
    delete[] arr;        // 3 次析构
}
```

### 5. placement new：在已有内存上构造

placement new 不分配内存，只在给定地址上调用构造函数，常用于内存池、容器实现、嵌入式固定缓冲区：

```cpp
#include <iostream>
#include <new>

struct P {
    int x;
    explicit P(int v) : x(v) {}
    ~P() { std::cout << "~P\n"; }
};

int main() {
    alignas(P) unsigned char buf[sizeof(P)];
    P* p = new (buf) P(42);   // 在 buf 上构造
    std::cout << p->x << '\n';
    p->~P();                  // 必须显式析构，不能 delete
}
```

注意：placement new 构造出来的对象**不能**用 `delete` 释放，因为内存不是 `operator new` 分配的；必须显式调用析构函数。

### 6. 重载 operator new/delete

可以全局或按类重载 `operator new` / `operator delete`，用于内存池、统计、对齐分配：

```cpp
#include <cstdlib>
#include <iostream>
#include <new>

struct Widget {
    static void* operator new(std::size_t n) {
        std::cout << "custom new " << n << '\n';
        return std::malloc(n);
    }
    static void operator delete(void* p) {
        std::free(p);
    }
    int x = 0;
};

int main() {
    Widget* w = new Widget;
    delete w;
}
```

类内重载只影响该类的 `new` 表达式，不影响 `new[]`（需要重载 `operator new[]`）和 placement new。

### 7. nothrow new

```cpp
#include <iostream>
#include <new>

int main() {
    int* p = new (std::nothrow) int[1000000000000ULL];
    if (!p) {
        std::cout << "allocation failed\n";
    }
    delete[] p;   // 对空指针 delete 是安全的
}
```

### 8. 为什么 delete void* 是错的

`delete p` 需要知道对象的动态类型才能调用正确的析构函数。如果 `p` 是 `void*`，编译器无法调用析构函数（且标准规定这是未定义行为）。同理，通过基类指针删除派生类对象时，基类析构必须是 `virtual`（见虚函数篇）。

## 高频追问

**Q1：`new` 和 `malloc` 最本质的区别是什么？**
A：`new` 在分配内存后调用构造函数，`delete` 在释放前调用析构函数；`malloc/free` 只管理原始内存，不涉及对象生命周期。

**Q2：`delete` 怎么知道要释放多少字节？**
A：释放函数 `operator delete(void*)` 本身不需要大小；分配器内部记录了块大小（通常在实际地址前方）。数组形式还会额外存元素个数 cookie，供 `delete[]` 调用相应次数的析构。

**Q3：placement new 的典型用途？**
A：在已分配的内存上构造对象，例如内存池、`std::vector` 的原地构造、嵌入式固定缓冲区；对象生命周期结束后要显式调用析构函数。

**Q4：`new[]` 配 `delete` 会怎样？**
A：未定义行为。可能只析构第一个元素，也可能因 cookie 解析错误导致堆损坏。必须 `new[]` 配 `delete[]`。

**Q5：可以重载 `operator new` 吗？**
A：可以，分全局重载和类内重载。重载的是分配函数，`new` 表达式仍会负责调用构造函数；不要重载 `new` 表达式本身。

**Q6：`delete void*` 为什么危险？**
A：编译器不知道真实类型，无法调用析构函数；标准规定这是未定义行为。释放前应先转换回正确类型。

**Q7：`malloc(0)` 返回什么？**
A：实现相关，可能返回空指针，也可能返回一个可传给 `free` 的唯一指针；不应依赖其行为。

## 易错点与加分点

- `new` 配 `free`、`malloc` 配 `delete` 都是 UB，虽然有时「看起来能跑」。
- 忘记 `new[]` / `delete[]` 配对。
- placement new 后忘记显式析构，或错误地 `delete`。
- 用 `delete` 释放 `void*`，析构函数不会执行。
- 重载 `operator new` 后忘记配套的 `operator delete`。
- 能指出 RAII 和智能指针才是现代 C++ 的首选，`new/delete` 应尽量封装。

## 相关文章

- [内存对齐与内存布局](内存对齐与内存布局.md)
- [智能指针](智能指针.md)
- [C++ 核心面试笔记索引](../README.md)
