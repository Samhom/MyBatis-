## Mybatis 和 Spring 事务中用的 Connection 是同一个吗

**正文**

不知道一些同学有没有这种疑问，为什么Mybtis中要配置dataSource，Spring的事务中也要配置dataSource？那么Mybatis和Spring事务中用的Connection是同一个吗？我们常用配置如下



```xml
<!--会话工厂 -->
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
  <property name="dataSource" ref="dataSource" />
</bean>

<!--spring事务管理 -->
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
  <property name="dataSource" ref="dataSource" />
</bean>

<!--使用注释事务 -->
<tx:annotation-driven  transaction-manager="transactionManager" />
```



看到没，**sqlSessionFactory中配置了****dataSource，****transactionManager也配置了****dataSource，我们来回忆一下****SqlSessionFactoryBean这个类**



```java
 1 protected SqlSessionFactory buildSqlSessionFactory() throws IOException {
 2 
 3     // 配置类
 4    Configuration configuration;
 5     // 解析mybatis-Config.xml文件，
 6     // 将相关配置信息保存到configuration
 7    XMLConfigBuilder xmlConfigBuilder = null;
 8    if (this.configuration != null) {
 9      configuration = this.configuration;
10      if (configuration.getVariables() == null) {
11        configuration.setVariables(this.configurationProperties);
12      } else if (this.configurationProperties != null) {
13        configuration.getVariables().putAll(this.configurationProperties);
14      }
15     //资源文件不为空
16    } else if (this.configLocation != null) {
17      //根据configLocation创建xmlConfigBuilder，XMLConfigBuilder构造器中会创建Configuration对象
18      xmlConfigBuilder = new XMLConfigBuilder(this.configLocation.getInputStream(), null, this.configurationProperties);
19      //将XMLConfigBuilder构造器中创建的Configuration对象直接赋值给configuration属性
20      configuration = xmlConfigBuilder.getConfiguration();
21    } 
22    
23     //略....
24 
25    if (xmlConfigBuilder != null) {
26      try {
27        //解析mybatis-Config.xml文件，并将相关配置信息保存到configuration
28        xmlConfigBuilder.parse();
29        if (LOGGER.isDebugEnabled()) {
30          LOGGER.debug("Parsed configuration file: '" + this.configLocation + "'");
31        }
32      } catch (Exception ex) {
33        throw new NestedIOException("Failed to parse config resource: " + this.configLocation, ex);
34      }
35    }
36     
37    if (this.transactionFactory == null) {
38      //事务默认采用SpringManagedTransaction，这一块非常重要
39      this.transactionFactory = new SpringManagedTransactionFactory();
40    }
41     // 为sqlSessionFactory绑定事务管理器和数据源
42     // 这样sqlSessionFactory在创建sqlSession的时候可以通过该事务管理器获取jdbc连接，从而执行SQL
43    configuration.setEnvironment(new Environment(this.environment, this.transactionFactory, this.dataSource));
44     // 解析mapper.xml
45    if (!isEmpty(this.mapperLocations)) {
46      for (Resource mapperLocation : this.mapperLocations) {
47        if (mapperLocation == null) {
48          continue;
49        }
50        try {
51          // 解析mapper.xml文件，并注册到configuration对象的mapperRegistry
52          XMLMapperBuilder xmlMapperBuilder = new XMLMapperBuilder(mapperLocation.getInputStream(),
53              configuration, mapperLocation.toString(), configuration.getSqlFragments());
54          xmlMapperBuilder.parse();
55        } catch (Exception e) {
56          throw new NestedIOException("Failed to parse mapping resource: '" + mapperLocation + "'", e);
57        } finally {
58          ErrorContext.instance().reset();
59        }
60 
61        if (LOGGER.isDebugEnabled()) {
62          LOGGER.debug("Parsed mapper file: '" + mapperLocation + "'");
63        }
64      }
65    } else {
66      if (LOGGER.isDebugEnabled()) {
67        LOGGER.debug("Property 'mapperLocations' was not specified or no matching resources found");
68      }
69    }
70 
71     // 将Configuration对象实例作为参数，
72     // 调用sqlSessionFactoryBuilder创建sqlSessionFactory对象实例
73    return this.sqlSessionFactoryBuilder.build(configuration);
74 }
```



