### 1. JDK 1.8 结构：数组 + 链表 + 红黑树
*   **数组 (Node[])**：作为主干，提供 $O(1)$ 的快速定位能力。通过 Hash 计算得出数组下标。
*   **链表 (Node)**：解决 Hash 冲突。当多个 Key 的 Hash 值映射到同一个数组下标时，以链表形式挂在数组节点后。
*   **红黑树 (TreeNode)**：解决链表过长导致的查询效率退化问题。当链表过长时，查询时间复杂度从 $O(n)$ 降为 $O(\log n)$。
*   **💡 面试加分项**：为什么 JDK 1.8 要引入红黑树？因为如果发生极端 Hash 冲突（如所有 key 的 hash 值一样），链表会无限长，导致 `get/put` 性能退化为 $O(n)$，甚至引发 DOS 攻击。红黑树是性能兜底方案。

### 2. 默认容量 16，负载因子 0.75，扩容 2 倍
*   **默认容量 16**：必须是 **2 的幂次方**。这是为了在计算数组索引时，能用位运算 `(n - 1) & hash` 代替耗时的取模运算 `hash % n`，且只有当 $n$ 是 2 的幂时，`(n - 1) & hash` 才等价于 `hash % n`。
*   **负载因子 0.75**：是时间和空间的折中。如果太大（如 1.0），链表会变长，查询变慢；如果太小（如 0.5），数组会频繁扩容，浪费内存。
*   **扩容 2 倍 (resize)**：当 `size > 容量 * 0.75` 时触发。
*   **💡 面试加分项（扩容优化）**：JDK 1.8 扩容时，**不需要重新计算 Hash 值**。因为容量是 2 倍，元素在新数组中的位置，要么**在原位置**，要么在**原位置 + 旧容量**的位置。源码通过 `hash & oldCap` 是否为 0 来极简判断，大大提高了扩容效率。

### 3. hash 计算：`(h = key.hashCode()) ^ (h >>> 16)`
*   **扰动函数**：这个操作叫“高低位异或”。
*   **为什么这么做？** 数组的长度通常不大，计算索引时只有 Hash 值的**低位**参与运算 `(n-1) & hash`。如果 Hash 函数生成的值高位差异很大但低位相似，就会发生大量冲突。
*   **原理**：将 `hashCode` 无符号右移 16 位（高位补 0），然后与原值异或。这样既保留了高位的特征，又让高位参与了低位运算，**大大降低了哈希冲突的概率**。

### 4. put 流程：算索引 → 判空 → 比 key → 走链表/树 → 扩容
这是面试必问的源码级流程，JDK 1.8 的 `put` 核心步骤如下：
1.  **算索引**：计算 key 的 hash 值，通过 `(n - 1) & hash` 算出数组下标 `i`。
2.  **判空**：如果数组 `table` 为空或长度为 0，先调用 `resize()` 初始化数组。
3.  **无冲突直接放**：如果 `table[i]` 为空，直接新建 Node 节点放入。
4.  **有冲突处理（比 key）**：
    *   如果 `table[i]` 的首节点 key 与当前 key 相同（hash 相等且 equals 为 true），直接覆盖 value。
    *   如果首节点是 `TreeNode`（红黑树节点），走红黑树的插入逻辑 `putTreeVal`。
    *   如果是链表，遍历链表（**尾插法**）。如果找到相同 key 则覆盖；如果遍历到末尾没找到，则将新节点插入链表尾部。
5.  **判断树化**：链表插入后，判断链表长度是否 $\ge 8$，如果是，尝试将链表转化为红黑树（`treeifyBin`）。
6.  **扩容**：插入完成后，`++size`，判断 `size` 是否大于阈值 `threshold`，大于则调用 `resize()` 扩容。

### 5. 树化条件：链表长度 ≥ 8 且数组长度 ≥ 64
*   **为什么是 8？** 根据泊松分布，在理想随机 Hash 码下，链表长度达到 8 的概率仅为 **0.00000006**（千万分之六）。引入红黑树只是为了应对极端恶劣情况下的性能兜底，正常情况下不会触发。
*   **为什么数组长度要 ≥ 64？** 
    *   如果数组长度 $< 64$，即使链表长度 $\ge 8$，也**不会树化**，而是优先选择**扩容**（`resize`）。因为数组太小，扩容能更有效地分散节点，降低冲突。
    *   只有当数组长度 $\ge 64$，且链表长度仍 $\ge 8$ 时，才会真正树化。
