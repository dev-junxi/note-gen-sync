这段代码是 Spring AOP 中 **筛选符合条件的增强器（Advisor）** 的核心逻辑，位于 `AbstractAdvisorAutoProxyCreator` 类中。它的作用是为当前正在处理的 Bean 找出所有需要应用的 AOP 增强逻辑。以下是逐行解析：

---

### 方法签名

```java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName)
```

- **输入**：目标 Bean 的 Class 对象和名称
- **输出**：需要应用到该 Bean 的所有 `Advisor`（按优先级排序）

---

### 执行步骤解析

#### 1. `findCandidateAdvisors()`

```java
List<Advisor> candidateAdvisors = findCandidateAdvisors();
```

- **作用**：获取容器中 **所有候选的 Advisor**（包括显式声明的和注解驱动的）
- **来源**：
  - 通过 `@Aspect` 注解定义的切面类（解析为 `Advisor`）
  - 编程式注册的 `Advisor` Bean（如 `TransactionInterceptor`）
  - 其他 `Advisor` 自动发现机制
- **关键点**：此时返回的是 **所有可能的 Advisor**，尚未与当前 Bean 匹配

---

#### 2. `findAdvisorsThatCanApply()`

```java
List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
```

- **作用**：从候选 Advisor 中筛选出 **能应用到当前 Bean** 的 Advisor
- **匹配逻辑**：
  - 对于 **注解驱动的 Advisor**（如 `@Transactional`）：检查目标类或方法是否有对应注解
  - 对于 **切面表达式驱动的 Advisor**：检查目标类是否匹配切点表达式（Pointcut）
- **实现细节**：
  - 调用 `AopUtils.canApply()` 进行实际匹配
  - 会检查类级别和方法级别的匹配条件

---

#### 3. `extendAdvisors()`

```java
extendAdvisors(eligibleAdvisors);
```

- **作用**：**扩展 Advisor 链**（留给子类的扩展点）
- **典型用途**：
  - 添加默认的 `ExposeInvocationInterceptor`（暴露当前调用上下文）
  - 子类可覆盖此方法插入自定义拦截器
- **示例**：
  ```java
  protected void extendAdvisors(List<Advisor> advisors) {
      // 默认添加一个拦截器到链首
      advisors.add(0, new DefaultAdvisor());
  }
  ```

---

#### 4. 排序处理

```java
if (!eligibleAdvisors.isEmpty()) {
    eligibleAdvisors = sortAdvisors(eligibleAdvisors);
}
```

- **作用**：对筛选后的 Advisor 按 **优先级排序**
- **排序规则**：
  - 通过 `Ordered` 接口或 `@Order` 注解确定优先级
  - 数值越小优先级越高（越先执行）
- **意义**：确保拦截器链的执行顺序符合预期（如：`@Transactional` 和 `@Cacheable` 的顺序影响行为）

---

### 方法调用场景

此方法在以下流程中被调用：

```java
AbstractAutoProxyCreator.wrapIfNecessary()
  → getAdvicesAndAdvisorsForBean()
    → findEligibleAdvisors()  // 走到当前方法
```

---

### 设计意义

1. **职责分离**：
   - `findCandidateAdvisors`：负责"找资源"
   - `findAdvisorsThatCanApply`：负责"过滤匹配"
2. **可扩展性**：
   - 通过覆盖 `extendAdvisors` 可插入自定义逻辑
3. **性能优化**：
   - 先获取所有候选，再按需过滤，避免重复查询容器

---

### 调试技巧

若需验证某个 Bean 的 Advisor 匹配情况：

1. 在 `findEligibleAdvisors` 方法设断点
2. 观察 `eligibleAdvisors` 的最终内容：
   - 对于 `@Transactional` Bean：应包含 `TransactionInterceptor`
   - 对于匹配切面的 Bean：应包含对应的 `AspectJAfterAdvice` 等

理解此方法对解决 AOP 不生效问题（如切面未触发）非常有帮助。
