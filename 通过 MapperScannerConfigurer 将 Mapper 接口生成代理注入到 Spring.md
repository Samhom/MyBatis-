## 通过 MapperScannerConfigurer 将 Mapper 接口生成代理注入到 Spring

## 扫描Mapper接口

我们上一篇文章介绍了扫描Mapper接口有两种方式，一种是通过bean.xml注册MapperScannerConfigurer对象，一种是通过@MapperScan("com.chenhao.mapper")注解的方式，如下

方式一：

```xml
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <property name="basePackage" value="com.chenhao.mapper" />
</bean>
```

方式二：

```java 
@Configuration
@MapperScan("com.chenhao.mapper")
public class AppConfig {
```



## @MapperScan

我们来看看**@MapperScan**这个注解

```java
@Import(MapperScannerRegistrar.class)
public @interface MapperScan {
```

MapperScan使用`@Import`将`MapperScannerRegistrar`导入。



### MapperScannerRegistrar



```java
 1 public class MapperScannerRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware {
 2   private ResourceLoader resourceLoader;
 3   @Override
 4   public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
 5     // 获取MapperScan 注解，如@MapperScan("com.chenhao.mapper")
 6     AnnotationAttributes annoAttrs = AnnotationAttributes.fromMap(importingClassMetadata.getAnnotationAttributes(MapperScan.class.getName()));
 7     // 创建路径扫描器，下面的一大段都是将MapperScan 中的属性设置到ClassPathMapperScanner ，做的就是一个set操作
 8     ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
 9     // this check is needed in Spring 3.1
10     if (resourceLoader != null) {
11       // 设置资源加载器，作用：扫描指定包下的class文件。
12       scanner.setResourceLoader(resourceLoader);
13     }
14     Class<? extends Annotation> annotationClass = annoAttrs.getClass("annotationClass");
15     if (!Annotation.class.equals(annotationClass)) {
16       scanner.setAnnotationClass(annotationClass);
17     }
18     Class<?> markerInterface = annoAttrs.getClass("markerInterface");
19     if (!Class.class.equals(markerInterface)) {
20       scanner.setMarkerInterface(markerInterface);
21     }
22     Class<? extends BeanNameGenerator> generatorClass = annoAttrs.getClass("nameGenerator");
23     if (!BeanNameGenerator.class.equals(generatorClass)) {
24       scanner.setBeanNameGenerator(BeanUtils.instantiateClass(generatorClass));
25     }
26     Class<? extends MapperFactoryBean> mapperFactoryBeanClass = annoAttrs.getClass("factoryBean");
27     if (!MapperFactoryBean.class.equals(mapperFactoryBeanClass)) {
28       scanner.setMapperFactoryBean(BeanUtils.instantiateClass(mapperFactoryBeanClass));
29     }
30     scanner.setSqlSessionTemplateBeanName(annoAttrs.getString("sqlSessionTemplateRef"));
31     //设置SqlSessionFactory的名称
32     scanner.setSqlSessionFactoryBeanName(annoAttrs.getString("sqlSessionFactoryRef"));
33     List<String> basePackages = new ArrayList<String>();
34     //获取配置的包路径，如com.chenhao.mapper
35     for (String pkg : annoAttrs.getStringArray("value")) {
36       if (StringUtils.hasText(pkg)) {
37         basePackages.add(pkg);
38       }
39     }
40     for (String pkg : annoAttrs.getStringArray("basePackages")) {
41       if (StringUtils.hasText(pkg)) {
42         basePackages.add(pkg);
43       }
44     }
45     for (Class<?> clazz : annoAttrs.getClassArray("basePackageClasses")) {
46       basePackages.add(ClassUtils.getPackageName(clazz));
47     }
48     // 注册过滤器，作用：什么类型的Mapper将会留下来。
49     scanner.registerFilters();
50     // 扫描指定包
51     scanner.doScan(StringUtils.toStringArray(basePackages));
52   }
53 }
```