*   **💡 补充考点（退化条件）**：当红黑树节点数 $\le 6$ 时（`UNTREEIFY_THRESHOLD`），会退化为链表。为什么是 6 而不是 8？为了防止在 8 的临界点频繁发生树化和退化的转换（避免震荡）。

### 6. 线程不安全：并发 put 可能死循环（1.7）/ 数据覆盖（1.8）
*   **JDK 1.7 的死循环问题**：
    *   1.7 采用**头插法**。在多线程并发扩容时，由于链表反转，极易导致链表形成**环形链表**。
    *   一旦成环，下次 `get` 或 `put` 遍历该链表时，就会陷入死循环，导致 CPU 飙升到 100%。
*   **JDK 1.8 的数据覆盖问题**：
    *   1.8 改为了**尾插法**，解决了扩容成环的死循环问题。
    *   **但是依然线程不安全**：如果两个线程同时 `put`，且计算出的数组下标相同，且该位置刚好为空，两个线程都会把自己的节点放进去，导致**后插入的覆盖先插入的**，造成数据丢失。此外，`size++` 也不是原子操作，会导致 size 统计不准。
*   **💡 解决方案**：多线程环境下必须使用 `ConcurrentHashMap`（JDK 1.8 采用 `CAS + synchronized` 锁住链表/树的头节点，并发度极高）。

---

### 7. 手写简化版 put/get（面试常考）
面试官让你手写，**不是**让你默写 JDK 源码（那有几千行），而是考察你对**核心数据结构、Hash 计算、冲突解决**的理解。

以下是一个极简版、可运行的 HashMap 实现，去掉了红黑树和复杂扩容，保留了最核心的灵魂：

```java
import java.util.Arrays;

public class SimpleHashMap<K, V> {
    
    // 默认容量 16
    private static final int DEFAULT_CAPACITY = 16;
    // 数组
    private Node<K, V>[] table;
    // 元素个数
    private int size;

    // 内部节点类
    static class Node<K, V> {
        final int hash;
        final K key;
        V value;
        Node<K, V> next;

        Node(int hash, K key, V value, Node<K, V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
    }

    public SimpleHashMap() {
        table = new Node[DEFAULT_CAPACITY];
    }

    // 核心：扰动函数
    private int hash(Object key) {
        if (key == null) return 0;
        int h = key.hashCode();
        return h ^ (h >>> 16);
    }

    // 核心：put 方法
    public V put(K key, V value) {
        int hash = hash(key);
        // 1. 算索引 (n-1) & hash
        int index = hash & (table.length - 1);

        // 2. 判空：如果该位置没有节点，直接放入
        if (table[index] == null) {
            table[index] = new Node<>(hash, key, value, null);
            size++;
            return null;
        }

        // 3. 有冲突：遍历链表
        Node<K, V> curr = table[index];
        Node<K, V> prev = null;
        
        while (curr != null) {
            // 比 key：如果 key 相同，覆盖 value
            if (curr.hash == hash && (curr.key == key || (key != null && key.equals(curr.key)))) {
                V oldValue = curr.value;
                curr.value = value;
                return oldValue;
            }
            prev = curr;
            curr = curr.next;
        }

        // 4. 走链表：尾插法插入新节点
        prev.next = new Node<>(hash, key, value, null);
        size++;
        
        // 5. 扩容判断 (简化版：假设阈值是 12，即 16 * 0.75)
        if (size > 12) {
            resize();
        }
        return null;
    }

    // 核心：get 方法
    public V get(K key) {
        int hash = hash(key);
        int index = hash & (table.length - 1);

        Node<K, V> curr = table[index];
        while (curr != null) {
            if (curr.hash == hash && (curr.key == key || (key != null && key.equals(curr.key)))) {
                return curr.value;
            }
            curr = curr.next;
        }
        return null;
    }

    // 简化版扩容：容量翻倍
    private void resize() {
        Node<K, V>[] oldTable = table;
        int newCapacity = oldTable.length * 2;
        Node<K, V>[] newTable = new Node[newCapacity];
        
        // 重新计算位置并迁移
        for (Node<K, V> node : oldTable) {
            Node<K, V> curr = node;
            while (curr != null) {
                Node<K, V> next = curr.next; // 保存下一个节点
                int newIndex = curr.hash & (newCapacity - 1); // 重新算索引
                // 头插法迁移（简化写法，实际 JDK 1.8 是尾插且优化了位置计算）
                curr.next = newTable[newIndex]; 
                newTable[newIndex] = curr;
                curr = next;
            }
        }
        table = newTable;
    }
    
    public int size() { return size; }
}
```