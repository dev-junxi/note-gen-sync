# HashMap.getNode() 方法深度解析

`getNode()` 是 HashMap 中实现键值查找的核心方法，它完成了 HashMap 根据 key 查找对应 value 的主要逻辑。下面我将从多个维度详细解析这个方法。

## 方法签名

```java
final Node<K,V> getNode(Object key)
```

- 功能：根据 key 查找对应的节点
- 返回值：找到返回 Node，未找到返回 null

## 方法执行流程

### 1. 初始检查和哈希计算

```java
Node<K,V>[] tab; Node<K,V> first, e; int n, hash; K k;
if ((tab = table) != null && (n = tab.length) > 0 &&
    (first = tab[(n - 1) & (hash = hash(key))]) != null) {
    // ...
}
```

- 检查哈希表是否初始化且不为空
- 计算 key 的哈希值：`hash = hash(key)`
- 计算桶位置：`(n - 1) & hash`
- 获取桶的第一个节点：`first`

### 2. 检查第一个节点

```java
if (first.hash == hash && // always check first node
    ((k = first.key) == key || (key != null && key.equals(k))))
    return first;
```

- 先检查第一个节点是否匹配
- 先比较内存地址(`==`)，再比较内容(`equals()`)
- 这是优化手段，因为大部分情况下第一个节点就是要找的

### 3. 处理冲突情况

```java
if ((e = first.next) != null) {
    // 处理链表或树
}
```

- 如果第一个节点不匹配且存在后续节点

#### 3.1 树节点查找

```java
if (first instanceof TreeNode)
    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
```

- 如果是树结构，调用树节点的查找方法
- 红黑树查找时间复杂度 O(log n)

#### 3.2 链表遍历

```java
do {
    if (e.hash == hash &&
        ((k = e.key) == key || (key != null && key.equals(k))))
        return e;
} while ((e = e.next) != null);
```

- 遍历链表查找匹配的节点
- 时间复杂度 O(n)，但在良好哈希函数下平均 O(1)

## 关键设计点

1. **哈希计算优化**：

   - 使用 `(n - 1) & hash` 替代取模运算
   - 利用了容量总是 2 的幂的特性
2. **快速路径优化**：

   - 优先检查第一个节点
   - 无冲突情况下只需一次比较
3. **多态处理**：

   - 自动识别处理链表和树两种结构
   - 保持接口统一性
4. **null 安全处理**：

   - 对 key 为 null 的情况有完善处理
   - 所有可能 NPE 的地方都有检查

## 性能分析

1. **最佳情况**：

   - 无哈希冲突：O(1) 时间复杂度
   - 第一个节点即匹配
2. **平均情况**：

   - 有少量冲突：接近 O(1)
   - 短链表遍历
3. **最坏情况**：

   - 所有 key 哈希冲突：O(n) 或 O(log n)
   - 取决于退化为链表还是转为红黑树

## 使用示例

```java
HashMap<String, Integer> map = new HashMap<>();
map.put("a", 1);
map.put("b", 2);

// 内部调用 getNode
Integer value = map.get("a"); // 返回 1
```

## 方法调用链

`get()` → `getNode()`
`containsKey()` → `getNode()`

## 总结

`getNode()` 方法体现了 HashMap 的核心设计思想：

1. 快速哈希定位
2. 优先检查最可能匹配的节点
3. 自动适应不同冲突解决策略（链表或树）
4. 全面的 null 安全性检查

这种设计使得 HashMap 在各种情况下都能保持较高的查找效率，是 Java 集合框架中最重要且优化最完善的方法之一。