### ClassPathMapperScanner

接着我们来看看扫描过程 **scanner.doScan(StringUtils.toStringArray(basePackages));**



```java
 1 @Override
 2 public Set<BeanDefinitionHolder> doScan(String... basePackages) {
 3     //调用父类进行扫描，并将basePackages下的class都封装成BeanDefinitionHolder，并注册进Spring容器的BeanDefinition
 4     Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);
 5  
 6     if (beanDefinitions.isEmpty()) {
 7       logger.warn("No MyBatis mapper was found in '" + Arrays.toString(basePackages) + "' package. Please check your configuration.");
 8     } else {
 9       //继续对beanDefinitions做处理，额外设置一些属性
10       processBeanDefinitions(beanDefinitions);
11     }
12  
13     return beanDefinitions;
14 }
15   
16 protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
17     Assert.notEmpty(basePackages, "At least one base package must be specified");
18     Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<BeanDefinitionHolder>();
19     //遍历basePackages进行扫描
20     for (String basePackage : basePackages) {
21         //找出匹配的类
22         Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
23         for (BeanDefinition candidate : candidates) {
24             ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
25             candidate.setScope(scopeMetadata.getScopeName());
26             String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
27             if (candidate instanceof AbstractBeanDefinition) {
28                 postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
29             }
30             if (candidate instanceof AnnotatedBeanDefinition) {
31                 AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
32             }
33             if (checkCandidate(beanName, candidate)) {
34                 //封装成BeanDefinitionHolder 对象
35                 BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
36                 definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
37                 beanDefinitions.add(definitionHolder);
38                 //将BeanDefinition对象注入spring的BeanDefinitionMap中，后续getBean时，就是从BeanDefinitionMap获取对应的BeanDefinition对象，取出其属性进行实例化Bean
39                 registerBeanDefinition(definitionHolder, this.registry);
40             }
41         }
42     }
43     return beanDefinitions;
44 }
```



我们重点看下第4行和第10行代码，第4行是调用父类的doScan方法，获取basePackages下的所有Class，并将其生成**BeanDefinition，**注入spring的**BeanDefini**tionMap中，也就是Class的描述类，在调用getBean方法时，获取BeanDefinition进行实例化。此时，所有的Mapper接口已经被生成了BeanDefinition。接着我们看下第10行，对生成的BeanDefinition做一些额外的处理。



### **processBeanDefinitions**

上面BeanDefinition已经注入进Spring容器了，接着我们看对BeanDefinition进行哪些额外的处理



```java
 1 private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
 2     GenericBeanDefinition definition;
 3     for (BeanDefinitionHolder holder : beanDefinitions) {
 4       definition = (GenericBeanDefinition) holder.getBeanDefinition();
 5  
 6       if (logger.isDebugEnabled()) {
 7         logger.debug("Creating MapperFactoryBean with name '" + holder.getBeanName() 
 8           + "' and '" + definition.getBeanClassName() + "' mapperInterface");
 9       }
10  
11       // 设置definition构造器的输入参数为definition.getBeanClassName()，这里就是com.chenhao.mapper.UserMapper
12       // 那么在getBean实例化时，通过反射调用构造器实例化时要将这个参数传进去
13       definition.getConstructorArgumentValues().addGenericArgumentValue(definition.getBeanClassName())
14       // 修改definition对应的Class
15       // 看过Spring源码的都知道，getBean()返回的就是BeanDefinitionHolder中beanClass属性对应的实例
16       // 所以我们后面ac.getBean(UserMapper.class)的返回值也就是MapperFactoryBean的实例
17       // 但是最终被注入到Spring容器的对象的并不是MapperFactoryBean的实例，根据名字看，我们就知道MapperFactoryBean实现了FactoryBean接口
18       // 那么最终注入Spring容器的必定是从MapperFactoryBean的实例的getObject()方法中返回
19       definition.setBeanClass(this.mapperFactoryBean.getClass());
20  
21       definition.getPropertyValues().add("addToConfig", this.addToConfig);
22  
23       boolean explicitFactoryUsed = false;
24       if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
25         definition.getPropertyValues().add("sqlSessionFactory", new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
26         explicitFactoryUsed = true;
27       } else if (this.sqlSessionFactory != null) {
28         //往definition设置属性值sqlSessionFactory，那么在MapperFactoryBean实例化后，进行属性赋值时populateBean(beanName, mbd, instanceWrapper);，会通过反射调用sqlSessionFactory的set方法进行赋值
29         //也就是在MapperFactoryBean创建实例后，要调用setSqlSessionFactory方法将sqlSessionFactory注入进MapperFactoryBean实例
30         definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
31         explicitFactoryUsed = true;
32       }
33  
34       if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
35         if (explicitFactoryUsed) {
36           logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
37         }
38         definition.getPropertyValues().add("sqlSessionTemplate", new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
39         explicitFactoryUsed = true;
40       } else if (this.sqlSessionTemplate != null) {
41         if (explicitFactoryUsed) {
42           logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
43         }
44         definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
45         explicitFactoryUsed = true;
46       }
47  
48       if (!explicitFactoryUsed) {
49         if (logger.isDebugEnabled()) {
50           logger.debug("Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
51         }
52         definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
53       }
54     }
55 }
```



