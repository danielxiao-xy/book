# 常用的多租户方案对比，以及用mybatis-plus插件实现多租户功能

### 技术栈
spring boot + postgresql + mybatis-plus

### 前菜
多租户的场景，大概率是要考虑以下问题的
1. 数据安全级别，和私有化部署能力
2. 是否会有二次开发，客户定制化
3. 开发成本和运维成本取舍（人工成本和硬件成本）
4. 租户间是否会有数据交互
5. 出现故障、遇到性能瓶颈，会不会相互影响


**直接上结论**

| 对比维度       | 独立数据库 | 共享数据库、独立schema | 共享数据库、共享数据架构 |
|------------|-------|----------------|--------------|
| 开发成本       | 低     | 一般             | 高            | 
| 运维成本       | 高     | 一般             | 低            | 
| 隔离性/安全性    | 高     | 一般             | 低            | 
| 租户间交互能力    | 低     | 一般             | 高            |
| 定制化空间      | 高     | 一般             | 低            |
| 可支持的最大租户数量 | 高     | 一般             | 高            |

## 快速实现多租户（共享数据库、共享数据架构的方式）

1. 步骤1：网关识别租户身份后，放在header中给到应用
2. 步骤2：应用中适配怎么区分多租户
3. 步骤3：数据库层面区分多租户

### 步骤1：应用中保留多租户信息
```java

/**
 * 用ThreadLocal保存租户信息
 */
public class TenantContext {

  private static final String tenantId;

  private static final ThreadLocal<String> currentTenant = new InheritableThreadLocal<>();
  
  public static String getCurrentTenantId() {
    return this.tenantId;
  }

  public static void setCurrentTenant(String tenantId) {
    this.tenantId=tenantId;
  }

  public static void clear() {
    currentTenant.remove();
  }
}


/**
 * 写一个Filter，从header中读取租户信息
 */
public class TenantFilter implements Filter {

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        String tenantId=getHeaderOrParam(servletRequest, "网关中增加的租户key").orElse("defaultTenantId");
        TenantContext.setCurrentTenant(tenantId);
        filterChain.doFilter(servletRequest, servletResponse);
    }

    private Optional<String> getHeaderOrParam(ServletRequest request, TenantEnum code) {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        return Optional.ofNullable(httpRequest.getHeader(code.getValue()) == null ?
                httpRequest.getParameter(code.getValue()) :
                httpRequest.getHeader(code.getValue()));
    }
}
```

### 步骤2：启用mybatis的多租户插件
```java

/**
 * 启用多租户插件
 */
@Configuration
public class MyBatisConfig {

  @Bean
  public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    interceptor.addInnerInterceptor(new TenantLineInnerInterceptor(new SystemTenantLineHandler()));
    return interceptor;
  }

  @Bean
  public CommentIntegrator commonInterceptor() {
    return new CommentIntegrator();
  }
}


/**
 * 租户功能配置
 */
public class HrCoreTenantLineHandler implements TenantLineHandler {
    @Override
    public Expression getTenantId() {
        return new StringValue(TenantContext.getCurrentTenant());
    }

    @Override
    public String getTenantIdColumn() {
        //这里对应的是数据库的列名
        return "tenant_id";
    }

    @Override
    public boolean ignoreTable(String tableName) {
        //如果那些表不需要做租户隔离的，在这里配置
        return false;
    }

    @Override
    public boolean ignoreInsert(List<Column> columns, String tenantIdColumn) {
        return TenantLineHandler.super.ignoreInsert(columns, tenantIdColumn);
    }
}

/**
 * entity 和 mapper
 */

@Data
class School{
    private  String id;
    private  String name;
}

@Data
class Student{
    private  String id;
    private  String school_id;
    private  String name;
}

<select id="findStudent" resultType="Student">
        SELECT *  FROM findStudent WHERE 1
</select>

```


### 步骤3：数据库设计
就用school表举例，每一张数据库表都需要加上tenant_id这一列，记住是每一张，每一张，每一张

| id  | name | tenant_id       |
|-----|------|-----------------|
| 1   | 实验三中 | defaultTenantId |


## mybatis-plus 实现多租户的原理解析

**mybatis会捕获 增删改查的sql，根据sql的类型，修改sql**

| 核心逻辑     | 原sql                                    | 插件会改成                                                     |
|----------|-----------------------------------------|-----------------------------------------------------------|
| select逻辑 | select * from School where 1            | select * from School where 1 and tenant_id='xxx'          |
| delete逻辑 | delete from School where 1              | delete from School where 1 and tenant_id='xxx'            |
| update逻辑 | update School set name ='xxx' where 1   | update School set name ='xxx' where 1 and tenant_id='xxx' |
| create逻辑 | insert into School(id,name) value (1,2) | insert into School(id,name,tenant_id) value (1,2,'xxx')   |

这里 mybatis-plus租户插件，代码的三分之二都在处理最复杂的join查询

```java
    例如：
    select * from Student a left join School b on a.school_id=b.id where b.name='xxx'

    会被处理成 
     select * from Student a
        left join  (select * from  School  where tenant_id='xxx')
        b on a.school_id=b.id where b.name='xxx' and a.tenant_id='xxx'
    

     所以这里无论是左表还是右表都加上了租户条件
```