# 自定义封装JDBC+Mybatis解析

视频讲解地址<a href="https://www.bilibili.com/video/BV1Kv41157vu" target="_blank">请点这里</a>

## JDBC

#### 操作步骤：

1. 建立连接：

   加载数据库驱动

   通过驱动管理类获取数据库连接

2. 准备SQL：

   定义SQL语句

   获取预处理statement

   设置参数

   向数据库发起操作请求

3. 处理结果集：

   遍历结果集

   处理结果

#### 问题分析：

1. 数据库配置信息存在硬编码问题——配置文件

2. 频繁创建、释放数据库连接——连接池

3. SQL语句、参数、结果集均存在硬编码问题——配置文件

4. 处理结果集较为繁琐——反射、内省

### 自定义框架问题分析：

1. 重复代码
2. statementId存在硬编码

### 解决思路：

1. 使用代理模式创建接口的代理对象

2. 代理模式：JDK代理（cglib代理）

   

### 插件

Executor/StatementHandler/ParameterHandler/ResultSetHandler

##### 原理

在四大对象创建的时候：

1. 每个创建出来的对象不是直接返回的，而是interceptorChain.pluginAll(parameterHandler);
2. 获取到所有的Interceptor (拦截器)(插件需要实现的接口)；调用 interceptor.plugin(target);返
   回 target 包装后的对象
3. 插件机制，我们可以使用插件为目标对象创建一个代理对象；AOP (面向切面)我们的插件可 以
   为四大对象创建出代理对象，代理对象就可以拦截到四大对象的每一个执行；

MyBatis所允许拦截的方法:

- 执行器Executor (update、query、commit、rollback等方法)；
- SQL语法构建器StatementHandler (prepare、parameterize、batch、updates query等方 法)；
- 参数处理器ParameterHandler (getParameterObject、setParameters方法)；
- 结果集处理器ResultSetHandler (handleResultSets、handleOutputParameters等方法)；

```java
//Configuration
public ParameterHandler newParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
    ParameterHandler parameterHandler = mappedStatement.getLang().createParameterHandler(mappedStatement, parameterObject, boundSql);
    parameterHandler = (ParameterHandler) interceptorChain.pluginAll(parameterHandler);
    return parameterHandler;
  }

//InterceptorChain
  public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }
//代理对象执行invoke()方法之前或之后拦截执行增强
```



##### 自定义插件

1. 实现Interceptor
2. plugin()，生成target代理对象
3. setProperties()，传递参数

```java
@Intercepts({//注意看这个大花括号，也就这说这里可以定义多个@Signature对多个地方拦截，都用这个拦截器
        @Signature(type = StatementHandler.class, //这是指拦截哪个接口
                method = "prepare",//这个接口内的哪个方法名，不要拼错了
                args = {Connection.class, Integer.class}),//// 这是拦截的方法的入参，按顺序写到这，不要多也不要少，如果方法重载，可是要通过方法名和入参来确定唯一的
})
public class MyPlugin implements Interceptor {
    private final Logger logger = LoggerFactory.getLogger(this.getClass());

    // //这里是每次执行操作的时候，都会进行这个拦截器的方法内
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        //增强逻辑
        System.out.println("对方法进行了增强....");
        return invocation.proceed(); //执行原方法
    }

    /**
     * //主要是为了把这个拦截器生成一个代理放到拦截器链中
     * ^Description包装目标对象 为目标对象创建代理对象
     *
     * @Param target为要拦截的对象
     * @Return代理对象
     */
    @Override
    public Object plugin(Object target) {
        System.out.println("将要包装的目标对象：" + target);
        return Plugin.wrap(target, this);
    }

    /**
     * 获取配置文件的属性
     **/
    //插件初始化的时候调用，也只调用一次，插件配置的属性从这里设置进来
    @Override
    public void setProperties(Properties properties) {
        System.out.println("插件配置的初始化参数：" + properties);
    }
}
```

##### PageHelper

pageInfo

##### Mapper

- @Table @Id @generatedValue @column
- Interface Mapper similar to jpa
- example

## 源码解析

### 初始化

