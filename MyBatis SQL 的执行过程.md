# MyBatis SQL 的执行过程

## 一、目录

1、前言

2、配置文件加载

3、配置文件解析

4、SQL执行

5、结果集映射

6、Mybatis中的设计模式

7、总结

## 二、前言

1、mybatis框架图

![img](https://pic2.zhimg.com/80/v2-6f75b98e3a207d7c9bbb145432da06b1_720w.jpg)



如上为mybatis的框架图，在这篇文章中通过源码来重点看下数据处理层中的参数映射，SQL解析，SQL执行，结果映射

2、配置使用

获取mapper并操作数据库代码如下：

```java
InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");	
 SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
 SqlSession sqlSession = sqlSessionFactory.openSession();
 LiveCourseMapper mapper = sqlSession.getMapper(LiveCourseMapper.class);	
 List<LiveCourse> liveCourseList = mapper.getLiveCourseList();
```

## 三、配置文件加载

配置文件加载最终还是通过ClassLoader.getResourceAsStream来加载文件，关键代码如下：

```java
public static InputStream getResourceAsStream(ClassLoader loader, String resource) throws IOException {
InputStream in = classLoaderWrapper.getResourceAsStream(resource, loader);
if (in == null) {
  throw new IOException("Could not find resource " + resource);
}
return in;
```

}

```java
InputStream getResourceAsStream(String resource, ClassLoader[] classLoader) {
for (ClassLoader cl : classLoader) {
  if (null != cl) {

    // try to find the resource as passed
    InputStream returnValue = cl.getResourceAsStream(resource);

    // now, some class loaders want this leading "/", so we'll add it and try again if we didn't find the resource
    if (null == returnValue) {
      returnValue = cl.getResourceAsStream("/" + resource);
    }

    if (null != returnValue) {
      return returnValue;
    }
  }
}
return null;
```

}

## 四、配置文件解析

SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream); 我们以 SqlSessionFactoryBuilder为入口，看下mybatis是如何解析配置文件，并创建SqlSessionFactory的。SqlSessionFactoryBuilder.build方法实现如下：

```java
XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
 //解析出configuration对象，并创建SqlSessionFactory
 return build(parser.parse());
```

重点为解析configuration对象，然后根据configuration创建DefualtSqlSessionFactory。

1、解析configuration

```java
private void parseConfiguration(XNode root) {
try {
  Properties settings = settingsAsPropertiess(root.evalNode("settings"));
  //issue #117 read properties first
  propertiesElement(root.evalNode("properties"));
  loadCustomVfs(settings);
  typeAliasesElement(root.evalNode("typeAliases"));
  pluginElement(root.evalNode("plugins"));
  objectFactoryElement(root.evalNode("objectFactory"));
  objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
  reflectionFactoryElement(root.evalNode("reflectionFactory"));
  settingsElement(settings);
  // read it after objectFactory and objectWrapperFactory issue #631
  environmentsElement(root.evalNode("environments"));
  databaseIdProviderElement(root.evalNode("databaseIdProvider"));
  typeHandlerElement(root.evalNode("typeHandlers"));
  mapperElement(root.evalNode("mappers"));
} catch (Exception e) {
  throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
}
  }
```

通过XPathParser解析configuration节点下的properties，settings，typeAliases，plugins，objectFactory，objectWrapperFactory，reflectionFactory，environments，databaseIdProvider，typeHandlers，mappers这些节点。解析过程大体相同，都是通过XPathParser解析相关属性、子节点，然后创建相关对象，并保存到configuration对象中。这块代码相对简单，大家阅读起来没什么难度。

（1）解析properties 解析properties，并设置到configuration对象下的variables属性，protected Properties variables = new Properties();

（2）解析settings 解析settings配置，如lazyLoadingEnabled（默认false），defaultExecutorType（默认SIMPLE），jdbcTypeForNull（默认OTHER），callSettersOnNulls（默认false）

