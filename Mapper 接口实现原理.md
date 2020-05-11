## Mapper 接口实现原理

```java
// 这里返回的是 EmployeeMapper 接口的代理对象
EmployeeMapper employeeMapper = sqlSession.getMapper(EmployeeMapper.class);
List<Employee> allEmployees = employeeMapper.getAll();
```

### 一、为 Mapper 接口创建代理对象

```java
public <T> T getMapper(Class<T> type) {
    return configuration.<T>getMapper(type, this);
}

// Configuration
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
}

// MapperRegistry
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    // 从 knownMappers 中获取与 type 对应的 MapperProxyFactory
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
        throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
        // 创建代理对象
        return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
        throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
}
```

#### 1、获取 MapperProxyFactory

​	这需要从 knownMappers 集合说起，knownMappers 集合中的元素是何时存入的？MyBatis 在解析配置文件的` <mappers> `节点的过程中，会调用 MapperRegistry 的 addMapper 方法将 Class 到 MapperProxyFactory 对象的映射关系存入到 knownMappers。看下源码：

```java
private void bindMapperForNamespace() {
    // 获取映射文件的命名空间
    String namespace = builderAssistant.getCurrentNamespace();
    if (namespace != null) {
        Class<?> boundType = null;
        try {
            // 根据命名空间解析 mapper 类型
            boundType = Resources.classForName(namespace);
        } catch (ClassNotFoundException e) {
        }
        if (boundType != null) {
            // 检测当前 mapper 类是否被绑定过
            if (!configuration.hasMapper(boundType)) {
                configuration.addLoadedResource("namespace:" + namespace);
                // 绑定 mapper 类
                configuration.addMapper(boundType);
            }
        }
    }
}

// Configuration
public <T> void addMapper(Class<T> type) {
    // 通过 MapperRegistry 绑定 mapper 类
    mapperRegistry.addMapper(type);
}

// MapperRegistry
public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
        if (hasMapper(type)) {
            throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
        }
        boolean loadCompleted = false;
        try {
            /*
             * 将 type 和 MapperProxyFactory 进行绑定，MapperProxyFactory 可为 mapper 接口生成代理类
             */
            knownMappers.put(type, new MapperProxyFactory<T>(type));
            
            MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
            // 解析注解中的信息
            parser.parse();
            loadCompleted = true;
        } finally {
            if (!loadCompleted) {
                knownMappers.remove(type);
            }
        }
    }
}
```

​	在解析Mapper.xml的最后阶段，获取到Mapper.xml的namespace，然后利用反射，获取到namespace的Class,并创建一个 MapperProxyFactory的实例，namespace的Class作为参数，最后将namespace的Class为key， MapperProxyFactory的实例为value存入 knownMappers。 

#### 2、生成代理对象 return mapperProxyFactory.newInstance(sqlSession)

用的是 JDK 的动态代理：

```java
public class MapperProxyFactory<T> {
    // 存放 Mapper 接口 Class
    private final Class<T> mapperInterface;
    private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap();

    public MapperProxyFactory(Class<T> mapperInterface) {
        this.mapperInterface = mapperInterface;
    }

    public Class<T> getMapperInterface() {
        return this.mapperInterface;
    }

    public Map<Method, MapperMethod> getMethodCache() {
        return this.methodCache;
    }

    protected T newInstance(MapperProxy<T> mapperProxy) {
        // 生成 mapperInterface 的代理类
        return Proxy.newProxyInstance(this.mapperInterface.getClassLoader(), new Class[]{this.mapperInterface}, mapperProxy);
    }

    public T newInstance(SqlSession sqlSession) {
         /*
         * 创建 MapperProxy 对象，MapperProxy 实现了 InvocationHandler 接口，代理逻辑封装在此类中
         * 将 sqlSession 传入 MapperProxy 对象中，第二个参数是 Mapper 的接口，并不是其实现类
         */
        MapperProxy<T> mapperProxy = new MapperProxy(sqlSession, this.mapperInterface, this.methodCache);
        return this.newInstance(mapperProxy);
    }
}
```

