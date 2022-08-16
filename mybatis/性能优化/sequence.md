# mybatis-plus 性能优化：【大数据量mybatis序列化和反序列化慢的问题】

## 成果
400M数据，30w条，从80秒干到8秒

## 技术栈
springboot+mybatis plus +postgresql

## 抛转引玉
 本人在开发多个项目中，都遇到过同样的问题，有些 **数据量（超过20w条）条数多**的接口，接口特别慢，列举两个我碰到过的问题。
1. 用spring data mongo 读mongo数据库25w条的时候，query响应只需要100毫秒，但是接口却需要30秒左右（50个字段左右，400m数据）。
2. 用mybatis plus 查询数据库的时候，query只需要6秒，但是接口响应却需要90秒左右（60个字段左右，400m数据），

这两个问题出现的原因都是类似的，**数据库的框架在对象序列化**的过程中，花费了大量的时间，以下展示一下具体的解题思路


## 错误的方式请不要模仿
    当你看到这里的时候，请大胆质疑，为什么系统中会设计这样的接口，一个接口需要返回这么多条的数据？
    答案：这种大数据量的接口服务，很多情况都是未了兼容老的业务需求（数据同步、数据订阅等）
    如果你是在设计新的系统，请认真思考，请选择最正确的技术路线去解决问题，例如消息队列、流计算、cdc等

## 解题思路
- 方案1：用stream的方式读数据库。
- 方案2：用并行读，分段读（多线程）。
- 方案3：修改mybatis-plus的源码。


 ### 方案1的问题（数据结构限制了发挥空间）


```java
/**
 * 这个是我们的统一接口返回的结构，所以用流数，没有本质上解决问题，还是要等所有结果响应完毕后，才能返回给客户端
 * @param <T>
 */
public class ResultDTO<T> implements Serializable {
  
    private boolean success;
  
    private T data;
 
    private String message;

    private String code;
}
```

### 方案2的问题（直接贴源码）
1. 没有找到本质问题所在，mybatis是游标读，并行的可能性很低
2. 开发难度大，无端增加无用的开发工作


```java
  private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)
      throws SQLException {
        DefaultResultContext<Object> resultContext = new DefaultResultContext<>();
        ResultSet resultSet = rsw.getResultSet();
        skipRows(resultSet, rowBounds);

        //这里 !resultSet.isClosed() && resultSet.next() 就是mybatis-plus的游标读，一个一个读
      while (shouldProcessMoreRows(resultContext, rowBounds) && !resultSet.isClosed() && resultSet.next()) {
        ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(resultSet, resultMap, null);
        Object rowValue = getRowValue(rsw, discriminatedResultMap, null);
        storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
      }
    }
```


### 方案3的结果（直接上结果，400M数据，30w条）

| 方式   | mybatis-plus自带的序列化功能 | 自己手写序列化方式     | 硬编码hardCode |
|------|:---------------------|---------------|-------------|
| 数据耗时 | 80秒                  | 8秒            | 1秒          | 
| 缺点   | 大量反射操作，数据量大的时候很慢     | 自定义字段类型处理可能失败 | 硬编码，会经常改代码  | 
| 优点   | 简单，准确                | 快             | 很快很快        | 
| 推荐   | 推荐                   | 推荐            | 不是很推荐       | 

#### 原理：充分利用mybatis-plus的typeHandle+拒绝反射