看第19行，将**definitio**n的beanClass属性设置为MapperFactoryBean.class，我们知道，在getBean的时候，会通过反射创建Bean的实例，也就是创建beanClass的实例，如下Spring的getBean的部分代码：



```java
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
    // 没有覆盖
    // 直接使用反射实例化即可
    if (!bd.hasMethodOverrides()) {
        // 重新检测获取下构造函数
        // 该构造函数是经过前面 N 多复杂过程确认的构造函数
        Constructor<?> constructorToUse;
        synchronized (bd.constructorArgumentLock) {
            // 获取已经解析的构造函数
            constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
            // 如果为 null，从 class 中解析获取，并设置
            if (constructorToUse == null) {
                final Class<?> clazz = bd.getBeanClass();
                if (clazz.isInterface()) {
                    throw new BeanInstantiationException(clazz, "Specified class is an interface");
                }
                try {
                    if (System.getSecurityManager() != null) {
                        constructorToUse = AccessController.doPrivileged(
                                (PrivilegedExceptionAction<Constructor<?>>) clazz::getDeclaredConstructor);
                    }
                    else {
                        //利用反射获取构造器
                        constructorToUse =  clazz.getDeclaredConstructor();
                    }
                    bd.resolvedConstructorOrFactoryMethod = constructorToUse;
                }
                catch (Throwable ex) {
                    throw new BeanInstantiationException(clazz, "No default constructor found", ex);
                }
            }
        }

        // 通过BeanUtils直接使用构造器对象实例化bean
        return BeanUtils.instantiateClass(constructorToUse);
    }
    else {
        // 生成CGLIB创建的子类对象
        return instantiateWithMethodInjection(bd, beanName, owner);
    }
}
```



看到没，是通过**bd.getBeanClass();从**BeanDefinition中获取beanClass属性，然后通过反射实例化Bean，如上，所有的Mapper接口扫描封装成的BeanDefinition的beanClass都设置成了MapperFactoryBean，我们知道在Spring初始化的最后，会获取所有的BeanDefinition，并通过getBean创建所有的实例注入进Spring容器，那么意思就是说，在getBean时，创建的的所有Mapper对象是MapperFactoryBean这个对象了。

**我们看下第13行处，设置了BeanDefinition构造器参数，那么当getBean中通过构造器创建实例时，需要将设置的构造器参数definition.getBeanClassName()，这里就是com.chenhao.mapper.UserMapper传进去。**

