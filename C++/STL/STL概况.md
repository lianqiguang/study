# C++ STL 概况

> 一句话结论：STL 由容器、算法、迭代器、仿函数、适配器、分配器六大组件构成；选容器先看访问模式（随机访问 / 有序 / 哈希 / 两端），再看插入删除的频繁程度。

## 为什么考 / 怎么考

STL 是 C++ 面试的「基本功」，中级岗位不会只问「vector 怎么用」，而是问容器底层、复杂度、失效规则和选型。典型追问链：

1. vector、list、deque 分别适合什么场景？
2. map 和 unordered_map 怎么选？
3. stack 和 queue 的底层是什么？
4. vector<bool> 为什么特殊？
5. 各容器的查找、插入、删除复杂度是多少？

## 原理

### 1. STL 六大组件

| 组件 | 作用 | 例子 |
|---|---|---|
| 容器（Container） | 管理元素集合 | `vector`、`map`、`list` |
| 算法（Algorithm） | 对区间施加操作 | `sort`、`find`、`copy` |
| 迭代器（Iterator） | 算法与容器之间的桥梁 | `begin()`、`end()` |
| 仿函数（Function Object） | 可调用对象，作为算法的策略参数 | `std::less`、lambda |
| 适配器（Adapter） | 改变接口或行为 | `stack`、`queue`、`priority_queue`、`reverse_iterator` |
| 分配器（Allocator） | 负责内存分配与释放 | `std::allocator` |

算法通过迭代器操作容器，不依赖具体容器类型，这是 STL 泛型设计的核心。

### 2. 常用容器速览

| 容器 | 类别 | 底层结构 | 一句话说明 |
|---|---|---|---|
| `vector` | 序列容器 | 连续动态数组 | 随机访问 O(1)，尾部增删均摊 O(1)，扩容会搬移全部元素 |
| `stack` | 容器适配器 | 默认 `deque` | 后进先出（LIFO），只暴露 `push/pop/top` |
| `queue` | 容器适配器 | 默认 `deque` | 先进先出（FIFO），只暴露 `push/pop/front/back` |
| `deque` | 序列容器 | 分段连续内存 + 中控数组 | 两端插入删除 O(1)，随机访问 O(1)，中间操作 O(n) |
| `priority_queue` | 容器适配器 | 默认 `vector` + 堆算法 | 默认大顶堆，`top()` 取最大元素，插入/删除 O(log n) |
| `map` | 有序关联容器 | 红黑树 | 键有序，查找/插入/删除 O(log n)，迭代器稳定 |
| `set` | 有序关联容器 | 红黑树 | 有序去重集合，操作复杂度同 `map` |
| `pair` | 值类型（非容器） | 两个成员的结构体 | 常用于 `map` 的 `value_type`，配合 `make_pair` 使用 |
| `string` | 序列容器（特化） | 连续字符存储 | 支持短字符串优化（SSO），`c_str()` 返回 C 风格字符串 |
| `bitset` | 位集 | 定长位数组 | 编译期确定长度，按位操作，节省空间 |
| `array` | 序列容器 | 定长连续数组 | 栈上定长数组，保留 `size()`、迭代器等 STL 接口 |
| `tuple` | 值类型（非容器） | 异构定长聚合 | 可存放多个不同类型元素，配合 `std::get` / 结构化绑定 |

补充常用容器：

- `list`：双向链表，任意位置插入删除 O(1)，不支持随机访问，缓存不友好。
- `unordered_map` / `unordered_set`：哈希表，平均 O(1) 查找，元素无序，rehash 会使迭代器失效。
- `multimap` / `multiset`：允许重复键的有序关联容器。

### 3. 容器选择决策

| 需求 | 首选 |
|---|---|
| 随机访问 + 尾部增删 | `vector` |
| 频繁在中间插入/删除，且已持有迭代器 | `list` |
| 两端都要进出 | `deque` |
| 需要有序遍历或范围查询 | `map` / `set` |
| 只要求平均 O(1) 查找，不要求有序 | `unordered_map` / `unordered_set` |
| LIFO / FIFO / 按优先级取最大最小 | `stack` / `queue` / `priority_queue` |
| 长度固定、追求零额外开销 | `array` |
| 大量布尔标志 | `bitset`（或 `vector<char>`） |

默认优先 `vector`：连续内存对缓存友好，绝大多数场景比链表快，即使中间插入是 O(n)。

### 4. 时间复杂度对照

| 容器 | 查找 | 插入 | 删除 | 随机访问 |
|---|---|---|---|---|
| `vector` | O(n) | 尾部均摊 O(1)，中间 O(n) | 尾部 O(1)，中间 O(n) | O(1) |
| `deque` | O(n) | 两端 O(1)，中间 O(n) | 两端 O(1)，中间 O(n) | O(1) |
| `list` | O(n) | O(1)（已知位置） | O(1)（已知位置） | 不支持 |
| `map` / `set` | O(log n) | O(log n) | O(log n) | 不支持（迭代器有序） |
| `unordered_map` / `unordered_set` | 平均 O(1)，最坏 O(n) | 平均 O(1)，最坏 O(n) | 平均 O(1)，最坏 O(n) | 不支持 |
| `stack` / `queue` | — | O(1) | O(1) | 只能访问端部 |
| `priority_queue` | O(1) 取堆顶 | O(log n) | O(log n) | 不支持 |