​	这里要注意一点，MapperProxy这个InvocationHandler 创建的时候，传入的参数并不是Mapper接口的实现类，以前是怎么创建JDK动态代理的？先创建一个接口，然后再创建一个接口的实现类，最后创建一个InvocationHandler并将实现类传入其中作为目标类，创建接口的代理类，然后调用代理类方法时会回调InvocationHandler的invoke方法，最后在invoke方法中调用目标类的方法，但是我们这里调用Mapper接口代理类的方法时，需要调用其实现类的方法吗？不需要，需要调用对应的配置文件的SQL，所以这里并不需要传入Mapper的实现类到MapperProxy中，那Mapper接口的代理对象是如何调用对应配置文件的SQL呢？

### 二、Mapper 代理类如何执行 SQL？

```java
List<Employee> allEmployees = employeeMapper.getAll();
```

```java
public class MapperProxy<T> implements InvocationHandler, Serializable {
    private final SqlSession sqlSession;
    private final Class<T> mapperInterface;
    private final Map<Method, MapperMethod> methodCache;

    public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, Map<Method, MapperMethod> methodCache) {
        this.sqlSession = sqlSession;
        this.mapperInterface = mapperInterface;
        this.methodCache = methodCache;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 如果方法是定义在 Object 类中的，则直接调用
        if (Object.class.equals(method.getDeclaringClass())) {
            try {
                return method.invoke(this, args);
            } catch (Throwable var5) {
                throw ExceptionUtil.unwrapThrowable(var5);
            }
        } else {
            // 从缓存中获取 MapperMethod 对象，若缓存未命中，则创建 MapperMethod 对象
            final MapperMethod mapperMethod = cachedMapperMethod(method);
            // 调用 execute 方法执行 SQL
            return mapperMethod.execute(this.sqlSession, args);
        }
    }

    private MapperMethod cachedMapperMethod(Method method) {
        MapperMethod mapperMethod = (MapperMethod)this.methodCache.get(method);
        if (mapperMethod == null) {
            // 创建一个 MapperMethod，参数为 mapperInterface 和 method 还有 Configuration
            mapperMethod = new MapperMethod(this.mapperInterface, method, this.sqlSession.getConfiguration());
            this.methodCache.put(method, mapperMethod);
        }
        return mapperMethod;
    }
}
```

##### 创建 MapperMethod 对象，MapperMethod 包含 SqlCommand 和 MethodSignature 对象

```java
public class MapperMethod {

    // 包含 SQL 相关信息，比喻 MappedStatement 的 id 属性，（mapper.EmployeeMapper.getAll）
    private final SqlCommand command;
    // 包含了关于执行的 Mapper 方法的参数类型和返回类型。
    private final MethodSignature method;

    public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
        // 创建 SqlCommand 对象，该对象包含一些和 SQL 相关的信息
        this.command = new SqlCommand(config, mapperInterface, method);
        // 创建 MethodSignature 对象，从类名中可知，该对象包含了被拦截方法的一些信息
        this.method = new MethodSignature(config, mapperInterface, method);
    }
}
```

#### 1、创建 SqlCommand 对象

```java
public static class SqlCommand {
    //name 为 MappedStatement 的 id，也就是namespace.methodName（mapper.EmployeeMapper.getAll）
    private final String name;
    // SQL 的类型，如 insert，delete，update
    private final SqlCommandType type;

    public SqlCommand(Configuration configuration, Class<?> mapperInterface, Method method) {
        // 拼接 Mapper 接口名和方法名，（mapper.EmployeeMapper.getAll）
        String statementName = mapperInterface.getName() + "." + method.getName();
        MappedStatement ms = null;
        // 检测 configuration 是否有 key 为 mapper.EmployeeMapper.getAll 的 MappedStatement
        if (configuration.hasStatement(statementName)) {
            // 获取 MappedStatement
            ms = configuration.getMappedStatement(statementName);
        } else if (!mapperInterface.equals(method.getDeclaringClass())) {
            String parentStatementName = method.getDeclaringClass().getName() + "." + method.getName();
            if (configuration.hasStatement(parentStatementName)) {
                ms = configuration.getMappedStatement(parentStatementName);
            }
        }
        
        // 检测当前方法是否有对应的 MappedStatement
        if (ms == null) {
            if (method.getAnnotation(Flush.class) != null) {
                name = null;
                type = SqlCommandType.FLUSH;
            } else {
                throw new BindingException("Invalid bound statement (not found): "
                    + mapperInterface.getName() + "." + methodName);
            }
        } else {
            // 设置 name 和 type 变量
            name = ms.getId();
            type = ms.getSqlCommandType();
            if (type == SqlCommandType.UNKNOWN) {
                throw new BindingException("Unknown execution method for: " + name);
            }
        }
    }
}

public boolean hasStatement(String statementName, boolean validateIncompleteStatements) {
    // 检测 configuration 是否有 key 为 statementNam e的 MappedStatement
    return this.mappedStatements.containsKey(statementName);
}
```

