# LRU 缓存

> 一句话结论：LRU = 哈希表 + 双向链表；哈希表 O(1) 定位节点，双向链表维护新旧顺序（头部最新、尾部最旧），`get`/`put` 都是 O(1)。

## 为什么考 / 怎么考

LRU 是「数据结构组合 + 工程实现」的经典题，考你能不能把 O(1) 的要求落实到每个操作。典型追问链：

1. 为什么必须用双向链表，单链表不行吗？
2. `get` 命中后要做什么？
3. 容量满了怎么淘汰？
4. 怎么保证 O(1)？
5. 多线程下怎么用？

## 思路

- **哈希表**：`key -> 链表节点迭代器`，O(1) 找到节点；
- **双向链表**：头部是最近使用，尾部是最久未使用；
- `get(key)`：找不到返回 -1；找到就把节点 `splice` 到头部，返回值；
- `put(key, value)`：已存在则更新值并移到头部；不存在则新建到头部，若超容量则删除尾部节点，并同步从哈希表删除；
- 删除/移动节点必须是 O(1)，所以链表要能拿到前驱——这就是双向链表的必要性。

`std::list::splice` 把一个节点从原位置摘下来接到另一位置，不拷贝元素、不失效迭代器，正好满足要求。

## 代码示例

`std::list` 的 `splice` 演示：

```cpp
#include <iostream>
#include <iterator>
#include <list>

int main() {
    std::list<int> l{1, 2, 3, 4};
    auto it = std::next(l.begin(), 2);   // 指向 3

    l.splice(l.begin(), l, it);          // O(1) 把 3 移到头部
    for (int x : l) std::cout << x << ' ';
    // 输出：3 1 2 4
}
```

完整实现：

```cpp
#include <list>
#include <unordered_map>
#include <utility>

class LRUCache {
public:
    explicit LRUCache(int capacity) : cap_(capacity) {}

    LRUCache(const LRUCache&) = delete;              // 拷贝会让 map 的迭代器指向原链表
    LRUCache& operator=(const LRUCache&) = delete;
    LRUCache(LRUCache&&) = default;                  // 移动保持节点地址，安全
    LRUCache& operator=(LRUCache&&) = default;

    int get(int key) {
        auto it = map_.find(key);
        if (it == map_.end()) return -1;

        items_.splice(items_.begin(), items_, it->second);   // O(1) 移到头部
        return it->second->second;
    }

    void put(int key, int value) {
        if (cap_ <= 0) return;

        auto it = map_.find(key);
        if (it != map_.end()) {           // 已存在：更新并移到头部
            it->second->second = value;
            items_.splice(items_.begin(), items_, it->second);
            return;
        }

        if (static_cast<int>(map_.size()) == cap_) {   // 淘汰最久未使用
            int oldKey = items_.back().first;
            map_.erase(oldKey);
            items_.pop_back();
        }

        items_.emplace_front(key, value);              // 新节点放头部
        map_[key] = items_.begin();
    }

private:
    int cap_;
    std::list<std::pair<int, int>> items_;   // 头部最新，尾部最旧
    std::unordered_map<int, std::list<std::pair<int, int>>::iterator> map_;
};
```

## 高频追问

**Q1：为什么必须用双向链表？**
A：淘汰和移动都需要 O(1) 访问前驱/后继；单链表删除节点要重新找前驱，退化为 O(n)。双向链表配合迭代器可以直接 `splice`/`erase`。

**Q2：`get` 命中后要做什么？**
A：把对应节点移到链表头部（标记为最近使用），然后返回值；不需要修改 value。

**Q3：`put` 已存在的 key 怎么处理？**
A：更新 value，并把节点移到头部；不要新建节点，否则哈希表里会出现重复 key 或旧节点残留。

**Q4：容量满了怎么淘汰？**
A：取链表尾部节点（最久未使用），先从哈希表删除它的 key，再 `pop_back`；顺序反了会留下悬空迭代器。

**Q5：为什么能做到 O(1)？**
A：哈希表平均 O(1) 定位；链表头部插入、尾部删除、已知迭代器的 `splice` 都是 O(1)。

**Q6：LRU 和 LFU 的区别？**
A：LRU 按「最近是否使用」淘汰，实现是哈希表 + 链表；LFU 按「使用频率」淘汰，通常需要频率桶 + 双向链表，或堆，实现更复杂。

**Q7：多线程下怎么用？**
A：给 `get/put` 加互斥锁；高并发场景可以分片（按 key 哈希到多个 LRU 实例）降低锁竞争。注意锁的顺序和锁粒度。

**Q8：缓存容量为 0 怎么办？**
A：`put` 直接返回，`get` 永远 -1；实现里加 `cap_ <= 0` 的判断，避免淘汰时访问空链表。

## 易错点与加分点

- 用单链表，删除时找不到前驱。
- `get` 命中后忘记移到头部，退化成 FIFO。
- 淘汰时先 `pop_back` 再删哈希表，留下悬空迭代器。
- `put` 已存在 key 时新建节点，导致链表和哈希表不一致。
- 忽略容量 0 的边界。
- 允许默认拷贝：`map_` 里的迭代器会指向原对象的链表，拷贝后使用是未定义行为；应删除拷贝（移动是安全的）。
- 能说出分片、读写锁、TTL 等工程扩展，是加分项。

## 相关文章

- [链表高频题](链表高频题.md)
- [二叉树遍历](二叉树遍历.md)
- [二分查找](二分查找.md)
- [算法索引](../README.md)
