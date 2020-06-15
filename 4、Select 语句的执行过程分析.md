## Select 语句的执行过程分析

#### MapperMethod 的 execute 方法如下：

```java
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    
    // 根据 SQL 类型执行相应的数据库操作
    switch (command.getType()) {
        case INSERT: {
            // 对用户传入的参数进行转换，下同
            Object param = method.convertArgsToSqlCommandParam(args);
            // 执行插入操作，rowCountResult 方法用于处理返回值
            result = rowCountResult(sqlSession.insert(command.getName(), param));
            break;
        }
        case UPDATE: {
            Object param = method.convertArgsToSqlCommandParam(args);
            // 执行更新操作
            result = rowCountResult(sqlSession.update(command.getName(), param));
            break;
        }
        case DELETE: {
            Object param = method.convertArgsToSqlCommandParam(args);
            // 执行删除操作
            result = rowCountResult(sqlSession.delete(command.getName(), param));
            break;
        }
        case SELECT:
            // 根据目标方法的返回类型进行相应的查询操作
            if (method.returnsVoid() && method.hasResultHandler()) {
                executeWithResultHandler(sqlSession, args);
                result = null;
            } else if (method.returnsMany()) {
                // 执行查询操作，并返回多个结果 
                result = executeForMany(sqlSession, args);
            } else if (method.returnsMap()) {
                // 执行查询操作，并将结果封装在 Map 中返回
                result = executeForMap(sqlSession, args);
            } else if (method.returnsCursor()) {
                // 执行查询操作，并返回一个 Cursor 对象
                result = executeForCursor(sqlSession, args);
            } else {
                Object param = method.convertArgsToSqlCommandParam(args);
                // 执行查询操作，并返回一个结果
                result = sqlSession.selectOne(command.getName(), param);
            }
            break;
        case FLUSH:
            // 执行刷新操作
            result = sqlSession.flushStatements();
            break;
        default:
            throw new BindingException("Unknown execution method for: " + command.getName());
    }
    return result;
}
```

#### selectOne 在内部会调用 selectList 方法：

```java
// 执行查询操作，并返回一个结果
// command.getName()和param，也就是 namespace.methodName（mapper.EmployeeMapper.getAll）和方法的运行参数。
result = sqlSession.selectOne(command.getName(), param);
```

#### DefaultSqlSession 中：

```java
public <T> T selectOne(String statement, Object parameter) {
    // 调用 selectList 获取结果
    List<T> list = this.<T>selectList(statement, parameter);
    if (list.size() == 1) {
        // 返回结果
        return list.get(0);
    } else if (list.size() > 1) {
        // 如果查询结果大于1则抛出异常
        throw new TooManyResultsException(
            "Expected one result (or null) to be returned by selectOne(), but found: " + list.size());
    } else {
        return null;
    }
}
```

```java
private final Executor executor;
public <E> List<E> selectList(String statement, Object parameter) {
    // 调用重载方法
    return this.selectList(statement, parameter, RowBounds.DEFAULT);
}

public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
        // 通过MappedStatement的Id获取 MappedStatement
        MappedStatement ms = configuration.getMappedStatement(statement);
        // 调用 Executor 实现类中的 query 方法
        return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```

​	从上可知，之前创建**DefaultSqlSession**的时候，是创建了一个Executor的实例作为其属性的，我们看到**通过MappedStatement的Id获取 MappedStatement后，就交由Executor去执行了。**

## 一、CachingExecutor：

```java
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    // 获取 BoundSql
    BoundSql boundSql = ms.getBoundSql(parameterObject);
   // 创建 CacheKey
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    // 调用重载方法
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

### 1、获取 BoundSql

​	调用了MappedStatement的getBoundSql方法，并将运行时参数传入其中。

​	SQL 是配置在映射文件中的，但由于映射文件中的 SQL 可能会包含占位符 #{}，以及动态 SQL 标签，比如 `<if>、<where>` 等。因此并不能直接使用映射文件中配置的 SQL。MyBatis 会将映射文件中的 SQL 解析成一组 SQL 片段。需要对这一组片段进行解析，从每个片段对象中获取相应的内容。然后将这些内容组合起来即可得到一个完成的 SQL 语句，这个完整的 SQL 以及其他的一些信息最终会存储在 BoundSql 对象中。

以下是 BoundSql 的属性信息：

```java
// 一个完整的 SQL 语句，可能会包含问号 ? 占位符
private final String sql;
// 参数映射列表，SQL 中的每个 #{xxx} 占位符都会被解析成相应的 ParameterMapping 对象
private final List<ParameterMapping> parameterMappings;
// 运行时参数，即用户传入的参数，比如 Article 对象，或是其他的参数
private final Object parameterObject;
// 附加参数集合，用于存储一些额外的信息，比如 datebaseId 等
private final Map<String, Object> additionalParameters;
// additionalParameters 的元信息对象
private final MetaObject metaParameters;
```

### 2、MappedStatement 的 getBoundSql 方法，代码如下：

```java
public BoundSql getBoundSql(Object parameterObject) {
  // 调用 sqlSource 的 getBoundSql 获取 BoundSql，把method运行时参数传进去
  BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    
  List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
  if (parameterMappings == null || parameterMappings.isEmpty()) {
    boundSql = new BoundSql(configuration, boundSql.getSql(), parameterMap.getParameterMappings(), parameterObject);
  }

  // check for nested result maps in parameter mappings (issue #30)
  for (ParameterMapping pm : boundSql.getParameterMappings()) {
    String rmId = pm.getResultMapId();
    if (rmId != null) {
      ResultMap rm = configuration.getResultMap(rmId);
      if (rm != null) {
        hasNestedResultMaps |= rm.hasNestedResultMaps();
      }
    }
  }

  return boundSql;
}
```

SqlSource 是一个接口，它有如下几个实现类：

- DynamicSqlSource
- RawSqlSource
- StaticSqlSource
- ProviderSqlSource

​	当 SQL 配置中包含 `${}`（不是 #{}）占位符，或者包含 `<if>、<where>` 等标签时，会被认为是动态 SQL，此时使用 DynamicSqlSource 存储 SQL 片段。否则，使用 RawSqlSource 存储 SQL 配置信息。

### 3、DynamicSqlSource

```java
public BoundSql getBoundSql(Object parameterObject) {
    // 创建 DynamicContext
    DynamicContext context = new DynamicContext(configuration, parameterObject);

    // 解析 SQL 片段，并将解析结果存储到 DynamicContext 中，这里会将${}替换成method对应的运行时参数，也会解析<if><where>等SqlNode
    rootSqlNode.apply(context);
    
    SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
    Class<?> parameterType = parameterObject == null ? Object.class : parameterObject.getClass();
    /*
     * 构建 StaticSqlSource，在此过程中将 sql 语句中的占位符 #{} 替换为问号 ?，
     * 并为每个占位符构建相应的 ParameterMapping
     */
    SqlSource sqlSource = sqlSourceParser.parse(context.getSql(), parameterType, context.getBindings());
    
 // 调用 StaticSqlSource 的 getBoundSql 获取 BoundSql
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);

    // 将 DynamicContext 的 ContextMap 中的内容拷贝到 BoundSql 中
    for (Map.Entry<String, Object> entry : context.getBindings().entrySet()) {
        boundSql.setAdditionalParameter(entry.getKey(), entry.getValue());
    }
    return boundSql;
}
```

该方法由数个步骤组成，这里总结一下：

1. 创建 DynamicContext
2. 解析 SQL 片段，并将解析结果存储到 DynamicContext 中
3. 解析 SQL 语句，并构建 StaticSqlSource
4. 调用 StaticSqlSource 的 getBoundSql 获取 BoundSql
5. 将 DynamicContext 的 ContextMap 中的内容拷贝到 BoundSql

#### 3.1、DynamicContext

​	DynamicContext 是 SQL 语句构建的上下文，每个 SQL 片段解析完成后，都会将解析结果存入 DynamicContext 中。待所有的 SQL 片段解析完毕后，一条完整的 SQL 语句就会出现在 DynamicContext 对象中。

```java
public class DynamicContext {

    public static final String PARAMETER_OBJECT_KEY = "_parameter";
    public static final String DATABASE_ID_KEY = "_databaseId";

    // bindings 则用于存储一些额外的信息，比如运行时参数
    private final ContextMap bindings;
    // sqlBuilder 变量用于存放 SQL 片段的解析结果
    private final StringBuilder sqlBuilder = new StringBuilder();

    public DynamicContext(Configuration configuration, Object parameterObject) {
        // 创建 ContextMap,并将运行时参数放入ContextMap中
        if (parameterObject != null && !(parameterObject instanceof Map)) {
            MetaObject metaObject = configuration.newMetaObject(parameterObject);
            bindings = new ContextMap(metaObject);
        } else {
            bindings = new ContextMap(null);
        }

        // 存放运行时参数 parameterObject 以及 databaseId
        bindings.put(PARAMETER_OBJECT_KEY, parameterObject);
        bindings.put(DATABASE_ID_KEY, configuration.getDatabaseId());
    }

    public void bind(String name, Object value) {
        this.bindings.put(name, value);
    }

    //拼接Sql片段
    public void appendSql(String sql) {
        this.sqlBuilder.append(sql);
        this.sqlBuilder.append(" ");
    }
    
    //得到sql字符串
    public String getSql() {
        return this.sqlBuilder.toString().trim();
    }

    //继承HashMap
    static class ContextMap extends HashMap<String, Object> {
        private MetaObject parameterMetaObject;
        public ContextMap(MetaObject parameterMetaObject) {
            this.parameterMetaObject = parameterMetaObject;
        }

        @Override
        public Object get(Object key) {
            String strKey = (String) key;
            // 检查是否包含 strKey，若包含则直接返回
            if (super.containsKey(strKey)) {
                return super.get(strKey);
            }
            if (parameterMetaObject != null) {
                // 从运行时参数中查找结果，这里会在${name}解析时，通过name获取运行时参数值，替换掉${name}字符串
                return parameterMetaObject.getValue(strKey);
            }
            return null;
        }
    }
    // 省略部分代码
}
```

#### 3.2、解析 SQL 片段 rootSqlNode.apply(context)

​	对于一个包含了 ${} 占位符，或 `<if>、<where>` 等标签的 SQL，在解析的过程中，会被分解成多个片段。每个片段都有对应的类型，每种类型的片段都有不同的解析逻辑。在源码中，片段这个概念等价于 sql 节点，即 SqlNode。

- StaticTextSqlNode 用于存储静态文本
- TextSqlNode 用于存储带有 ${} 占位符的文本
- IfSqlNode 则用于存储 `<if>` 节点的内容
- MixedSqlNode 内部维护了一个 SqlNode 集合，用于存储各种各样的 SqlNode
- WhereSqlNode 
- TrimSqlNode

##### 3.2.1、MixedSqlNode

​	MixedSqlNode 可以看做是 SqlNode 实现类对象的容器，凡是实现了 SqlNode 接口的类都可以存储到 MixedSqlNode 中，包括它自己。MixedSqlNode 解析方法 apply 逻辑比较简单，即遍历 SqlNode 集合，并调用其他 SqlNode实现类对象的 apply 方法解析 sql。

```java
public class MixedSqlNode implements SqlNode {
    private final List<SqlNode> contents;

    public MixedSqlNode(List<SqlNode> contents) {
        this.contents = contents;
    }

    @Override
    public boolean apply(DynamicContext context) {
        // 遍历 SqlNode 集合
        for (SqlNode sqlNode : contents) {
            // 调用 salNode 对象本身的 apply 方法解析 sql
            sqlNode.apply(context);
        }
        return true;
    }
}
```

##### 3.2.2、StaticTextSqlNode

​	StaticTextSqlNode 用于存储静态文本，直接将其存储的 SQL 的文本值拼接到 DynamicContext 的**sqlBuilder**中即可。

```java
public class StaticTextSqlNode implements SqlNode {

    private final String text;

    public StaticTextSqlNode(String text) {
        this.text = text;
    }

    @Override
    public boolean apply(DynamicContext context) {
        //直接拼接当前sql片段的文本到DynamicContext的sqlBuilder中
        context.appendSql(text);
        return true;
    }
}
```

##### 3.2.3、TextSqlNode

```java
public class TextSqlNode implements SqlNode {

    private final String text;
    private final Pattern injectionFilter;

    @Override
    public boolean apply(DynamicContext context) {
        // 创建 ${} 占位符解析器
        GenericTokenParser parser = createParser(new BindingTokenParser(context, injectionFilter));
        // 解析 ${} 占位符，通过ONGL 从用户传入的参数中获取结果，替换text中的${} 占位符
        // 并将解析结果的文本拼接到DynamicContext的sqlBuilder中
        context.appendSql(parser.parse(text));
        return true;
    }

    private GenericTokenParser createParser(TokenHandler handler) {
        // 创建占位符解析器
        return new GenericTokenParser("${", "}", handler);
    }

    private static class BindingTokenParser implements TokenHandler {

        private DynamicContext context;
        private Pattern injectionFilter;

        public BindingTokenParser(DynamicContext context, Pattern injectionFilter) {
            this.context = context;
            this.injectionFilter = injectionFilter;
        }

        @Override
        public String handleToken(String content) {
            Object parameter = context.getBindings().get("_parameter");
            if (parameter == null) {
                context.getBindings().put("value", null);
            } else if (SimpleTypeRegistry.isSimpleType(parameter.getClass())) {
                context.getBindings().put("value", parameter);
            }
            // 通过 ONGL 从用户传入的参数中获取结果
            Object value = OgnlCache.getValue(content, context.getBindings());
            String srtValue = (value == null ? "" : String.valueOf(value));
            // 通过正则表达式检测 srtValue 有效性
            checkInjection(srtValue);
            return srtValue;
        }
    }
}
```

​	GenericTokenParser 是一个通用的标记解析器，用于解析形如 ${name}，#{id} 等标记。此时是解析 ${name}的形式，从运行时参数的Map中获取到key为name的值，直接用运行时参数替换掉 ${name}字符串，将替换后的text字符串拼接到DynamicContext的sqlBuilder中

举个例子吧，比喻我们有如下SQL：

```
SELECT * FROM user WHERE name = '${name}' and id= ${id}
```

假如我们传的参数 Map中name值为 chenhao,id为1，那么该 SQL 最终会被解析成如下的结果：

```
SELECT * FROM user WHERE name = 'chenhao' and id= 1
```

​	很明显这种直接拼接值很容易造成SQL注入，假如我们传入的参数为name值为 chenhao'; DROP TABLE user;# ，解析得到的结果为：

```
SELECT * FROM user WHERE name = 'chenhao'; DROP TABLE user;#'
```

​	由于传入的参数没有经过转义，最终导致了一条 SQL 被恶意参数拼接成了两条 SQL。这就是为什么我们不应该在 SQL 语句中是用 ${} 占位符，风险太大。

##### 3.3.3、IfSqlNode

```java
public class IfSqlNode implements SqlNode {

    private final ExpressionEvaluator evaluator;
    private final String test;
    private final SqlNode contents;

    public IfSqlNode(SqlNode contents, String test) {
        this.test = test;
        this.contents = contents;
        this.evaluator = new ExpressionEvaluator();
    }

