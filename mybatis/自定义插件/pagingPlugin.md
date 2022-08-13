# mybatis-plus实现自定义插件

## 分页插件是干什么的
1. 当写sql的时候，不需要实现 limit 和 offset语句
2. 不需要重复实现 select count(1) 的逻辑

## 使用mybatis默认的分页插件

#### 步骤1.启用mybatis插件config
```java
//下面这两个是对应的包路径
import com.baomidou.mybatisplus.extension.plugins.MybatisPlusInterceptor;
import com.baomidou.mybatisplus.extension.plugins.inner.PaginationInnerInterceptor;

@Configuration
public class MyBatisConfig {
    
  @Bean
  public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    //配置需要启动的插件
    interceptor.addInnerInterceptor(new PaginationInnerInterceptor());
    return interceptor;
  }

  @Bean
  public CommentIntegrator commonInterceptor() {
    return new CommentIntegrator();
  }
}

```
#### 步骤2.定义mapper的接口
```java

  //查询所有组织架构（使用分页插件）
  IPage<OrganizationInfo> getOrgList(@Param("page") IPage<?> page,@Param("condition") QueryCondition<condition> condition);

  //查询所有组织架构（不使用分页插件）
  List<OrganizationInfo> getOrgListWithoutPaging(@Param("condition") QueryCondition<condition> condition);
```

#### 步骤3.实现sql
```java
<select id="getOrgList" resultType="xxx.xxx.xxx.OrganizationInfo">
    SELECT *  FROM OrganizationInfo WHERE xxxx
</select>

<select id="getOrgListWithoutPaging" resultType="xxx.xxx.xxx.OrganizationInfo">
        SELECT *  FROM OrganizationInfo WHERE xxxx
</select>
```
####步骤4.对别区别
**mapper.xml的实现方式是一模一样的，使用分页插件的话，不需要在sql中写 limit和offset信息会有插件自动注入**


## 自定义分页插件（直接放结果）
1. pageSize(Integer) : 当前页面的大小，当pageSize<=0时，不分页
2. pageIndex(Integer): 当前第几页，当pageIndex<=0时，表示查询第一页
3. 是否分页，每种情况的默认值，是否计数等，参考以下用例

#### 举例说明

| pageSize | pageIndex | 含义         | 对应SQL              | 是否count(*)           |
|----------|-----------|------------|--------------------|----------------------|
| 0        | 0         | 不分页        | 不分页                | 是，pageCount=1        |
| 0        | 1         | 不分页        | 不分页                | 是，pageCount=1        |
| 0        | -1        | 不分页        | 不分页                | 是，pageCount=1        |
| -1       | 0         | 不分页        | 不分页                | 是，pageCount=1        |
| -1       | 1         | 不分页        | 不分页                | 是，pageCount=1        |
| -1       | -1        | 不分页        | 不分页                | 是，pageCount=1        |
| 1        | 0         | 第一页，大小1    | limit 1 offset 0   | 是，pageCount=count/1  |
| 1        | -1        | 第一页，大小1    | limit 1 offset 0   | 是，pageCount=count/1  |
| 1        | 1         | 第一页，大小1    | limit 1 offset 0   | 是，pageCount=count/1  |
| 10       | 2         | 第2页，大小10   | limit 10 offset 10 | 是，pageCount=count/10 |
| null     | null      | 默认第一页，大小50 | limit 50 offset 0  | 是，pageCount=count/50 |
| -2       | 1         | 不分页        | 不分页                | 否，pageCount=1        |

#### 实现原理，自定义分页插件
1. 我的思路：先研究mybatis-plus的插件是怎么写的，然后自己找到关键的地方，进行自定义逻辑的补全
2. mybatis的分页插件做了那些事，1sql注入+select count(*) 计数
3. 所以按照以上思路，需要控制注入逻辑和count逻辑

```java
/*展示最关键的几个步骤*/
public class PaginationInnerInterceptor implements InnerInterceptor{
    
  public boolean willDoQuery(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    IPage<?> page = (IPage)ParameterUtils.findPage(parameter).orElse(null);
    
    //这里修改了mybatis的分页默认逻辑
    if (page != null && page.getSize() >= -1L && page.searchCount()) {
      MappedStatement countMs = this.buildCountMappedStatement(ms, page.countId());
      BoundSql countSql;
      if (countMs != null) {
        countSql = countMs.getBoundSql(parameter);
      } else {
        countMs = this.buildAutoCountMappedStatement(ms);
        String countSqlStr = this.autoCountSql(page, boundSql.getSql());
        MPBoundSql mpBoundSql = PluginUtils.mpBoundSql(boundSql);
        countSql = new BoundSql(countMs.getConfiguration(), countSqlStr, mpBoundSql.parameterMappings(), parameter);
        PluginUtils.setAdditionalParameter(countSql, mpBoundSql.additionalParameters());
      }

      CacheKey cacheKey = executor.createCacheKey(countMs, parameter, rowBounds, countSql);
      //这里是mybatis去执行count的核心步骤
      List<Object> result = executor.query(countMs, parameter, rowBounds, resultHandler, cacheKey, countSql);
      long total = 0L;
      if (CollectionUtils.isNotEmpty(result)) {
        Object o = result.get(0);
        if (o != null) {
          total = Long.parseLong(o.toString());
        }
      }

      page.setTotal(total);
      return this.continuePage(page);
    } else {
      return true;
    }
  }

    /**
     * 这里判断是否需要继续执行select操作
     * @param page
     * @return
     */
  protected boolean continuePage(IPage<?> page) {
    if (page.getTotal() <= 0L) {
      //如果count的结果不大于0，那直接不执行了
      return false;
    } else {
      if (page.getCurrent() > page.getPages()) {
        if (!this.overflow) {
          return true;
        }
        this.handlerOverflow(page);
      }
      return true;
    }
  }
  
}

```