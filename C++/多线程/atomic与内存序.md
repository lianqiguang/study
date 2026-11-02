# atomic 与内存序

> 一句话结论：`std::atomic` 保证单个操作的原子性，但不自动保证多个变量之间的顺序；`memory_order_relaxed` 只保证原子、不建立同步，`acquire/release` 配对才保证普通数据的可见性。

## 为什么考 / 怎么考

中级面试里，`atomic` 常被用来区分「会用」和「理解」：会用的只会写 `counter++`，理解的能说清 `memory_order` 和 happens-before。典型追问链：

1. `atomic` 和 `mutex` 怎么选？
2. `memory_order` 有哪几种？默认是什么？
3. `relaxed` 为什么不能用来做线程间同步？
4. `compare_exchange_weak` 和 `strong` 有什么区别？
5. `volatile` 能不能替代 `atomic`？

## 原理

### 1. atomic 做什么

`std::atomic<T>` 提供对 `T` 的原子读、写、读改写操作。多个线程并发操作同一个 atomic 对象不会产生数据竞争。

常用成员：

- `load()` / `store()`：原子读写；
- `exchange()`：写入新值并返回旧值；
- `fetch_add()` / `fetch_sub()`：返回**旧值**（`++` 返回新值）；
- `compare_exchange_weak/strong()`：CAS（compare-and-swap）。

```cpp
#include <atomic>
#include <iostream>
#include <thread>

std::atomic<int> counter{0};

void add() {
    for (int i = 0; i < 1000; ++i) {
        counter.fetch_add(1, std::memory_order_relaxed);
    }
}

int main() {
    std::thread t1(add), t2(add);
    t1.join();
    t2.join();
    std::cout << counter.load() << '\n'; // 2000
}
```

`atomic<T>` 不保证一定无锁：`is_lock_free()` 用来查询；`std::atomic_flag` 则保证无锁。

### 2. CAS 与 ABA

```cpp
#include <atomic>

int main() {
    std::atomic<int> v{10};

    int expected = 10;
    bool ok = v.compare_exchange_strong(expected, 20); // 成功，v = 20

    int target = 20;
    while (!v.compare_exchange_weak(target, 30)) {
        // 失败时 target 会被更新为 v 的当前值，可重试
    }
    return ok ? 0 : 1;
}
```

- `weak` 允许「伪失败」（即使值相等也可能返回失败），通常放在循环里；`strong` 不会伪失败。
- **ABA 问题**：CAS 只比较值，如果值从 A 变成 B 又变回 A，CAS 会误以为没变过。解决方法是加版本号（`atomic<pair<T, uint64_t>>` 或带 tag 的指针）。

### 3. memory_order

| 枚举 | 语义 |
|---|---|
| `memory_order_relaxed` | 只保证原子性，不建立任何跨线程顺序 |
| `memory_order_acquire` | 读操作；之后的读写不能重排到它之前；与 release 配对 |
| `memory_order_release` | 写操作；之前的读写不能重排到它之后；与 acquire 配对 |
| `memory_order_acq_rel` | 读改写操作同时具备 acquire 和 release |
| `memory_order_seq_cst` | 全局总顺序，默认值，最强也最慢 |
| `memory_order_consume` | 依赖顺序，语义复杂，实践中不推荐 |

`load/store` 默认是 `seq_cst`；`fetch_*` 默认也是 `seq_cst`。

### 4. release/acquire：消息传递

`release` 保证「它之前的所有写」对「与它配对的 acquire 之后的所有读」可见：

```cpp
#include <atomic>
#include <cassert>
#include <thread>

int data = 0;
std::atomic<bool> ready{false};

void producer() {
    data = 42;                                     // 普通写
    ready.store(true, std::memory_order_release);  // 之前的写先完成
}

void consumer() {
    while (!ready.load(std::memory_order_acquire)) {}
    assert(data == 42);                            // 保证能看到 42
}

int main() {
    std::thread p(producer), c(consumer);
    p.join();
    c.join();
}
```