### 5. 常见坑

- **`vector<bool>` 是特化**：按位存储，`operator[]` 返回代理对象而不是 `bool&`，不能取地址，行为与普通容器不同。
- **`map::operator[]` 会插入默认值**：只查询时用 `find()` 或 `at()`。
- **`priority_queue` 默认是大顶堆**：要小顶堆需指定 `std::greater<T>`。
- **`stack` / `queue` 是适配器**：不是独立容器。`stack` 默认底层是 `deque`，可以换 `vector` 或 `list`；`queue` 默认 `deque`，需要 `pop_front`，不能用 `vector`，可换 `list`。
- **`unordered_map` 迭代顺序不确定**：不要依赖遍历顺序；rehash 会使迭代器失效。
- **`size()` 返回无符号类型**：倒序遍历时 `i >= 0` 会下溢。

## 代码示例

```cpp
#include <iostream>
#include <map>
#include <queue>
#include <string>
#include <unordered_map>
#include <vector>

int main() {
    std::vector<int> v = {3, 1, 2};
    v.push_back(4);

    std::map<std::string, int> m;
    m["apple"] = 3;                 // 有序映射

    std::unordered_map<std::string, int> um;
    um["banana"] = 5;               // 哈希映射

    std::priority_queue<int> pq;    // 默认大顶堆
    pq.push(10);
    pq.push(20);
    std::cout << pq.top() << '\n';  // 20

    std::queue<int> q;              // FIFO 适配器
    q.push(1);
    q.pop();
}
```

`map::operator[]` 的副作用：

```cpp
#include <iostream>
#include <map>
#include <string>

int main() {
    std::map<std::string, int> m{{"a", 1}};
    std::cout << m.size() << '\n';     // 1

    int x = m["missing"];              // 插入 {"missing", 0}
    std::cout << m.size() << '\n';     // 2
    (void)x;

    auto it = m.find("missing2");      // 不插入
    if (it == m.end()) {
        std::cout << "not found\n";
    }
}
```

`vector<bool>` 的代理对象：

```cpp
#include <iostream>
#include <vector>

int main() {
    std::vector<bool> bits(4, false);
    bits[1] = true;

    auto proxy = bits[1];      // 代理对象，不是 bool&
    std::cout << proxy << '\n';

    // bool* p = &bits[1];     // 编译错误：不能取地址
}
```

小顶堆与定长数组：

```cpp
#include <array>
#include <functional>
#include <queue>
#include <vector>

int main() {
    std::priority_queue<int, std::vector<int>, std::greater<int>> minHeap;
    minHeap.push(3);
    minHeap.push(1);
    // minHeap.top() == 1

    std::array<int, 3> a = {1, 2, 3};
    static_assert(a.size() == 3);
}
```

## 高频追问

**Q1：`vector` 和 `list` 怎么选？**
A：默认选 `vector`。`vector` 连续内存、缓存友好，尾部插入均摊 O(1)；`list` 只有在频繁中间插入/删除且已持有位置迭代器时才有优势，但遍历和查找更慢。

**Q2：`map` 和 `unordered_map` 怎么选？**
A：需要有序遍历或范围查询用 `map`；只做键查找、追求平均 O(1) 用 `unordered_map`。`unordered_map` 最坏 O(n)，且 rehash 会使迭代器失效；`map` 迭代器稳定。

**Q3：`deque` 适合什么场景？**
A：两端频繁插入删除、又需要随机访问的场景，例如滑动窗口、双端队列。它由分段连续内存 + 中控数组实现，两端 O(1)，中间插入 O(n)。

**Q4：`stack` 和 `queue` 的底层默认容器是什么？**
A：默认都是 `deque`。`stack` 也可以用 `vector` 或 `list`；`queue` 要求 `front/pop_front`，`vector` 不适合，通常用 `deque` 或 `list`。

**Q5：`array` 和 C 数组有什么区别？**
A：`std::array` 是值语义的聚合类型，支持 `size()`、迭代器、`at()`、比较运算，可整体赋值；C 数组会退化为指针，丢失长度信息。

**Q6：`vector<bool>` 为什么特殊？**
A：标准为 `vector<bool>` 做了空间优化，按位存储。`operator[]` 返回代理对象，不能取地址，`data()` 也不存在，因此通常不建议用。

**Q7：`priority_queue` 默认是大顶堆还是小顶堆？**
A：默认大顶堆。小顶堆需写成 `std::priority_queue<T, std::vector<T>, std::greater<T>>`。

## 易错点与加分点

- 不知道 `stack/queue` 是适配器，误以为它们是独立容器。
- 用 `map::operator[]` 做只读查询，意外插入元素。
- 依赖 `unordered_map` 的遍历顺序。
- 用 `list` 替代 `vector` 却仍在频繁遍历，性能反而更差。
- 知道 `vector` 的均摊复杂度来自「成倍扩容」，细节见容器底层篇。
- 能说出 `vector<bool>` 是特化而不是普通容器。

## 相关文章

- [容器底层与迭代器失效](容器底层与迭代器失效.md)
- [C++ 核心面试笔记索引](../README.md)