（3）解析typeAliases 通过typeAliasRegistry来注册别名，别名通过key，value的方式来进行存储，mybatis默认会创建一些基础类型的别名，如string->String.class，int->Integer.class，map->Map.class，hashmap->HashMap.class，list->List.class。别名和class关系通过HashMap来存储，

```java
private final Map<String, Class<?>> TYPE_ALIASES = new HashMap<String, Class<?>>();
```

（4）解析plugins 解析插件，然后设置Configuration的InterceptorChain。 Configuration： protected final InterceptorChain interceptorChain = new InterceptorChain();

InterceptorChain：

```java
private final List<Interceptor> interceptors = new ArrayList<Interceptor>();
public void addInterceptor(Interceptor interceptor) {
    interceptors.add(interceptor);
}
```

在创建的时候构造了拦截器链，在执行的时候也会经过拦截器链，此处为典型的责任链模式

（5）解析objectFactory 可以自定义ObjectFactory，对象工厂，默认为DefaultObjectFactory

（6）解析objectWrapperFactory 默认为DefaultObjectWrapperFactory

（7）reflectionFactory 反射工厂，在通过反射创建对象时（如结果集对象），可以通过自定义的反射工厂来创建对象。objectFactory，objectWrapperFactory，reflectionFactory这又是典型的工厂模式，将对象的创建交由相应的工厂来创建。

（8）databaseIdProvider 用来支持不同的数据库，很少在项目中用到

（9）解析typeHandlers 解析TypeHandler并通过typeHandlerRegistry注册到configuration中，通过TYPE_HANDLER_MAP保存typeHandler：

```java
private final Map<Type, Map<JdbcType, TypeHandler<?>>> TYPE_HANDLER_MAP = new HashMap<Type, Map<JdbcType, TypeHandler<?>>>();复制代码
```

（10）解析mappers 读取通过url指定的配置文件，然后通过XmlMapperBuilder进行解析

2、解析mapper

解析mapper的入口为XmlMapperBuilder.parse方法，在解析的时候会解析cache-ref，cache，parameterMap，resultMap，sql，select|insert|update|delete。cache-ref，cache和缓存相关，parameterMap目前已很少使用，这里就不再说明了。

2.1、解析resultMap

入口方法为XmlMapperBuilder.resultMapElement，解析resultMap主要包含如下步骤：

（1）解析resultMap属性

解析id，type，autoMapping属性，type取值的优先级为 type -> ofType -> resultType -> javaType

```java
String type = resultMapNode.getStringAttribute("type",
    resultMapNode.getStringAttribute("ofType",
        resultMapNode.getStringAttribute("resultType",
            resultMapNode.getStringAttribute("javaType"))));
```

（2）解析resultMap下的result子节点，创建ResultMapping对象

```java
resultMappings.add(buildResultMappingFromContext(resultChild, typeClass, flags));
```

解析result节点的property，column，javaType，jdbcType，select，resultMap，notNullColumn，typeHandler，resultSet，foreignColumn，lazy属性。

此处需要注意的点为：解析select属性与resultMap属性，因为这块涉及嵌套查询与嵌套映射（后面在结果集映射时会讲下这块）。如果result节点中存在select属性则认为是嵌套查询，而嵌套映射的判断条件如下：

```java
String nestedResultMap = context.getStringAttribute("resultMap",
    processNestedResultMappings(context, Collections.<ResultMapping> emptyList()));
```

如果result节点存在resultMap则肯定是嵌套映射

```java
private String processNestedResultMappings(XNode context, List<ResultMapping> resultMappings) throws Exception {
if ("association".equals(context.getName())
    || "collection".equals(context.getName())
    || "case".equals(context.getName())) {
  if (context.getStringAttribute("select") == null) {
    ResultMap resultMap = resultMapElement(context, resultMappings);
    return resultMap.getId();
  }
}
return null;
```

}

如果是association，collection，case这些节点，并且select属性为空的话，则认为是嵌套映射

（3）注册ResultMap 通过resultMapResolver.resolve()来解析resultMap属性，然后创建ResultMap对象，并保存到resultMaps 属性中。

