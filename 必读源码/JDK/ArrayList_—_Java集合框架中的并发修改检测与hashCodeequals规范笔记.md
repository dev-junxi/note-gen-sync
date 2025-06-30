# ArrayList — Java集合框架中的并发修改检测与hashCode/equals规范笔记

## 关于@IntrinsicCandidate注解

`@IntrinsicCandidate`是JDK内部使用的性能优化注解，主要作用于一些会被JVM特殊处理的核心方法：

```java
@IntrinsicCandidate
public static native void arraycopy(Object src, int srcPos, Object dest, int destPos, int length);
```

这种注解的方法会被JVM用底层指令优化，比如：

- `System.arraycopy()`可能被替换为`memcpy`指令
- `Math.sqrt()`可能直接使用CPU的平方根指令

## 为什么重写equals()必须重写hashCode()

这是Java对象一致性最重要的契约之一：(hash集合)

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;
    Person person = (Person) o;
    return age == person.age && 
           Objects.equals(name, person.name);
}

@Override
public int hashCode() {
    return Objects.hash(name, age);
}
```

**关键原因**：当两个对象`equals()`为true时，必须保证它们的`hashCode()`也相同，否则在HashSet/HashMap等集合中会出现对象"消失"的诡异现象。

## ConcurrentModificationException详解

这个异常是Java集合的"自我保护机制"，主要发生在：

```java
List<String> list = new ArrayList<>();
list.add("A");
for (String s : list) {
    list.add("B"); // 抛出异常！
}
```

### 解决方案对比


| 场景   | 解决方案       | 示例                           |
| ------ | -------------- | ------------------------------ |
| 单线程 | 使用迭代器删除 | `iterator.remove()`            |
| 多线程 | 并发集合       | `new CopyOnWriteArrayList<>()` |
| 通用   | 同步控制       | `synchronized(list)`           |

## ArrayList的修改检测机制

在`equalsArrayList`方法中，Java用两种方式检测并发修改：

1. **Size与容量检查**：

```java
if (s > es.length || s > otherEs.length) {
    throw new ConcurrentModificationException();
}
```

2. **modCount验证**：

```java
void checkForComodification(int expectedModCount) {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```

## 实用案例分析

假设我们要实现一个线程安全的颜色管理器：

```java
public class ColorManager {
    private final List<String> colors = new CopyOnWriteArrayList<>();
  
    public void addColor(String color) {
        colors.add(color);
    }
  
    public boolean containsAll(Collection<String> otherColors) {
        // 线程安全地比较
        return colors.containsAll(otherColors);
    }
}
```

**关键点**：

1. 使用`CopyOnWriteArrayList`避免并发修改异常
2. 不需要额外同步
3. 适合读多写少的场景

## **✅ 使用迭代器（Iterator）删除元素 —— ✅ 推荐、安全**

```
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
Iterator<String> iterator = list.iterator();
while (iterator.hasNext()) {
    String item = iterator.next();
    if ("b".equals(item)) {
        iterator.remove(); // ✅ 安全地删除
    }
}
```

* **调用的是** iterator.remove()**，这是** **迭代器支持的安全删除方式**。
* 底层知道你删除了当前元素，会自动更新内部状态，避免异常。

---

### **❌ 直接在** ******for**** 循环中删除元素 —— ❌ 会抛出异常（****ConcurrentModificationException****）**

```
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
for (String item : list) {
    if ("b".equals(item)) {
        list.remove(item); // ❌ 抛 ConcurrentModificationException
    }
}
```

* **for-each** **循环其实是** **基于迭代器 Iterator 实现的语法糖****。**
* 你直接** **list.remove()** **会导致集合结构发生变化，**但迭代器并不知道**。
* **下一次调用** iterator.next()** **时，会发现集合被修改了，于是抛出** **ConcurrentModificationException**。**

---

## **🧠 为什么会抛** ******ConcurrentModificationException****？**

这是 Java 为了避免** ****“遍历过程中结构被破坏”** 而设计的保护机制。

* 每个集合类（如** **ArrayList）内部维护一个** **modCount（修改次数）。
* 迭代器也有一个** **expectedModCount，记录创建迭代器时的版本。
* **一旦检测到** modCount != expectedModCount**，就抛出异常。**

---

## **✅ 正确删除元素的几种方式**

### **1. 使用迭代器（推荐）**

```
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (shouldRemove(it.next())) {
        it.remove();
    }
}
```

---

### **2. 反向遍历** ******for**** 循环（适用于** ******ArrayList****）**

```
for (int i = list.size() - 1; i >= 0; i--) {
    if (shouldRemove(list.get(i))) {
        list.remove(i); // ✅ 安全：不会影响未遍历的部分
    }
}
```

---

### **3. 使用** ******removeIf****（Java 8+）**

```
list.removeIf(item -> "b".equals(item)); // ✅ 简洁又安全
```

---

## **❗ 总结**


| **方式**                        | **是否安全** | **是否推荐**       |
| ------------------------------- | ------------ | ------------------ |
| iterator.remove()               | ✅ 安全      | ✅ 推荐            |
| for-each**中** list.remove()    | ❌ 抛异常    | ❌ 禁止            |
| for**正序循环中** list.remove() | ❌ 有坑      | ❌ 慎用            |
| **for** 倒序删除                | ✅ 安全      | ✅ 可用            |
| removeIf(Predicate)             | ✅ 安全      | ✅ 推荐（Java 8+） |

```
Java集合安全使用指南
├─ 单线程环境
│  ├─ 避免遍历时修改 → 用迭代器删除
│  └─ 需要修改时 → 创建副本
└─ 多线程环境
   ├─ 读多写少 → CopyOnWriteArrayList
   └─ 写频繁 → Collections.synchronizedList + 同步块
```

记住这些原则，就能避免大多数集合相关的并发问题！
