
Spring Boot 的自动配置机制是它的重要特性之一，极大地简化了 Spring 应用的配置工作。自动配置的核心思想是基于类路径中的依赖、环境配置以及自定义代码进行智能化配置，避免了开发者手动编写大量的样板代码。


接下来，我将详细介绍 Spring Boot 自动配置的过程，核心原理以及涉及的关键组件，并结合源码进行深入解析。


### 一、Spring Boot 自动配置的工作流程


1. **`@SpringBootApplication` 注解**
自动配置的起点通常是 `@SpringBootApplication` 注解，它是一个组合注解，包含了三个重要注解：


	* `@SpringBootConfiguration`：标记为一个 Spring 配置类，类似于 `@Configuration`。
	* `@EnableAutoConfiguration`：启用 Spring Boot 的自动配置机制。
	* `@ComponentScan`：扫描当前包及其子包下的所有 Spring 组件。其中 `@EnableAutoConfiguration` 是自动配置的核心，它引导自动配置机制。
2. **`@EnableAutoConfiguration` 和 `AutoConfigurationImportSelector`**
`@EnableAutoConfiguration` 注解的作用是告诉 Spring Boot 启动时自动配置 Spring 应用上下文。该注解引入了 `AutoConfigurationImportSelector`，这是自动配置的核心处理器。



```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
}

```

`AutoConfigurationImportSelector` 类会从配置文件中（通常是 `spring.factories`）读取所有的自动配置类，并将它们导入到应用上下文中。
3. **`spring.factories` 文件**
自动配置类是通过 `spring-boot-autoconfigure` 模块的 `META-INF/spring.factories` 文件来配置的。这个文件中列出了所有可以被自动加载的配置类：



```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration,\
...

```

这些配置类会在 Spring Boot 启动时根据当前环境条件被选择性加载。
4. **条件装配（`@Conditional` 系列注解）**
Spring Boot 并不是盲目地加载所有的自动配置类。每个自动配置类通常都会使用 `@Conditional` 系列注解来进行有条件的加载。最常见的条件注解有：


	* **`@ConditionalOnClass`**：当类路径中存在某个类时才生效。
	* **`@ConditionalOnMissingBean`**：当 Spring 上下文中不存在某个 Bean 时才生效。
	* **`@ConditionalOnProperty`**：当某个配置属性满足特定条件时才生效。
	* **`@ConditionalOnBean`**：当 Spring 上下文中存在某个 Bean 时才生效。例如，`DataSourceAutoConfiguration` 只有在项目中存在数据源相关的依赖（如 `javax.sql.DataSource` 类）时才会被加载。
5. **自动配置类示例：`DataSourceAutoConfiguration`**


Spring Boot 中 `DataSourceAutoConfiguration` 是配置数据源的自动配置类，它的源码如下：



```
@Configuration
@ConditionalOnClass(DataSource.class)
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({DataSourcePoolMetadataProvidersConfiguration.class,
         DataSourceInitializationConfiguration.class})
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource(DataSourceProperties properties) {
        // 创建并返回 DataSource 对象
        return properties.initializeDataSourceBuilder().build();
    }
}

```

	* `@ConditionalOnClass(DataSource.class)`：只有当类路径下存在 `DataSource` 类时，才进行数据源的自动配置。
	* `@ConditionalOnMissingBean`：如果 Spring 上下文中没有其他 `DataSource` Bean，则自动配置一个。这种基于条件的配置方式确保了 Spring Boot 的灵活性，允许用户通过覆盖默认 Bean 或不满足条件的方式跳过某些自动配置。


### 二、Spring Boot 自动配置的核心步骤


1. **收集自动配置类**
启动时，`AutoConfigurationImportSelector` 从 `spring.factories` 文件中读取所有的自动配置类，并通过 `@Import` 导入这些类。
2. **条件检查**
自动配置类的加载不是无条件的，Spring Boot 会根据 `@Conditional` 注解进行条件检查，确保只有符合条件的自动配置类才会生效。
3. **注入所需的 Bean**
一旦自动配置类通过条件检查，Spring Boot 就会根据这些配置类注册所需的 Bean。例如，`DataSourceAutoConfiguration` 会自动配置数据源相关的 Bean。
4. **允许用户覆盖自动配置**
自动配置并不是强制的。用户可以通过显式声明自己的 Bean 来覆盖自动配置的默认行为。例如，如果用户在自己的配置类中定义了 `DataSource`，那么 Spring Boot 就不会再自动配置数据源。


### 三、Spring Boot 自动配置的实际案例


1. **Web 应用自动配置**
在 Spring Boot Web 应用中，`DispatcherServletAutoConfiguration` 负责自动配置 Spring MVC 的核心组件，例如 `DispatcherServlet`、`RequestMappingHandlerMapping` 等。


	* 如果项目中存在 `spring-web` 依赖，那么 `DispatcherServletAutoConfiguration` 会自动加载。
	* 如果没有手动定义 `DispatcherServlet`，Spring Boot 会自动创建一个 `DispatcherServlet` 并配置到 Spring 容器中。
2. **数据库连接池自动配置**
Spring Boot 还会自动配置数据库连接池（如 HikariCP、Tomcat JDBC 等），这依赖于项目中的 `spring-boot-starter-data-jpa` 或者 `spring-boot-starter-jdbc` 依赖。


	* `DataSourceAutoConfiguration` 和 `DataSourceProperties` 共同负责自动配置数据源。
	* 如果类路径中存在连接池类（如 `HikariDataSource`），那么 Spring Boot 就会自动配置连接池。同时，用户可以通过 `application.properties` 文件来自定义连接池配置：



```
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.datasource.password=secret
spring.datasource.hikari.maximum-pool-size=10

```
3. **Spring Security 自动配置**
当引入 `spring-boot-starter-security` 依赖时，Spring Boot 会自动配置安全机制，默认提供 HTTP Basic 认证机制。


	* `SecurityAutoConfiguration` 负责自动配置 Spring Security 的基础设施。
	* 如果需要定制安全策略，可以通过自定义 `WebSecurityConfigurerAdapter` 来覆盖默认配置。


### 四、Spring Boot 自动配置的好处


1. **极大地简化配置工作**：开发者不再需要为每个基础设施组件编写配置代码，自动配置机制根据项目依赖自动注入所需的 Bean。
2. **灵活性**：自动配置并不会束缚开发者。开发者可以通过自定义配置轻松覆盖默认的自动配置。
3. **约定优于配置**：Spring Boot 遵循 "约定优于配置" 的原则，只需少量的配置，Spring Boot 就能完成复杂的初始化工作。


### 五、Spring Boot 自动配置与 Spring 自动装配的区别


* **Spring 自动装配**：指通过 `@Autowired` 等注解，根据类型自动注入依赖 Bean。它侧重于注入已经配置好的 Bean。
* **Spring Boot 自动配置**：是根据类路径中的依赖和环境信息自动配置 Spring 组件的过程。它负责创建并配置所需的基础设施 Bean。


总结来说，Spring Boot 的自动配置机制通过 `@EnableAutoConfiguration` 启动，基于 `spring.factories` 中的配置和 `@Conditional` 条件判断，自动注入所需的 Bean，简化了开发者的配置工作，同时保留了灵活的定制能力。


 本博客参考[蓝猫机场加速器](https://dahelaoshi.com)。转载请注明出处！