```java
    //这个是处理mybatis-plus的序列化的主要类
   package org.apache.ibatis.executor.resultset.DefaultResultSetHandler;

   //关键1：将typeHandler的集合做一个缓存
    private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)
        throws SQLException {
        DefaultResultContext<Object> resultContext = new DefaultResultContext<>();
        ResultSet resultSet = rsw.getResultSet();
        skipRows(resultSet, rowBounds);
    
        //这里举个例子，当类为Student类的时候，先把typeHandler缓存，这里不用反射的方式，所以快了很多
        if (resultMap.getType().equals(Student.class)){
        Map<String, Field> logicPropertyField=new HashMap<>();
        Map<String, TypeHandler<?>>logicPropertyTypeHandler=new HashMap<>();


        List<Field> allPropertyField=new ArrayList<>();
        Class<?> temp=resultMap.getType();
        while(temp!=null){
            //获取对象的所有字段
            allPropertyField.addAll(Arrays.asList(temp.getDeclaredFields()));
            //处理对象的继承关系
            temp=temp.getSuperclass();
            temp=null;
        }

        //字段映射关系  数据库返回字段 to 逻辑字段
        List<ResultMapping> resultMappings=resultMap.getResultMappings();
        if(resultMappings!=null&&!resultMappings.isEmpty()){
        for(ResultMapping mappingRule:resultMappings){
        Field fieldTarget=allPropertyField.stream().filter(field->field.getName().equals(mappingRule.getProperty())).findFirst().orElse(null);
        if(fieldTarget!=null){
        //先做string验证
        fieldTarget.setAccessible(true);
            logicPropertyField.put(mappingRule.getColumn(),fieldTarget);
            logicPropertyTypeHandler.put(mappingRule.getColumn(),mappingRule.getTypeHandler());
        }
        }
        }
        /*以下片段是result的demo用法，后续变动时，可以参考如下逻辑*/
        //ResultMapping demo=new ResultMapping();
        //demo.getProperty() //逻辑字段
        //demo.getColumn() //数据库返回的字段
        //demo.getJavaType() //逻辑字段类型
        //demo.getJdbcType() //数据库字段类型，可能为null
        //demo.getTypeHandler() // 该字段的typeHandler

        while(shouldProcessMoreRows(resultContext,rowBounds)&&!resultSet.isClosed()&&resultSet.next()){
            ResultMap discriminatedResultMap=resolveDiscriminatedResultMap(resultSet,resultMap,null);
            Object rowValue=getRowValue(rsw,discriminatedResultMap,null,logicPropertyField,logicPropertyTypeHandler);
            storeObject(resultHandler,resultContext,rowValue,parentMapping,resultSet);
        }
        
        }   
        else{
            while (shouldProcessMoreRows(resultContext, rowBounds) && !resultSet.isClosed() && resultSet.next()) {
            ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(resultSet, resultMap, null);
            //这里是原生的序列化方式
            Object rowValue = getRowValue(rsw, discriminatedResultMap, null);
            storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
            }
        }
        
    }


    // 展示一下用typeHandler处理的序列化逻辑
    // GET VALUE FROM ROW FOR SIMPLE RESULT MAP
    //
    private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap, String columnPrefix,Map<String,Field> logicPropertyField,Map<String,TypeHandler<?>> logicPropertyTypeHandler) throws SQLException {
        Object rowValue=null;
        try{
            rowValue=resultMap.getType().newInstance();
            ResultSet temp=rsw.getResultSet();
            //这里遍历可以有优化空间（遍历算法还可以提升）
            for (String columnName : logicPropertyField.keySet()) {
                Field field=logicPropertyField.get(columnName);
                if (logicPropertyTypeHandler.containsKey(columnName)){
                    field.set(rowValue,logicPropertyTypeHandler.get(columnName).getResult(temp,(columnName)));
                }else{
                    field.set(rowValue,temp.getObject(columnName));
                }
            }
        }catch (Exception e){
            //这里自定义你自己的异常逻辑
        }
        return rowValue;
    }


    //下面这个是硬编码的方式，虽然最快，但是改动起来会很痛苦
    private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap, String columnPrefix) throws SQLException {
    final ResultLoaderMap lazyLoader = new ResultLoaderMap();
            Object rowValue = createResultObject(rsw, resultMap, lazyLoader, columnPrefix);
            if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
             if (resultMap.getType().equals(Student.class)){
                //这个是硬编码的方式，通过getter setter直接赋值  
                ResultSet temp=rsw.getResultSet();
                Student dbItem=new Student();
                dbItem.setName(temp.getString("name"));
                rowValue=dbItem;
              }else{
                final MetaObject metaObject = configuration.newMetaObject(rowValue);
                boolean foundValues = this.useConstructorMappings;
                if (shouldApplyAutomaticMappings(resultMap, false)) {
                foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, columnPrefix) || foundValues;
                }
                foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, columnPrefix) || foundValues;
                foundValues = lazyLoader.size() > 0 || foundValues;
                rowValue = foundValues || configuration.isReturnInstanceForEmptyRow() ? rowValue : null;
             }
        }
        return rowValue;
    }
    
```

## 有没有更好的方式

这里提供一个破题思路
1. 充分利用mybatis-plus的上下文，如果不满足，自己写一个上下文，结合IPage分页查询使用
2. 分页操作是才query操作前的，所以可以先得知这次查询会返回多少条数据，根据返回数据的条数动态去选择序列化方式
3. 如果结果条数大于5000，用typeHandler缓存的方式序列化对象，否则用mybatis-plus自带的反射机制进行序列化（动态选择）
4. 看代码思路


```java
 //这个是mybatis-plus的分页插件源码
  public boolean willDoQuery(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) thr
         ows SQLException {
    IPage<?> page = (IPage)ParameterUtils.findPage(parameter).orElse(null);
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
      List<Object> result = executor.query(countMs, parameter, rowBounds, resultHandler, cacheKey, countSql);
      long total = 0L;
      if (CollectionUtils.isNotEmpty(result)) {
        Object o = result.get(0);
        if (o != null) {
          total = Long.parseLong(o.toString());
        }
      }
      //这里已经获取到数据总数了，在这里根据你的需求把序列化方式放在上下文中即可
      page.setTotal(total);
      
      //往下才会执行真正的query 语句，所以这种方式更优雅
      return this.continuePage(page);
    } else {
      return true;
    }
  }
```