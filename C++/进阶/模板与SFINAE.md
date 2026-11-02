# 模板与 SFINAE

> 一句话结论：模板让同一份代码适配不同类型；SFINAE 的意思是「模板参数替换失败不算错误，只是把这个候选从重载集中去掉」，它只在推导的立即上下文中生效。

## 为什么考 / 怎么考

模板是 C++ 泛型编程的核心，中级面试通常不会让你手写一个完整的 traits 库，而是考你能否解释特化、SFINAE、`enable_if` 和 `if constexpr` 的关系。典型追问链：

1. 函数模板怎么推导类型？
2. 全特化和偏特化有什么区别？
3. SFINAE 是什么？为什么叫「替换失败不是错误」？
4. `enable_if` 怎么用？
5. `if constexpr` 能替代 SFINAE 吗？

## 原理

### 1. 函数模板与类型推导

函数模板根据实参推导模板参数；也可以显式指定：

```cpp
#include <iostream>

template <typename T>
T maxv(T a, T b) {
    return a > b ? a : b;
}

int main() {
    std::cout << maxv(3, 5) << ' ' << maxv<double>(1, 2.5) << '\n';
}
```

注意：`maxv(1, 2.5)` 无法推导出唯一的 `T`（`int` 与 `double` 冲突），需要显式指定或写两个模板参数。

### 2. 类模板与特化

- **全特化**：为某一组具体类型提供完全不同的实现；
- **偏特化**：为一类类型（如所有指针 `T*`）提供实现；函数模板不支持偏特化，只能用重载。

```cpp
template <typename T>
struct Box { static constexpr int kind = 0; };

template <typename T>
struct Box<T*> { static constexpr int kind = 1; };   // 偏特化

template <>
struct Box<int> { static constexpr int kind = 2; };  // 全特化

static_assert(Box<double>::kind == 0);
static_assert(Box<int*>::kind == 1);
static_assert(Box<int>::kind == 2);
```

匹配时优先选择最特化的版本：`Box<int*>` 用偏特化，`Box<int>` 用全特化。

### 3. 非类型模板参数

模板参数可以是整数、枚举、指针等编译期常量：

```cpp
#include <array>

template <typename T, int N>
struct FixedArray {
    std::array<T, N> data;
};

int main() {
    FixedArray<int, 3> a{};
    (void)a;
}
```

### 4. 可变参数模板

`typename... Ts` 是参数包，`sizeof...(Ts)` 得到包中元素个数，折叠表达式可以展开：

```cpp
#include <cstddef>
#include <iostream>

template <typename... Ts>
std::size_t count(Ts...) {
    return sizeof...(Ts);
}

template <typename... Ts>
auto sum(Ts... xs) {
    return (xs + ...);   // C++17 折叠表达式
}

int main() {
    std::cout << count(1, 2.0, "x") << ' ' << sum(1, 2, 3) << '\n';
}
```

空包时 `(xs + ...)` 不合法，而 `(xs && ...)` 为 `true`、`(xs || ...)` 为 `false`。

### 5. SFINAE

SFINAE = Substitution Failure Is Not An Error。编译器在推导模板参数、把实参代入模板签名时，如果某个候选的替换失败（例如 `T::value_type` 不存在），不会报错，只是把这个候选从重载集中移除。

关键限制：**只发生在推导的立即上下文**（函数签名、模板参数列表）。如果错误发生在函数体内部（模板实例化之后），就是硬错误。

```cpp
#include <type_traits>

template <typename T>
std::enable_if_t<std::is_integral_v<T>, T> twice(T x) {
    return x * 2;
}

int main() {
    twice(3);
    // twice(2.5);   // 没有匹配的重载：double 不满足 is_integral
}
```

`enable_if<cond, T>` 在 `cond` 为真时提供 `T`，否则不提供 `type`，从而导致替换失败。常见写法：

- 返回类型：`std::enable_if_t<cond, T>`
- 默认模板参数：`template <typename T, typename = std::enable_if_t<cond>>`
- 非类型模板参数：`std::enable_if_t<cond, int> = 0`

