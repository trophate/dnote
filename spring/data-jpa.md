# 配置

```java
@Configuration
@EnableJpaRepositories("**.repository")// 扫描repository包
@EnableTransactionManagement// 开启事务
public class JpaConfig {

}
```

yml：

```java
spring:
  jpa:
    ...
```

yml配置参数：

show-sql：输出执行sql

ddl-auto：自动生成表策略

1. create：应用运行时会删除并重新创建数据库。
2. create-drop：当sessionFactory关闭，表会被删除。
3. none：无处理。
4. update：应用运行时将新增表/字段同步，不会修改或删除原有结构和数据。
5. validate：应用运行时验证实体与表是否匹配，如果不匹配则报错。





# crud

```java
public interface xxxRepository extends JpaRepository<Entity, IdType> {

}
```

增删改被默认实现，只要继承JpaRepository即可直接使用。



## 查询

通过jpa约定的命名构建查询方法：

```java
User findUserByUsername(String username);
```