    @Override
    public boolean apply(DynamicContext context) {
        // 通过 ONGL 评估 test 表达式的结果
        if (evaluator.evaluateBoolean(test, context.getBindings())) {
            // 若 test 表达式中的条件成立，则调用其子节点节点的 apply 方法进行解析
            // 如果是静态SQL节点，则会直接拼接到DynamicContext中
            contents.apply(context);
            return true;
        }
        return false;
    }
}
```

​	IfSqlNode 对应的是 `<if test='xxx'>` 节点，首先是通过 ONGL 检测 test 表达式是否为 true，如果为 true，则调用其子节点的 apply 方法继续进行解析。如果子节点是静态SQL节点，则子节点的文本值会直接拼接到 DynamicContext 中。

#### 3.3、解析 #{} 占位符

​	经过前面的解析已经能从 DynamicContext 获取到完整的 SQL 语句了。但这并不意味着解析过程就结束了，因为当前的 SQL 语句中还有一种占位符没有处理，即 #{}。与 ${} 占位符的处理方式不同，MyBatis 并不会直接将 #{} 占位符替换为相应的参数值，而是将其替换成**？**。其解析是在如下代码中实现的：

```java
// 将前面解析过的sql字符串和运行时参数的Map作为参数
SqlSource sqlSource = sqlSourceParser.parse(context.getSql(), parameterType, context.getBindings());
```

```java 
public SqlSource parse(String originalSql, Class<?> parameterType, Map<String, Object> additionalParameters) {
    // 创建 #{} 占位符处理器
    ParameterMappingTokenHandler handler = new ParameterMappingTokenHandler(configuration, parameterType, additionalParameters);
    // 创建 #{} 占位符解析器
    GenericTokenParser parser = new GenericTokenParser("#{", "}", handler);
    // 解析 #{} 占位符，并返回解析结果字符串
    String sql = parser.parse(originalSql);
    // 封装解析结果到 StaticSqlSource 中，并返回,因为所有的动态参数都已经解析了，可以封装成一个静态的SqlSource
    return new StaticSqlSource(configuration, sql, handler.getParameterMappings());
}

public String handleToken(String content) {
    // 获取 content 的对应的 ParameterMapping
    parameterMappings.add(buildParameterMapping(content));
    // 返回 ?
    return "?";
}
```

​	将Sql中的 #{} 占位符替换成 "?"，并且将对应的参数转化成ParameterMapping 对象，通过buildParameterMapping 完成,最后创建一个 StaticSqlSource， 将 sql 字符串和 ParameterMappings 为参数传入，返回这个 StaticSqlSource：

```java
private ParameterMapping buildParameterMapping(String content) {
    /*
     * 将#{xxx} 占位符中的内容解析成 Map。
     *   #{age,javaType=int,jdbcType=NUMERIC,typeHandler=MyTypeHandler}
     *      上面占位符中的内容最终会被解析成如下的结果：
     *  {
     *      "property": "age",
     *      "typeHandler": "MyTypeHandler", 
     *      "jdbcType": "NUMERIC", 
     *      "javaType": "int"
     *  }
     */
    Map<String, String> propertiesMap = parseParameterMapping(content);
    String property = propertiesMap.get("property");
    Class<?> propertyType;
    // metaParameters 为 DynamicContext 成员变量 bindings 的元信息对象
    if (metaParameters.hasGetter(property)) {
        propertyType = metaParameters.getGetterType(property);
    
    /*
     * parameterType 是运行时参数的类型。如果用户传入的是单个参数，比如 Employe 对象，此时 
     * parameterType 为 Employe.class。如果用户传入的多个参数，比如 [id = 1, author = "chenhao"]，
     * MyBatis 会使用 ParamMap 封装这些参数，此时 parameterType 为 ParamMap.class。
     */
    } else if (typeHandlerRegistry.hasTypeHandler(parameterType)) {
        propertyType = parameterType;
    } else if (JdbcType.CURSOR.name().equals(propertiesMap.get("jdbcType"))) {
        propertyType = java.sql.ResultSet.class;
    } else if (property == null || Map.class.isAssignableFrom(parameterType)) {
        propertyType = Object.class;
    } else {
        /*
         * 代码逻辑走到此分支中，表明 parameterType 是一个自定义的类，
         * 比如 Employe，此时为该类创建一个元信息对象
         */
        MetaClass metaClass = MetaClass.forClass(parameterType, configuration.getReflectorFactory());
        // 检测参数对象有没有与 property 想对应的 getter 方法
        if (metaClass.hasGetter(property)) {
            // 获取成员变量的类型
            propertyType = metaClass.getGetterType(property);
        } else {
            propertyType = Object.class;
        }
    }
    
    ParameterMapping.Builder builder = new ParameterMapping.Builder(configuration, property, propertyType);
    
    // 将 propertyType 赋值给 javaType
    Class<?> javaType = propertyType;
    String typeHandlerAlias = null;
    
    // 遍历 propertiesMap
    for (Map.Entry<String, String> entry : propertiesMap.entrySet()) {
        String name = entry.getKey();
        String value = entry.getValue();
        if ("javaType".equals(name)) {
            // 如果用户明确配置了 javaType，则以用户的配置为准
            javaType = resolveClass(value);
            builder.javaType(javaType);
        } else if ("jdbcType".equals(name)) {
            // 解析 jdbcType
            builder.jdbcType(resolveJdbcType(value));
        } else if ("mode".equals(name)) {...} 
        else if ("numericScale".equals(name)) {...} 
        else if ("resultMap".equals(name)) {...} 
        else if ("typeHandler".equals(name)) {
            typeHandlerAlias = value;    
        } 
        else if ("jdbcTypeName".equals(name)) {...} 
        else if ("property".equals(name)) {...} 
        else if ("expression".equals(name)) {
            throw new BuilderException("Expression based parameters are not supported yet");
        } else {
            throw new BuilderException("An invalid property '" + name + "' was found in mapping #{" + content
                + "}.  Valid properties are " + parameterProperties);
        }
    }
    if (typeHandlerAlias != null) {
        builder.typeHandler(resolveTypeHandler(javaType, typeHandlerAlias));
    }
    
    // 构建 ParameterMapping 对象
    return builder.build();
}
```

​	SQL 中的 #{name, ...} 占位符被替换成了问号 ?。#{name, ...} 也被解析成了一个 ParameterMapping 对象。

##### 3.3.1、StaticSqlSource 的创建过程

```java
public class StaticSqlSource implements SqlSource {

    private final String sql;
    private final List<ParameterMapping> parameterMappings;
    private final Configuration configuration;

    public StaticSqlSource(Configuration configuration, String sql) {
        this(configuration, sql, null);
    }

    public StaticSqlSource(Configuration configuration, String sql, List<ParameterMapping> parameterMappings) {
        this.sql = sql;
        this.parameterMappings = parameterMappings;
        this.configuration = configuration;
    }

    @Override
    public BoundSql getBoundSql(Object parameterObject) {
        // 创建 BoundSql 对象
        return new BoundSql(configuration, sql, parameterMappings, parameterObject);
    }
}
```

​	调用上面创建的StaticSqlSource 中的getBoundSql方法，这是简单的 **return new** **BoundSql(configuration, sql, parameterMappings, parameterObject);** ，接着看看BoundSql：

```java
public class BoundSql {
    private String sql;
    private List<ParameterMapping> parameterMappings;
   private Object parameterObject;
    private Map<String, Object> additionalParameters;
    private MetaObject metaParameters;

    public BoundSql(Configuration configuration, String sql, List<ParameterMapping> parameterMappings, Object parameterObject) {
        this.sql = sql;
        this.parameterMappings = parameterMappings;
        this.parameterObject = parameterObject;
        this.additionalParameters = new HashMap();
        this.metaParameters = configuration.newMetaObject(this.additionalParameters);
    }

    public String getSql() {
        return this.sql;
    }
    //略
}
```

### 4、CachingExecutor 的 query(ms, parameterObject, rowBounds, resultHandler, key, boundSql) 方法

```java
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    // 从 MappedStatement 中获取缓存
    Cache cache = ms.getCache();
    // 若映射文件中未配置缓存或参照缓存，此时 cache = null
    if (cache != null) {
        flushCacheIfRequired(ms);
        if (ms.isUseCache() && resultHandler == null) {
            ensureNoOutParams(ms, boundSql);
            List<E> list = (List<E>) tcm.getObject(cache, key);
            if (list == null) {
                // 若缓存未命中，则调用被装饰类的 query 方法，也就是 SimpleExecutor 的 query 方法
                list = delegate.<E>query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
                tcm.putObject(cache, key, list); // issue #578 and #116
            }
            return list;
        }
    }
    // 调用被装饰类的 query 方法,这里的delegate我们知道应该是SimpleExecutor
    return delegate.<E>query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

​	上面的代码涉及到了二级缓存，若二级缓存为空，或未命中，则调用被装饰类的 query 方法。被装饰类为SimpleExecutor，而 SimpleExecutor 继承 BaseExecutor，那我们来看看 BaseExecutor 的 query 方法。

#### 4.1、BseExecutor 继续查一级缓存

```java
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
        clearLocalCache();
    }
    List<E> list;
    try {
        queryStack++;
        // 从一级缓存中获取缓存项
        list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
        if (list != null) {
            handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
        } else {
            // 一级缓存未命中，则从数据库中查询
            list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
        }
    } finally {
        queryStack--;
    }
    if (queryStack == 0) {
        for (DeferredLoad deferredLoad : deferredLoads) {
            deferredLoad.load();
        }
        deferredLoads.clear();
        if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
            clearLocalCache();
        }
    }
    return list;
}
```

​	从一级缓存中查找查询结果。若缓存未命中，再向数据库进行查询。至此明白了一级二级缓存的大概思路，先从二级缓存中查找，若未命中二级缓存，再从一级缓存中查找，若未命中一级缓存，再从数据库查询数据。

#### 4.2、queryFromDatabase

```java
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds,
    ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    // 向缓存中存储一个占位符
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
        // 调用 doQuery 进行查询
        list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
        // 移除占位符
        localCache.removeObject(key);
    }
    // 缓存查询结果
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
        localOutputParameterCache.putObject(key, parameter);
    }
    return list;
}
```

调用了**doQuery ** 方法进行查询，最后将查询结果放入一级缓存，接下来看看 doQuery，在 SimpleExecutor 中。

#### 4.3、数据库中查找—SimpleExecutor

```java
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
        Configuration configuration = ms.getConfiguration();
        // 创建 StatementHandler
        StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
        // 创建 Statement
        stmt = prepareStatement(handler, ms.getStatementLog());
        // 执行查询操作
        return handler.<E>query(stmt, resultHandler);
    } finally {
        // 关闭 Statement
        closeStatement(stmt);
    }
}
```

##### 4.3.1、第一步创建**StatementHandler** 

通过 **StatementHandler**  这个对象获取 Statement 对象，然后填充运行时参数，最后调用query完成查询。

```java
public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement,
    Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    // 创建具有路由功能的 StatementHandler
    StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
    // 应用插件到 StatementHandler 上
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
}
```

##### 4.3.2、RoutingStatementHandler 的构造方法

```java
public class RoutingStatementHandler implements StatementHandler {

    private final StatementHandler delegate;

    public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds,
        ResultHandler resultHandler, BoundSql boundSql) {

        // 根据 StatementType 创建不同的 StatementHandler 
        switch (ms.getStatementType()) {
            case STATEMENT:
                delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
                break;
            case PREPARED:
                delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
                break;
            case CALLABLE:
                delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
                break;
            default:
                throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
        }
    }
    
}
```

​	RoutingStatementHandler 的构造方法会根据 MappedStatement 中的 statementType 变量创建不同的 StatementHandler 实现类。那 statementType 是什么呢？还要回顾一下 MappedStatement 的创建过程：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200514172134562.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NvbWh1,size_16,color_FFFFFF,t_70)

看到 statementType 的默认类型为 PREPARED，这里将会创建 PreparedStatementHandler。

##### 4.3.3、创建 Statement

创建 Statement 在 stmt = prepareStatement(handler, ms.getStatementLog()); 这句代码：

```java
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    // 获取数据库连接
    Connection connection = getConnection(statementLog);
    // 创建 Statement，
    stmt = handler.prepare(connection, transaction.getTimeout());
  // 为 Statement 设置参数
    handler.parameterize(stmt);
    return stmt;
}
```

上面的代码中终于看到了和 jdbc 相关的内容了，创建完  Statement，最后就可以 执行查询操作了。

## 二、数据库层面的操作

先来回顾一下 jdbc 的用法：

```java
public class Login {
    /**
     *    第一步，加载驱动，创建数据库的连接
     *    第二步，编写sql
     *    第三步，需要对sql进行预编译
     *    第四步，向sql里面设置参数
     *    第五步，执行sql
     *    第六步，释放资源 
     * @throws Exception 
     */
    public static final String URL = "jdbc:mysql://localhost:3306/chenhao";
    public static final String USER = "liulx";
    public static final String PASSWORD = "123456";
    public static void main(String[] args) throws Exception {
        login("lucy","123");
    }
    
    public static void login(String username , String password) throws Exception{
        Connection conn = null; 
        PreparedStatement psmt = null;
        ResultSet rs = null;
        try {
            // 加载驱动程序
            Class.forName("com.mysql.jdbc.Driver");
            // 获得数据库连接
            conn = DriverManager.getConnection(URL, USER, PASSWORD);
            // 编写 sql
            String sql = "select * from user where name =? and password = ?";// 问号相当于一个占位符
            // 对 sql 进行预编译
            psmt = conn.prepareStatement(sql);
            // 设置参数
            psmt.setString(1, username);
            psmt.setString(2, password);
            // 执行 sql ,返回一个结果集
            rs = psmt.executeQuery();
            // 输出结果
            while(rs.next()){
                System.out.println(rs.getString("user_name")+" 年龄："+rs.getInt("age"));
            }
        } catch (Exception e) {
            e.printStackTrace();
        }finally{
            // 释放资源
            conn.close();
            psmt.close();
            rs.close();
        }
    }
}
```

接着上边的**SimpleExecutor**：

```java
/**
  * 大概分为下面三个步骤：
  * 获取数据库连接
  * 创建 PreparedStatement
  * 为 PreparedStatement 设置运行时参数
  */
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    // 获取数据库连接
    Connection connection = getConnection(statementLog);
    // 创建 Statement，
    stmt = handler.prepare(connection, transaction.getTimeout());
    // 为 Statement 设置参数
    handler.parameterize(stmt);
    return stmt;
}
```

**BaseExecutor**:

```java 
protected Connection getConnection(Log statementLog) throws SQLException {
    //通过transaction来获取Connection
    Connection connection = this.transaction.getConnection();
    return statementLog.isDebugEnabled() ? ConnectionLogger.newInstance(connection, statementLog, this.queryStack) : connection;
}
```

​	看到是通过**Executor中的**transaction属性来获取Connection，那就先来看看transaction，根据前面的文章中的配置**` ``<``transactionManager` `type="jdbc"/>，`**则MyBatis会创建一个**JdbcTransactionFactory.class** 实例，Executor中的transaction是一个**JdbcTransaction.class** 实例，其实现Transaction接口。

### 1、JdbcTransaction

#### 1.1、Transaction 接口：

```java
public interface Transaction {
    // 获取数据库连接
    Connection getConnection() throws SQLException;
    // 提交事务
    void commit() throws SQLException;
    // 回滚事务
    void rollback() throws SQLException;
    // 关闭事务
    void close() throws SQLException;
    // 获取超时时间
    Integer getTimeout() throws SQLException;
}
```

#### 1.2、JdbcTransaction 类：

