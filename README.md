# mybatis-plus-practice

# 主启动类配置
```java
@MapperScan("org.lyflexi.debug_mybatis.dao")
@SpringBootApplication
public class DebugMybatisApplication {

    public static void main(String[] args) {
        SpringApplication.run(DebugMybatisApplication.class, args);
    }

}
```

# 日志开启
```yaml
spring:
  jpa:
    properties:
      hibernate:
      dialect: org.hibernate.dialect.PostgreSQLDialect
    show-sql: true
```

# 分页插件
插件配置：
```java
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.POSTGRE_SQL)); // 如果配置多个插件, 切记分页最后添加
        // 如果有多数据源可以不配具体类型, 否则都建议配上具体的 DbType
        return interceptor;
    }
```
自定义分页模型：
> Page 类继承了 IPage 类，实现了简单分页模型。如果你需要实现自己的分页模型，可以继承 Page 类或实现 IPage 类。
> 
> https://baomidou.com/plugins/pagination/
```java
@Data
public class PageParam<T> extends Page<T> {
    // 默认页码
    private static final long DEFAULT_CURRENT = 1;
    // 默认每页条数
    private static final long DEFAULT_SIZE = 10;

    // 扩展字段：其他查询条件参数可以添加到这里
    private String orderBy; // 排序字段
    private boolean asc;    // 是否升序

    // 默认构造函数
    public PageParam() {
        super(DEFAULT_CURRENT, DEFAULT_SIZE);
    }

    // 自定义构造函数
    public PageParam(long current, long size) {
        super(current, size);
    }

    public IPage<T> getPage() {
        return new Page<>(getCurrent(), getSize());
    }

    // 添加更多的参数逻辑，如果需要的话
}
```

# 逻辑删除设置
```yaml
mybatis-plus:
  global-config:
    db-config:
      logic-delete-field: deleted # 全局逻辑删除字段名
      logic-delete-value: 1 # 逻辑已删除值
      logic-not-delete-value: 0 # 逻辑未删除值
```

# 字段拦截器
第一步，在实体类中，你需要使用 @TableField 注解来标记哪些字段需要自动填充，并指定填充的策略。
```java
@Data
public class BasePo {
    @TableField(fill = FieldFill.INSERT)
    LocalDateTime addTime;
    @TableField(fill = FieldFill.INSERT_UPDATE)
    LocalDateTime editTime;
    @TableField(fill = FieldFill.INSERT)
    String addUserCode;
    @TableField(fill = FieldFill.INSERT)
    String addUserName;
    @TableField(fill = FieldFill.INSERT_UPDATE)
    String editUserCode;
    @TableField(fill = FieldFill.INSERT_UPDATE)
    String editUserName;
}
```
第二步，创建一个类来实现 MetaObjectHandler 接口，并重写 insertFill 和 updateFill 方法。
```java
@Slf4j
@Component
public class MyMetaObjectHandler implements MetaObjectHandler {

    @Override
    public void insertFill(MetaObject metaObject) {
        LoginUserVo context = UserContextHolder.getInstance().getContext();
        log.info("开始插入填充...");
        this.strictInsertFill(metaObject, "addTime", LocalDateTime.class, LocalDateTime.now())
                .strictInsertFill(metaObject, "editTime", LocalDateTime.class, LocalDateTime.now())
                .strictInsertFill(metaObject, "addUserCode", String.class, context.getUserCode())
                .strictInsertFill(metaObject, "addUserName", String.class, context.getUserName())
                .strictInsertFill(metaObject, "editUserCode", String.class, context.getUserCode())
                .strictInsertFill(metaObject, "editUserName", String.class, context.getUserName());
        log.info("插入填充完成...");
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        LoginUserVo context = UserContextHolder.getInstance().getContext();
        log.info("开始更新填充...");
        this.strictUpdateFill(metaObject, "editTime", LocalDateTime.class, LocalDateTime.now())
                .strictUpdateFill(metaObject, "editUserCode", String.class, context.getUserCode())
                .strictUpdateFill(metaObject, "editUserName", String.class, context.getUserName());
        log.info("更新填充完成...");
    }
}
```