MyBatis在初始化的时候，会将MyBatis的配置信息全部加载到内存中，使用
org.apache.ibatis.session.Configuratio n 实例来维护

```java
Inputstream inputstream = Resources.getResourceAsStream("mybatis-
config.xml");
    //这一行代码正是初始化工作的开始。
    SqlSessionFactory factory = new
SqlSessionFactoryBuilder().build(inputStream);

// 1.我们最初调用的build
    public SqlSessionFactory build (InputStream inputStream){
      //调用了重载方法
      return build(inputStream, null, null);
   }
   
    // 2.调用的重载方法
    public SqlSessionFactory build (InputStream inputStream, String
environment,
        Properties properties){
      try {
        // XMLConfigBuilder是专门解析mybatis的配置文件的类
        XMLConfigBuilder parser = new XMLConfigBuilder(inputstream,
environment, properties);
     //这里又调用了一个重载方法。parser.parse()的返回值是Configuration对象
        return build(parser.parse());
     } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error building
SqlSession.", e)
     }
```

### 执行SQL流程

#### 传统方式

DefaultSqlSession query-->Executor query-->StatementHandler query--> ParameterHandler setParameters-->TypeHandler setParameter-->execute-->ResultSetHandler handleResultSet

Executor的功能和作用是：

1. 根据传递的参数，完成SQL语句的动态解析，生成BoundSql对象，供StatementHandler使用；
2. 为查询创建缓存，提高性能
3. 创建JDBC的Statement连接对象，传递给*StatementHandler*对象，返回List查询结果。

StatementHandler对象主要完成两个工作：

1. 对于JDBC的PreparedStatement类型的对象，创建的过程中，我们使用的是SQL语句字符串会包
   含若干个？占位符，我们其后再对占位符进行设值。StatementHandler通过
   parameterize(statement) 方法对 Statement 进行设值；
2. StatementHandler 通过 List query(Statement statement, ResultHandler resultHandler)方法来
   完成执行Statement，和将Statement对象返回的resultSet封装成List；

#### Mapper方式

```java
SqlSession sqlSession = factory.openSession();
    //这里不再调用SqlSession的api,而是获得了接口对象，调用接口中的方法。
UserMapper mapper = sqlSession.getMapper(UserMapper.class);
```

一个问题：通常的Mapper接口我们都没有实现的方法却可以使用，是为什么呢？答案很简单，动态
代理。

MapperRegistry是Configuration中的一个属性，它内部维护一个HashMap用于存放mapper接口的工厂类，每个接口对应一个工厂类。mappers中可以配置接口的包路径，或者某个具体的接口类。

```xml
<mappers>
	<mapper class="com.spring.mapper.UserMapper"/>
	<package name="com.spring.mapper"/>
</mappers>
```

当解析mappers标签时，它会判断解析到的是mapper配置文件时，会再将对应配置文件中的增删 改查
标签 封装成MappedStatement对象，存入mappedStatements中。(上文介绍了)当判断解析到接口时，会创建此接口对应的MapperProxyFactory对象，存入HashMap中，key =接口的字节码对象，value =此接
口对应的MapperProxyFactory对象。

##### getMapper()

```java
// Configuration
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }

// MapperRegistry
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
        // 动态代理，生成实例
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }

// MapperProxyFactory<T>
  public T newInstance(SqlSession sqlSession) {
      // 创建jdk动态代理InvocationHandler接口的实现类MapperProxy
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
      // 重载
    return newInstance(mapperProxy);
  }

  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

//MapperProxy
public class MapperProxy<T> implements InvocationHandler, Serializable {

  private static final long serialVersionUID = -4724728412955527868L;
  private static final int ALLOWED_MODES = MethodHandles.Lookup.PRIVATE | MethodHandles.Lookup.PROTECTED
      | MethodHandles.Lookup.PACKAGE | MethodHandles.Lookup.PUBLIC;
  private static final Constructor<Lookup> lookupConstructor;
  private static final Method privateLookupInMethod;
  private final SqlSession sqlSession;
  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethodInvoker> methodCache;
    
// 构造方法传入了sqlSession，sqlSession中的每个代理对象是不同的
  public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, Map<Method, MapperMethodInvoker> methodCache) {
    this.sqlSession = sqlSession;
    this.mapperInterface = mapperInterface;
    this.methodCache = methodCache;
  }
```

