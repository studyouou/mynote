# spring上下文刷新

在spring初始化中，最关键的步骤就是refresh方法。下面是代码

```
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			//初始化前准备
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			//获取beanFactory
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			//对beanFactory进行设置等
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```

首先是prepareRefresh()方法。

1. 记录当前时间
2. 设置属性ApplicationContext中closed为false，设置属性active为true。
3. 初始化propertysource

第二步时obtainFreshBeanFactory。在这里会将我们定义的bean通过配置文件加载进去。

```
loadBeanDefinitions(beanFactory);
```

第三步是prepareBeanFactory，这里会设置很多忽略自动装配的设置，是为了防止Aware 的bean内部自动注入属性（主要是依赖ignoreDependencyInterface方法，该bean内部如果有setter方法注入会被忽略），其中会添加BeanPostProcessor，BeanPostProcessor是bean的后置处理，在bean初始化（initmethod或@PostConstruct注解方法执行）之前和初始化（initmethod执行或@PostConstruct注解方法执行）之后对bean进行设置。这里主要会添加2个BeanPostProcessor，一个是ApplicationContextAwareProcessor，另一个是ApplicationListenerDetector。

```
//ApplicationContextAwareProcessor处理初始化之前bean，主要代码如下。大致作用是，如果该bean是Aware接口
//的一个实列，会根据类型判断将该ApplicationContext的属性传入进去调用。下面是它主要逻辑
if (bean instanceof Aware) {
			if (bean instanceof EnvironmentAware) {
				((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
			}
			if (bean instanceof EmbeddedValueResolverAware) {
				((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
			}
			if (bean instanceof ResourceLoaderAware) {
				((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
			}
			if (bean instanceof ApplicationEventPublisherAware) {
				((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
			}
			if (bean instanceof MessageSourceAware) {
				((MessageSourceAware) bean).setMessageSource(this.applicationContext);
			}
			if (bean instanceof ApplicationContextAware) {
				((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
			}
		}
//ApplicationListenerDetector处理后置逻辑，它的作用是，如果该bean是一个listener，就将该bean加入到
//listener的list中，主要逻辑如下
public Object postProcessAfterInitialization(Object bean, String beanName) {
		if (bean instanceof ApplicationListener) {
			// potentially not detected as a listener by getBeanNamesForType retrieval
			//判断该listener是否监听有Application中存在的事件，如果不存在监听的事件，则不加入
			Boolean flag = this.singletonNames.get(beanName);
			if (Boolean.TRUE.equals(flag)) {
				// singleton bean (top-level or inner): register on the fly
				this.applicationContext.addApplicationListener((ApplicationListener<?>) bean);
			}
			else if (Boolean.FALSE.equals(flag)) {
				if (logger.isWarnEnabled() && !this.applicationContext.containsBean(beanName)) {
					// inner bean with other scope - can't reliably process events
					logger.warn("Inner bean '" + beanName + "' implements ApplicationListener interface " +
							"but is not reachable for event multicasting by its containing ApplicationContext " +
							"because it does not have singleton scope. Only top-level listener beans are allowed " +
							"to be of non-singleton scope.");
				}
				this.singletonNames.remove(beanName);
			}
		}
		return bean;
	}
			
```

第四步是postProcessBeanFactory方法，每个实现类不同，就WebApplicationContext来说，主要是添加ServletContextAwareProcessor，该方法处理bean初始化之前。主要作用是如果该bean是一个ServletContextAware或ServletConfigAware的实现类，会将该环境下的ServlertContext或ServletConfig传递给该bean。

```
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		if (getServletContext() != null && bean instanceof ServletContextAware) {
			((ServletContextAware) bean).setServletContext(getServletContext());
		}
		if (getServletConfig() != null && bean instanceof ServletConfigAware) {
			((ServletConfigAware) bean).setServletConfig(getServletConfig());
		}
		return bean;
	}
```

第五步方法是invokeBeanFactoryPostProcessors。BeanFactoryPostProcessor该接口是在这时候进行处理的。在完成普通bean初始化后（并没有执行指定的初始化方法，如init-method或是被@PostConstruct注解标注的方法），进行该方法处理（处理顺序，具体的还没看懂）。

```

```

第六步就是registerBeanPostProcessors方法了，也是按一定的顺序将实现了BeanPostProcessor的类注入到BeanFactory中（大致依次是）。BeanPostProcessor这个类的作用就是进行初始化方法调用前后调用后的处理。

