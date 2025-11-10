# db-router-spring-boot-starter

## 使用方法 How to use

1. **引入 Starter**

```
<dependency>
    <groupId>cn.bugstack.middleware</groupId>
    <artifactId>db-router-spring-boot-starter</artifactId>
    <version>1.0.1-SNAPSHOT</version>
</dependency>
```

1. **配置数据源**

```
mini-db-router:
  jdbc:
    datasource:
      dbCount: 2
      tbCount: 4
      routerKey: uId
      list: db0,db1
      default: db0
      db0:
        url: jdbc:mysql://localhost:3306/db0
        username: root
        password: 123456
      db1:
        url: jdbc:mysql://localhost:3306/db1
        username: root
        password: 123456
```

1. **业务方法使用**

```
@DBRouter(key = "uId")
public User getUser(Long uId) {
    ...
}
```

## 文件夹结构与功能对应图示意

```
db-router-spring-boot-starter
├── src/main/java/cn/zly/middleware/db/router
│   ├── annotation/          ──> 自定义注解（@DBRouter, @DBRouterStrategy）
│   ├── config/              ──> 自动配置类（DataSourceAutoConfig）
│   ├── dynamic/             ──> 动态数据源 & MyBatis 分表插件
│   │     ├── DynamicDataSource.java  ──> 选择数据源
│   │     └── DynamicMybatisPlugin.java ──> SQL 表名替换
│   ├── strategy/            ──> 路由策略接口与实现
│   │     ├── IDBRouterStrategy.java
│   │     └── DBRouterStrategyHashCode.java
│   ├── util/                ──> 工具类（配置读取）
│   │     └── PropertyUtil.java
│   ├── DBContextHolder.java ──> 上下文 ThreadLocal 保存 dbKey/tbKey
│   ├── properties/            ──> 从配置文件获取参数相关
│   │     └── DbRouterProperties.java  ──> 封装 dbCount、tbCount、routerKey 配置
│   └── aspect ──> AOP 切面，拦截注解方法
│     	  └── DBRouterJoinPoint.java  ──> aop相关
├── src/main/resources
│   └── META-INF/spring.factories ──> 自动配置声明
└── pom.xml                   ──> 项目依赖与打包配置

```









## 作用流程

```
┌────────────────────┐
│    业务方法调用     │
│ getUser(uId)       │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│      AOP拦截       │
│ @DBRouter 注解     │
│ 获取路由字段 uId   │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│   DBRouterStrategy  │
│ 计算 dbKey / tbKey  │
│ dbCount / tbCount   │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│   DBContextHolder   │
│ setDBKey / setTBKey │
│ (ThreadLocal)       │
└─────────┬──────────┘
          │
 ┌────────┴─────────┐
 │                  │
 ▼                  ▼
┌──────────────┐  ┌─────────────────────────┐
│ DynamicDataSource │  │ DynamicMybatisPlugin   │
│ determineCurrent  │  │ prepare SQL            │
│ LookupKey()       │  │ 替换表名 user → user_03│
└──────────────┘  └─────────────────────────┘
          │                  │
          └────────┬─────────┘
                   ▼
           ┌─────────────┐
           │   执行 SQL  │
           │   查询/修改 │
           └─────────────┘
                   │
                   ▼
           ┌─────────────┐
           │ 清理上下文  │
           │ DBContextHolder.clear() │
           └─────────────┘

```





## 1️⃣ 核心概念

1. **Context（上下文）**
   - 指 `DBContextHolder` 中保存的线程级变量：
     - `dbKey` → 当前线程要访问的库
     - `tbKey` → 当前线程要访问的表
   - **作用**：动态数据源和 SQL 分表插件都依赖这个上下文去决定操作目标
2. **路由策略（DBRouterStrategy）**
   - 决定如何根据路由字段计算库和表索引
   - 例：`DBRouterStrategyHashCode` 使用 `hash(uId) % dbCount` 计算库
   - 参数来源：
     - `dbCount` 和 `tbCount` → 配置文件中指定
     - `uId` → 注解中指定的路由字段，从方法参数中获取

## 2️⃣ 执行流程

1. **方法拦截（AOP）**

```
@Around("@annotation(DBRouter)")
public Object doRouter(ProceedingJoinPoint jp, DBRouter dbRouter)
```

- 拦截带有 `@DBRouter` 注解的方法
- 获取路由字段，例如 `uId`
- 调用 `DBRouterStrategy.doRouter(dbKeyAttr)`
- **路由策略计算结果后，会把 dbKey 和 tbKey 写入 DBContextHolder**

------

1. **动态数据源（DynamicDataSource）**

```
@Override
protected Object determineCurrentLookupKey() {
    return "db" + DBContextHolder.getDBKey();
}
```

- Spring 在获取数据源时调用 `determineCurrentLookupKey()`
- 从 `DBContextHolder` 获取当前线程对应的库索引
- 返回的 key 用于从 `targetDataSources` 中选择具体库

------

1. **MyBatis 插件（DynamicMybatisPlugin）**

- 在 SQL 执行前拦截 `StatementHandler.prepare()`
- 从 `DBContextHolder` 获取当前线程的 `tbKey`
- 根据规则替换 SQL 中的表名，例如：

```
SELECT * FROM user → SELECT * FROM user_03
```

## 3️⃣ 数据流总结

```
方法参数(uId)
      │
      ▼
  DBRouter 注解
      │
      ▼
DBRouterStrategy 计算
(dbCount, tbCount, uId)
      │
      ▼
DBContextHolder.setDBKey / setTBKey
      │
      ├─> DynamicDataSource 获取库 → 执行对应库的数据源
      │
      └─> MyBatis 插件替换表名 → 执行对应表的 SQL
```

------

## 4️⃣ 核心点

1. **Context 由路由策略控制**
   - AOP 拦截 → 解析路由字段 → 调用路由策略 → 写入 ThreadLocal 上下文
2. **DBRouterStrategy 参数来源**
   - `dbCount`、`tbCount` → 配置文件
   - 路由字段（如 uId） → 注解中指定
3. **上下文决定后续操作**
   - 动态选择数据源
   - 动态修改 SQL 表名
4. **整个流程是线程安全的**
   - ThreadLocal 保证每个请求线程独立上下文
   - 避免并发冲突