```java
protected final Map<String, ResultMap> resultMaps = new StrictMap<ResultMap>("Result Maps collection");复制代码
```

2.2、解析sql

sql解析相对简单，主要是解析sql节点，然后保存到sqlFragments

2.3、解析select|insert|update|delete

入口方法为XMLStatementBuilder.parseStatementNode，解析statementNode主要包含如下步骤：

（1）解析statementNode属性 属性主要包括id，parameterMap，parameterType，resultMap，resultType，statementType（默认为PREPARED，预处理的statement）

（2）解析include 将include替换为sql片段，然后移除include节点

（3）解析selectKey Parse selectKey after includes and remove them.

（4）创建sqlSource SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass); langDriver默认为XMLLanguageDriver，此处很重要，请允许我多列点代码

```java
public SqlSource createSqlSource(Configuration configuration, XNode script, Class<?> parameterType) {
XMLScriptBuilder builder = new XMLScriptBuilder(configuration, script, parameterType);
return builder.parseScriptNode();
```

}

XMLScriptBuilder.parseScriptNode：

```java
public SqlSource parseScriptNode() {
    List<SqlNode> contents = parseDynamicTags(context);
    MixedSqlNode rootSqlNode = new MixedSqlNode(contents);
    SqlSource sqlSource = null;
    if (isDynamic) {
      sqlSource = new DynamicSqlSource(configuration, rootSqlNode);
    } else {
      sqlSource = new RawSqlSource(configuration, rootSqlNode, parameterType);
    }
    return sqlSource;
  }
```

1、解析动态节点

```java
List<SqlNode> parseDynamicTags(XNode node) {
    List<SqlNode> contents = new ArrayList<SqlNode>();
    NodeList children = node.getNode().getChildNodes();
    for (int i = 0; i < children.getLength(); i++) {
      XNode child = node.newXNode(children.item(i));
      if (child.getNode().getNodeType() == Node.CDATA_SECTION_NODE || child.getNode().getNodeType() == Node.TEXT_NODE) {
        String data = child.getStringBody("");
        TextSqlNode textSqlNode = new TextSqlNode(data);
        //如果包含${}的话则认为是动态节点
        if (textSqlNode.isDynamic()) {
          contents.add(textSqlNode);
          isDynamic = true;
        } else {
          contents.add(new StaticTextSqlNode(data));
        }
      } else if (child.getNode().getNodeType() == Node.ELEMENT_NODE) { // issue #628
        String nodeName = child.getNode().getNodeName();
        NodeHandler handler = nodeHandlers(nodeName);
        if (handler == null) {
          throw new BuilderException("Unknown element <" + nodeName + "> in SQL statement.");
        }
        handler.handleNode(child, contents);
        isDynamic = true;
      }
    }
    return contents;
  }
```

如果statement节点下存在子节点，如trim，if，where，那么statement肯定是动态节点；如果statement节点下不存在子节点，但是文本中包含${}，那么也任务是动态节点。

2、创建SqlSource

如果包含动态节点创建DynamicSqlSource，否则创建RawSqlSource

（5）创建MappedStatement并注册

根据解析出的属性创建MappedStatement对象，然后注册到configuration对象中

```java
protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>("Mapped Statements collection");复制代码
```

## 五、SQL执行

在配置文件解析这一节，我们解析了configuration，mapper等节点，并创建了SqlSessionFactory，下面我们就来分析下SQL执行的过程。

（1）创建SqlSession

```java
SqlSession sqlSession = sqlSessionFactory.openSession();

final Environment environment = configuration.getEnvironment();
  final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
  tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
  final Executor executor = configuration.newExecutor(tx, execType);
  return new DefaultSqlSession(configuration, executor, autoCommit);
```

因为没有和spring进行整合，事务为JdbcTransaction，executor为默认的SimpleExecutor，autoCommit为false

（2）创建mapper代理类

我们顺着DefaultSqlSession.getMapper方法来看下mybatis是如何创建mapper代理类的，

```java
public <T> T getMapper(Class<T> type) {
    return configuration.<T>getMapper(type, this);
  }

  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }

  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```