我们看第39行，Mybatis集成Spring后，默认使用的transactionFactory是SpringManagedTransactionFactory，那我们就来看看其获取Transaction的方法



```java
private SqlSession openSessionFromConnection(ExecutorType execType, Connection connection) {
    try {
      boolean autoCommit;
      try {
        autoCommit = connection.getAutoCommit();
      } catch (SQLException e) {
        // Failover to true, as most poor drivers
        // or databases won't support transactions
        autoCommit = true;
      }      
      //从configuration中取出environment对象
      final Environment environment = configuration.getEnvironment();
      //从environment中取出TransactionFactory
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      //创建Transaction
      final Transaction tx = transactionFactory.newTransaction(connection);
      //创建包含事务操作的执行器
      final Executor executor = configuration.newExecutor(tx, execType);
      //构建包含执行器的SqlSession
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
}

private TransactionFactory getTransactionFactoryFromEnvironment(Environment environment) {
    if (environment == null || environment.getTransactionFactory() == null) {
      return new ManagedTransactionFactory();
    }
    //这里返回SpringManagedTransactionFactory
    return environment.getTransactionFactory();
}

@Override
public Transaction newTransaction(DataSource dataSource, TransactionIsolationLevel level, boolean autoCommit) {
    //创建SpringManagedTransaction
    return new SpringManagedTransaction(dataSource);
}
```





## SpringManagedTransaction

也就是说mybatis的执行事务的事务管理器就切换成了SpringManagedTransaction，下面我们再去看看SpringManagedTransactionFactory类的源码：



```java
public class SpringManagedTransaction implements Transaction {
    private static final Log LOGGER = LogFactory.getLog(SpringManagedTransaction.class);
    private final DataSource dataSource;
    private Connection connection;
    private boolean isConnectionTransactional;
    private boolean autoCommit;

    public SpringManagedTransaction(DataSource dataSource) {
        Assert.notNull(dataSource, "No DataSource specified");
        this.dataSource = dataSource;
    }

    public Connection getConnection() throws SQLException {
        if (this.connection == null) {
            this.openConnection();
        }

        return this.connection;
    }

    private void openConnection() throws SQLException {
        //通过DataSourceUtils获取connection，这里和JdbcTransaction不一样
        this.connection = DataSourceUtils.getConnection(this.dataSource);
        this.autoCommit = this.connection.getAutoCommit();
        this.isConnectionTransactional = DataSourceUtils.isConnectionTransactional(this.connection, this.dataSource);
        if (LOGGER.isDebugEnabled()) {
            LOGGER.debug("JDBC Connection [" + this.connection + "] will" + (this.isConnectionTransactional ? " " : " not ") + "be managed by Spring");
        }

    }

    public void commit() throws SQLException {
        if (this.connection != null && !this.isConnectionTransactional && !this.autoCommit) {
            if (LOGGER.isDebugEnabled()) {
                LOGGER.debug("Committing JDBC Connection [" + this.connection + "]");
            }
            //通过connection提交，这里和JdbcTransaction一样
            this.connection.commit();
        }

    }

    public void rollback() throws SQLException {
        if (this.connection != null && !this.isConnectionTransactional && !this.autoCommit) {
            if (LOGGER.isDebugEnabled()) {
                LOGGER.debug("Rolling back JDBC Connection [" + this.connection + "]");
            }
            //通过connection回滚，这里和JdbcTransaction一样
            this.connection.rollback();
        }

    }

    public void close() throws SQLException {
        DataSourceUtils.releaseConnection(this.connection, this.dataSource);
    }

    public Integer getTimeout() throws SQLException {
        ConnectionHolder holder = (ConnectionHolder)TransactionSynchronizationManager.getResource(this.dataSource);
        return holder != null && holder.hasTimeout() ? holder.getTimeToLiveInSeconds() : null;
    }
}
```





### org.springframework.jdbc.datasource.DataSourceUtils#getConnection