还有一个点要注意，在第30行处，往BeanDefinition的PropertyValues设置了sqlSessionFactory，那么在创建完MapperFactoryBean的实例后，会对MapperFactoryBean进行属性赋值，也就是Spring创建Bean的这句代码，**populateBean(beanName, mbd, instanceWrapper);**，这里会通过反射调用MapperFactoryBean的**setSqlSessionFactory方法将sqlSessionFactory注入进MapperFactoryBean实例，所以我们接下来重点看的就是****MapperFactoryBean这个对象了。**



## MapperFactoryBean

接下来我们看最重要的一个类MapperFactoryBean



```java
//继承SqlSessionDaoSupport、实现FactoryBean，那么最终注入Spring容器的对象要从getObject()中取
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {
    private Class<T> mapperInterface;
    private boolean addToConfig = true;

    public MapperFactoryBean() {
    }

    //构造器，我们上一节中在BeanDefinition中已经设置了构造器输入参数
    //所以在通过反射调用构造器实例化时，会获取在BeanDefinition设置的构造器输入参数
    //也就是对应得每个Mapper接口Class
    public MapperFactoryBean(Class<T> mapperInterface) {
        this.mapperInterface = mapperInterface;
    }

    protected void checkDaoConfig() {
        super.checkDaoConfig();
        Assert.notNull(this.mapperInterface, "Property 'mapperInterface' is required");
        Configuration configuration = this.getSqlSession().getConfiguration();
        if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
            try {
                configuration.addMapper(this.mapperInterface);
            } catch (Exception var6) {
                this.logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", var6);
                throw new IllegalArgumentException(var6);
            } finally {
                ErrorContext.instance().reset();
            }
        }

    }
    //最终注入Spring容器的就是这里的返回对象
    public T getObject() throws Exception {
        //获取父类setSqlSessionFactory方法中创建的SqlSessionTemplate
        //通过SqlSessionTemplate获取mapperInterface的代理类
        //我们例子中就是通过SqlSessionTemplate获取com.chenhao.mapper.UserMapper的代理类
        //获取到Mapper接口的代理类后，就把这个Mapper的代理类对象注入Spring容器
        return this.getSqlSession().getMapper(this.mapperInterface);
    }

    public Class<T> getObjectType() {
        return this.mapperInterface;
    }

    public boolean isSingleton() {
        return true;
    }

    public void setMapperInterface(Class<T> mapperInterface) {
        this.mapperInterface = mapperInterface;
    }

    public Class<T> getMapperInterface() {
        return this.mapperInterface;
    }

    public void setAddToConfig(boolean addToConfig) {
        this.addToConfig = addToConfig;
    }

    public boolean isAddToConfig() {
        return this.addToConfig;
    }
}


public abstract class SqlSessionDaoSupport extends DaoSupport {
    private SqlSession sqlSession;
    private boolean externalSqlSession;

    public SqlSessionDaoSupport() {
    }
    //还记得上一节中我们往BeanDefinition中设置的sqlSessionFactory这个属性吗？
    //在实例化MapperFactoryBean后，进行属性赋值时，就会通过反射调用setSqlSessionFactory
    public void setSqlSessionFactory(SqlSessionFactory sqlSessionFactory) {
        if (!this.externalSqlSession) {
            //创建一个SqlSessionTemplate并赋值给sqlSession
            this.sqlSession = new SqlSessionTemplate(sqlSessionFactory);
        }
    }

    public void setSqlSessionTemplate(SqlSessionTemplate sqlSessionTemplate) {
        this.sqlSession = sqlSessionTemplate;
        this.externalSqlSession = true;
    }

    public SqlSession getSqlSession() {
        return this.sqlSession;
    }

    protected void checkDaoConfig() {
        Assert.notNull(this.sqlSession, "Property 'sqlSessionFactory' or 'sqlSessionTemplate' are required");
    }
}
```



我们看到MapperFactoryBean **extends SqlSessionDaoSupport implements FactoryBean，那么getBean获取的对象是从其getObject()中获取，并且MapperFactoryBean是一个单例，那么其中的属性****SqlSessionTemplate对象也是一个单例，全局唯一，供所有的Mapper代理类使用。**