可以看到最终是会通过mapperProxyFactory来创建MapperProxy代理类，实现代码如下：

```java
public T newInstance(SqlSession sqlSession) {
	final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
	return newInstance(mapperProxy);
}

protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }
```

通过jdk动态代理来创建最终的Proxy代理类，最终类结构如下图所示：



![img](https://pic2.zhimg.com/80/v2-fe3f5b7fb546a6e2733dbbda8f9ea1c1_720w.jpg)



MapperProxy实现InvocationHandler接口，在执行mapper方法的时候实际执行的是MapperProxy的invoke方法（对动态代理有疑问的同学可以自行补习下）。

（3）调用mapper方法

MapperProxy.invoke方法实现如下：

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if (Object.class.equals(method.getDeclaringClass())) {
      try {
        return method.invoke(this, args);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }
```

如果执行的是Object类的方法，那么直接执行方法即可；其它方法的话通过MapperMethod来执行。实现如下：

1. 如果是insert命令，则调用sqlSession.insert方法；
2. 如果是update命令，则调用sqlSession.update方法；
3. 如果是delete命令，则调用sqlSession.delete方法；
4. 如果是select命令，相对insert，update，delete命令来说稍微复杂些，要区分方法的返回值，如果返回List集合的话则调用executeForMany，如果返回单个对象的话则调用selectOne，返回map的话则调用executeForMap

insert，update，delete，select命令它们实现原理都差不多，select只是比它们多了结果集映射这一步，我们就以select命令的executeForMany方法为例来说明sql的执行过程。

MapperMethod.executeMany会调用DefaultSqlSession.selectList，而selectList方法实现如下：

```java
//获取MappedStatement，在mapper解析的时候注册到configuration对象中的
MappedStatement ms = configuration.getMappedStatement(statement);
//默认为SimpleExecutor，sql的执行类
return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
```

Executor.query：

```java
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
//获取BoundSql，在此处处理if，where，choose动态节点，很重要
BoundSql boundSql = ms.getBoundSql(parameter);
CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
```

}

4.1、getBoundSql

```java
public class BoundSql {
  private String sql;
  private List<ParameterMapping> parameterMappings;
  private Object parameterObject;
  private Map<String, Object> additionalParameters;
  private MetaObject metaParameters;
```

BoundSql为最终执行的sql，为处理完动态节点后的sql。通过SqlSource来获取BoundSql，通过前面我们了解到存在两种SqlSource：DynamicSqlSource，RawSqlSource

4.1.1、DynamicSqlSource.getBoundSql

```java
public BoundSql getBoundSql(Object parameterObject) {
    DynamicContext context = new DynamicContext(configuration, parameterObject);
    rootSqlNode.apply(context);
    SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
    Class<?> parameterType = parameterObject == null ? Object.class : parameterObject.getClass();
    SqlSource sqlSource = sqlSourceParser.parse(context.getSql(), parameterType, context.getBindings());
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    for (Map.Entry<String, Object> entry : context.getBindings().entrySet()) {
      boundSql.setAdditionalParameter(entry.getKey(), entry.getValue());
    }
    return boundSql;
}
```

在getBoundSql时主要包含如下几个步骤：

（1）SqlNode.apply

```java
public boolean apply(DynamicContext context) {
    for (SqlNode sqlNode : contents) {
      sqlNode.apply(context);
    }
    return true;
  }
```

在此处处理IfSqlNode，MixedSqlNode，ForEachSqlNode，TrimSqlNode这些动态节点

（2）sqlSourceParser.parse

```java
public SqlSource parse(String originalSql, Class<?> parameterType, Map<String, Object> additionalParameters) {
ParameterMappingTokenHandler handler = new ParameterMappingTokenHandler(configuration, parameterType, additionalParameters);
GenericTokenParser parser = new GenericTokenParser("#{", "}", handler);
//将#{}替换为?，解析出ParameterMappings
String sql = parser.parse(originalSql);
return new StaticSqlSource(configuration, sql, handler.getParameterMappings());
```

}

解析SqlSource，将#{}替换为?，解析出ParameterMappings，最终生成静态的StaticSqlSource

```java
public String handleToken(String content) {
  parameterMappings.add(buildParameterMapping(content));
  return "?";
}
```

ParameterMapping主要包括property名称，jdbcType，javaType，typeHandler。如果未指定javaType的话默认取得是传递的参数对象中属性的类型。

StaticSqlSource.getBoundSql最终返回结果如下：

```java
public BoundSql getBoundSql(Object parameterObject) {
  return new BoundSql(configuration, sql, parameterMappings, parameterObject);
}
```

4.1.2、RawSqlSource.getBoundSql

RawSqlSource相比DynamicSqlSource就简单多了，在创建RawSqlSource时直接就将sql解析了，getBoundSql时直接创建BoundSql返回即可：

```java
public RawSqlSource(Configuration configuration, SqlNode rootSqlNode, Class<?> parameterType) {
    this(configuration, getSql(configuration, rootSqlNode), parameterType);
  }

  public RawSqlSource(Configuration configuration, String sql, Class<?> parameterType) {
    SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
    Class<?> clazz = parameterType == null ? Object.class : parameterType;
    sqlSource = sqlSourceParser.parse(sql, clazz, new HashMap<String, Object>());
  }

  private static String getSql(Configuration configuration, SqlNode rootSqlNode) {
    DynamicContext context = new DynamicContext(configuration, null);
    rootSqlNode.apply(context);
    return context.getSql();
  }


  public BoundSql getBoundSql(Object parameterObject) {
    return sqlSource.getBoundSql(parameterObject);
  }
```

4.2、query

在上面的小节中生成了最终的sql，下面就可以执行sql了。我们以SimpleExecutor为例来看下sql的执行过程：

```java
Configuration configuration = ms.getConfiguration();
  //创建StatementHandler，默认为PreparedStatementHandler
  StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
  stmt = prepareStatement(handler, ms.getStatementLog());
  return handler.<E>query(stmt, resultHandler);
```

（1）prepareStatement

```java
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException { Statement stmt;
Connection connection = getConnection(statementLog);
//设置fetchSize，timeout
stmt = handler.prepare(connection, transaction.getTimeout());
//statement.setParameter sql实际执行参数设置
handler.parameterize(stmt);
return stmt;
```

}

```java
public void parameterize(Statement statement) throws SQLException {
   parameterHandler.setParameters((PreparedStatement) statement);
}
```

最终通过typeHandler.setParameter(ps, i + 1, value, jdbcType);来设置参数

（2）query

```java
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
  PreparedStatement ps = (PreparedStatement) statement;
  ps.execute();  //sql执行
  return resultSetHandler.<E> handleResultSets(ps); //处理结果集
}
```

处理结果集也块相对也比较重要，我们单独来讲下。

## 五、结果集映射

方法入口为DefaultResultSetHandler.handleResultSets，关键代码如下：

```java
public void handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
    if (resultMap.hasNestedResultMaps()) {
      ensureNoRowBounds();
      checkResultHandler();
      handleRowValuesForNestedResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    } else {
      handleRowValuesForSimpleResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    }
  }
```

在处理结果集行值时分为两部分，处理简单resultMap对应的行值和处理嵌套resultMap对应的行值，是否嵌套映射在解析mapper resultMap的时候已经解释过了，这里不再重复。处理简单resultMap对应的行值稍微简单些，我们先从简单的看起吧

5.1、简单映射

```java
private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)
  throws SQLException {
DefaultResultContext<Object> resultContext = new DefaultResultContext<Object>();
//处理分页，跳过指定的行，如果rs类型不是TYPE_FORWARD_ONLY，直接absolute，否则的话循环rs.next
skipRows(rsw.getResultSet(), rowBounds);
//循环处理结果集，获取下一行值
while (shouldProcessMoreRows(resultContext, rowBounds) && rsw.getResultSet().next()) {
  ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(rsw.getResultSet(), resultMap, null);
  //处理行值，重点分析
  Object rowValue = getRowValue(rsw, discriminatedResultMap);
  //保存对象，通过list保存生成的对象Object
  storeObject(resultHandler, resultContext, rowValue, parentMapping, rsw.getResultSet());
}
```

}

5.1.1、getRowValue

```java
private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap) throws SQLException {
final ResultLoaderMap lazyLoader = new ResultLoaderMap();
Object resultObject = createResultObject(rsw, resultMap, lazyLoader, null);
if (resultObject != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
  final MetaObject metaObject = configuration.newMetaObject(resultObject);
  boolean foundValues = !resultMap.getConstructorResultMappings().isEmpty();
  if (shouldApplyAutomaticMappings(resultMap, false)) {
    foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, null) || foundValues;
  }
  foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, null) || foundValues;
  foundValues = lazyLoader.size() > 0 || foundValues;
  resultObject = foundValues ? resultObject : null;
  return resultObject;
}
return resultObject;
```

}

获取行值主要包含如下3个步骤：

（1）createResultObject创建结果集对象 根据resultType，通过ObjectFactory.create来创建对象，其实现原理还是通过反射来创建对象。在创建对象时如果resultMap未配置constructor，通过默认构造方法来创建对象，否则通过有参的构造方法来创建对象

（2）自动映射属性 如果ResultMap配置了autoMapping="true"，或者AutoMappingBehavior为PARTIAL会自动映射在resultSet查询列中存在但是未在resultMap中配置的列。

（3）人工映射属性 映射在resultMap中配置的列，主要包括两步：获取属性的值和设置属性的值。

```java
//获取属性的值
Object value = getPropertyMappingValue(rsw.getResultSet(), metaObject, propertyMapping, lazyLoader, columnPrefix);
//设置属性的值,通过反射来设置
metaObject.setValue(property, value);
```

获取属性的值：

```java
private Object getPropertyMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix) throws SQLException {
//获取嵌套查询对应的属性值，最终还是通过Executor.query来获取属性值
if (propertyMapping.getNestedQueryId() != null) {
  return getNestedQueryMappingValue(rs, metaResultObject, propertyMapping, lazyLoader, columnPrefix);
} else if (propertyMapping.getResultSet() != null) {
  addPendingChildRelation(rs, metaResultObject, propertyMapping);   // TODO is that OK?
  return DEFERED;
} else {
  final TypeHandler<?> typeHandler = propertyMapping.getTypeHandler();
  final String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
  //通过typeHandler来获取属性的值，如StringTypeHandler获取属性值：rs.getString(columnName)
  return typeHandler.getResult(rs, column);
}
```

}

5.2、嵌套映射

嵌套resultMap主要用来处理collection，association属性，并且select属性为空，如：

```java
<resultMap id="liveCourseMap" type="com.jd.mybatis.entity.LiveCourse">
		<result column="id" property="id"></result>
		<result column="course_name" property="courseName"></result>
		<!-- 通过嵌套映射来获取关联属性的值 -->
		<collection property="users" ofType="com.jd.mybatis.entity.LiveCourseUser">
			<result column="uid" property="id"></result>
			<result column="user_name" property="userName"></result>
			<result column="id" property="liveCourseId"></result>
		</collection>
</resultMap>
```

处理嵌套映射主要包括如下几个步骤：

（1）skipRows(rsw.getResultSet(), rowBounds); 同简单映射

（2）createRowKey，根据resultMap下的列创建rowKey，很有用。在如上liveCourseMap配置中，mybatis将会根据id列和course_name列的值来创建rowKey，类似于类似于-1421739516:769980325:com.jd.mybatis.mapper.LiveCourseMapper.liveCourseMap:id:121:course_name:j2ee

（3）getRowValue

这块代码稍微有点绕，我通过例子来说明吧。

```java
select l.id,course_name,u.id uid,u.user_name from jdams_school_live l left join jdams_school_live_users u on l.id = u.live_id where l.yn =1 and l.id = 121 order by course_start_time复制代码
```

我的sql很简单，查询出课程和参加课程的用户，结果集如下：



![img](https://pic2.zhimg.com/80/v2-536c146474cd7f8d61c4431e05c9b1f9_720w.jpg)



mybatis的处理过程为：

1、处理第一行的值，创建LiveCourse对象（id=121，courseName=j2ee），同时创建User对象（id=1，userName=张三），并放到List中，然后设置LiveCourse的users属性

2、处理第二行的值，因为在第一行已经创建了LiveCourse对象，所以这一次不会再创建LiveCourse对象，根据rowKey来判断创没创建LiveCourse对象（创建完对象会保存）

3、创建User对象（id=2，userName=李四），然后放到List中

大体过程如上所述，我们再来看下相应的源码：

```java
//创建rowKey，根据rowKey判断对应创建没创建
final CacheKey rowKey = createRowKey(discriminatedResultMap, rsw, null);
//创建的LiveCourse对象会保存到nestedResultObjects
Object partialObject = nestedResultObjects.get(rowKey);

 rowValue = getRowValue(rsw, discriminatedResultMap, rowKey, null, partialObject);
 //只有是第一次创建LiveCourse时才会进行保存
 if (partialObject == null) {
     storeObject(resultHandler, resultContext, rowValue, parentMapping, rsw.getResultSet());
    }
```

getRowValue：

```java
Object resultObject = partialObject;
//如果已经创建LiveCoure对象
if (resultObject != null) {
  final MetaObject metaObject = configuration.newMetaObject(resultObject);
  putAncestor(resultObject, resultMapId, columnPrefix);
  //不用创建LiveCouse对象，直接处理嵌套映射即可
  applyNestedResultMappings(rsw, resultMap, metaObject, columnPrefix, combinedKey, false);
  ancestorObjects.remove(resultMapId);
} else {
  final ResultLoaderMap lazyLoader = new ResultLoaderMap();
  //创建LiveCoure对象，同简单映射
  resultObject = createResultObject(rsw, resultMap, lazyLoader, columnPrefix);
  if (resultObject != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
    final MetaObject metaObject = configuration.newMetaObject(resultObject);
    boolean foundValues = !resultMap.getConstructorResultMappings().isEmpty();
    //自动映射，同简单映射
    if (shouldApplyAutomaticMappings(resultMap, true)) {
      foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, columnPrefix) || foundValues;
    }
    //人工映射，同简单映射
    foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, columnPrefix) || foundValues;
    putAncestor(resultObject, resultMapId, columnPrefix);
    //处理嵌套映射
    foundValues = applyNestedResultMappings(rsw, resultMap, metaObject, columnPrefix, combinedKey, true) || foundValues;
    ancestorObjects.remove(resultMapId);
    foundValues = lazyLoader.size() > 0 || foundValues;
    resultObject = foundValues ? resultObject : null;
  }
  if (combinedKey != CacheKey.NULL_CACHE_KEY) {
    nestedResultObjects.put(combinedKey, resultObject);
  }
}
```

在处理嵌套映射属性时，主要是创建对象，设置属性值，然后添加到外层对象的colletion属性中

```java
private void linkObjects(MetaObject metaObject, ResultMapping resultMapping, Object rowValue) {
final Object collectionProperty = instantiateCollectionPropertyIfAppropriate(resultMapping, metaObject);
//如果外层对象已经有集合属性值时，直接将创建的对象添加到集合中
if (collectionProperty != null) {
  final MetaObject targetMetaObject = configuration.newMetaObject(collectionProperty);
  targetMetaObject.add(rowValue);
} else {
  //创建集合，然后设置属性值
  metaObject.setValue(resultMapping.getProperty(), rowValue);
}
```

}

（4）storeObject

保存创建的对象，同简单映射

## 六、Mybatis中的设计模式

（1）工厂模式 SqlSessionFactory

（2）动态代理 MapperProxy

（3）组合模式 SqlNode，MixSqlNode，处理SqlNode

（4）模板方法

（5）建造者模式

（6）责任链模式 plugins插件处理