```java
public static Connection getConnection(DataSource dataSource) throws CannotGetJdbcConnectionException {
    try {
        return doGetConnection(dataSource);
    }
    catch (SQLException ex) {
        throw new CannotGetJdbcConnectionException("Could not get JDBC Connection", ex);
    }
}

public static Connection doGetConnection(DataSource dataSource) throws SQLException {
    Assert.notNull(dataSource, "No DataSource specified");
    //TransactionSynchronizationManager重点！！！有没有很熟悉的感觉？？
    //还记得我们前面Spring事务源码的分析吗？@Transaction会创建Connection，并放入ThreadLocal中
    //这里从ThreadLocal中获取ConnectionHolder
    ConnectionHolder conHolder = (ConnectionHolder)TransactionSynchronizationManager.getResource(dataSource);
    if (conHolder == null || !conHolder.hasConnection() && !conHolder.isSynchronizedWithTransaction()) {
        logger.debug("Fetching JDBC Connection from DataSource");
        //如果没有使用@Transaction，那调用Mapper接口方法时，也是通过Spring的方法获取Connection
        Connection con = fetchConnection(dataSource);
        if (TransactionSynchronizationManager.isSynchronizationActive()) {
            logger.debug("Registering transaction synchronization for JDBC Connection");
            ConnectionHolder holderToUse = conHolder;
            if (conHolder == null) {
                holderToUse = new ConnectionHolder(con);
            } else {
                conHolder.setConnection(con);
            }

            holderToUse.requested();
            TransactionSynchronizationManager.registerSynchronization(new DataSourceUtils.ConnectionSynchronization(holderToUse, dataSource));
            holderToUse.setSynchronizedWithTransaction(true);
            if (holderToUse != conHolder) {
                //将获取到的ConnectionHolder放入ThreadLocal中，那么当前线程调用下一个接口，下一个接口使用了Spring事务，那Spring事务也可以直接取到Mybatis创建的Connection
                //通过ThreadLocal保证了同一线程中Spring事务使用的Connection和Mapper代理类使用的Connection是同一个
                TransactionSynchronizationManager.bindResource(dataSource, holderToUse);
            }
        }

        return con;
    } else {
        conHolder.requested();
        if (!conHolder.hasConnection()) {
            logger.debug("Fetching resumed JDBC Connection from DataSource");
            conHolder.setConnection(fetchConnection(dataSource));
        }

        //所以如果我们业务代码使用了@Transaction注解，在Spring中就已经通过dataSource创建了一个Connection并放入ThreadLocal中
        //那么当Mapper代理对象调用方法时，通过SqlSession的SpringManagedTransaction获取连接时，就直接获取到了当前线程中Spring事务创建的Connection并返回
        return conHolder.getConnection();
    }
}
```



想看怎么获取connHolder 



### org.springframework.transaction.support.TransactionSynchronizationManager#getResource



```java
//保存数据库连接的ThreadLocal
private static final ThreadLocal<Map<Object, Object>> resources = new NamedThreadLocal<>("Transactional resources");
@Nullable
public static Object getResource(Object key) {
    Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
    //获取ConnectionHolder
    Object value = doGetResource(actualKey);
    ....
    return value;
}

@Nullable
private static Object doGetResource(Object actualKey) {
    /**
     * 从threadlocal <Map<Object, Object>>中取出来当前线程绑定的map
     * map里面存的是<dataSource,ConnectionHolder>
     */
    Map<Object, Object> map = resources.get();
    if (map == null) {
        return null;
    }
    //map中取出来对应dataSource的ConnectionHolder
    Object value = map.get(actualKey);
    // Transparently remove ResourceHolder that was marked as void...
    if (value instanceof ResourceHolder && ((ResourceHolder) value).isVoid()) {
        map.remove(actualKey);
        // Remove entire ThreadLocal if empty...
        if (map.isEmpty()) {
            resources.remove();
        }
        value = null;
    }
    return value;
}
```



我们看到直接从ThreadLocal中取出来的conn,而spring自己的事务也是操作的这个ThreadLocal中的conn来进行事务的开启和回滚，由此我们知道了在同一线程中Spring事务中的Connection和Mybaits中Mapper代理对象中操作数据库的Connection是同一个，当取出来的conn为空时候,调用org.springframework.jdbc.datasource.DataSourceUtils#fetchConnection获取，然后把从数据源取出来的连接返回