```java 
public class JdbcTransaction implements Transaction {
  
  private static final Log log = LogFactory.getLog(JdbcTransaction.class);
  
  // 数据库连接
  protected Connection connection;
  // 数据源信息
  protected DataSource dataSource;
  // 隔离级别
  protected TransactionIsolationLevel level;
  // 是否为自动提交
  protected boolean autoCommmit;
  
  public JdbcTransaction(DataSource ds, TransactionIsolationLevel desiredLevel, boolean desiredAutoCommit) {
    dataSource = ds;
    level = desiredLevel;
    autoCommmit = desiredAutoCommit;
  }
  
  public JdbcTransaction(Connection connection) {
    this.connection = connection;
  }
  
  public Connection getConnection() throws SQLException {
    // 如果事务中不存在 connection，则获取一个 connection 并放入 connection 属性中
    // 第一次肯定为空
    if (connection == null) {
      openConnection();
    }
    // 如果事务中已经存在 connection，则直接返回这个 connection
    return connection;
  }
  
    /**
     * commit()功能 
     * @throws SQLException
     */
  public void commit() throws SQLException {
    if (connection != null && !connection.getAutoCommit()) {
      if (log.isDebugEnabled()) {
        log.debug("Committing JDBC Connection [" + connection + "]");
      }
      // 使用connection的commit()
      connection.commit();
    }
  }
  
    /**
     * 事务不关闭，connection 也不会关闭
     * @throws SQLException
     */
  public void rollback() throws SQLException {
    if (connection != null && !connection.getAutoCommit()) {
      if (log.isDebugEnabled()) {
        log.debug("Rolling back JDBC Connection [" + connection + "]");
      }
      // 使用 connection 的 rollback()
      connection.rollback();
    }
  }
  
    /**
     * close()功能 
     * @throws SQLException
     */
  public void close() throws SQLException {
    if (connection != null) {
      resetAutoCommit();
      if (log.isDebugEnabled()) {
        log.debug("Closing JDBC Connection [" + connection + "]");
      }
      // 使用 connection 的 close()
      connection.close();
    }
  }
  
  protected void openConnection() throws SQLException {
    if (log.isDebugEnabled()) {
      log.debug("Opening JDBC Connection");
    }
    // 通过 dataSource 来获取 connection，并设置到 transaction 的 connection 属性中
    connection = dataSource.getConnection();
    if (level != null) {
      // 通过 connection 设置事务的隔离级别
      connection.setTransactionIsolation(level.getLevel());
    }
    // 设置事务是否自动提交
    setDesiredAutoCommit(autoCommmit);
  }
  
  protected void setDesiredAutoCommit(boolean desiredAutoCommit) {
    try {
        if (this.connection.getAutoCommit() != desiredAutoCommit) {
            if (log.isDebugEnabled()) {
                log.debug("Setting autocommit to " + desiredAutoCommit + " on JDBC Connection [" + this.connection + "]");
            }
            // 通过connection设置事务是否自动提交
            this.connection.setAutoCommit(desiredAutoCommit);
        }

    } catch (SQLException var3) {
        throw new TransactionException("Error configuring AutoCommit.  Your driver may not support getAutoCommit() or setAutoCommit(). Requested setting: " + desiredAutoCommit + ".  Cause: " + var3, var3);
    }
  }
  
}
```

​	看到 JdbcTransaction 中有一个 Connection 属性和 dataSource 属性，使用 connection 来进行提交、回滚、关闭等操作，也就是说 JdbcTransaction 其实只是在 jdbc 的 connection 上面封装了一下，实际使用的其实还是jdbc 的事务。接下来看看 getConnection() 方法：

​	transaction 是 SimpleExecutor 中的属性，SimpleExecutor 又是 SqlSession 中的属性，那可以这样说，同一个 SqlSession 中只有一个 SimpleExecutor，SimpleExecutor 中有一个 Transaction，Transaction 有一个connection。

#### 1.3、来看看如下例子

```java
public static void main(String[] args) throws IOException {
    String resource = "mybatis-config.xml";
    InputStream inputStream = Resources.getResourceAsStream(resource);
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
    //创建一个SqlSession
    SqlSession sqlSession = sqlSessionFactory.openSession();
    try {
         EmployeeMapper employeeMapper = sqlSession.getMapper(Employee.class);
         UserMapper userMapper = sqlSession.getMapper(User.class);
         List<Employee> allEmployee = employeeMapper.getAll();
         List<User> allUser = userMapper.getAll();
         Employee employee = employeeMapper.getOne();
    } finally {
        sqlSession.close();
    }
}
```

​	看到同一个 sqlSession 可以获取多个 Mapper 代理对象，则多个 Mapper 代理对象中的 sqlSession 引用应该是同一个，那么多个 Mapper 代理对象调用方法应该是同一个 Connection，直到调用 close()。所以说 sqlSession 是线程不安全的，如果所有的业务都使用一个 sqlSession，那 Connection 也是同一个，一个业务执行完了就将其关闭，那其他的业务还没执行完呢。

​	回归到源码，connection = dataSource.getConnection();，最终还是调用 dataSource 来获取连接，那是不是要来看看 dataSource 呢？还是从前面的配置文件来看 UNPOOLED 和 POOLED 两种  DataSource， UNPOOLED 将会创将 new UnpooledDataSource() 实例， POOLED 将会 new pooledDataSource() 实例，都实现  DataSource 接口，那先来看看 DataSource 接口。

##### 1.3.1、DataSource 接口

```java
public interface DataSource  extends CommonDataSource,Wrapper {
  //获取数据库连接
  Connection getConnection() throws SQLException;
  Connection getConnection(String username, String password)
    throws SQLException;
}
```

##### 1.3.2、DataSource 接口的实现类 UnpooledDataSource

​	该种数据源不具有池化特性。该种数据源每次会返回一个新的数据库连接，而非复用旧的连接。其核心的方法有三个，分别如下：

1. initializeDriver - 初始化数据库驱动
2. doGetConnection - 获取数据连接
3. configureConnection - 配置数据库连接

```java
public class UnpooledDataSource implements DataSource {
    private ClassLoader driverClassLoader;
    private Properties driverProperties;
    private static Map<String, Driver> registeredDrivers = new ConcurrentHashMap();
    private String driver;
    private String url;
    private String username;
    private String password;
    private Boolean autoCommit;
    private Integer defaultTransactionIsolationLevel;

    public UnpooledDataSource() {
    }

    public UnpooledDataSource(String driver, String url, String username, String password) {
        this.driver = driver;
        this.url = url;
        this.username = username;
        this.password = password;
    }

    private synchronized void initializeDriver() throws SQLException {
        // 检测当前 driver 对应的驱动实例是否已经注册
        if (!registeredDrivers.containsKey(driver)) {
            Class<?> driverType;
            try {
                // 加载驱动类型
                if (driverClassLoader != null) {
                    // 使用 driverClassLoader 加载驱动
                    driverType = Class.forName(driver, true, driverClassLoader);
                } else {
                    // 通过其他 ClassLoader 加载驱动
                    driverType = Resources.classForName(driver);
                }

                // 通过反射创建驱动实例
                Driver driverInstance = (Driver) driverType.newInstance();
                /*
                 * 注册驱动，注意这里是将 Driver 代理类 DriverProxy 对象注册到 DriverManager 中的，而非 Driver 对象本身。
                 */
                DriverManager.registerDriver(new DriverProxy(driverInstance));
                // 缓存驱动类名和实例，防止多次注册
                registeredDrivers.put(driver, driverInstance);
            } catch (Exception e) {
                throw new SQLException("Error setting driver on UnpooledDataSource. Cause: " + e);
            }
        }
    }
    //略...
}

// DriverManager
private final static CopyOnWriteArrayList<DriverInfo> registeredDrivers = new CopyOnWriteArrayList<DriverInfo>();
public static synchronized void registerDriver(java.sql.Driver driver)
    throws SQLException {

    if(driver != null) {
        registeredDrivers.addIfAbsent(new DriverInfo(driver));
    } else {
        throw new NullPointerException();
    }
}
```

​	通过反射机制加载驱动 Driver，并将其注册到 DriverManager 中的一个常量集合中，供后面获取连接时使用，为什么这里是一个 List 呢？实际开发中有可能使用到了多种数据库类型，如 Mysql、Oracle 等，其驱动都是不同的，不同的数据源获取连接时使用的是不同的驱动。

​	在使用 JDBC 的时候，也没有通过 DriverManager.registerDriver(new DriverProxy(driverInstance)); 去注册Driver，如果使用的是 Mysql 数据源，那来看 Class.forName("com.mysql.jdbc.Driver"); 这句代码发生了什么
它主要是要求 JVM 查找并装载指定的类。这样类 com.mysql.jdbc.Driver 就被装载进来了。而且在类被装载进 JVM 的时候，它的静态方法就会被执行。来看 com.mysql.jdbc.Driver 的实现代码。在它的实现里有这么一段代码：

```java
static {  
    try {  
        // 这里使用了 DriverManager 并将该类给注册上去了
        java.sql.DriverManager.registerDriver(new Driver());  
    } catch (SQLException E) {  
        throw new RuntimeException("Can't register driver!");  
    }  
}
```

获取数据库连接：

​	在上面例子中使用 JDBC 时都是通过 DriverManager 的接口方法获取数据库连接。来看看UnpooledDataSource是如何获取的：

```java
public Connection getConnection() throws SQLException {
    return doGetConnection(username, password);
}
    
private Connection doGetConnection(String username, String password) throws SQLException {
    Properties props = new Properties();
    if (driverProperties != null) {
        props.putAll(driverProperties);
    }
    if (username != null) {
        // 存储 user 配置
        props.setProperty("user", username);
    }
    if (password != null) {
        // 存储 password 配置
        props.setProperty("password", password);
    }
    // 调用重载方法
    return doGetConnection(props);
}

private Connection doGetConnection(Properties properties) throws SQLException {
    // 初始化驱动,我们上一节已经讲过了，只用初始化一次
    initializeDriver();
    // 获取连接
    Connection connection = DriverManager.getConnection(url, properties);
    // 配置连接，包括自动提交以及事务等级
    configureConnection(connection);
    return connection;
}

private void configureConnection(Connection conn) throws SQLException {
    if (autoCommit != null && autoCommit != conn.getAutoCommit()) {
        // 设置自动提交
        conn.setAutoCommit(autoCommit);
    }
    if (defaultTransactionIsolationLevel != null) {
        // 设置事务隔离级别
        conn.setTransactionIsolation(defaultTransactionIsolationLevel);
    }
}
```

DriverManager 的 getConnection 方法即可获取到数据库连接：

```java
private static Connection getConnection(String url, java.util.Properties info, Class<?> caller) throws SQLException {
    // 获取类加载器
    ClassLoader callerCL = caller != null ? caller.getClassLoader() : null;
    synchronized(DriverManager.class) {
      if (callerCL == null) {
        callerCL = Thread.currentThread().getContextClassLoader();
      }
    }
    // 此处省略部分代码 
    // 这里遍历的是在 registerDriver(Driver driver) 方法中注册的驱动对象
    // 每个DriverInfo包含了驱动对象和其信息
    for(DriverInfo aDriver : registeredDrivers) {

      // 判断是否为当前线程类加载器加载的驱动类
      if(isDriverAllowed(aDriver.driver, callerCL)) {
        try {
          println("trying " + aDriver.driver.getClass().getName());

          // 获取连接对象，这里调用了 Driver 的父类的方法
          // 如果这里有多个 DriverInfo，比喻 Mysql 和 Oracle 的 Driver 都注册 registeredDrivers了
          // 这里所有的 Driver 都会尝试使用 url 和 info 去连接，哪个连接上了就返回
          // 会不会所有的都会连接上呢？不会，因为 url 的写法不同，不同的 Driver 会判断 url 是否适合当前驱动
          Connection con = aDriver.driver.connect(url, info);
          if (con != null) {
            // 打印连接成功信息
            println("getConnection returning " + aDriver.driver.getClass().getName());
            // 返回连接对像
            return (con);
          }
        } catch (SQLException ex) {
          if (reason == null) {
            reason = ex;
          }
        }
      } else {
        println("    skipping: " + aDriver.getClass().getName());
      }
    }  
}
```

​	代码中循环所有注册的驱动，然后通过驱动进行连接，所有的驱动都会尝试连接，但是不同的驱动，连接的URL是不同的，如 Mysql 的url是 jdbc:mysql://localhost:3306/chenhao，以 dbc:mysql://开头，则其Mysql的驱动肯定会判断获取连接的url符合，Oracle的也类似，来看看Mysql的驱动获取连接：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200514164633213.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NvbWh1,size_16,color_FFFFFF,t_70)

##### 1.3.3、DataSource 接口的实现类 PooledDataSource

​	PooledDataSource 内部实现了连接池功能，用于复用数据库连接。因此，从效率上来说，PooledDataSource 要高于 UnpooledDataSource。但是最终获取Connection还是通过UnpooledDataSource，只不过PooledDataSource 提供一个存储 Connection 的功能。

​	PooledDataSource 需要借助两个辅助类帮其完成功能，这两个辅助类分别是 PoolState 和 PooledConnection。PoolState 用于记录连接池运行时的状态，比如连接获取次数，无效连接数量等。同时 PoolState 内部定义了两个 PooledConnection 集合，用于存储空闲连接和活跃连接。PooledConnection 内部定义了一个 Connection 类型的变量，用于指向真实的数据库连接。以及一个 Connection 的代理类，用于对部分方法调用进行拦截。至于为什么要拦截，随后将进行分析。除此之外，PooledConnection 内部也定义了一些字段，用于记录数据库连接的一些运行时状态。接下来来看一下 PooledConnection 的定义：

```java
class PooledConnection implements InvocationHandler {

    private static final String CLOSE = "close";
    private static final Class<?>[] IFACES = new Class<?>[]{Connection.class};

    private final int hashCode;
    private final PooledDataSource dataSource;
    // 真实的数据库连接
    private final Connection realConnection;
    // 数据库连接代理
    private final Connection proxyConnection;
    
    // 从连接池中取出连接时的时间戳
    private long checkoutTimestamp;
    // 数据库连接创建时间
    private long createdTimestamp;
    // 数据库连接最后使用时间
    private long lastUsedTimestamp;
    // connectionTypeCode = (url + username + password).hashCode()
    private int connectionTypeCode;
    // 表示连接是否有效
    private boolean valid;

    public PooledConnection(Connection connection, PooledDataSource dataSource) {
        this.hashCode = connection.hashCode();
        this.realConnection = connection;
        this.dataSource = dataSource;
        this.createdTimestamp = System.currentTimeMillis();
        this.lastUsedTimestamp = System.currentTimeMillis();
        this.valid = true;
        // 创建 Connection 的代理类对象
        this.proxyConnection = (Connection) Proxy.newProxyInstance(Connection.class.getClassLoader(), IFACES, this);
    }
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {...}
    
    // 省略部分代码
}
```

PoolState 的定义：

```java 
public class PoolState {

    protected PooledDataSource dataSource;

    // 空闲连接列表
    protected final List<PooledConnection> idleConnections = new ArrayList<PooledConnection>();
    // 活跃连接列表
    protected final List<PooledConnection> activeConnections = new ArrayList<PooledConnection>();
    // 从连接池中获取连接的次数
    protected long requestCount = 0;
    // 请求连接总耗时（单位：毫秒）
    protected long accumulatedRequestTime = 0;
    // 连接执行时间总耗时
    protected long accumulatedCheckoutTime = 0;
    // 执行时间超时的连接数
    protected long claimedOverdueConnectionCount = 0;
    // 超时时间累加值
    protected long accumulatedCheckoutTimeOfOverdueConnections = 0;
    // 等待时间累加值
    protected long accumulatedWaitTime = 0;
    // 等待次数
    protected long hadToWaitCount = 0;
    // 无效连接数
    protected long badConnectionCount = 0;
}
```

获取连接：

​	从 PooledDataSource 获取连接时，如果空闲链接列表里有连接时，可直接取用。那如果没有空闲连接怎么办呢？此时有两种解决办法，要么创建新连接，要么等待其他连接完成任务。

PoolDataSource 源码：