### 6. void_t 检测惯用法

`std::void_t<Ts...>` 把任意类型映射为 `void`，配合偏特化可以检测某个表达式是否合法：

```cpp
#include <type_traits>
#include <utility>
#include <vector>

template <typename T, typename = void>
struct has_size : std::false_type {};

template <typename T>
struct has_size<T, std::void_t<decltype(std::declval<T>().size())>> : std::true_type {};

static_assert(has_size<std::vector<int>>::value);
static_assert(!has_size<int>::value);
```

### 7. if constexpr

C++17 的 `if constexpr` 在模板中按编译期条件选择分支，未选中的分支不会被实例化，很多场景可以替代 SFINAE：

```cpp
#include <iostream>
#include <type_traits>

template <typename T>
void show(T v) {
    if constexpr (std::is_integral_v<T>) {
        std::cout << "integer: " << v << '\n';
    } else {
        std::cout << "other\n";
    }
}

int main() {
    show(1);
    show(1.5);
}
```

注意：只有在模板中、且分支依赖模板参数时，未选中的分支才不会被实例化；非模板代码里的 `if constexpr` 分支仍要能通过语法检查。

### 8. 依赖名与 typename

在模板里，依赖模板参数的名字默认被当作非类型；如果它是类型，要加 `typename`：

```cpp
template <typename T>
void use(typename T::value_type v) {
    (void)v;
}
```

### 9. 实例化与代码膨胀

模板在编译期按需实例化，不同参数会生成不同代码，可能导致二进制膨胀。缓解手段：显式实例化、把实现移出模板、用类型擦除（如 `std::function`）。C++20 的 concepts 把约束写在签名上，错误信息更清晰，也更容易表达重载意图。

## 高频追问

**Q1：函数模板和普通函数重载怎么选？**
A：逻辑相同、只是类型不同用模板；需要完全不同的实现或需要隐式转换时用重载。模板实例化后也会参与重载决议。

**Q2：SFINAE 是什么？**
A：模板参数替换失败不算编译错误，只是把该候选从重载集中移除；它只作用于推导的立即上下文，函数体里的错误仍是硬错误。

**Q3：`enable_if` 怎么用？**
A：放在返回类型、默认模板参数或非类型模板参数上，条件为真时提供类型，否则让替换失败。常用于按类型约束重载。

**Q4：全特化和偏特化有什么区别？**
A：全特化指定所有模板参数的具体类型；偏特化只指定一部分模式（如 `T*`），仍保留模板参数。函数模板不支持偏特化。

**Q5：可变参数模板怎么展开参数包？**
A：可以用 `sizeof...(Ts)` 计数、递归处理首元素，或 C++17 折叠表达式 `(xs + ...)`。空包要按运算符选择合法形式。

**Q6：`if constexpr` 能替代 SFINAE 吗？**
A：很多场景可以，尤其是按类型选择实现；但 `if constexpr` 不能参与重载决议，需要「根据类型启用/禁用某个重载」时仍要用 SFINAE 或 concepts。

**Q7：模板为什么容易导致代码膨胀？**
A：每个不同的模板实参都会实例化一份代码；可以通过显式实例化、减少不必要的模板参数、类型擦除来控制。

## 易错点与加分点

- 把 SFINAE 说成「任何模板错误都会被忽略」——只限立即上下文。
- 混淆全特化和偏特化，或以为函数模板可以偏特化。
- 忘记 `typename` 标注依赖类型名。
- 在非模板代码里依赖 `if constexpr` 丢弃非法分支（仍会检查）。
- 空参数包直接写 `(xs + ...)`，编译失败。
- 能说出 concepts 相对 SFINAE 的可读性优势。

## 相关文章

- [C++11–17 高频特性](../新特性/C++11-17高频特性.md)
- [类型转换](../基础/类型转换.md)
- [C++ 核心面试笔记索引](../README.md)