第七步就是initMessageSource，初始化关于国际化的MessageSource。

第八步是initApplicationEventMulticaster，这里初始化事件发布者的SimpleApplicationEventMulticaster。

第九步是onRefresh，初始化特殊bean，在springboot启动时，这里会初始化webService（因为springboot时内嵌tomcat，所以需要自己启动tomcat，ssm中时外置tomcat，所以在这里也就不需要了）

第十步注册listener到multicaster中

第十一步，创建bean并放入DefaultSingletonBeanRegistry里面（我们平时用的单列都存入在DefaultSingletonBeanRegistry的singletonObjects的map中）。前面说的BeanPostProcess也就是在这里起作用的。当创建bean时，最后会调用initializeBean方法。

```
//AbstractAutowireCapableBeanFactory的initializeBean方法
if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
//applyMergedBeanDefinitionPostProcessors
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		//根据前面注册到BeanFactory的BeanPostProcess来调用postProcessBeforeInitialization方法
		//这里还没有执行初始化方法，所以是在初始化方法之前调用的。并且通过
		//CommonAnotationBeanPostProcess通过放射来执行我们的@PostConstruct方法（前面通过有序排列放
		//入list的作用就出现啦，这里我们定义的是排在前面的，所以先执行，顺便这里的执行顺序也出来了。首
		//先是调用我们beanpostprocess的postProcessBeforeInitialization前置方法，然后调用bean中加
		//有@PostConstruct注解方法，然后调用bean指定的init-method方法，最后调用
		//方法postProcessAfterInitialization）
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
			Object current = processor.postProcessBeforeInitialization(result, beanName);
			if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}
```

最后一步就是完成，感觉没做啥。这里会发布一个ContextRefreshedEvent事件。（是由第八步的SimpleApplicationEventMulticaster来广播发布，也就是方法只要监听了该事件的监听器，都会收到消息。这里发布消息是如果没有task，则是单线程发布，直接调用listener的onApplicationEvent）

整个spring上下文就刷新完了。



# 关于WebMvcConfigurationSupport实现动态注入问题。

看了点代码，前半段没看，也就是继承WebMvcConfigurationSupport这个类是如何将自己beanName注入到目标类的factoryBeanName属性。猜想是在准备实例DispacherServlet中的几个类（HandlerMapping，HandlerMappingAdapter等，很多，只要在WebMvcConfigurationSupport类中名字是注册的bean的name都是，这里以HandlerMappingAdapter为例）时，通过扫描父bean中是否有继承了WebMvcConfigurationSupport的bean，如果有，这将该bean的name注入到HandlerMappingAdapter的RootBeanDefinition的factoryBeanName属性中。所以实验了以下如果有2个该类实现，会使用字母顺序在前的WebMvcConfigurationSupport。

后面的时根据HandlerMappingAdapter的RootBeanDefinition（如无说明，后面统一用HandlerMappingAdapter代替该RootBeanDefinition）中是否有factoryBeanName属性，如果有该属性，则去factorybean中获取该（实现了WebMvcConfigurationSupport的类）bean的Class（在这里发现了，好像所有标记有@Configuration注解的类，该类都是cglib动态代理实现的（是基于继承的代理））。然后会将isStatic属性设置为false。后面就会通过（实现了WebMvcConfigurationSupport的类）该class是否通过cglib代理，如果是，则返回其父类class，如果不是，则返回当前class。紧接着获取该类所有方法，然后的逻辑大致就是判断现在要初始化的bean的name是否和刚才所获得的所有方法中的唯一一个方法的名字相同并且该方法。如果有，则认为该bean是需要由（实现了WebMvcConfigurationSupport的类）工厂生成（在后续的方法中，也是通过方法反射机制完成的）。所有我们的ArgumentResolvers也是通过这样动态注入进去的。相当于重新用放射方法生成我们定义的bean(最好要改那些生成了bean的方法，我们只需要改里面动态注入的方法就好了，就比如注册ArgumentResolvers)。下面主要代码