```java
public class PooledDataSource implements DataSource {
    private static final Log log = LogFactory.getLog(PooledDataSource.class);
    //这里有辅助类PoolState
    private final PoolState state = new PoolState(this);
    //还有一个UnpooledDataSource属性，其实真正获取Connection是由UnpooledDataSource来完成的
    private final UnpooledDataSource dataSource;
    protected int poolMaximumActiveConnections = 10;
    protected int poolMaximumIdleConnections = 5;
    protected int poolMaximumCheckoutTime = 20000;
    protected int poolTimeToWait = 20000;
    protected String poolPingQuery = "NO PING QUERY SET";
    protected boolean poolPingEnabled = false;
    protected int poolPingConnectionsNotUsedFor = 0;
    private int expectedConnectionTypeCode;
    
    public PooledDataSource() {
        this.dataSource = new UnpooledDataSource();
    }
    
    public PooledDataSource(String driver, String url, String username, String password) {
        //构造器中创建UnpooledDataSource对象
        this.dataSource = new UnpooledDataSource(driver, url, username, password);
    }
    
    public Connection getConnection() throws SQLException {
        return this.popConnection(this.dataSource.getUsername(), this.dataSource.getPassword()).getProxyConnection();
    }
    
    private PooledConnection popConnection(String username, String password) throws SQLException {
        boolean countedWait = false;
        PooledConnection conn = null;
        long t = System.currentTimeMillis();
        int localBadConnectionCount = 0;

        while (conn == null) {
            synchronized (state) {
                // 检测空闲连接集合（idleConnections）是否为空
                if (!state.idleConnections.isEmpty()) {
                    // idleConnections 不为空，表示有空闲连接可以使用，直接从空闲连接集合中取出一个连接
                    conn = state.idleConnections.remove(0);
                } else {
                    /*
                     * 暂无空闲连接可用，但如果活跃连接数还未超出限制
                     *（poolMaximumActiveConnections），则可创建新的连接
                     */
                    if (state.activeConnections.size() < poolMaximumActiveConnections) {
                        // 创建新连接，看到没，还是通过dataSource获取连接，也就是UnpooledDataSource获取连接
                        conn = new PooledConnection(dataSource.getConnection(), this);
                    } else {    // 连接池已满，不能创建新连接
                        // 取出运行时间最长的连接
                        PooledConnection oldestActiveConnection = state.activeConnections.get(0);
                        // 获取运行时长
                        long longestCheckoutTime = oldestActiveConnection.getCheckoutTime();
                        // 检测运行时长是否超出限制，即超时
                        if (longestCheckoutTime > poolMaximumCheckoutTime) {
                            // 累加超时相关的统计字段
                            state.claimedOverdueConnectionCount++;
                            state.accumulatedCheckoutTimeOfOverdueConnections += longestCheckoutTime;
                            state.accumulatedCheckoutTime += longestCheckoutTime;

                            // 从活跃连接集合中移除超时连接
                            state.activeConnections.remove(oldestActiveConnection);
                            // 若连接未设置自动提交，此处进行回滚操作
                            if (!oldestActiveConnection.getRealConnection().getAutoCommit()) {
                                try {
                                    oldestActiveConnection.getRealConnection().rollback();
                                } catch (SQLException e) {...}
                            }
                            /*
                             * 创建一个新的 PooledConnection，注意，
                             * 此处复用 oldestActiveConnection 的 realConnection 变量
                             */
                            conn = new PooledConnection(oldestActiveConnection.getRealConnection(), this);
                            /*
                             * 复用 oldestActiveConnection 的一些信息，注意 PooledConnection 中的 
                             * createdTimestamp 用于记录 Connection 的创建时间，而非 PooledConnection 
                             * 的创建时间。所以这里要复用原连接的时间信息。
                             */
                            conn.setCreatedTimestamp(oldestActiveConnection.getCreatedTimestamp());
                            conn.setLastUsedTimestamp(oldestActiveConnection.getLastUsedTimestamp());

                            // 设置连接为无效状态
                            oldestActiveConnection.invalidate();
                            
                        } else {// 运行时间最长的连接并未超时
                            try {
                                if (!countedWait) {
                                    state.hadToWaitCount++;
                                    countedWait = true;
                                }
                                long wt = System.currentTimeMillis();
                                // 当前线程进入等待状态
                                state.wait(poolTimeToWait);
                                state.accumulatedWaitTime += System.currentTimeMillis() - wt;
                            } catch (InterruptedException e) {
                                break;
                            }
                        }
                    }
                }
                if (conn != null) {
                    if (conn.isValid()) {
                        if (!conn.getRealConnection().getAutoCommit()) {
                            // 进行回滚操作
                            conn.getRealConnection().rollback();
                        }
                        conn.setConnectionTypeCode(assembleConnectionTypeCode(dataSource.getUrl(), username, password));
                        // 设置统计字段
                        conn.setCheckoutTimestamp(System.currentTimeMillis());
                        conn.setLastUsedTimestamp(System.currentTimeMillis());
                        state.activeConnections.add(conn);
                        state.requestCount++;
                        state.accumulatedRequestTime += System.currentTimeMillis() - t;
                    } else {
                        // 连接无效，此时累加无效连接相关的统计字段
                        state.badConnectionCount++;
                        localBadConnectionCount++;
                        conn = null;
                        if (localBadConnectionCount > (poolMaximumIdleConnections
                            + poolMaximumLocalBadConnectionTolerance)) {
                            throw new SQLException(...);
                        }
                    }
                }
            }

        }
        if (conn == null) {
            throw new SQLException(...);
        }

        return conn;
    }
}
```

从连接池中获取连接首先会遇到两种情况：

1. 连接池中有空闲连接
2. 连接池中无空闲连接

对于第一种情况，把连接取出返回即可。对于第二种情况，则要进行细分，会有如下的情况：

1. 活跃连接数没有超出最大活跃连接数
2. 活跃连接数超出最大活跃连接数

对于上面两种情况，第一种情况比较好处理，直接创建新的连接即可。至于第二种情况，需要再次进行细分：

1. 活跃连接的运行时间超出限制，即超时了
2. 活跃连接未超时

对于第一种情况，直接将超时连接强行中断，并进行回滚，然后复用部分字段重新创建 PooledConnection 即可。对于第二种情况，目前没有更好的处理方式了，只能等待了。

##### 1.3.4、回收连接

​	相比于获取连接，回收连接的逻辑要简单的多。回收连接成功与否只取决于空闲连接集合的状态，所需处理情况很少，因此比较简单。

```java
public Connection getConnection() throws SQLException {
    return this.popConnection(this.dataSource.getUsername(), this.dataSource.getPassword()).getProxyConnection();
}
```

​	返回的是 PooledConnection 的一个代理类，为什么不直接使用 PooledConnection 的 realConnection 呢？看下 PooledConnection 这个类：

```java 
class PooledConnection implements InvocationHandler {
```

标准的代理类用法，看下其 invoke 方法：

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    String methodName = method.getName();
    // 重点在这里，如果调用了其 close 方法，则实际执行的是将连接放回连接池的操作
    if (CLOSE.hashCode() == methodName.hashCode() && CLOSE.equals(methodName)) {
        dataSource.pushConnection(this);
        return null;
    } else {
        try {
            if (!Object.class.equals(method.getDeclaringClass())) {
                // issue #579 toString() should never fail
                // throw an SQLException instead of a Runtime
                checkConnection();
            }
            // 其他的操作都交给 realConnection 执行
            return method.invoke(realConnection, args);
        } catch (Throwable t) {
            throw ExceptionUtil.unwrapThrowable(t);
        }
    }
}
```

看看 **pushConnection** 做了什么：

```java
protected void pushConnection(PooledConnection conn) throws SQLException {
    synchronized (state) {
        // 从活跃连接池中移除连接
        state.activeConnections.remove(conn);
        if (conn.isValid()) {
            // 空闲连接集合未满
            if (state.idleConnections.size() < poolMaximumIdleConnections
                && conn.getConnectionTypeCode() == expectedConnectionTypeCode) {
                state.accumulatedCheckoutTime += conn.getCheckoutTime();

                // 回滚未提交的事务
                if (!conn.getRealConnection().getAutoCommit()) {
                    conn.getRealConnection().rollback();
                }

                // 创建新的 PooledConnection
                PooledConnection newConn = new PooledConnection(conn.getRealConnection(), this);
                state.idleConnections.add(newConn);
                // 复用时间信息
                newConn.setCreatedTimestamp(conn.getCreatedTimestamp());
                newConn.setLastUsedTimestamp(conn.getLastUsedTimestamp());

                // 将原连接置为无效状态
                conn.invalidate();

                // 通知等待的线程
                state.notifyAll();
                
            } else {// 空闲连接集合已满
                state.accumulatedCheckoutTime += conn.getCheckoutTime();
                // 回滚未提交的事务
                if (!conn.getRealConnection().getAutoCommit()) {
                    conn.getRealConnection().rollback();
                }

                // 关闭数据库连接
                conn.getRealConnection().close();
                conn.invalidate();
            }
        } else {
            state.badConnectionCount++;
        }
    }
}
```

​	先将连接从活跃连接集合中移除，如果空闲集合未满，此时复用原连接的字段信息创建新的连接，并将其放入空闲集合中即可；若空闲集合已满，此时无需回收连接，直接关闭即可。

​	已经获取到了数据库连接，接下来要创建**PrepareStatement**了，JDBC 例子中获取 PrepareStatement 的方式为：

```java
psmt = conn.prepareStatement(sql);
```

**直接通过Connection来获取，并且把sql传进去了**，接下来看看 Mybaits 中是怎么创建 PrepareStatement 的。

### 2、创建 PreparedStatement

#### 2.1、准备 ServerPreparedStatement

PreparedStatementHandler：

```java
stmt = handler.prepare(connection, transaction.getTimeout());

public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
    Statement statement = null;
    try {
        // 创建 Statement
        statement = instantiateStatement(connection);
        // 设置超时和 FetchSize
        setStatementTimeout(statement, transactionTimeout);
        setFetchSize(statement);
        return statement;
    } catch (SQLException e) {
        closeStatement(statement);
        throw e;
    } catch (Exception e) {
        closeStatement(statement);
        throw new ExecutorException("Error preparing statement.  Cause: " + e, e);
    }
}

protected Statement instantiateStatement(Connection connection) throws SQLException {
    // 获取sql字符串，比如"select * from user where id= ?"
    String sql = boundSql.getSql();
    // 根据条件调用不同的 prepareStatement 方法创建 PreparedStatement
    if (mappedStatement.getKeyGenerator() instanceof Jdbc3KeyGenerator) {
        String[] keyColumnNames = mappedStatement.getKeyColumns();
        if (keyColumnNames == null) {
            // 通过 connection 获取 Statement，将 sql 语句传进去
            return connection.prepareStatement(sql, PreparedStatement.RETURN_GENERATED_KEYS);
        } else {
            return connection.prepareStatement(sql, keyColumnNames);
        }
    } else if (mappedStatement.getResultSetType() != null) {
        return connection.prepareStatement(sql, mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
    } else {
        return connection.prepareStatement(sql);
    }
}
```

和 jdbc 的形式一模一样，我们具体来看看 **connection.prepareStatement** 做了什么：

```java
public PreparedStatement prepareStatement(String sql, int resultSetType, int resultSetConcurrency) throws SQLException {
       
    boolean canServerPrepare = true;

    String nativeSql = getProcessEscapeCodesForPrepStmts() ? nativeSQL(sql) : sql;

    if (this.useServerPreparedStmts && getEmulateUnsupportedPstmts()) {
        canServerPrepare = canHandleAsServerPreparedStatement(nativeSql);
    }

    if (this.useServerPreparedStmts && getEmulateUnsupportedPstmts()) {
        canServerPrepare = canHandleAsServerPreparedStatement(nativeSql);
    }

    if (this.useServerPreparedStmts && canServerPrepare) {
        if (this.getCachePreparedStatements()) {
            ......
        } else {
            try {
                // 这里使用的是 ServerPreparedStatement 创建 PreparedStatement
                pStmt = ServerPreparedStatement.getInstance(getMultiHostSafeProxy(), nativeSql, this.database, resultSetType, resultSetConcurrency);

                pStmt.setResultSetType(resultSetType);
                pStmt.setResultSetConcurrency(resultSetConcurrency);
            } catch (SQLException sqlEx) {
                // Punt, if necessary
                if (getEmulateUnsupportedPstmts()) {
                    pStmt = (PreparedStatement) clientPrepareStatement(nativeSql, resultSetType, resultSetConcurrency, false);
                } else {
                    throw sqlEx;
                }
            }
        }
    } else {
        pStmt = (PreparedStatement) clientPrepareStatement(nativeSql, resultSetType, resultSetConcurrency, false);
    }
}
```

使用**ServerPreparedStatement的getInstance返回一个**PreparedStatement，其实本质上	**ServerPreparedStatement继承了PreparedStatement对象**，看看其构造方法：

```java 
protected ServerPreparedStatement(ConnectionImpl conn, String sql, String catalog, int resultSetType, int resultSetConcurrency) throws SQLException {
    //略...

    try {
        this.serverPrepare(sql);
    } catch (SQLException var10) {
        this.realClose(false, true);
        throw var10;
    } catch (Exception var11) {
        this.realClose(false, true);
        SQLException sqlEx = SQLError.createSQLException(var11.toString(), "S1000", this.getExceptionInterceptor());
        sqlEx.initCause(var11);
        throw sqlEx;
    }
    //略...

}
```

继续调用 this.serverPrepare(sql) ：

```java
public class ServerPreparedStatement extends PreparedStatement {
    // 存放运行时参数的数组
    private ServerPreparedStatement.BindValue[] parameterBindings;
    // 服务器预编译好的 sql 语句返回的 serverStatementId
    private long serverStatementId;
    private void serverPrepare(String sql) throws SQLException {
        synchronized(this.connection.getMutex()) {
            MysqlIO mysql = this.connection.getIO();
            try {
                // 向 sql 服务器发送了一条 PREPARE 指令
                Buffer prepareResultPacket = mysql.sendCommand(MysqlDefs.COM_PREPARE, sql, (Buffer)null, false, characterEncoding, 0);
                // 记录下了预编译好的 sql 语句所对应的 serverStatementId
                this.serverStatementId = prepareResultPacket.readLong();
                this.fieldCount = prepareResultPacket.readInt();
                // 获取参数个数，比喻 select * from user where id= ？and name = ？，其中有两个？，则这里返回的参数个数应该为 2
                this.parameterCount = prepareResultPacket.readInt();
                this.parameterBindings = new ServerPreparedStatement.BindValue[this.parameterCount];

                for(int i = 0; i < this.parameterCount; ++i) {
                    // 根据参数个数，初始化数组
                    this.parameterBindings[i] = new ServerPreparedStatement.BindValue();
                }
            } catch (SQLException var16) {
                throw sqlEx;
            } finally {
                this.connection.getIO().clearInputStream();
            }

        }
    }
}
```

​	ServerPreparedStatement 继承 PreparedStatement，ServerPreparedStatement 初始化的时候就向 sql 服务器发送了一条 PREPARE 指令，把 SQL 语句传到 mysql 服务器，如 select * from user where id= ？and name = ？，mysql 服务器会对 sql 进行编译，并保存在服务器，返回预编译语句对应的 id，并保存在  ServerPreparedStatement 中，同时创建 BindValue[] parameterBindings 数组，后面设置参数就直接添加到此数组中。此时创建了一个 ServerPreparedStatement 并返回，下面就是设置运行时参数了：

#### 2.2、设置运行时参数到 SQL 中

已经获取到了 PreparedStatement，接下来就是将运行时参数设置到 PreparedStatement中，如下代码：

```java
handler.parameterize(stmt);
```

JDBC是怎么设置的呢？看看上面的例子：

```java
psmt = conn.prepareStatement(sql);
//设置参数
psmt.setString(1, username);
psmt.setString(2, password);
```

看看 parameterize 方法：

```java
public void parameterize(Statement statement) throws SQLException {
    // 通过参数处理器 ParameterHandler 设置运行时参数到 PreparedStatement 中
    parameterHandler.setParameters((PreparedStatement) statement);
}

public class DefaultParameterHandler implements ParameterHandler {
    private final TypeHandlerRegistry typeHandlerRegistry;
    private final MappedStatement mappedStatement;
    private final Object parameterObject;
    private final BoundSql boundSql;
    private final Configuration configuration;

    public void setParameters(PreparedStatement ps) {
        /*
         * 从 BoundSql 中获取 ParameterMapping 列表，每个 ParameterMapping 与原始 SQL 中的 #{xxx} 占位符一一对应
         */
        List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
        if (parameterMappings != null) {
            for (int i = 0; i < parameterMappings.size(); i++) {
                ParameterMapping parameterMapping = parameterMappings.get(i);
                if (parameterMapping.getMode() != ParameterMode.OUT) {
                    Object value;
                    // 获取属性名
                    String propertyName = parameterMapping.getProperty();
                    if (boundSql.hasAdditionalParameter(propertyName)) {
                        value = boundSql.getAdditionalParameter(propertyName);
                    } else if (parameterObject == null) {
                        value = null;
                    } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
                        value = parameterObject;
                    } else {
                        // 为用户传入的参数 parameterObject 创建元信息对象
                        MetaObject metaObject = configuration.newMetaObject(parameterObject);
                        // 从用户传入的参数中获取 propertyName 对应的值
                        value = metaObject.getValue(propertyName);
                    }

                    TypeHandler typeHandler = parameterMapping.getTypeHandler();
                    JdbcType jdbcType = parameterMapping.getJdbcType();
                    if (value == null && jdbcType == null) {
                        jdbcType = configuration.getJdbcTypeForNull();
                    }
                    try {
                        // 由类型处理器 typeHandler 向 ParameterHandler 设置参数
                        typeHandler.setParameter(ps, i + 1, value, jdbcType);
                    } catch (TypeException e) {
                        throw new TypeException(...);
                    } catch (SQLException e) {
                        throw new TypeException(...);
                    }
                }
            }
        }
    }
}
```

​	首先从  boundSql 中获取 parameterMappings 集合，然后遍历获取 parameterMapping 中的 propertyName ，如  #{name} 中的 name，然后从运行时参数 parameterObject 中获取 name 对应的参数值，最后设置到 PreparedStatement 中，主要来看是如何设置参数的。也就是：

```java
typeHandler.setParameter(ps, i + 1, value, jdbcType);
```

这句代码最终如下：

```
public void setNonNullParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType) throws SQLException {
    ps.setString(i, parameter);
}
```

​	还记得 PreparedStatement是什么吗？是 ServerPreparedStatement，那来看看 ServerPreparedStatement的 setString 方法：

```java
public void setString(int parameterIndex, String x) throws SQLException {
    this.checkClosed();
    if (x == null) {
        this.setNull(parameterIndex, 1);
    } else {
        // 根据参数下标从 parameterBindings 数组总获取 BindValue
        ServerPreparedStatement.BindValue binding = this.getBinding(parameterIndex, false);
        this.setType(binding, this.stringTypeCode);
        // 设置参数值
        binding.value = x;
        binding.isNull = false;
        binding.isLongData = false;
    }
}