##### invoke()

```java
//MapperProxy
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else {
          // 获得MapperMethod 代理对象，调用invoke方法
        return cachedInvoker(method).invoke(proxy, method, args, sqlSession);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
  }
// MapperMethodInvoker
  interface MapperMethodInvoker {
    Object invoke(Object proxy, Method method, Object[] args, SqlSession sqlSession) throws Throwable;
  }
// PlainMethodInvoker
  private static class PlainMethodInvoker implements MapperMethodInvoker {
    private final MapperMethod mapperMethod;

    public PlainMethodInvoker(MapperMethod mapperMethod) {
      super();
      this.mapperMethod = mapperMethod;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args, SqlSession sqlSession) throws Throwable {
        // 最终调用执行方法
      return mapperMethod.execute(sqlSession, args);
    }
  }
// 进入execute方法
  public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
      // 判断mapper中的方法类型，最终调用的还是SqlSession中的方法
    switch (command.getType()) {
      case INSERT: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      case DELETE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));
        break;
      }
      case SELECT:
            // 无返回，并且有ResultHandler方法参数，将查询结果提交给ResultHandler处理
        if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
          if (method.returnsOptional()
              && (result == null || !method.getReturnType().equals(result.getClass()))) {
            result = Optional.ofNullable(result);
          }
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
      // 返回结果为null，并且返回类型为基本类型，则抛出BindingException异常
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName()
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }

```

## Mybatis缓存

### 一级缓存：

- sqlsession级别，底层hashMap，默认开启

- 增删改等操作(sqlSession.commit())事物提交刷新缓存

- 创建缓存：

  ```java
  Executor
  CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql);
  ```

- 清理缓存：

  ```java
  SqlSession clearCache()--> DefaultSqlSesion executor.clearLocalCache()--> BaseExecutor localCache.clear()--> PerpetualCache cache.clear()--> cache = new HashMap<>()
  ```

- 调用缓存：

  ```java
  Executor
  <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException;
  ```



### 二级缓存：

- 跨sqlsession，mapper(namespace)级别，底层hashMap

- 缓存数据，不是对象

- 默认不开启，需要手动配置开启

  sqlMapConfig.xml:

  ```xml
  <!--开启二级缓存-->
  <settings>
      <setting name="cacheEnabled" value="true"/>
   </settings>
  ```

  UserMapper.xml

  ```xml
  <!--开启二级缓存-->
  <cache></cache>
  ```

  注解：@CacheNamespace

- 问题：无法实现分布式缓存，不能跨服务器共享缓存

  解决：对接分布式缓存Redis、memcached、ehcache等

- useCache和flushCache

分为三步走：
1）开启全局二级缓存配置

```xml
<settings>
  <setting name="cacheEnabled" value="true"/>
</settings>
```

2) 在需要使用二级缓存的Mapper配置文件中配置标签

```xml
 <cache></cache>
```

3）在具体CURD标签上配置 useCache=true

```xml
<select id="findById" resultType="com.lagou.pojo.User" useCache="true">
   select * from user where id = #{id}
</select>
```

疑问：

1. 如何解析< cache/>标签？

2. 同时开启的情况下，为什么先走二级缓存，再走一级缓存？

3. 为什么要commit之后才生效？
4. 

#### 标签 < cache/> 的解析

根据之前的mybatis源码剖析，xml的解析工作主要交给XMLConfigBuilder.parse()方法来实现

 



#### Mybatis-redis

Rediscache

- mybatis创建时创建

- RedisConfig

- putObject

- Hset-->hash



### 延迟加载：

嵌套查询，多次单表查询

动态代理，默认JavasistProxy、CglibProxy

## mybatis设计模式

##### *构建者模式：创建型模式*

构建多个简单对象，组装成复杂对象 sqlSessionFactoryBuilder XMLConfigBuilder buildConfiguration

##### *工厂模式：创建型模式*

根据需求生产不同对象，通常有相同父类 Factory openSession

##### *代理模式：对象结构模式*

MapperProxyFactory mapper.geMapper()

 

 