```
//AbstractAutowireCapableBeanFactory的createBeanInstance，也就是在准备创建bean的时候，会判断该mbd中
//属性factoryMethodName是否为空，如果不为空，则根据FactoryMethodName来构建
if (mbd.getFactoryMethodName() != null) {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}
------------------------------------------------------------		
//ConstructorResolver的instantiateUsingFactoryMethod方法
public BeanWrapper instantiateUsingFactoryMethod(
			String beanName, RootBeanDefinition mbd, @Nullable Object[] explicitArgs) {
		String factoryBeanName = mbd.getFactoryBeanName();
		if (factoryBeanName != null) {
			if (factoryBeanName.equals(beanName)) {
				throw new BeanDefinitionStoreException(mbd.getResourceDescription(), beanName,
						"factory-bean reference points back to the same bean definition");
			}
			factoryBean = this.beanFactory.getBean(factoryBeanName);
			if (mbd.isSingleton() && this.beanFactory.containsSingleton(beanName)) {
				throw new ImplicitlyAppearedSingletonException();
			}
			factoryClass = factoryBean.getClass();
			isStatic = false;
		}else{
			******************************//省略中间代码
			isStatic=true;
		}
		*****************************************************************
		if (factoryMethodToUse == null || argsToUse == null) {
			// Need to determine the factory method...
			// Try all methods with this name to see if they match the given arguments.
			factoryClass = ClassUtils.getUserClass(factoryClass);

			Method[] rawCandidates = getCandidateMethods(factoryClass, mbd);
			List<Method> candidateList = new ArrayList<>();
			//下面几行就是判断该bean的name是否与只有一个方法名相同，差不多意思就是寻找工厂方法。
			//而我们的
			for (Method candidate : rawCandidates) {
				if (Modifier.isStatic(candidate.getModifiers()) == isStatic && mbd.isFactoryMethod(candidate)) {
					candidateList.add(candidate);
				}
			}

			if (candidateList.size() == 1 && explicitArgs == null && !mbd.hasConstructorArgumentValues()) {
				Method uniqueCandidate = candidateList.get(0);
				if (uniqueCandidate.getParameterCount() == 0) {
					mbd.factoryMethodToIntrospect = uniqueCandidate;
					synchronized (mbd.constructorArgumentLock) {
						mbd.resolvedConstructorOrFactoryMethod = uniqueCandidate;
						mbd.constructorArgumentsResolved = true;
						mbd.resolvedConstructorArguments = EMPTY_ARGS;
					}
					//如果找到，这里就用工厂方法生成beanwapper，通过反射去调用工厂方法。
					bw.setBeanInstance(instantiate(beanName, mbd, factoryBean, uniqueCandidate, EMPTY_ARGS));
					return bw;
				}
			}
		//下面的是工厂方法有参数的，参数参数的类型和名字去注册bean，太麻烦了，后面需要再看。
------------------------------------------------
```

大致逻辑差不多是这样的。



# 问题

今天问题。关于factorybean。

我们想要获取factorybean，如果factorybean中的getObjectType没有被覆写的话，spring会在名字前面加一个&符号，后面判断会根据这个符号来判断（代码如下），这样就不会去获取getObject中的bean了。（这里说下Factorybean加载，当一个factorybean不认为是想获取的实例，当加载到他时，如果判断它是factorybean，并且前面没有&的话，他就会进入factorybean调用getObject方法注册该实列，mybatis的代理原理也就是这样）

```

//AbstractAutowireCapableBeanFactory的getTypeForFactoryBean方法，当result为null时，也就是我们没有
//覆写getObjectType方法时，会进入AbstractFactory.getTypeForFactoryBean方法
if (fb != null) {
			// Try to obtain the FactoryBean's object type from this early stage of the instance.
			Class<?> result = getTypeForFactoryBean(fb);
			if (result != null) {
				return result;
			}
			else {
				// No type found for shortcut FactoryBean instance:
				// fall back to full creation of the FactoryBean instance.
				return super.getTypeForFactoryBean(beanName, mbd);
			}
		}
------------------------------------------------------------		
//AbstractFactory.getTypeForFactoryBean片段，这样还是会被打上&这个标签
try {
			FactoryBean<?> factoryBean = doGetBean(FACTORY_BEAN_PREFIX + beanName, FactoryBean.class, null, true);
			return getTypeForFactoryBean(factoryBean);
		}
---------------------------------------------------
//AbstractBeanFactory的getObjectForBeanInstance方法，当上述所有条件达成成，这里就不会直接返回
//factorybean了
if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
			return beanInstance;
	}
```