```java
private static Connection fetchConnection(DataSource dataSource) throws SQLException {
    //从数据源取出来conn
    Connection con = dataSource.getConnection();
    if (con == null) {
        throw new IllegalStateException("DataSource returned null from getConnection(): " + dataSource);
    }
    return con;
}
```



我们再来回顾一下上篇文章中的SqlSessionInterceptor



```java
 1 private class SqlSessionInterceptor implements InvocationHandler {
 2     private SqlSessionInterceptor() {
 3     }
 4 
 5     public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
 6         SqlSession sqlSession = SqlSessionUtils.getSqlSession(SqlSessionTemplate.this.sqlSessionFactory, SqlSessionTemplate.this.executorType, SqlSessionTemplate.this.exceptionTranslator);
 7 
 8         Object unwrapped;
 9         try {
10             Object result = method.invoke(sqlSession, args);
11             // 如果当前操作没有在一个Spring事务中，则手动commit一下
12             // 如果当前业务没有使用@Transation,那么每次执行了Mapper接口的方法直接commit
13             // 还记得我们前面讲的Mybatis的一级缓存吗，这里一级缓存不能起作用了，因为每执行一个Mapper的方法，sqlSession都提交了
14             // sqlSession提交，会清空一级缓存
15             if (!SqlSessionUtils.isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
16                 sqlSession.commit(true);
17             }
18 
19             unwrapped = result;
20         } catch (Throwable var11) {
21             unwrapped = ExceptionUtil.unwrapThrowable(var11);
22             if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
23                 SqlSessionUtils.closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
24                 sqlSession = null;
25                 Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException)unwrapped);
26                 if (translated != null) {
27                     unwrapped = translated;
28                 }
29             }
30 
31             throw (Throwable)unwrapped;
32         } finally {
33             if (sqlSession != null) {
34                 SqlSessionUtils.closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
35             }
36 
37         }
38         return unwrapped;
39     }
40 }
```



看第15和16行，如果我们没有使用@Transation，Mapper方法执行完后，sqlSession将会提交，也就是说通过org.springframework.jdbc.datasource.DataSourceUtils#fetchConnection获取到的Connection将会commit，相当于Connection是自动提交的，也就是说如果不使用@Transation，Mybatis将没有事务可言。

Mybatis和Spring整合后SpringManagedTransaction和Spring的Transaction的关系：

- 如果开启Spring事务，则先有Spring的Transaction，然后mybatis创建sqlSession时，会创建SpringManagedTransaction并加入sqlSession中，SpringManagedTransaction中的connection会从Spring的Transaction创建的Connection并放入ThreadLocal中获取
- 如果没有开启Spring事务或者第一个方法没有事务后面的方法有事务，则SpringManagedTransaction创建Connection并放入ThreadLocal中

spring结合mybatis后mybaits一级缓存失效分为两种情况：

- 如果没有开启事务，每一次sql都是用的新的SqlSession，这时mybatis的一级缓存是失效的。
- 如果有事务，同一个事务中相同的查询使用的相同的SqlSessioon，此时一级缓存是生效的。

如果使用了@Transation呢？那在调用Mapper代理类的方法之前就已经通过Spring的事务生成了Connection并放入ThreadLocal，并且设置事务不自动提交，当前线程多个Mapper代理对象调用数据库操作方法时，将从ThreadLocal获取Spring创建的connection,在所有的Mapper方法调用完后，Spring事务提交或者回滚，到此mybatis的事务是怎么被spring管理的就显而易见了

还有文章开头的问题，为什么Mybtis中要配置dataSource，Spring的事务中也要配置dataSource？

因为Spring事务在没调用Mapper方法之前就需要开一个Connection，并设置事务不自动提交，那么transactionManager中自然要配置dataSource。那如果我们的Service没有用到Spring事务呢，难道就不需要获取数据库连接了吗？当然不是，此时通过SpringManagedTransaction调用org.springframework.jdbc.datasource.DataSourceUtils#getConnection#fetchConnection方法获取，并将dataSource作为参数传进去，实际上获取的Connection都是通过dataSource来获取的。