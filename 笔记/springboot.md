# springcloud 的bootstrap加载过程

在初始化加载时，会首先将springboot启动包下的spring.factories的类，将其对应放在静态工具类SpringFactoriesLoader的一个map属性cache下（在我们自定义包下自定义的META-INF的spring.factories也会加载进去，这些都是当需要的时候就会调用方法加载）这里map的key属性是ClassLoad，这里springboot中是通过当前线程获取classLoader来保证都是一个classLoader。node：这里这个value也是个map类型，其键是根据spring.factories中的“=”号前面的字符串，而值则是 “=” 后面对应的类名，相关代码如下

```
//SpringFactoriesLoader的静态属性
public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
private static final Log logger = LogFactory.getLog(SpringFactoriesLoader.class);
//储存各种factories下加载的类。
private static final Map<ClassLoader, MultiValueMap<String, String>> cache = new ConcurrentReferenceHashMap<>();

```

```
//SpringApplication中获取classLoader的方法
public ClassLoader getClassLoader() {
		if (this.resourceLoader != null) {
			return this.resourceLoader.getClassLoader();
		}
		return ClassUtils.getDefaultClassLoader();
	}
```

在springboot启动过程中，首先会初始化环境（Enviroment实现类，主要功能就是加载配置类，然后可以通过属性获取值。），在初始化时，会将SpringApplicationRunListeners类（该类内部主要就是一个list储存SpringApplicationRunListener实现类。根据spring.factories，EventPublishingRunListener会传入list中，该类生成时内部会初始化initialMulticaster用来发布事件）传入当作参数，该类会遍历内部listeners集合，用来发布ApplicationEnvironmentPreparedEvent事件。这里在spring-cloud-context包的spring.factories又有一个初始化类BootstrapApplicationListener（只要pom中引用的包下的spring.factories都会被加载到上面说的cache中），这个类就是监听该事件的。该事件的onApplicationEvent方法通过调用内部bootstrapServiceContext去构建一个bootstrap环境的ConfigurableApplicationContext，通过build方式构建SpringApplication。

```
SpringApplicationBuilder builder = new SpringApplicationBuilder()
				.profiles(environment.getActiveProfiles()).bannerMode(Mode.OFF)
				.environment(bootstrapEnvironment)
				// Don't use the default properties in this builder
				.registerShutdownHook(false).logStartupInfo(false)
				.web(WebApplicationType.NONE);

//调用build的run方法
final ConfigurableApplicationContext context = builder.run();
//后面的逻辑
if (this.running.compareAndSet(false, true)) {
			synchronized (this.running) {
            // If not already running copy the sources over and then run.
				this.context = build().run(args);
			}
		}
//这里会向新的SpringApplication添加一个properties格式的键值对进去。后面根据这个就不会再循环调用了
bootstrapProperties.addFirst(
				new MapPropertySource(BOOTSTRAP_PROPERTY_SOURCE_NAME, bootstrapMap));
public SpringApplication build(String... args) {
		configureAsChildIfNecessary(args);
		//这里会添加一个这个BootstrapImportSelectorConfiguration进去
		this.application.addPrimarySources(this.sources);
		return this.application;
	}
```

这里就又会去调用上面同一个过程，但是当执行到BootstrapApplicationListener的listener的onApplicationEvent，因为刚刚上个步骤存入了该键，所以直接跳出去了。

```
if (environment.getPropertySources().contains(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
			return;
		}
```

这里面还有一些其他监听器（没看）。

注意，这里的初始化已经是基于bootstrap的了。

# SpringApplication启动过程

首先调用SpringApplication的静态run方法，后面一点就是初始化一个SpringApplication调用run方法，初始化构造方法是，一些解释都在下面标注。

```
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		//赋值resourceLoader，第一次为null
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
		//获取webApplication的类型，这里为servlet
		this.webApplicationType = WebApplicationType.deduceFromClasspath();
		//获取并初始化ApplicationContextInitializer类，配置在spring.factories中
setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
		//和上面一样，配置在spring.factories中，（spring.factories加载代码会放在最后），后面的
		//参数属性就是根据其全类名获取相应的实现类的集合，核心代码也在最后面
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		this.mainApplicationClass = deduceMainApplicationClass();
	}
	
```

接下来就是run方法了

```
public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		//计时器，记录当前时间
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		//设置系统属性java.awt.headless，（后面来了解）
		configureHeadlessProperty();
		//新建一个SpringApplicationRunListeners，同样的从spring.factories中获取
		//SpringApplicationRunListener注册了的实现类。也就一个EventPublishingRunListener
		//该类构造器有2个参数，一个是SpringApplication，一个是String[],springboot在用反射通过构造
		//器创建该类时，会将上一步初始化的listener传入进去，如下
		/***
		*	for (ApplicationListener<?> listener : application.getListeners()) {
		*		this.initialMulticaster.addApplicationListener(listener);
		*	}
		*/
		SpringApplicationRunListeners listeners = getRunListeners(args);
		// 这里就是发布事件了。this.initialMulticaster.multicastEvent(
		// new ApplicationStartingEvent(this.application, this.args));	
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
			//这里就是最上面讲的就是springcloud的bootstrap初始化了。这里注意，在springboot中是没有
			//bootstrap的，因为springcloud中的spring.factories有一个bootstrap的
			//BootstrapApplicationListener，会在这里监听ApplicationEnvironmentPreparedEvent事件
			//这里这个listener就会去完成bootstrap的初始化，如最上面所展示的。
			ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
			//通过环境变量中是否有spring.beaninfo.ignore属性，如果有，则将其覆盖值设置为true，这里有
			configureIgnoreBeanInfo(environment);
			
			Banner printedBanner = printBanner(environment);
			//默认创建AnnotationConfigApplicationContext，该类会初始化2个类
			//AnnotatedBeanDefinitionReader、ClassPathBeanDefinitionScanner
			context = createApplicationContext();
			exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
					
			//refresh前准备，里面进行设置环境，以及initializers集合的ApplicationContextInitializer
			//调用执行。并且发布事件ApplicationContextInitializedEvent。
			prepareContext(context, environment, listeners, applicationArguments, printedBanner);
			//刷新上下文
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
			}
			//发布ApplicationStartedEvent
			listeners.started(context);
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, listeners);
			throw new IllegalStateException(ex);
		}

		try {
			//发布ApplicationReadyEvent
			listeners.running(context);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, null);
			throw new IllegalStateException(ex);
		}
		return context;
	}
```









```
public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
   String factoryClassName = factoryClass.getName();
   return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
}

private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
   MultiValueMap<String, String> result = cache.get(classLoader);
   if (result != null) {
      return result;
   }

   try {
      Enumeration<URL> urls = (classLoader != null ?
            classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
            ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
      result = new LinkedMultiValueMap<>();
      while (urls.hasMoreElements()) {
         URL url = urls.nextElement();
         UrlResource resource = new UrlResource(url);
         Properties properties = PropertiesLoaderUtils.loadProperties(resource);
         for (Map.Entry<?, ?> entry : properties.entrySet()) {
            String factoryClassName = ((String) entry.getKey()).trim();
            for (String factoryName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
               result.add(factoryClassName, factoryName.trim());
            }
         }
      }
      cache.put(classLoader, result);
      return result;
   }
   catch (IOException ex) {
      throw new IllegalArgumentException("Unable to load factories from location [" +
            FACTORIES_RESOURCE_LOCATION + "]", ex);
   }
}
```