#### 2、创建 MethodSignature 对象

**MethodSignature** 包含了被拦截方法的一些信息，如目标方法的返回类型，目标方法的参数列表信息等。

```java
public static class MethodSignature {

    private final boolean returnsMany;
    private final boolean returnsMap;
    private final boolean returnsVoid;
    private final boolean returnsCursor;
    private final Class<?> returnType;
    private final String mapKey;
    private final Integer resultHandlerIndex;
    private final Integer rowBoundsIndex;
    private final ParamNameResolver paramNameResolver;

    public MethodSignature(Configuration configuration, Class<?> mapperInterface, Method method) {

        // 通过反射解析方法返回类型
        Type resolvedReturnType = TypeParameterResolver.resolveReturnType(method, mapperInterface);
        if (resolvedReturnType instanceof Class<?>) {
            this.returnType = (Class<?>) resolvedReturnType;
        } else if (resolvedReturnType instanceof ParameterizedType) {
            this.returnType = (Class<?>) ((ParameterizedType) resolvedReturnType).getRawType();
        } else {
            this.returnType = method.getReturnType();
        }
        
        // 检测返回值类型是否是 void、集合或数组、Cursor、Map 等
        this.returnsVoid = void.class.equals(this.returnType);
        this.returnsMany = configuration.getObjectFactory().isCollection(this.returnType) || this.returnType.isArray();
        this.returnsCursor = Cursor.class.equals(this.returnType);
        // 解析 @MapKey 注解，获取注解内容
        this.mapKey = getMapKey(method);
        this.returnsMap = this.mapKey != null;
        /*
         * 获取 RowBounds 参数在参数列表中的位置，如果参数列表中
         * 包含多个 RowBounds 参数，此方法会抛出异常
         */ 
        this.rowBoundsIndex = getUniqueParamIndex(method, RowBounds.class);
        // 获取 ResultHandler 参数在参数列表中的位置
        this.resultHandlerIndex = getUniqueParamIndex(method, ResultHandler.class);
        // 解析参数列表
        this.paramNameResolver = new ParamNameResolver(configuration, method);
    }
}
```

### 三、执行 execute 方法：mapperMethod.execute(this.sqlSession, args)

​	接下来要做的事情是调用 MapperMethod 的 execute 方法，执行 SQL。传递参数 sqlSession 和 method 的运行参数 args：

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

重点看下参数的处理方法convertArgsToSqlCommandParam是如何将方法参数数组转化成Map的：

```java
public Object convertArgsToSqlCommandParam(Object[] args) {
    return paramNameResolver.getNamedParams(args);
}

public Object getNamedParams(Object[] args) {
    final int paramCount = names.size();
    if (args == null || paramCount == 0) {
        return null;
    } else if (!hasParamAnnotation && paramCount == 1) {
        return args[names.firstKey()];
    } else {
        //创建一个Map，key为method的参数名，值为method的运行时参数值
        final Map<String, Object> param = new ParamMap<Object>();
        int i = 0;
        for (Map.Entry<Integer, String> entry : names.entrySet()) {
            // 添加 <参数名, 参数值> 键值对到 param 中
            param.put(entry.getValue(), args[entry.getKey()]);
            final String genericParamName = GENERIC_NAME_PREFIX + String.valueOf(i + 1);
            if (!names.containsValue(genericParamName)) {
                param.put(genericParamName, args[entry.getKey()]);
            }
            i++;
        }
        return param;
    }
}
```

​	可以看到是通过sqlSession来执行查询的，并且传入的参数为command.getName()和param，也就是**namespace.methodName（mapper.EmployeeMapper.getAll）和方法的运行参数。**