protected ServerPreparedStatement.BindValue getBinding(int parameterIndex, boolean forLongData) throws SQLException {
    this.checkClosed();
    if (this.parameterBindings.length == 0) {
        throw SQLError.createSQLException(Messages.getString("ServerPreparedStatement.8"), "S1009", this.getExceptionInterceptor());
    } else {
        --parameterIndex;
        if (parameterIndex >= 0 && parameterIndex < this.parameterBindings.length) {
            if (this.parameterBindings[parameterIndex] == null) {
                this.parameterBindings[parameterIndex] = new ServerPreparedStatement.BindValue();
            } else if (this.parameterBindings[parameterIndex].isLongData && !forLongData) {
                this.detectedLongParameterSwitch = true;
            }

            this.parameterBindings[parameterIndex].isSet = true;
            this.parameterBindings[parameterIndex].boundBeforeExecutionNum = (long)this.numberOfExecutions;
            // 根据参数下标从 parameterBindings 数组总获取 BindValue
            return this.parameterBindings[parameterIndex];
        } else {
            throw SQLError.createSQLException(Messages.getString("ServerPreparedStatement.9") + (parameterIndex + 1) + Messages.getString("ServerPreparedStatement.10") + this.parameterBindings.length, "S1009", this.getExceptionInterceptor());
        }
    }
}
```

​	就是根据参数下标从 ServerPreparedStatement 的参数数组 parameterBindings 中获取 BindValue 对象，然后设置值，好了现在 ServerPreparedStatement 包含了预编译 SQL 语句的 Id 和参数数组，最后一步便是执行 SQL 了。

### 3、执行查询

执行查询操作就是文章开头的最后一行代码，如下：

```java
return handler.<E>query(stmt, resultHandler);
```

```java
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement)statement;
    // 直接执行 ServerPreparedStatement 的 execute 方法
    ps.execute();
    return this.resultSetHandler.handleResultSets(ps);
}

public boolean execute() throws SQLException {
    this.checkClosed();
    ConnectionImpl locallyScopedConn = this.connection;
    if (!this.checkReadOnlySafeStatement()) {
        throw SQLError.createSQLException(Messages.getString("PreparedStatement.20") + Messages.getString("PreparedStatement.21"), "S1009", this.getExceptionInterceptor());
    } else {
        ResultSetInternalMethods rs = null;
        CachedResultSetMetaData cachedMetadata = null;
        synchronized(locallyScopedConn.getMutex()) {
            //略....
            rs = this.executeInternal(rowLimit, sendPacket, doStreaming, this.firstCharOfStmt == 'S', metadataFromCache, false);
            //略....
        }
        return rs != null && rs.reallyResult();
    }
}
```

省略了很多代码，只看最关键的**executeInternal**：

ServerPreparedStatement：

```java
protected ResultSetInternalMethods executeInternal(int maxRowsToRetrieve, Buffer sendPacket, boolean createStreamingResultSet, boolean queryIsSelectOnly, Field[] metadataFromCache, boolean isBatch) throws SQLException {
    try {
        return this.serverExecute(maxRowsToRetrieve, createStreamingResultSet, metadataFromCache);
    } catch (SQLException var11) {
        throw sqlEx;
    } 
}

private ResultSetInternalMethods serverExecute(int maxRowsToRetrieve, boolean createStreamingResultSet, Field[] metadataFromCache) throws SQLException {
    synchronized(this.connection.getMutex()) {
        // 略....
        MysqlIO mysql = this.connection.getIO();
        Buffer packet = mysql.getSharedSendPacket();
        packet.clear();
        packet.writeByte((byte)MysqlDefs.COM_EXECUTE);
        // 将该语句对应的 id 写入数据包
        packet.writeLong(this.serverStatementId);

        int i;
        // 将对应的参数写入数据包
        for(i = 0; i < this.parameterCount; ++i) {
            if (!this.parameterBindings[i].isLongData) {
                if (!this.parameterBindings[i].isNull) {
                    this.storeBinding(packet, this.parameterBindings[i], mysql);
                } else {
                    nullBitsBuffer[i / 8] = (byte)(nullBitsBuffer[i / 8] | 1 << (i & 7));
                }
            }
        }
        // 发送数据包,表示执行 id 对应的预编译 sql
        Buffer resultPacket = mysql.sendCommand(MysqlDefs.COM_EXECUTE, (String)null, packet, false, (String)null, 0);
        // 略....
        ResultSetImpl rs = mysql.readAllResults(this,  this.resultSetType,  resultPacket, true, (long)this.fieldCount, metadataFromCache);
        //返回结果
        return rs;
    }
}
```

​	ServerPreparedStatement 在记录下 serverStatementId 后，对于相同 SQL 模板的操作，每次只是发送 serverStatementId 和对应的参数，省去了编译 sql 的过程。 至此已经从数据库拿到了查询结果，但是结果是 ResultSetImpl 类型，还需要将返回结果转化成我们的 java 对象。且看接下来分解。

## 三、映射结果入口

我们来看看上次看源码的位置

```java 
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement)statement;
    //执行数据库SQL
    ps.execute();
    //进行resultSet自动映射
    return this.resultSetHandler.handleResultSets(ps);
}
```

结果集的处理入口方法是 handleResultSets

```java 
public List<Object> handleResultSets(Statement stmt) throws SQLException {
    
    final List<Object> multipleResults = new ArrayList<Object>();

    int resultSetCount = 0;
    //获取第一个ResultSet,通常只会有一个
    ResultSetWrapper rsw = getFirstResultSet(stmt);
    //从配置中读取对应的ResultMap，通常也只会有一个，设置多个是通过逗号来分隔，我们平时有这样设置吗？
    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount);

    while (rsw != null && resultMapCount > resultSetCount) {
        ResultMap resultMap = resultMaps.get(resultSetCount);
        // 处理结果集
        handleResultSet(rsw, resultMap, multipleResults, null);
        rsw = getNextResultSet(stmt);
        cleanUpAfterHandlingResultSet();
        resultSetCount++;
    }

    // 以下逻辑均与多结果集有关，就不分析了，代码省略
    String[] resultSets = mappedStatement.getResultSets();
    if (resultSets != null) {...}

    return collapseSingleResultList(multipleResults);
}
```

在实际运行过程中，通常情况下一个Sql语句只返回一个结果集，对多个结果集的情况不做分析 。实际很少用到。继续看handleResultSet方法

```java 
private void handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping) throws SQLException {
    try {
        if (parentMapping != null) {
            handleRowValues(rsw, resultMap, null, RowBounds.DEFAULT, parentMapping);
        } else {
            if (resultHandler == null) {
                // 创建默认的结果处理器
                DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);
                // 处理结果集的行数据
                handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);
                // 将结果加入multipleResults中
                multipleResults.add(defaultResultHandler.getResultList());
            } else {
                handleRowValues(rsw, resultMap, resultHandler, rowBounds, null);
            }
        }
    } finally {
        closeResultSet(rsw.getResultSet());
    }
}
```

通过handleRowValues 映射ResultSet结果，最后映射的结果会在defaultResultHandler的ResultList集合中，最后将结果加入到multipleResults中就可以返回了，我们继续跟进**handleRowValues这个核心方法**

```java 
public void handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler,
        RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
    if (resultMap.hasNestedResultMaps()) {
        ensureNoRowBounds();
        checkResultHandler();
        // 处理嵌套映射，关于嵌套映射我们下一篇文章单独分析
        handleRowValuesForNestedResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    } else {
        // 处理简单映射，本文先只分析简单映射
        handleRowValuesForSimpleResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    }
}
```

我们可以通过resultMap.hasNestedResultMaps()知道查询语句是否是嵌套查询，如果resultMap中包含<association> 和 <collection>且其select属性不为空，则为嵌套查询，大家可以看看我第三篇文章关于[解析 resultMap 节点](https://www.cnblogs.com/java-chen-hao/p/11743442.html#_label2_1)。本文先分析简单的映射

```java 
private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap,
        ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {

    DefaultResultContext<Object> resultContext = new DefaultResultContext<Object>();
    // 根据 RowBounds 定位到指定行记录
    skipRows(rsw.getResultSet(), rowBounds);
    // ResultSet是一个集合，很有可能我们查询的就是一个List，这就就每条数据遍历处理
    while (shouldProcessMoreRows(resultContext, rowBounds) && rsw.getResultSet().next()) {
        ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(rsw.getResultSet(), resultMap, null);
        // 从 resultSet 中获取结果
        Object rowValue = getRowValue(rsw, discriminatedResultMap);
        // 存储结果到resultHandler的ResultList，最后ResultList加入multipleResults中返回
        storeObject(resultHandler, resultContext, rowValue, parentMapping, rsw.getResultSet());
    }
}
```

我们查询的结果很有可能是一个集合，所以这里要遍历集合，每条结果单独进行映射，最后映射的结果加入到**resultHandler的ResultList**

**MyBatis 默认提供了 RowBounds 用于分页，从上面的代码中可以看出，这并非是一个高效的分页方式，是查出所有的数据，进行内存分页。除了使用 RowBounds，还可以使用一些第三方分页插件进行分页。**我们后面文章来讲，我们来看关键代码getRowValue，处理一行数据

```java 
private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap) throws SQLException {
    // 这个Map是用来存储延迟加载的BountSql的，我们下面来看
    final ResultLoaderMap lazyLoader = new ResultLoaderMap();
 // 创建实体类对象，比如 Employ 对象
    Object rowValue = createResultObject(rsw, resultMap, lazyLoader, null);
    if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
        final MetaObject metaObject = configuration.newMetaObject(rowValue);
        boolean foundValues = this.useConstructorMappings;
        
        if (shouldApplyAutomaticMappings(resultMap, false)) {
            //自动映射,结果集中有的column，但resultMap中并没有配置  
            foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, null) || foundValues;
        }
      // 根据 <resultMap> 节点中配置的映射关系进行映射
        foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, null) || foundValues;
        foundValues = lazyLoader.size() > 0 || foundValues;
        rowValue = foundValues || configuration.isReturnInstanceForEmptyRow() ? rowValue : null;
    }
    return rowValue;
}
```

重要的逻辑已经注释出来了。分别如下：

1. 创建实体类对象
2. 自动映射结果集中有的column，但resultMap中并没有配置
3. 根据 <resultMap> 节点中配置的映射关系进行映射

#### 3.1、创建实体类对象

我们想将查询结果映射成实体类对象，第一步当然是要创建实体类对象了，下面我们来看一下 MyBatis 创建实体类对象的过程。

```java
private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, ResultLoaderMap lazyLoader, String columnPrefix) throws SQLException {

    this.useConstructorMappings = false;
    final List<Class<?>> constructorArgTypes = new ArrayList<Class<?>>();
    final List<Object> constructorArgs = new ArrayList<Object>();

    // 调用重载方法创建实体类对象
    Object resultObject = createResultObject(rsw, resultMap, constructorArgTypes, constructorArgs, columnPrefix);
    if (resultObject != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
        final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
        for (ResultMapping propertyMapping : propertyMappings) {
            // 如果开启了延迟加载，则为 resultObject 生成代理类，如果仅仅是配置的关联查询，没有开启延迟加载，是不会创建代理类
            if (propertyMapping.getNestedQueryId() != null && propertyMapping.isLazy()) {
                /*
                 * 创建代理类，默认使用 Javassist 框架生成代理类。
                 * 由于实体类通常不会实现接口，所以不能使用 JDK 动态代理 API 为实体类生成代理。
                 * 并且将lazyLoader传进去了
                 */
                resultObject = configuration.getProxyFactory()
                    .createProxy(resultObject, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
                break;
            }
        }
    }
    this.useConstructorMappings =
        resultObject != null && !constructorArgTypes.isEmpty();
    return resultObject;
}
```

我们先来看 createResultObject 重载方法的逻辑

```java
private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix) throws SQLException {

    final Class<?> resultType = resultMap.getType();
    final MetaClass metaType = MetaClass.forClass(resultType, reflectorFactory);
    final List<ResultMapping> constructorMappings = resultMap.getConstructorResultMappings();

    if (hasTypeHandlerForResultObject(rsw, resultType)) {
        return createPrimitiveResultObject(rsw, resultMap, columnPrefix);
    } else if (!constructorMappings.isEmpty()) {
        return createParameterizedResultObject(rsw, resultType, constructorMappings, constructorArgTypes, constructorArgs, columnPrefix);
    } else if (resultType.isInterface() || metaType.hasDefaultConstructor()) {
        // 通过 ObjectFactory 调用目标类的默认构造方法创建实例
        return objectFactory.create(resultType);
    } else if (shouldApplyAutomaticMappings(resultMap, false)) {
        return createByConstructorSignature(rsw, resultType, constructorArgTypes, constructorArgs, columnPrefix);
    }
    throw new ExecutorException("Do not know how to create an instance of " + resultType);
}
```

一般情况下，MyBatis 会通过 ObjectFactory 调用默认构造方法创建实体类对象。看看是如何创建的

```java
public <T> T create(Class<T> type, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
    Class<?> classToCreate = this.resolveInterface(type);
    return this.instantiateClass(classToCreate, constructorArgTypes, constructorArgs);
}

