在 Spring Boot 中，`ConfigurationPropertySources` 是用于统一管理所有环境配置属性的工具类，但它本身并不直接存储环境变量，而是**提供对多种配置源的访问适配**。以下是其核心作用和存储的配置源类型：

---

### **1. `ConfigurationPropertySources` 的职责**

- **功能**：
  将 Spring 的 `Environment` 中的原始属性源（`PropertySource`）适配为 `ConfigurationPropertySource` 接口，供 Spring Boot 的配置绑定（如 `@ConfigurationProperties`）使用。
- **特点**：
  提供**统一视图**，屏蔽不同属性源（如系统变量、配置文件）的差异，支持层级化属性查找（如 `server.port`）。

---

### **2. 存储的配置源类型**

通过 `ConfigurationPropertySources.attach(Environment)` 方法，会将 `Environment` 中的以下常见属性源纳入管理：


| **属性源类型**       | **示例来源**                                                               | **优先级（从高到低）** |
| -------------------- | -------------------------------------------------------------------------- | ---------------------- |
| **命令行参数**       | `--server.port=8080`                                                       | 最高（直接覆盖其他）   |
| **Java 系统属性**    | `-Dapp.name=MyApp`                                                         | 高                     |
| **操作系统环境变量** | `export SERVER_PORT=8080`（Linux/Mac）或 `set SERVER_PORT=8080`（Windows） | 中                     |
| **应用配置文件**     | `application.yml` 或 `application.properties`                              | 低                     |
| **随机数属性**       | `random.*`（如 `random.int`）                                              | 特殊                   |
| **默认属性**         | 通过`SpringApplication.setDefaultProperties()` 设置                        | 最低                   |

---

### **3. 如何查看所有配置源？**

通过 `Environment` 接口可以获取所有属性源及其内容：

```java
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(App.class, args);
        ConfigurableEnvironment env = context.getEnvironment();

        // 打印所有属性源及内容
        env.getPropertySources().forEach(propertySource -> {
            System.out.println("=== PropertySource: " + propertySource.getName() + " ===");
            if (propertySource instanceof MapPropertySource) {
                ((MapPropertySource) propertySource).getSource().forEach((key, value) -> {
                    System.out.println(key + " = " + value);
                });
            }
        });
    }
}
```

---

### **4. 配置源的优先级验证**

Spring Boot 的属性加载遵循 **"就近优先"** 原则。例如：

1. 如果同时在 `application.properties` 和系统属性中定义 `server.port`，系统属性会生效。
2. 命令行参数 (`--server.port`) 的优先级最高。

---

### **5. 特殊场景处理**

#### **(1) 属性名称转换**

- **环境变量**：`SERVER_PORT` 会自动匹配 `server.port`（驼峰转下划线+大写）。
- **配置绑定**：`@ConfigurationProperties(prefix="server")` 可绑定到 `server.port`。

#### **(2) 动态属性源**

通过代码动态添加属性源：

```java
env.getPropertySources().addFirst(
    new MapPropertySource("custom-source", Collections.singletonMap("app.key", "value"))
);
```

---

### **6. 与 `@Value` 和 `@ConfigurationProperties` 的关系**

- **`@Value`**：
  直接从 `Environment` 解析属性，受 `ConfigurationPropertySources` 管理。
  ```java
  @Value("${server.port}")
  private int port;
  ```
- **`@ConfigurationProperties`**：
  通过 `ConfigurationPropertySources` 实现类型安全的绑定。
  ```java
  @ConfigurationProperties(prefix="server")
  public class ServerProperties {
      private int port;
      // getter/setter
  }
  ```

---

### **总结**

- `ConfigurationPropertySources` **不直接存储变量**，而是统一适配 `Environment` 中的多种属性源。
- **存储范围**：包括命令行参数、系统属性、环境变量、配置文件等。
- **优先级**：命令行 > 系统属性 > 环境变量 > 配置文件 > 默认值。
- **调试技巧**：通过 `Environment.getPropertySources()` 查看所有属性源及其内容。
