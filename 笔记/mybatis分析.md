# mybatis分析

当调用mapper的方法时，调用mybatis为每个类的代理对象。并且会根据每个方法的传入生成对应的MapperMethod信息，里面包含了method方法是对应什么语句的信息，并且会存入缓存。接着调用execute方法。execute方法主要是根据MapperMethod方法的command.getType()（也就是select或insert这些方法）和result来判断执行什么什么语句。接着根据是我们方法中返回类型（list，map，bean）来选择怎么执行，但是查询单个和查询list最后都会使用selectList方法。以单个为列最后选择执行sqlSession.selectOne(command.getName(), param);其中第一个参数是xml中的id（会根据这个查询到MapperStatement），接下来进入执行器Excutor(默认是SimpleExecutor执行query方法）。
进入query方法，首先获得BoundSql对象（包含我们查询语句，以及#{}里面的参数，参数和参数值会封装成ParameterMapping，并且有对应的TypeHandler），接着会将MapperStatement里的关于sql的语句和参数根据hash算法得到一个hash值，存入到CacheKey的list集合里面去（）。最后调用doQuery
开始创建StatementHandler，我们所有的StatementHandler都是RoutingStatementHandler，但是RoutingStatementHandler是用来包装PreparedStatementHandler，CallableStatementHandler和SimpleStatementHandler的，里是有个delegate（默认是PreparedStatementHandler，）
属性就是这个。接着就是PreparedStatementHandler执行prepare方法，返回PrepareStatement。prepare方法就是用来预编译，就是对原生的JDBC的prepareStatement功能做了一样的作用。防止sql注入。下一步就是parameterHandler注入参数值了。parameterHandler底层就是通过TypeHandler来进行参数值设定的
（比如TypeHandler实现类为IntegerTypeHandler，就会调用prepareStatement的setInt方法设置参数了），最后处理的结果交给ResultSetHandler
来处理。
大致执行流程代码
当执行查询方法时，代理Mapper会根据xml中配置生成MapperMethod调用sqlSession执行查询，而sqlSession则通过内部封装的Executor
去调用StatementHandler预编译sql，并且插入参数，最后执行sql语句，statementHandler并将结果交给ResultHandler去处理返回结果。


mybatis自定义插件分析
mybatis的每个插件都需要继承Interceptor，并且标注注解@Intercepts，一个自定义插件大致如下
```
@Intercepts(@Signature(type = StatementHandler.class,method = "prepare",args = {Connection.class,Integer.class}))
public class GetOver5Interceptor implements Interceptor {
	@Override
    public Object intercept(Invocation invocation) throws Throwable {
    	return invocation.proceed();
    }

	@Override
    public Object plugin(Object o) {
        return Plugin.wrap(o,this);
    }

    @Override
    public void setProperties(Properties properties) {

    }
}

```

注册自定义插件

```
<plugins>
        <plugin interceptor=""></plugin>
</plugins>
```

当我们通过全局设置注册了Intercept,mybatis在解析configuration时就会去注册插件

```
private void pluginElement(XNode parent) throws Exception {
    if (parent != null) {
       //获取所有插件标签
      for (XNode child : parent.getChildren()) {
       //获取插件名
        String interceptor = child.getStringAttribute("interceptor");
        //获取插件内配置的属性值
        Properties properties = child.getChildrenAsProperties();
        //实例化一个插件
        Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).newInstance();
        //设置属性
        interceptorInstance.setProperties(properties);
        //加入intercepteChain中
        configuration.addInterceptor(interceptorInstance);
      }
    }
  }
```

通过调用addInterceptor方法去将interceptor注入到configuration的interceptorChain的list中。

自定义插件用在4中类型中。分别是

1. StatementHandler实现类中
2. ParameterHandler实现类中
3. ResultHandler实现类中
4. Executor实现类中  

因为在创建这四个类的实现类是，会调用interceptorChain.pluginAll方法，如下

```
public ParameterHandler newParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
    ParameterHandler parameterHandler = mappedStatement.getLang().createParameterHandler(mappedStatement, parameterObject, boundSql);
    parameterHandler = (ParameterHandler) interceptorChain.pluginAll(parameterHandler);
    return parameterHandler;
}

public ResultSetHandler newResultSetHandler(Executor executor, MappedStatement mappedStatement, RowBounds rowBounds, ParameterHandler parameterHandler,
  ResultHandler resultHandler, BoundSql boundSql) {
    ResultSetHandler resultSetHandler = new DefaultResultSetHandler(executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds);
    resultSetHandler = (ResultSetHandler) interceptorChain.pluginAll(resultSetHandler);
    return resultSetHandler;
}

public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
}

public Executor newExecutor(Transaction transaction, ExecutorType executorType, boolean autoCommit) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    if (cacheEnabled) {
      executor = new CachingExecutor(executor, autoCommit);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
}
```

通过执行pluginAll返回一个代理对象类。

pluginAll的方法如下：

```
 public Object pluginAll(Object target) {
 	//获取每个插件并执行plugin方法，并返回一个代理类
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }
```

看出会执行每个interceptor的plugin方法，plugin方法再调用Plugin.wrap(target, this)，而Plugin本身就是实现了JDK的动态代理InvocationHandler接口，wrap内容如下

```
public static Object wrap(Object target, Interceptor interceptor) {
	//获取插件上拦截的所有方法
    Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
    Class<?> type = target.getClass();
    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
    if (interfaces.length > 0) {
      return Proxy.newProxyInstance(
          type.getClassLoader(),
          interfaces,
          new Plugin(target, interceptor, signatureMap));
    }
    return target;
  }
```

getSignatureMap的作用是根据当前拦截器的所有@Signature标签中的拦截类，拦截方法以及参数，获得需要拦截的Method。方法如下：

```
private static Map<Class<?>, Set<Method>> getSignatureMap(Interceptor interceptor) {
	//获取注解@Intercepts
    Intercepts interceptsAnnotation = interceptor.getClass().getAnnotation(Intercepts.class);
    if (interceptsAnnotation == null) {
      throw new PluginException("No @Intercepts annotation was found in interceptor " + interceptor.getClass().getName());
    }
    //获取注解上@Signature注解
    Signature[] sigs = interceptsAnnotation.value();
    Map<Class<?>, Set<Method>> signatureMap = new HashMap<>();
    //遍历每一个Signature注解
    for (Signature sig : sigs) {
      //computeIfAbsent方法是如果key对应有value，可返回原来的value，如果没有，则添加后面Fuction函数
      //所生成的值作为value存进map中。如果不存在sig.type()，则new个set并返回，存在则返回原来的值。
      //（sig.type()是要拦截的类，比如StatementHandler）
      Set<Method> methods = signatureMap.computeIfAbsent(sig.type(), k -> new HashSet<>());
      try {
        //根据参数以及方法名获取Method，然后添加到set中，如果反法找不到或则不匹配参数会报错
        Method method = sig.type().getMethod(sig.method(), sig.args());
        methods.add(method);
      } catch (NoSuchMethodException e) {
        throw new PluginException("Could not find method on " + sig.type() + " named " + sig.method() + ". Cause: " + e, e);
      }
    }
    //返回map（注意这里map的key是拦截类型的Class）
    return signatureMap;
  }
```

通过过去拦截器上标注的内容，

```
private static Class<?>[] getAllInterfaces(Class<?> type, Map<Class<?>, Set<Method>> signatureMap) {
    Set<Class<?>> interfaces = new HashSet<>();
    //这里这是根据被拦截的类的class是否在该拦截器上存在（就是如果拦截器上的type = StatementHandler这个     //属性和执行刚刚pluginAll方法类的所有接口有一样的，就会代理这一个接口）
    while (type != null) {
      for (Class<?> c : type.getInterfaces()) {
        if (signatureMap.containsKey(c)) {
          interfaces.add(c);
        }
      }
      //获取父类type，后面传递父类加载器用（我感觉一样，父类加载器都一样的）
      type = type.getSuperclass();
    }
    return interfaces.toArray(new Class<?>[interfaces.size()]);
  }
```

最后就是生成代理类了，然后进入下一个拦截器的代理生成，大概就是一层接着一层代理，然后最后看代理方法如下：

```
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      Set<Method> methods = signatureMap.get(method.getDeclaringClass());
      //如果执行的方法为methods含有的方法（就是刚刚从@Signature读取的方法）
      //就执行我们自己拦截器的方法，如果不含就执行原来的method方法
      if (methods != null && methods.contains(method)) {
        return interceptor.intercept(new Invocation(target, method, args));
      }
      return method.invoke(target, args);
    } catch (Exception e) {
      throw ExceptionUtil.unwrapThrowable(e);
    }
  }
```



mybatis的一级缓存是存在与SqlSession范围的，只有当同一个sqlSession查询时才会生效。二级缓存存在与mapper内的，开启二级缓存首先要设置（<setting name="cacheEnabled" value="true"></setting>，一级缓存不会受其影响，因为一级缓存取数据都是从BaseExecutor的localCache取，所以同一个sqlSession都时共享的），因为当设置为true时，sqlSession内部的Executor就为CachingExecutor，而CachingExecutor内部还是有一个SimpleExecutor充当当缓存没中时，实际的调用查询的执行器。和其他查询器的区别就是当sqlSession调用query方法时，它先会获取MappedStatement.getCache（实质上mapper.xml中每个id都会对应有一个MappedStatement，但是他们都使用一个SynchronizedCache），这也就是需要开启二级缓存的第二个条件，在mapper中添加<cache></cache>标签。这样才能开启。代码如下

```
 MappedStatement.Builder statementBuilder = new MappedStatement.Builder(configuration, id, sqlSource, sqlCommandType)
        .resource(resource)
        .fetchSize(fetchSize)
        .timeout(timeout)
        .statementType(statementType)
        .keyGenerator(keyGenerator)
        .keyProperty(keyProperty)
        .keyColumn(keyColumn)
        .databaseId(databaseId)
        .lang(lang)
        .resultOrdered(resultOrdered)
        .resultSets(resultSets)
        .resultMaps(getStatementResultMaps(resultMap, resultType, id))
        .resultSetType(resultSetType)
        .flushCacheRequired(valueOrDefault(flushCache, !isSelect))
        .useCache(valueOrDefault(useCache, isSelect))
		//这里就是添加二级缓存Cache，currentCache是构建该Mapper当有<Cache>时生成的
        .cache(currentCache);

```

```
//XMLMapperBuilder的configurationElement方法
cacheElement(context.evalNode("cache"));
//XMLMapperBuilder方法
private void cacheElement(XNode context) {
    if (context != null) {
      String type = context.getStringAttribute("type", "PERPETUAL");
      Class<? extends Cache> typeClass = typeAliasRegistry.resolveAlias(type);
      String eviction = context.getStringAttribute("eviction", "LRU");
      Class<? extends Cache> evictionClass = typeAliasRegistry.resolveAlias(eviction);
      Long flushInterval = context.getLongAttribute("flushInterval");
      Integer size = context.getIntAttribute("size");
      boolean readWrite = !context.getBooleanAttribute("readOnly", false);
      boolean blocking = context.getBooleanAttribute("blocking", false);
      Properties props = context.getChildrenAsProperties();
      //当cache标签存在时，构建Cache，初始化一个Cache，并且会添加到configuration的caches中
      builderAssistant.useNewCache(typeClass, evictionClass, flushInterval, size, readWrite, blocking, props);
    }
  }

```

```
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    //当配置的cacheEnabled为true时
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
```

```
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    //MappedStatement，当开启二级缓存时，每个MappedStatement都维护了自己的Cache（自己的缓存）。
    Cache cache = ms.getCache();
    if (cache != null) {
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, boundSql);
        //如果cache中是否存在key（key是上面根据条件生成的CacheKye）
        @SuppressWarnings("unchecked")
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          //不存在后面添加进去。tcm相当与每个CacheExecutor维护的保存每个MappedStatement的缓存，它内部		  //实际储存由map当容器，当当前CacheExecutor存在在Cache时，则不该Cache到map中，不存在则添加。
          list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```

最后在每个CachingExecutor都有一个管理Cache的类TransactionalCacheManager，其内部封装了Map<Cache, TransactionalCache>这样一个map，这个map的key是SynchronizedCache，value是对SynchronizedCache封装了的TransactionalCache。当只有查询时，首先会判断是否在缓存中，调用TransactionalCache.getObject(key)实际调用的是SynchronizedCache.getObject(key)，是同步的，所以在同一时间，只能一个CachingExecutor来操作所有MappedStatement中的共同的Cache。当缓存未命中时，就会调用entriesMissedInCache.add(key)（注意这里TransactionalCache内部封装了1个map和一个set，set用来当未命中时，存入CacheKey，map用来添加未命中缓存的CacheKey以及所查询出的内容），当从数据库取得数据时候，会存入Cache中;当有操作数据的sql语句（update，insert，delete）中，对应的StatementMapped中flushCacheRequired为true，就会使得将TransactionalCache中clearOnCommit = true;最后在提交的时候，会清除该mapper.xml中所有缓存，然后清楚该内部entriesToAddOnCommit的内容（除非查询语句在插入语句之后）。一个二级缓存的过程差不多就完成了。