<T> T instantiateClass(Class<T> type, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
    try {
        Constructor constructor;
        if (constructorArgTypes != null && constructorArgs != null) {
            constructor = type.getDeclaredConstructor((Class[])constructorArgTypes.toArray(new Class[constructorArgTypes.size()]));
            if (!constructor.isAccessible()) {
                constructor.setAccessible(true);
            }

            return constructor.newInstance(constructorArgs.toArray(new Object[constructorArgs.size()]));
        } else {
            //通过反射获取构造器
            constructor = type.getDeclaredConstructor();
            if (!constructor.isAccessible()) {
                constructor.setAccessible(true);
            }
            //通过构造器来实例化对象
            return constructor.newInstance();
        }
    } catch (Exception var9) {
        throw new ReflectionException("Error instantiating " + type + " with invalid types (" + argTypes + ") or values (" + argValues + "). Cause: " + var9, var9);
    }
}
```

很简单，就是通过反射创建对象

#### 3.2、结果集映射

映射结果集分为两种情况：一种是自动映射(结果集有但在resultMap里没有配置的字段)，在实际应用中，都会使用自动映射，减少配置的工作。自动映射在Mybatis中也是默认开启的。第二种是映射ResultMap中配置的，我们分这两者映射来看

#### 3.3、自动映射

```java
private boolean applyAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String columnPrefix) throws SQLException {

    // 获取 UnMappedColumnAutoMapping 列表
    List<UnMappedColumnAutoMapping> autoMapping = createAutomaticMappings(rsw, resultMap, metaObject, columnPrefix);
    boolean foundValues = false;
    if (!autoMapping.isEmpty()) {
        for (UnMappedColumnAutoMapping mapping : autoMapping) {
            // 通过 TypeHandler 从结果集中获取指定列的数据
            final Object value = mapping.typeHandler.getResult(rsw.getResultSet(), mapping.column);
            if (value != null) {
                foundValues = true;
            }
            if (value != null || (configuration.isCallSettersOnNulls() && !mapping.primitive)) {
                // 通过元信息对象设置 value 到实体类对象的指定字段上
                metaObject.setValue(mapping.property, value);
            }
        }
    }
    return foundValues;
}
```

首先是获取 UnMappedColumnAutoMapping 集合，然后遍历该集合，并通过 TypeHandler 从结果集中获取数据，最后再将获取到的数据设置到实体类对象中。

UnMappedColumnAutoMapping 用于记录未配置在 <resultMap> 节点中的映射关系。它的代码如下：

```java
private static class UnMappedColumnAutoMapping {

    private final String column;
    private final String property;
    private final TypeHandler<?> typeHandler;
    private final boolean primitive;

    public UnMappedColumnAutoMapping(String column, String property, TypeHandler<?> typeHandler, boolean primitive) {
        this.column = column;
        this.property = property;
        this.typeHandler = typeHandler;
        this.primitive = primitive;
    }
}
```

仅用于记录映射关系。下面看一下获取 UnMappedColumnAutoMapping 集合的过程，如下：

```java
private List<UnMappedColumnAutoMapping> createAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String columnPrefix) throws SQLException {

    final String mapKey = resultMap.getId() + ":" + columnPrefix;
    // 从缓存中获取 UnMappedColumnAutoMapping 列表
    List<UnMappedColumnAutoMapping> autoMapping = autoMappingsCache.get(mapKey);
    // 缓存未命中
    if (autoMapping == null) {
        autoMapping = new ArrayList<UnMappedColumnAutoMapping>();
        // 从 ResultSetWrapper 中获取未配置在 <resultMap> 中的列名
        final List<String> unmappedColumnNames = rsw.getUnmappedColumnNames(resultMap, columnPrefix);
        for (String columnName : unmappedColumnNames) {
            String propertyName = columnName;
            if (columnPrefix != null && !columnPrefix.isEmpty()) {
                if (columnName.toUpperCase(Locale.ENGLISH).startsWith(columnPrefix)) {
                    propertyName = columnName.substring(columnPrefix.length());
                } else {
                    continue;
                }
            }
            // 将下划线形式的列名转成驼峰式，比如 AUTHOR_NAME -> authorName
            final String property = metaObject.findProperty(propertyName, configuration.isMapUnderscoreToCamelCase());
            if (property != null && metaObject.hasSetter(property)) {
                // 检测当前属性是否存在于 resultMap 中
                if (resultMap.getMappedProperties().contains(property)) {
                    continue;
                }
                // 获取属性对于的类型
                final Class<?> propertyType = metaObject.getSetterType(property);
                if (typeHandlerRegistry.hasTypeHandler(propertyType, rsw.getJdbcType(columnName))) {
                    final TypeHandler<?> typeHandler = rsw.getTypeHandler(propertyType, columnName);
                    // 封装上面获取到的信息到 UnMappedColumnAutoMapping 对象中
                    autoMapping.add(new UnMappedColumnAutoMapping(columnName, property, typeHandler, propertyType.isPrimitive()));
                } else {
                    configuration.getAutoMappingUnknownColumnBehavior()
                        .doAction(mappedStatement, columnName, property, propertyType);
                }
            } else {
                configuration.getAutoMappingUnknownColumnBehavior()
                    .doAction(mappedStatement, columnName, (property != null) ? property : propertyName, null);
            }
        }
        // 写入缓存
        autoMappingsCache.put(mapKey, autoMapping);
    }
    return autoMapping;
}
```

先来看看 ResultSetWrapper 中获取未配置在 <resultMap> 中的列名

```java
public List<String> getUnmappedColumnNames(ResultMap resultMap, String columnPrefix) throws SQLException {
    List<String> unMappedColumnNames = unMappedColumnNamesMap.get(getMapKey(resultMap, columnPrefix));
    if (unMappedColumnNames == null) {
        // 加载已映射与未映射列名
        loadMappedAndUnmappedColumnNames(resultMap, columnPrefix);
        // 获取未映射列名
        unMappedColumnNames = unMappedColumnNamesMap.get(getMapKey(resultMap, columnPrefix));
    }
    return unMappedColumnNames;
}

private void loadMappedAndUnmappedColumnNames(ResultMap resultMap, String columnPrefix) throws SQLException {
    List<String> mappedColumnNames = new ArrayList<String>();
    List<String> unmappedColumnNames = new ArrayList<String>();
    final String upperColumnPrefix = columnPrefix == null ? null : columnPrefix.toUpperCase(Locale.ENGLISH);
    // 获取 <resultMap> 中配置的所有列名
    final Set<String> mappedColumns = prependPrefixes(resultMap.getMappedColumns(), upperColumnPrefix);
    /*
     * 遍历 columnNames，columnNames 是 ResultSetWrapper 的成员变量，保存了当前结果集中的所有列名
     * 这里是通过ResultSet中的所有列名来获取没有在resultMap中配置的列名
     * 意思是后面进行自动赋值时，只赋值查出来的列名
     */
    for (String columnName : columnNames) {
        final String upperColumnName = columnName.toUpperCase(Locale.ENGLISH);
        // 检测已映射列名集合中是否包含当前列名
        if (mappedColumns.contains(upperColumnName)) {
            mappedColumnNames.add(upperColumnName);
        } else {
            // 将列名存入 unmappedColumnNames 中
            unmappedColumnNames.add(columnName);
        }
    }
    // 缓存列名集合
    mappedColumnNamesMap.put(getMapKey(resultMap, columnPrefix), mappedColumnNames);
    unMappedColumnNamesMap.put(getMapKey(resultMap, columnPrefix), unmappedColumnNames);
}
```

首先是从当前数据集中获取列名集合，然后获取 <resultMap> 中配置的列名集合。之后遍历数据集中的列名集合，并判断列名是否被配置在了 <resultMap> 节点中。若配置了，则表明该列名已有映射关系，此时该列名存入 mappedColumnNames 中。若未配置，则表明列名未与实体类的某个字段形成映射关系，此时该列名存入 unmappedColumnNames 中。

#### 3.3、映射result节点

接下来分析一下 MyBatis 是如何将结果集中的数据填充到已配置ResultMap映射的实体类字段中的。

```java
private boolean applyPropertyMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject,ResultLoaderMap lazyLoader, String columnPrefix) throws SQLException {
    
    // 获取已映射的列名
    final List<String> mappedColumnNames = rsw.getMappedColumnNames(resultMap, columnPrefix);
    boolean foundValues = false;
    // 获取 ResultMapping集合
    final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
    // 所有的ResultMapping遍历进行映射
    for (ResultMapping propertyMapping : propertyMappings) {
        String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
        if (propertyMapping.getNestedResultMapId() != null) {
            column = null;
        }
        if (propertyMapping.isCompositeResult()
            || (column != null && mappedColumnNames.contains(column.toUpperCase(Locale.ENGLISH)))
            || propertyMapping.getResultSet() != null) {
            
            // 从结果集中获取指定列的数据
            Object value = getPropertyMappingValue(rsw.getResultSet(), metaObject, propertyMapping, lazyLoader, columnPrefix);
            
            final String property = propertyMapping.getProperty();
            if (property == null) {
                continue;

            // 若获取到的值为 DEFERED，则延迟加载该值
            } else if (value == DEFERED) {
                foundValues = true;
                continue;
            }
            if (value != null) {
                foundValues = true;
            }
            if (value != null || (configuration.isCallSettersOnNulls() && !metaObject.getSetterType(property).isPrimitive())) {
                // 将获取到的值设置到实体类对象中
                metaObject.setValue(property, value);
            }
        }
    }
    return foundValues;
}

private Object getPropertyMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping,ResultLoaderMap lazyLoader, String columnPrefix) throws SQLException {

    if (propertyMapping.getNestedQueryId() != null) {
        // 获取关联查询结果
        return getNestedQueryMappingValue(rs, metaResultObject, propertyMapping, lazyLoader, columnPrefix);
    } else if (propertyMapping.getResultSet() != null) {
        addPendingChildRelation(rs, metaResultObject, propertyMapping);
        return DEFERED;
    } else {
        final TypeHandler<?> typeHandler = propertyMapping.getTypeHandler();
        final String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
        // 从 ResultSet 中获取指定列的值
        return typeHandler.getResult(rs, column);
    }
}
```

从 ResultMap 获取映射对象 ResultMapping 集合。然后遍历 ResultMapping 集合，再此过程中调用 getPropertyMappingValue 获取指定指定列的数据，最后将获取到的数据设置到实体类对象中。

这里和自动映射有一点不同，**自动映射是从直接从ResultSet 中获取指定列的值，但是通过ResultMap多了一种情况，那就是关联查询，也可以说是延迟查询，此关联查询如果没有配置延迟加载，那么就要获取关联查询的值，如果配置了延迟加载，则返回DEFERED**

#### 3.4、关联查询与延迟加载

我们的查询经常会碰到一对一，一对多的情况，通常我们可以用一条 SQL 进行多表查询完成任务。当然我们也可以使用关联查询，将一条 SQL 拆成两条去完成查询任务。MyBatis 提供了两个标签用于支持一对一和一对多的使用场景，分别是 <association> 和 <collection>。下面我来演示一下如何使用 <association> 完成一对一的关联查询。先来看看实体类的定义：

```java
/** 作者类 */
public class Author {
    private Integer id;
    private String name;
    private Integer age;
    private Integer sex;
    private String email;
    
    // 省略 getter/setter
}

/** 文章类 */
public class Article {
    private Integer id;
    private String title;
    // 一对一关系
    private Author author;
    private String content;
    private Date createTime;
    
    // 省略 getter/setter
}
```

接下来看一下 Mapper 接口与映射文件的定义。

```java
public interface ArticleDao {
    Article findOne(@Param("id") int id);
    Author findAuthor(@Param("id") int authorId);
}
```

```java
<mapper namespace="xyz.coolblog.dao.ArticleDao">
    <resultMap id="articleResult" type="Article">
        <result property="createTime" column="create_time"/>
        //column 属性值仅包含列信息，参数类型为 author_id 列对应的类型，这里为 Integer
        //意思是将author_id做为参数传给关联的查询语句findAuthor
        <association property="author" column="author_id" javaType="Author" select="findAuthor"/>
    </resultMap>

    <select id="findOne" resultMap="articleResult">
        SELECT
            id, author_id, title, content, create_time
        FROM
            article
        WHERE
            id = #{id}
    </select>

    <select id="findAuthor" resultType="Author">
        SELECT
            id, name, age, sex, email
        FROM
            author
        WHERE
            id = #{id}
    </select>