这里我大概讲一下getBean时，这个类的过程：

1、MapperFactoryBean通过反射调用构造器实例化出一个对象，并且通过上一节中definition.getConstructorArgumentValues().addGenericArgumentValue(definition.getBeanClassName())设置的构造器参数，在构造器实例化时，传入Mapper接口的Class,并设置为MapperFactoryBean的mapperInterface属性。

2、进行属性赋值，通过上一节中definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);设置的属性值，在populateBean属性赋值过程中通过反射调用setSqlSessionFactory方法，并创建SqlSessionTemplate对象设置到sqlSession属性中。

3、由于MapperFactoryBean实现了FactoryBean，最终注册进Spring容器的对象是从getObject()方法中取，接着获取SqlSessionTemplate这个SqlSession调用getMapper(this.mapperInterface);生成Mapper接口的代理对象，将Mapper接口的代理对象注册进Spring容器

至此，所有com.chenhao.mapper中的Mapper接口都生成了代理类，并注入到Spring容器了。接着我们就可以在Service中直接从Spring的BeanFactory中获取了，如下

![img](https://img2018.cnblogs.com/i-beta/1168971/201911/1168971-20191102151929081-185356019.png)



## SqlSessionTemplate

还记得我们前面分析Mybatis源码时，获取的SqlSession实例是什么吗？我们简单回顾一下



```java
/**
 * ExecutorType 指定Executor的类型，分为三种：SIMPLE, REUSE, BATCH，默认使用的是SIMPLE
 * TransactionIsolationLevel 指定事务隔离级别，使用null,则表示使用数据库默认的事务隔离界别
 * autoCommit 是否自动提交，传过来的参数为false，表示不自动提交
 */
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
        // 获取配置中的环境信息，包括了数据源信息、事务等
        final Environment environment = configuration.getEnvironment();
        // 创建事务工厂
        final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
        // 创建事务，配置事务属性
        tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
        // 创建Executor，即执行器
        // 它是真正用来Java和数据库交互操作的类，后面会展开说。
        final Executor executor = configuration.newExecutor(tx, execType);
        // 创建DefaultSqlSession对象返回，其实现了SqlSession接口
        return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
        closeTransaction(tx);
        throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```



大家应该还有印象，就是上面的DefaultSqlSession，那上一节的SqlSessionTemplate是什么鬼？？？我们来看看



```java
// 实现SqlSession接口，单例、线程安全，使用spring的事务管理器的sqlSession，
// 具体的SqlSession的功能，则是通过内部包含的sqlSessionProxy来来实现，这也是静态代理的一种实现。
// 同时内部的sqlSessionProxy实现InvocationHandler接口，则是动态代理的一种实现，而线程安全也是在这里实现的。
// 注意mybatis默认的sqlSession不是线程安全的，需要每个线程有一个单例的对象实例。
// SqlSession的主要作用是提供SQL操作的API，执行指定的SQL语句，mapper需要依赖SqlSession来执行其方法对应的SQL。
public class SqlSessionTemplate implements SqlSession, DisposableBean {
    private final SqlSessionFactory sqlSessionFactory;
    private final ExecutorType executorType;
    // 一个代理类，由于SqlSessionTemplate为单例的，被所有mapper，所有线程共享，
    // 所以sqlSessionProxy要保证这些mapper中方法调用的线程安全特性：
    // sqlSessionProxy的实现方式主要为实现了InvocationHandler接口实现了动态代理，
    // 由动态代理的知识可知，InvocationHandler的invoke方法会拦截所有mapper的所有方法调用，
    // 故这里的实现方式是在invoke方法内部创建一个sqlSession局部变量，从而实现了每个mapper的每个方法调用都使用
    private final SqlSession sqlSessionProxy;
    private final PersistenceExceptionTranslator exceptionTranslator;

    public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
        this(sqlSessionFactory, sqlSessionFactory.getConfiguration().getDefaultExecutorType());
    }

    public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType) {
        this(sqlSessionFactory, executorType, new MyBatisExceptionTranslator(sqlSessionFactory.getConfiguration().getEnvironment().getDataSource(), true));
    }

    public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {
        Assert.notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
        Assert.notNull(executorType, "Property 'executorType' is required");
        this.sqlSessionFactory = sqlSessionFactory;
        this.executorType = executorType;
        this.exceptionTranslator = exceptionTranslator;
        this.sqlSessionProxy = (SqlSession)Proxy.newProxyInstance(SqlSessionFactory.class.getClassLoader(), new Class[]{SqlSession.class}, new SqlSessionTemplate.SqlSessionInterceptor());
    }

    public SqlSessionFactory getSqlSessionFactory() {
        return this.sqlSessionFactory;
    }

    public ExecutorType getExecutorType() {
        return this.executorType;
    }

    public PersistenceExceptionTranslator getPersistenceExceptionTranslator() {
        return this.exceptionTranslator;
    }

    public <T> T selectOne(String statement) {
        //由真实的对象sqlSessionProxy执行查询
        return this.sqlSessionProxy.selectOne(statement);
    }

    public <T> T selectOne(String statement, Object parameter) {
        //由真实的对象sqlSessionProxy执行查询
        return this.sqlSessionProxy.selectOne(statement, parameter);
    }

    public <K, V> Map<K, V> selectMap(String statement, String mapKey) {
        //由真实的对象sqlSessionProxy执行查询
        return this.sqlSessionProxy.selectMap(statement, mapKey);
    }

    public <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey) {
        //由真实的对象sqlSessionProxy执行查询
        return this.sqlSessionProxy.selectMap(statement, parameter, mapKey);
    }

    public <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey, RowBounds rowBounds) {
        //由真实的对象sqlSessionProxy执行查询
        return this.sqlSessionProxy.selectMap(statement, parameter, mapKey, rowBounds);
    }

    public <T> Cursor<T> selectCursor(String statement) {
        return this.sqlSessionProxy.selectCursor(statement);
    }

    public <T> Cursor<T> selectCursor(String statement, Object parameter) {
        return this.sqlSessionProxy.selectCursor(statement, parameter);
    }

    public <T> Cursor<T> selectCursor(String statement, Object parameter, RowBounds rowBounds) {
        return this.sqlSessionProxy.selectCursor(statement, parameter, rowBounds);
    }

    public <E> List<E> selectList(String statement) {
        return this.sqlSessionProxy.selectList(statement);
    }

    public <E> List<E> selectList(String statement, Object parameter) {
        return this.sqlSessionProxy.selectList(statement, parameter);
    }

    public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
        return this.sqlSessionProxy.selectList(statement, parameter, rowBounds);
    }

    public void select(String statement, ResultHandler handler) {
        this.sqlSessionProxy.select(statement, handler);
    }

    public void select(String statement, Object parameter, ResultHandler handler) {
        this.sqlSessionProxy.select(statement, parameter, handler);
    }

    public void select(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler) {
        this.sqlSessionProxy.select(statement, parameter, rowBounds, handler);
    }

    public int insert(String statement) {
        return this.sqlSessionProxy.insert(statement);
    }

    public int insert(String statement, Object parameter) {
        return this.sqlSessionProxy.insert(statement, parameter);
    }

    public int update(String statement) {
        return this.sqlSessionProxy.update(statement);
    }

    public int update(String statement, Object parameter) {
        return this.sqlSessionProxy.update(statement, parameter);
    }

    public int delete(String statement) {
        return this.sqlSessionProxy.delete(statement);
    }

    public int delete(String statement, Object parameter) {
        return this.sqlSessionProxy.delete(statement, parameter);
    }

    public <T> T getMapper(Class<T> type) {
        return this.getConfiguration().getMapper(type, this);
    }

    public void commit() {
        throw new UnsupportedOperationException("Manual commit is not allowed over a Spring managed SqlSession");
    }

    public void commit(boolean force) {
        throw new UnsupportedOperationException("Manual commit is not allowed over a Spring managed SqlSession");
    }

    public void rollback() {
        throw new UnsupportedOperationException("Manual rollback is not allowed over a Spring managed SqlSession");
    }

    public void rollback(boolean force) {
        throw new UnsupportedOperationException("Manual rollback is not allowed over a Spring managed SqlSession");
    }

    public void close() {
        throw new UnsupportedOperationException("Manual close is not allowed over a Spring managed SqlSession");
    }

    public void clearCache() {
        this.sqlSessionProxy.clearCache();
    }

    public Configuration getConfiguration() {
        return this.sqlSessionFactory.getConfiguration();
    }

    public Connection getConnection() {
        return this.sqlSessionProxy.getConnection();
    }

    public List<BatchResult> flushStatements() {
        return this.sqlSessionProxy.flushStatements();
    }

    public void destroy() throws Exception {
    }

    private class SqlSessionInterceptor implements InvocationHandler {
        private SqlSessionInterceptor() {
        }

        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            SqlSession sqlSession = SqlSessionUtils.getSqlSession(SqlSessionTemplate.this.sqlSessionFactory, SqlSessionTemplate.this.executorType, SqlSessionTemplate.this.exceptionTranslator);

            Object unwrapped;
            try {
                Object result = method.invoke(sqlSession, args);
                if (!SqlSessionUtils.isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
                    sqlSession.commit(true);
                }

                unwrapped = result;
            } catch (Throwable var11) {
                unwrapped = ExceptionUtil.unwrapThrowable(var11);
                if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
                    SqlSessionUtils.closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
                    sqlSession = null;
                    Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException)unwrapped);
                    if (translated != null) {
                        unwrapped = translated;
                    }
                }

                throw (Throwable)unwrapped;
            } finally {
                if (sqlSession != null) {
                    SqlSessionUtils.closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
                }

            }

            return unwrapped;
        }
    }
}
```



我们看到SqlSessionTemplate实现了**SqlSession接口，那么Mapper代理类中执行所有的数据库操作，都是通过SqlSessionTemplate来执行，如上我们看到所有的数据库操作都由****对象sqlSessionProxy执行查询**



### 静态代理的使用

SqlSessionTemplate在内部访问数据库时，其实是委派给**sqlSessionProxy**来执行数据库操作的，SqlSessionTemplate不是自身重新实现了一套mybatis数据库访问的逻辑。

SqlSessionTemplate通过静态代理机制来提供SqlSession接口的行为，即实现SqlSession接口来获取SqlSession的所有方法；SqlSessionTemplate的定义如下：标准的静态代理实现模式，即实现SqlSession接口并在内部包含一个SqlSession接口实现类引用sqlSessionProxy。那我们就要看看sqlSessionProxy这个SqlSession，我们先来看看SqlSessionTemplate的构造方法



```java
 public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {
        Assert.notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
        Assert.notNull(executorType, "Property 'executorType' is required");
        this.sqlSessionFactory = sqlSessionFactory;
        this.executorType = executorType;
        this.exceptionTranslator = exceptionTranslator;
        this.sqlSessionProxy = (SqlSession)Proxy.newProxyInstance(SqlSessionFactory.class.getClassLoader(), new Class[]{SqlSession.class}, new SqlSessionTemplate.SqlSessionInterceptor());
    }
```





### 动态代理的使用

不是吧，又使用了动态代理？？真够曲折的，那我们接着看 new SqlSessionTemplate.SqlSessionInterceptor() 这个InvocationHandler



```java
private class SqlSessionInterceptor implements InvocationHandler {
    //很奇怪，这里并没有真实目标对象？
    private SqlSessionInterceptor() {
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 获取一个sqlSession来执行proxy的method对应的SQL,
        // 每次调用都获取创建一个sqlSession线程局部变量，故不同线程相互不影响，在这里实现了SqlSessionTemplate的线程安全性
        SqlSession sqlSession = SqlSessionUtils.getSqlSession(SqlSessionTemplate.this.sqlSessionFactory, SqlSessionTemplate.this.executorType, SqlSessionTemplate.this.exceptionTranslator);

        Object unwrapped;
        try {
            //直接通过新创建的SqlSession反射调用method
            //这也就解释了为什么不需要目标类属性了，这里每次都会创建一个
            Object result = method.invoke(sqlSession, args);
            // 如果当前操作没有在一个Spring事务中，则手动commit一下
            // 如果当前业务没有使用@Transation,那么每次执行了Mapper接口的方法直接commit
            // 还记得我们前面讲的Mybatis的一级缓存吗，这里一级缓存不能起作用了，因为每执行一个Mapper的方法，sqlSession都提交了
            // sqlSession提交，会清空一级缓存
            if (!SqlSessionUtils.isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
                sqlSession.commit(true);
            }

            unwrapped = result;
        } catch (Throwable var11) {
            unwrapped = ExceptionUtil.unwrapThrowable(var11);
            if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
                SqlSessionUtils.closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
                sqlSession = null;
                Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException)unwrapped);
                if (translated != null) {
                    unwrapped = translated;
                }
            }

            throw (Throwable)unwrapped;
        } finally {
            if (sqlSession != null) {
                SqlSessionUtils.closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
            }

        }
        return unwrapped;
    }
}
```



这里大概讲一下Mapper代理类调用方法执行逻辑：

1、SqlSessionTemplate生成Mapper代理类时，将本身传进去做为Mapper代理类的属性，调用Mapper代理类的方法时，最后会通过SqlSession类执行，也就是调用SqlSessionTemplate中的方法。

2、SqlSessionTemplate中操作数据库的方法中又交给了**sqlSessionProxy**这个代理类去执行，那么每次执行的方法都会回调其SqlSessionInterceptor这个InvocationHandler的invoke方法

3、在invoke方法中，为每个线程创建一个新的SqlSession，并通过反射调用SqlSession的method。这里sqlSession是一个线程局部变量，不同线程相互不影响，实现了SqlSessionTemplate的线程安全性

4、如果当前操作并没有在Spring事务中，那么每次执行一个方法，都会提交，相当于数据库的事务自动提交，Mysql的一级缓存也将不可用

接下来我们还要看一个地方，invoke中是如何创建**SqlSession的？**



```java
public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {
    Assert.notNull(sessionFactory, "No SqlSessionFactory specified");
    Assert.notNull(executorType, "No ExecutorType specified");
    //通过TransactionSynchronizationManager内部的ThreadLocal中获取
    SqlSessionHolder holder = (SqlSessionHolder)TransactionSynchronizationManager.getResource(sessionFactory);
    SqlSession session = sessionHolder(executorType, holder);
    if(session != null) {
        return session;
    } else {
        if(LOGGER.isDebugEnabled()) {
            LOGGER.debug("Creating a new SqlSession");
        }
        //这里我们知道实际上是创建了一个DefaultSqlSession
        session = sessionFactory.openSession(executorType);
        //将创建的SqlSession对象放入TransactionSynchronizationManager内部的ThreadLocal中
        registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);
        return session;
    }
}
```



通过**sessionFactory.openSession(executorType)实际创建的****SqlSession还是****DefaultSqlSession。**如果读过我前面Spring源码的朋友，肯定知道TransactionSynchronizationManager这个类，其内部维护了一个**ThreadLocal的**Map，这里同一线程创建了**SqlSession后放入\**ThreadLocal中，同一线程中其他Mapper接口调用方法时，将会直接从\*\*ThreadLocal中获取。\*\**\***