如果把两端都换成 `relaxed`，`data = 42` 和 `ready = true` 之间没有顺序保证，消费者可能看到 `ready == true` 但 `data == 0`——这仍然构成数据竞争：

```cpp
#include <atomic>
#include <thread>

int data = 0;
std::atomic<bool> ready{false};

void writer() {
    data = 1;
    ready.store(true, std::memory_order_relaxed);  // 反例：不建立同步
}

void reader() {
    while (!ready.load(std::memory_order_relaxed)) {}
    // data 可能仍是 0；这里的读写构成数据竞争
}

int main() {
    std::thread w(writer), r(reader);
    w.join();
    r.join();
}
```

### 5. 自旋锁

`atomic_flag` 是最简单的原子标志，保证无锁，常用于实现自旋锁：

```cpp
#include <atomic>

class SpinLock {
public:
    void lock() {
        while (flag_.test_and_set(std::memory_order_acquire)) {
            // 自旋
        }
    }
    void unlock() {
        flag_.clear(std::memory_order_release);
    }

private:
    std::atomic_flag flag_ = ATOMIC_FLAG_INIT;
};
```

自旋锁适合临界区极短、线程数不多的场景；临界区长时浪费 CPU，应改用 `mutex`。

### 6. volatile 与 atomic 的区别

- `volatile` 只阻止编译器优化访问，不保证原子性，也不建立内存序；
- `atomic` 保证原子性，并可指定内存序；
- 两者解决的问题不同，不能互相替代。线程间共享数据用 `atomic` 或锁，MMIO 用 `volatile`。

### 7. C++20 的 atomic<shared_ptr>

C++20 提供 `std::atomic<std::shared_ptr<T>>`，支持原子的 shared_ptr 加载/存储/交换/CAS；在 C++17 及以前，通常用锁或 `atomic_load/atomic_store` 的自由函数（C++11 起对 `shared_ptr` 有特化，但能力有限）。

## 高频追问

**Q1：`atomic` 和 `mutex` 怎么选？**
A：单个变量、简单计数/标志用 `atomic`，无锁、开销小；需要保护多个变量或复合不变量时用 `mutex`。`atomic` 不能替代锁来保证一段代码的互斥。

**Q2：`memory_order` 有哪几种？默认是什么？**
A：`relaxed`、`consume`、`acquire`、`release`、`acq_rel`、`seq_cst`；默认是 `seq_cst`，最强也最慢。

**Q3：`relaxed` 为什么不能做线程间同步？**
A：它只保证单个原子操作的原子性，不建立 happens-before，普通数据的读写可能被重排或不可见。

**Q4：`compare_exchange_weak` 和 `strong` 的区别？**
A：`weak` 允许伪失败，适合循环重试；`strong` 不会伪失败，单次尝试更直观。

**Q5：`atomic` 一定无锁吗？**
A：不一定。`atomic<T>` 可能是用锁实现的（例如大对象）；用 `is_lock_free()` 查询。`atomic_flag` 保证无锁。

**Q6：`volatile` 为什么不能替代 `atomic`？**
A：`volatile` 不保证原子性，也不建立内存序，多线程下仍会数据竞争；它只用于 MMIO、信号处理等「访问不能被优化」的场景。

**Q7：ABA 问题怎么解决？**
A：给值加版本号或使用带 tag 的指针，让 CAS 能区分「没变」和「变了又变回来」。

## 易错点与加分点

- 用 `relaxed` 保护普通数据，误以为 atomic 自带同步。
- 把 `volatile` 当线程同步（Java/C# 的语义与 C++ 不同）。
- `fetch_add` 返回旧值、`operator++` 返回新值，记反。
- 在 CAS 循环里忘记用失败时更新的 expected 值重试。
- 忽视 `atomic<T>` 可能不是 lock-free。
- 能说出 `acquire`/`release` 的配对关系和 happens-before。

## 相关文章

- [线程与锁](线程与锁.md)
- [条件变量与线程池](条件变量与线程池.md)
- [const / static / volatile](../基础/const-static-volatile.md)
- [C++ 核心面试笔记索引](../README.md)