</mapper>
```

**开启延迟加载**

```java
<!-- 开启延迟加载 -->
<setting name="lazyLoadingEnabled" value="true"/>
<!-- 关闭积极的加载策略 -->
<setting name="aggressiveLazyLoading" value="false"/>
<!-- 延迟加载的触发方法 -->
<setting name="lazyLoadTriggerMethods" value="equals,hashCode"/>
```

**此时association节点使用了select指向另外一个查询语句，并且将 author_id作为参数传给关联查询的语句**

**此时如果不开启延迟加载，那么会生成两条SQL，先执行findOne，然后通过findOne的返回结果做为参数，执行findAuthor语句，并将结果设置到****author属性**

**如果开启了延迟加载呢？那么只会执行findOne一条SQL，当调用article.getAuthor()方法时，才会去执行findAuthor进行查询，我们下面来看看是如何实现的**

我们还是要从上面映射result节点说起



```java
private Object getPropertyMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping,ResultLoaderMap lazyLoader, String columnPrefix) throws SQLException {

    if (propertyMapping.getNestedQueryId() != null) {
        // 获取关联查询结果
        return getNestedQueryMappingValue(rs, metaResultObject, propertyMapping, lazyLoader, columnPrefix);
    } else if (propertyMapping.getResultSet() != null) {
        addPendingChildRelation(rs, metaResultObject, propertyMapping);
        return DEFERED;
    } else {
        final TypeHandler<?> typeHandler = propertyMapping.getTypeHandler();
        final String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
        // 从 ResultSet 中获取指定列的值
        return typeHandler.getResult(rs, column);
    }
}
```

我们看到，如果ResultMapping设置了关联查询，也就是**association或者collection配置了select，那么就要通过关联语句来查询结果，并设置到实体类对象的属性中了。如果没配置select，那就简单，直接从ResultSet中通过列名获取结果。**那我们来看看getNestedQueryMappingValue

```java
private Object getNestedQueryMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix) throws SQLException {

    // 获取关联查询 id，id = 命名空间 + <association> 的 select 属性值
    final String nestedQueryId = propertyMapping.getNestedQueryId();
    final String property = propertyMapping.getProperty();
    // 根据 nestedQueryId 获取关联的 MappedStatement
    final MappedStatement nestedQuery = configuration.getMappedStatement(nestedQueryId);
    //获取关联查询MappedStatement的参数类型
    final Class<?> nestedQueryParameterType = nestedQuery.getParameterMap().getType();
    /*
     * 生成关联查询语句参数对象，参数类型可能是一些包装类，Map 或是自定义的实体类，
     * 具体类型取决于配置信息。以上面的例子为基础，下面分析不同配置对参数类型的影响：
     *   1. <association column="author_id"> 
     *      column 属性值仅包含列信息，参数类型为 author_id 列对应的类型，这里为 Integer
     * 
     *   2. <association column="{id=author_id, name=title}"> 
     *      column 属性值包含了属性名与列名的复合信息，MyBatis 会根据列名从 ResultSet 中
     *      获取列数据，并将列数据设置到实体类对象的指定属性中，比如：
     *          Author{id=1, name="陈浩"}
     *      或是以键值对 <属性, 列数据> 的形式，将两者存入 Map 中。比如：
     *          {"id": 1, "name": "陈浩"}
     *
     *      至于参数类型到底为实体类还是 Map，取决于关联查询语句的配置信息。比如：
     *          <select id="findAuthor">  ->  参数类型为 Map
     *          <select id="findAuthor" parameterType="Author"> -> 参数类型为实体类
     */
    final Object nestedQueryParameterObject = prepareParameterForNestedQuery(rs, propertyMapping, nestedQueryParameterType, columnPrefix);
    Object value = null;
    if (nestedQueryParameterObject != null) {
        // 获取 BoundSql，这里设置了运行时参数，所以这里是能直接执行的
        final BoundSql nestedBoundSql = nestedQuery.getBoundSql(nestedQueryParameterObject);
        final CacheKey key = executor.createCacheKey(nestedQuery, nestedQueryParameterObject, RowBounds.DEFAULT, nestedBoundSql);
        final Class<?> targetType = propertyMapping.getJavaType();

        if (executor.isCached(nestedQuery, key)) {
            executor.deferLoad(nestedQuery, metaResultObject, property, key, targetType);
            value = DEFERED;
        } else {
            // 创建结果加载器
            final ResultLoader resultLoader = new ResultLoader(configuration, executor, nestedQuery, nestedQueryParameterObject, targetType, key, nestedBoundSql);
            // 检测当前属性是否需要延迟加载
            if (propertyMapping.isLazy()) {
                // 添加延迟加载相关的对象到 loaderMap 集合中
                lazyLoader.addLoader(property, metaResultObject, resultLoader);
                value = DEFERED;
            } else {
                // 直接执行关联查询
                // 如果只是配置关联查询，但是没有开启懒加载，则直接执行关联查询，并返回结果，设置到实体类对象的属性中
                value = resultLoader.loadResult();
            }
        }
    }
    return value;
}
```

下面先来总结一下该方法的逻辑：

1. 根据 nestedQueryId 获取 MappedStatement
2. 生成参数对象
3. 获取 BoundSql
4. 创建结果加载器 ResultLoader
5. 检测当前属性是否需要进行延迟加载，若需要，则添加延迟加载相关的对象到 loaderMap 集合中
6. 如不需要延迟加载，则直接通过结果加载器加载结果

以上流程中针对一级缓存的检查是十分有必要的，若缓存命中，可直接取用结果，无需再在执行关联查询 SQL。若缓存未命中，接下来就要按部就班执行延迟加载相关逻辑

我们来看一下添加延迟加载相关对象到 loaderMap 集合中的逻辑，如下：

```java
public void addLoader(String property, MetaObject metaResultObject, ResultLoader resultLoader) {
    // 将属性名转为大写
    String upperFirst = getUppercaseFirstProperty(property);
    if (!upperFirst.equalsIgnoreCase(property) && loaderMap.containsKey(upperFirst)) {
        throw new ExecutorException("Nested lazy loaded result property '" + property +
                                    "' for query id '" + resultLoader.mappedStatement.getId() +
                                    " already exists in the result map. The leftmost property of all lazy loaded properties must be unique within a result map.");
    }
    // 创建 LoadPair，并将 <大写属性名，LoadPair对象> 键值对添加到 loaderMap 中
    loaderMap.put(upperFirst, new LoadPair(property, metaResultObject, resultLoader));
}
```

我们再来回顾一下文章开始的创建实体类

```java
private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, ResultLoaderMap lazyLoader, String columnPrefix) throws SQLException {

    this.useConstructorMappings = false;
    final List<Class<?>> constructorArgTypes = new ArrayList<Class<?>>();
    final List<Object> constructorArgs = new ArrayList<Object>();

    // 调用重载方法创建实体类对象
    Object resultObject = createResultObject(rsw, resultMap, constructorArgTypes, constructorArgs, columnPrefix);
    if (resultObject != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
        final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
        for (ResultMapping propertyMapping : propertyMappings) {
            // 如果开启了延迟加载，则为 resultObject 生成代理类，如果仅仅是配置的关联查询，没有开启延迟加载，是不会创建代理类
            if (propertyMapping.getNestedQueryId() != null && propertyMapping.isLazy()) {
                /*
                 * 创建代理类，默认使用 Javassist 框架生成代理类。
                 * 由于实体类通常不会实现接口，所以不能使用 JDK 动态代理 API 为实体类生成代理。
                 * 并且将lazyLoader传进去了
                 */
                resultObject = configuration.getProxyFactory()
                    .createProxy(resultObject, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
                break;
            }
        }
    }
    this.useConstructorMappings =
        resultObject != null && !constructorArgTypes.isEmpty();
    return resultObject;
}
```

如果开启了延迟加载，并且有关联查询，此时是要创建一个代理对象的，**将上面存放BondSql的lazyLoader和创建的目标对象****resultObject 作为参数传进去。**

Mybatis提供了两个实现类CglibProxyFactory和JavassistProxyFactory，分别基于org.javassist:javassist和cglib:cglib进行实现。createProxy方法就是实现懒加载逻辑的核心方法，也是我们分析的目标。

##### 3.4.1、CglibProxyFactory

CglibProxyFactory基于cglib动态代理模式，**通过继承父类的方式生成动态代理类。**

```java
@Override
public Object createProxy(Object target, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
  return EnhancedResultObjectProxyImpl.createProxy(target, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
}

public static Object createProxy(Object target, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
  final Class<?> type = target.getClass();
  EnhancedResultObjectProxyImpl callback = new EnhancedResultObjectProxyImpl(type, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
  //由CglibProxyFactory生成对象
  Object enhanced = crateProxy(type, callback, constructorArgTypes, constructorArgs);
  //复制属性
  PropertyCopier.copyBeanProperties(type, target, enhanced);
  return enhanced;
}

static Object crateProxy(Class<?> type, Callback callback, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
  Enhancer enhancer = new Enhancer();
  enhancer.setCallback(callback);
  //设置父类对象
  enhancer.setSuperclass(type);
  try {
    type.getDeclaredMethod(WRITE_REPLACE_METHOD);
    // ObjectOutputStream will call writeReplace of objects returned by writeReplace
    if (log.isDebugEnabled()) {
      log.debug(WRITE_REPLACE_METHOD + " method was found on bean " + type + ", make sure it returns this");
    }
  } catch (NoSuchMethodException e) {
    enhancer.setInterfaces(new Class[]{WriteReplaceInterface.class});
  } catch (SecurityException e) {
    // nothing to do here
  }
  Object enhanced;
  if (constructorArgTypes.isEmpty()) {
    enhanced = enhancer.create();
  } else {
    Class<?>[] typesArray = constructorArgTypes.toArray(new Class[constructorArgTypes.size()]);
    Object[] valuesArray = constructorArgs.toArray(new Object[constructorArgs.size()]);
    enhanced = enhancer.create(typesArray, valuesArray);
  }
  return enhanced;
}
```

 可以看到，初始化Enhancer，并调用构造方法，生成对象。从`enhancer.setSuperclass(type);`也能看出cglib采用的是继承父类的方式。

**EnhancedResultObjectProxyImpl**

EnhancedResultObjectProxyImpl实现了MethodInterceptor接口，此接口是Cglib拦截目标对象方法的入口，对目标对象方法的调用都会通过此接口的intercept的方法。

```java
@Override
public Object intercept(Object enhanced, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
  final String methodName = method.getName();
  try {
    synchronized (lazyLoader) {
      if (WRITE_REPLACE_METHOD.equals(methodName)) {
        Object original;
        if (constructorArgTypes.isEmpty()) {
          original = objectFactory.create(type);
        } else {
          original = objectFactory.create(type, constructorArgTypes, constructorArgs);
        }
        PropertyCopier.copyBeanProperties(type, enhanced, original);
        if (lazyLoader.size() > 0) {
          return new CglibSerialStateHolder(original, lazyLoader.getProperties(), objectFactory, constructorArgTypes, constructorArgs);
        } else {
          return original;
        }
      } else {
        if (lazyLoader.size() > 0 && !FINALIZE_METHOD.equals(methodName)) {
        /*
         * 如果 aggressive 为 true，或触发方法（比如 equals，hashCode 等）被调用，
         * 则加载所有的所有延迟加载的数据
         */
          if (aggressive || lazyLoadTriggerMethods.contains(methodName)) {
            lazyLoader.loadAll();
          } else if (PropertyNamer.isSetter(methodName)) {
            // 如果使用者显示调用了 setter 方法，则将相应的延迟加载类从 loaderMap 中移除
            final String property = PropertyNamer.methodToProperty(methodName);
            lazyLoader.remove(property);
          // 检测使用者是否调用 getter 方法
          } else if (PropertyNamer.isGetter(methodName)) {
            final String property = PropertyNamer.methodToProperty(methodName);
            if (lazyLoader.hasLoader(property)) {
              // 执行延迟加载逻辑
              lazyLoader.load(property);
            }
          }
        }
      }
    }
    //执行原方法（即父类方法）
    return methodProxy.invokeSuper(enhanced, args);
  } catch (Throwable t) {
    throw ExceptionUtil.unwrapThrowable(t);
  }
}
```

完整的代码

```java
import java.lang.reflect.Method;
import java.util.List;
import java.util.Map;
import java.util.Properties;
import java.util.Set;

import net.sf.cglib.proxy.Callback;
import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

import org.apache.ibatis.executor.loader.AbstractEnhancedDeserializationProxy;
import org.apache.ibatis.executor.loader.AbstractSerialStateHolder;
import org.apache.ibatis.executor.loader.ProxyFactory;
import org.apache.ibatis.executor.loader.ResultLoaderMap;
import org.apache.ibatis.executor.loader.WriteReplaceInterface;
import org.apache.ibatis.io.Resources;
import org.apache.ibatis.logging.Log;
import org.apache.ibatis.logging.LogFactory;
import org.apache.ibatis.reflection.ExceptionUtil;
import org.apache.ibatis.reflection.factory.ObjectFactory;
import org.apache.ibatis.reflection.property.PropertyCopier;
import org.apache.ibatis.reflection.property.PropertyNamer;
import org.apache.ibatis.session.Configuration;

/**
 * cglib代理工厂类，实现延迟加载属性
 * @author Clinton Begin
 */
public class CglibProxyFactory implements ProxyFactory {

  /**
   * finalize方法
   */
  private static final String FINALIZE_METHOD = "finalize";
  /**
   * writeReplace方法
   */
  private static final String WRITE_REPLACE_METHOD = "writeReplace";

  /**
   * 加载Enhancer，这个是Cglib的入口
   */
  public CglibProxyFactory() {
    try {
      Resources.classForName("net.sf.cglib.proxy.Enhancer");
    } catch (Throwable e) {
      throw new IllegalStateException("Cannot enable lazy loading because CGLIB is not available. Add CGLIB to your classpath.", e);
    }
  }

  /**
   * 创建代理对象
   * @param target 目标对象
   * @param lazyLoader 延迟加载器
   * @param configuration 配置类
   * @param objectFactory 对象工厂
   * @param constructorArgTypes 构造函数类型[]
   * @param constructorArgs  构造函数的值[]
   * @return
   */
  @Override
  public Object createProxy(Object target, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
    return EnhancedResultObjectProxyImpl.createProxy(target, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
  }

  /**
   * 创建一个反序列化代理
   * @param target 目标
   * @param unloadedProperties
   * @param objectFactory 对象工厂
   * @param constructorArgTypes 构造函数类型数组
   * @param constructorArgs 构造函数值
   * @return
   */
  public Object createDeserializationProxy(Object target, Map<String, ResultLoaderMap.LoadPair> unloadedProperties, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
    return EnhancedDeserializationProxyImpl.createProxy(target, unloadedProperties, objectFactory, constructorArgTypes, constructorArgs);
  }

  @Override
  public void setProperties(Properties properties) {
      // Not Implemented
  }

  /**
   * 返回代理对象， 这个代理对象在调用任何方法都会调用本类的intercept方法
   * Enhancer 认为这个就是自定义类的工厂，比如这个类需要实现什么接口
   * @param type 目标类型
   * @param callback 结果对象代理实现类，当中有invoke回调方法
   * @param constructorArgTypes 构造函数类型数组
   * @param constructorArgs 构造函数对应字段的值数组
   * @return
   */
  static Object crateProxy(Class<?> type, Callback callback, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
    // enhancer 配置调节代理对象的一些参数
    // 设置回调方法
    // 设置超类
    //判断当传入目标类型是否有writeReplace方法，没有则配置一个有writeReplace方法的接口（序列化写出）
    Enhancer enhancer = new Enhancer();
    enhancer.setCallback(callback);
    enhancer.setSuperclass(type);
    try {
      type.getDeclaredMethod(WRITE_REPLACE_METHOD);
      // ObjectOutputStream will call writeReplace of objects returned by writeReplace
      if (LogHolder.log.isDebugEnabled()) {
        LogHolder.log.debug(WRITE_REPLACE_METHOD + " method was found on bean " + type + ", make sure it returns this");
      }
    } catch (NoSuchMethodException e) {
      //这个enhancer增加一个WriteReplaceInterface接口
      enhancer.setInterfaces(new Class[]{WriteReplaceInterface.class});
    } catch (SecurityException e) {
      // nothing to do here
    }
    //根据构造函数创建一个对象
    //无参构造
    //有参构造
    Object enhanced;
    if (constructorArgTypes.isEmpty()) {
      enhanced = enhancer.create();
    } else {
      Class<?>[] typesArray = constructorArgTypes.toArray(new Class[constructorArgTypes.size()]);
      Object[] valuesArray = constructorArgs.toArray(new Object[constructorArgs.size()]);
      enhanced = enhancer.create(typesArray, valuesArray);
    }
    return enhanced;
  }

  /**
   * 结果对象代理实现类，
   * 它实现方法拦截器的intercept方法
   */
  private static class EnhancedResultObjectProxyImpl implements MethodInterceptor {

    private final Class<?> type;
    private final ResultLoaderMap lazyLoader;
    private final boolean aggressive;
    private final Set<String> lazyLoadTriggerMethods;
    private final ObjectFactory objectFactory;
    private final List<Class<?>> constructorArgTypes;
    private final List<Object> constructorArgs;

    /**
     * 代理对象创建
     * @param type 目标class类型
     * @param lazyLoader 延迟加载器
     * @param configuration 配置信息
     * @param objectFactory 对象工厂
     * @param constructorArgTypes 构造函数类型数组
     * @param constructorArgs 构造函数值数组
     */
    private EnhancedResultObjectProxyImpl(Class<?> type, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
      this.type = type;
      this.lazyLoader = lazyLoader;
      this.aggressive = configuration.isAggressiveLazyLoading();
      this.lazyLoadTriggerMethods = configuration.getLazyLoadTriggerMethods();
      this.objectFactory = objectFactory;
      this.constructorArgTypes = constructorArgTypes;
      this.constructorArgs = constructorArgs;
    }

    /**
     * 创建代理对象, 将源对象值赋值给代理对象
     * @param target 目标对象
     * @param lazyLoader 延迟加载器
     * @param configuration 配置对象
     * @param objectFactory 对象工厂
     * @param constructorArgTypes 构造函数类型数组
     * @param constructorArgs 构造函数值数组
     * @return
     */
    public static Object createProxy(Object target, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
      //获取目标的类型
      //创建一个结果对象代理实现类（它实现cglib的MethodInterface接口，完成回调作用invoke方法）
      final Class<?> type = target.getClass();
      EnhancedResultObjectProxyImpl callback = new EnhancedResultObjectProxyImpl(type, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
      Object enhanced = crateProxy(type, callback, constructorArgTypes, constructorArgs);
      PropertyCopier.copyBeanProperties(type, target, enhanced);
      return enhanced;
    }

    /**
     * 回调方法
     * @param enhanced 代理对象
     * @param method 方法
     * @param args 方法参数
     * @param methodProxy 代理方法
     * @return
     * @throws Throwable
     */
    @Override
    public Object intercept(Object enhanced, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
      //获取方法名
      final String methodName = method.getName();
      try {
        // 同步获取延迟加载对象
        // 如果是执行writeReplace方法(序列化写出）
        // 实例化一个目标对象的实例
        synchronized (lazyLoader) {
          if (WRITE_REPLACE_METHOD.equals(methodName)) {
            Object original;
            if (constructorArgTypes.isEmpty()) {
              original = objectFactory.create(type);
            } else {
              original = objectFactory.create(type, constructorArgTypes, constructorArgs);
            }
            // 将enhanced中的属性复制到orignal对象中
            // 如果延迟加载数量>0,
            PropertyCopier.copyBeanProperties(type, enhanced, original);
            if (lazyLoader.size() > 0) {
              return new CglibSerialStateHolder(original, lazyLoader.getProperties(), objectFactory, constructorArgTypes, constructorArgs);
            } else {
              return original;
            }
          } else {
            //不是writeReplace方法
            // 延迟加载长度大于0， 且不是finalize方法
            // configuration配置延迟加载参数，延迟加载触发的方法包含这个方法
            // 延迟加载所有数据
            if (lazyLoader.size() > 0 && !FINALIZE_METHOD.equals(methodName)) {
              if (aggressive || lazyLoadTriggerMethods.contains(methodName)) {
                lazyLoader.loadAll();
                // setter方法，直接移除
              } else if (PropertyNamer.isSetter(methodName)) {
                final String property = PropertyNamer.methodToProperty(methodName);
                lazyLoader.remove(property);
                // getter方法， 加载该属性
              } else if (PropertyNamer.isGetter(methodName)) {
                final String property = PropertyNamer.methodToProperty(methodName);
                if (lazyLoader.hasLoader(property)) {
                  lazyLoader.load(property);
                }
              }
            }
          }
        }
        return methodProxy.invokeSuper(enhanced, args);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    }
  }

  /**
   * 他继承抽象反序列化代理和实现了方法拦截
   */
  private static class EnhancedDeserializationProxyImpl extends AbstractEnhancedDeserializationProxy implements MethodInterceptor {

    private EnhancedDeserializationProxyImpl(Class<?> type, Map<String, ResultLoaderMap.LoadPair> unloadedProperties, ObjectFactory objectFactory,
            List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
      super(type, unloadedProperties, objectFactory, constructorArgTypes, constructorArgs);
    }

    /**
     * 创建代理对象
     * @param target
     * @param unloadedProperties
     * @param objectFactory
     * @param constructorArgTypes
     * @param constructorArgs
     * @return
     */
    public static Object createProxy(Object target, Map<String, ResultLoaderMap.LoadPair> unloadedProperties, ObjectFactory objectFactory,
            List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
      final Class<?> type = target.getClass();
      EnhancedDeserializationProxyImpl callback = new EnhancedDeserializationProxyImpl(type, unloadedProperties, objectFactory, constructorArgTypes, constructorArgs);
      Object enhanced = crateProxy(type, callback, constructorArgTypes, constructorArgs);
      PropertyCopier.copyBeanProperties(type, target, enhanced);
      return enhanced;
    }

    @Override
    public Object intercept(Object enhanced, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
      final Object o = super.invoke(enhanced, method, args);
      return o instanceof AbstractSerialStateHolder ? o : methodProxy.invokeSuper(o, args);
    }

    @Override
    protected AbstractSerialStateHolder newSerialStateHolder(Object userBean, Map<String, ResultLoaderMap.LoadPair> unloadedProperties, ObjectFactory objectFactory,
            List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
      return new CglibSerialStateHolder(userBean, unloadedProperties, objectFactory, constructorArgTypes, constructorArgs);
    }
  }


}
```



如上，代理方法首先会检查 aggressive 是否为 true，如果不满足，再去检查 lazyLoadTriggerMethods 是否包含当前方法名。这里两个条件只要一个为 true，当前实体类中所有需要延迟加载。aggressive 和 lazyLoadTriggerMethods 两个变量的值取决于下面的配置。

```java
<setting name="aggressiveLazyLoading" value="false"/>
<setting name="lazyLoadTriggerMethods" value="equals,hashCode"/>
```

然后代理逻辑会检查使用者是不是调用了实体类的 setter 方法，如果调用了，就将该属性对应的 LoadPair 从 loaderMap 中移除。为什么要这么做呢？答案是：使用者既然手动调用 setter 方法，说明使用者想自定义某个属性的值。此时，延迟加载逻辑不应该再修改该属性的值，所以这里从 loaderMap 中移除属性对于的 LoadPair。

最后如果使用者调用的是某个属性的 **getter 方法**，且该属性配置了延迟加载，此时延迟加载逻辑就会被触发。那接下来，我们来看看延迟加载逻辑是怎样实现的的。



```java
public boolean load(String property) throws SQLException {
    // 从 loaderMap 中移除 property 所对应的 LoadPair
    LoadPair pair = loaderMap.remove(property.toUpperCase(Locale.ENGLISH));
    if (pair != null) {
        // 加载结果
        pair.load();
        return true;
    }
    return false;
}

public void load(final Object userObject) throws SQLException {
    /*
     * 调用 ResultLoader 的 loadResult 方法加载结果，
     * 并通过 metaResultObject 设置结果到实体类对象中
     */
    this.metaResultObject.setValue(property, this.resultLoader.loadResult());
}

public Object loadResult() throws SQLException {
    // 执行关联查询
    List<Object> list = selectList();
    // 抽取结果
    resultObject = resultExtractor.extractObjectFromList(list, targetType);
    return resultObject;
}

private <E> List<E> selectList() throws SQLException {
    Executor localExecutor = executor;
    if (Thread.currentThread().getId() != this.creatorThreadId || localExecutor.isClosed()) {
        localExecutor = newExecutor();
    }
    try {
        // 通过 Executor 就行查询，这个之前已经分析过了
        // 这里的parameterObject和boundSql就是我们之前存放在LoadPair中的，现在直接拿来执行了
        return localExecutor.<E>query(mappedStatement, parameterObject, RowBounds.DEFAULT,
                                      Executor.NO_RESULT_HANDLER, cacheKey, boundSql);
    } finally {
        if (localExecutor != executor) {
            localExecutor.close(false);
        }
    }
}
```



好了，延迟加载我们基本已经讲清楚了，我们介绍一下另外的一种代理方式

##### 3.4.2、JavassistProxyFactory

JavassistProxyFactory使用的是javassist方式，直接修改class文件的字节码格式。

```java
import java.lang.reflect.Method;
import java.util.List;
import java.util.Map;
import java.util.Properties;
import java.util.Set;

import javassist.util.proxy.MethodHandler;
import javassist.util.proxy.Proxy;
import javassist.util.proxy.ProxyFactory;

import org.apache.ibatis.executor.ExecutorException;
import org.apache.ibatis.executor.loader.AbstractEnhancedDeserializationProxy;
import org.apache.ibatis.executor.loader.AbstractSerialStateHolder;
import org.apache.ibatis.executor.loader.ResultLoaderMap;
import org.apache.ibatis.executor.loader.WriteReplaceInterface;
import org.apache.ibatis.io.Resources;
import org.apache.ibatis.logging.Log;
import org.apache.ibatis.logging.LogFactory;
import org.apache.ibatis.reflection.ExceptionUtil;
import org.apache.ibatis.reflection.factory.ObjectFactory;
import org.apache.ibatis.reflection.property.PropertyCopier;
import org.apache.ibatis.reflection.property.PropertyNamer;
import org.apache.ibatis.session.Configuration;

/**JavassistProxy字节码生成代理
 * 1.创建一个代理对象然后将目标对象的值赋值给代理对象，这个代理对象是可以实现其他的接口
 * 2. JavassistProxyFactory实现ProxyFactory接口createProxy(创建代理对象的方法)
 * @author Eduardo Macarron
 */
public class JavassistProxyFactory implements org.apache.ibatis.executor.loader.ProxyFactory {

  /**
   * finalize方法（垃圾回收）
   */
  private static final String FINALIZE_METHOD = "finalize";

  /**
   * writeReplace(序列化写出方法）
   */
  private static final String WRITE_REPLACE_METHOD = "writeReplace";

  /**
   * 加载ProxyFactory, 也就是JavassistProxy的入口
   */
  public JavassistProxyFactory() {
    try {
      Resources.classForName("javassist.util.proxy.ProxyFactory");
    } catch (Throwable e) {
      throw new IllegalStateException("Cannot enable lazy loading because Javassist is not available. Add Javassist to your classpath.", e);
    }
  }

  /**
   * 创建代理
   * @param target 目标对象
   * @param lazyLoader 延迟加载Map集合（那些属性是需要延迟加载的）
   * @param configuration 配置类
   * @param objectFactory 对象工厂
   * @param constructorArgTypes 构造函数类型[]
   * @param constructorArgs  构造函数的值[]
   * @return
   */
  @Override
  public Object createProxy(Object target, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
    return EnhancedResultObjectProxyImpl.createProxy(target, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
  }

  public Object createDeserializationProxy(Object target, Map<String, ResultLoaderMap.LoadPair> unloadedProperties, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
    return EnhancedDeserializationProxyImpl.createProxy(target, unloadedProperties, objectFactory, constructorArgTypes, constructorArgs);
  }

  @Override
  public void setProperties(Properties properties) {
      // Not Implemented
  }

  /**
   * 获取代理对象， 也就是说在执行方法之前首先调用MethodHanlder的invoke方法
   * @param type 目标类型
   * @param callback 回调对象
   * @param constructorArgTypes 构造函数类型数组
   * @param constructorArgs 构造函数值的数组
   * @return
   */
  static Object crateProxy(Class<?> type, MethodHandler callback, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
    // 创建一个代理工厂类
    // 配置超类
    ProxyFactory enhancer = new ProxyFactory();
    enhancer.setSuperclass(type);
    //判断是否有writeReplace方法，如果没有将这个代理对象实现WriteReplaceInterface接口，这个接口只有一个writeReplace方法
    try {
      type.getDeclaredMethod(WRITE_REPLACE_METHOD);
      // ObjectOutputStream will call writeReplace of objects returned by writeReplace
      if (LogHolder.log.isDebugEnabled()) {
        LogHolder.log.debug(WRITE_REPLACE_METHOD + " method was found on bean " + type + ", make sure it returns this");
      }
    } catch (NoSuchMethodException e) {
      enhancer.setInterfaces(new Class[]{WriteReplaceInterface.class});
    } catch (SecurityException e) {
      // nothing to do here
    }

    Object enhanced;
    Class<?>[] typesArray = constructorArgTypes.toArray(new Class[constructorArgTypes.size()]);
    Object[] valuesArray = constructorArgs.toArray(new Object[constructorArgs.size()]);
    try {
      // 根据构造函数创建一个代理对象
      enhanced = enhancer.create(typesArray, valuesArray);
    } catch (Exception e) {
      throw new ExecutorException("Error creating lazy proxy.  Cause: " + e, e);
    }
    // 设置回调对象
    ((Proxy) enhanced).setHandler(callback);
    return enhanced;
  }

  /**
   * 实现Javassist的MethodHandler接口， 相对于Cglib的MethodInterceptor
   * 他们接口的方法名也是不一样的，Javassist的是invoke, 而cglib是intercept，叫法不同，实现功能是一样的
   */
  private static class EnhancedResultObjectProxyImpl implements MethodHandler {

    /**
     * 目标类型
     */
    private final Class<?> type;
    /**
     * 延迟加载Map集合
     */
    private final ResultLoaderMap lazyLoader;

    /**
     * 是否配置延迟加载
     */
    private final boolean aggressive;

    /**
     * 延迟加载触发的方法
     */
    private final Set<String> lazyLoadTriggerMethods;

    /**
     * 对象工厂
     */
    private final ObjectFactory objectFactory;

    /**
     * 构造函数类型数组
     */
    private final List<Class<?>> constructorArgTypes;

    /**
     * 构造函数类型的值数组
     */
    private final List<Object> constructorArgs;

    /**
     * 构造函数私有化了
     * @param type
     * @param lazyLoader
     * @param configuration
     * @param objectFactory
     * @param constructorArgTypes
     * @param constructorArgs
     */
    private EnhancedResultObjectProxyImpl(Class<?> type, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
      this.type = type;
      this.lazyLoader = lazyLoader;
      this.aggressive = configuration.isAggressiveLazyLoading();
      this.lazyLoadTriggerMethods = configuration.getLazyLoadTriggerMethods();
      this.objectFactory = objectFactory;
      this.constructorArgTypes = constructorArgTypes;
      this.constructorArgs = constructorArgs;
    }

    public static Object createProxy(Object target, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
      // 获取目标类型
      // 创建一个EnhancedResultObjectProxyImpl对象，回调对象
      final Class<?> type = target.getClass();
      EnhancedResultObjectProxyImpl callback = new EnhancedResultObjectProxyImpl(type, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
      Object enhanced = crateProxy(type, callback, constructorArgTypes, constructorArgs);
      PropertyCopier.copyBeanProperties(type, target, enhanced);
      return enhanced;
    }

    /**
     * 回调方法
     * @param enhanced 代理对象
     * @param method 方法
     * @param methodProxy 代理方法
     * @param args 入参
     * @return
     * @throws Throwable
     */
    @Override
    public Object invoke(Object enhanced, Method method, Method methodProxy, Object[] args) throws Throwable {
      //获取方法名称
      final String methodName = method.getName();
      try {
        synchronized (lazyLoader) {
          if (WRITE_REPLACE_METHOD.equals(methodName)) {
            //如果方法是writeReplace
            Object original;
            if (constructorArgTypes.isEmpty()) {
              original = objectFactory.create(type);
            } else {
              original = objectFactory.create(type, constructorArgTypes, constructorArgs);
            }
            PropertyCopier.copyBeanProperties(type, enhanced, original);
            if (lazyLoader.size() > 0) {
              return new JavassistSerialStateHolder(original, lazyLoader.getProperties(), objectFactory, constructorArgTypes, constructorArgs);
            } else {
              return original;
            }
          } else {
            //不是writeReplace方法
            // 延迟加载长度大于0， 且不是finalize方法
            // configuration配置延迟加载参数，延迟加载触发的方法包含这个方法
            // 延迟加载所有数据
            if (lazyLoader.size() > 0 && !FINALIZE_METHOD.equals(methodName)) {
              if (aggressive || lazyLoadTriggerMethods.contains(methodName)) {
                lazyLoader.loadAll();
              } else if (PropertyNamer.isSetter(methodName)) {
                final String property = PropertyNamer.methodToProperty(methodName);
                lazyLoader.remove(property);
              } else if (PropertyNamer.isGetter(methodName)) {
                final String property = PropertyNamer.methodToProperty(methodName);
                if (lazyLoader.hasLoader(property)) {
                  lazyLoader.load(property);
                }
              }
            }
          }
        }
        return methodProxy.invoke(enhanced, args);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    }
  }

  private static class EnhancedDeserializationProxyImpl extends AbstractEnhancedDeserializationProxy implements MethodHandler {

    private EnhancedDeserializationProxyImpl(Class<?> type, Map<String, ResultLoaderMap.LoadPair> unloadedProperties, ObjectFactory objectFactory,
            List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
      super(type, unloadedProperties, objectFactory, constructorArgTypes, constructorArgs);
    }

    public static Object createProxy(Object target, Map<String, ResultLoaderMap.LoadPair> unloadedProperties, ObjectFactory objectFactory,
            List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
      final Class<?> type = target.getClass();
      EnhancedDeserializationProxyImpl callback = new EnhancedDeserializationProxyImpl(type, unloadedProperties, objectFactory, constructorArgTypes, constructorArgs);
      Object enhanced = crateProxy(type, callback, constructorArgTypes, constructorArgs);
      PropertyCopier.copyBeanProperties(type, target, enhanced);
      return enhanced;
    }

    @Override
    public Object invoke(Object enhanced, Method method, Method methodProxy, Object[] args) throws Throwable {
      final Object o = super.invoke(enhanced, method, args);
      return o instanceof AbstractSerialStateHolder ? o : methodProxy.invoke(o, args);
    }

    @Override
    protected AbstractSerialStateHolder newSerialStateHolder(Object userBean, Map<String, ResultLoaderMap.LoadPair> unloadedProperties, ObjectFactory objectFactory,
            List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
      return new JavassistSerialStateHolder(userBean, unloadedProperties, objectFactory, constructorArgTypes, constructorArgs);
    }
  }

  private static class LogHolder {
    private static final Log log = LogFactory.getLog(JavassistProxyFactory.class);
  }

}
```
