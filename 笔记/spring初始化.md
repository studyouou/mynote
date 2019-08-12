# spring������ˢ��

��spring��ʼ���У���ؼ��Ĳ������refresh�����������Ǵ���

```
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			//��ʼ��ǰ׼��
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			//��ȡbeanFactory
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			//��beanFactory�������õ�
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

������prepareRefresh()������

1. ��¼��ǰʱ��
2. ��������ApplicationContext��closedΪfalse����������activeΪtrue��
3. ��ʼ��propertysource

�ڶ���ʱobtainFreshBeanFactory��������Ὣ���Ƕ����beanͨ�������ļ����ؽ�ȥ��

```
loadBeanDefinitions(beanFactory);
```

��������prepareBeanFactory����������úܶ�����Զ�װ������ã���Ϊ�˷�ֹAware ��bean�ڲ��Զ�ע�����ԣ���Ҫ������ignoreDependencyInterface��������bean�ڲ������setter����ע��ᱻ���ԣ������л����BeanPostProcessor��BeanPostProcessor��bean�ĺ��ô�����bean��ʼ����initmethod��@PostConstructע�ⷽ��ִ�У�֮ǰ�ͳ�ʼ����initmethodִ�л�@PostConstructע�ⷽ��ִ�У�֮���bean�������á�������Ҫ�����2��BeanPostProcessor��һ����ApplicationContextAwareProcessor����һ����ApplicationListenerDetector��

```
//ApplicationContextAwareProcessor�����ʼ��֮ǰbean����Ҫ�������¡����������ǣ������bean��Aware�ӿ�
//��һ��ʵ�У�����������жϽ���ApplicationContext�����Դ����ȥ���á�����������Ҫ�߼�
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
//ApplicationListenerDetector��������߼������������ǣ������bean��һ��listener���ͽ���bean���뵽
//listener��list�У���Ҫ�߼�����
public Object postProcessAfterInitialization(Object bean, String beanName) {
		if (bean instanceof ApplicationListener) {
			// potentially not detected as a listener by getBeanNamesForType retrieval
			//�жϸ�listener�Ƿ������Application�д��ڵ��¼�����������ڼ������¼����򲻼���
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

���Ĳ���postProcessBeanFactory������ÿ��ʵ���಻ͬ����WebApplicationContext��˵����Ҫ�����ServletContextAwareProcessor���÷�������bean��ʼ��֮ǰ����Ҫ�����������bean��һ��ServletContextAware��ServletConfigAware��ʵ���࣬�Ὣ�û����µ�ServlertContext��ServletConfig���ݸ���bean��

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

���岽������invokeBeanFactoryPostProcessors��BeanFactoryPostProcessor�ýӿ�������ʱ����д���ġ��������ͨbean��ʼ���󣨲�û��ִ��ָ���ĳ�ʼ����������init-method���Ǳ�@PostConstructע���ע�ķ����������и÷�����������˳�򣬾���Ļ�û��������

```

```

����������registerBeanPostProcessors�����ˣ�Ҳ�ǰ�һ����˳��ʵ����BeanPostProcessor����ע�뵽BeanFactory�У����������ǣ���BeanPostProcessor���������þ��ǽ��г�ʼ����������ǰ����ú�Ĵ���

���߲�����initMessageSource����ʼ�����ڹ��ʻ���MessageSource��

�ڰ˲���initApplicationEventMulticaster�������ʼ���¼������ߵ�SimpleApplicationEventMulticaster��

�ھŲ���onRefresh����ʼ������bean����springboot����ʱ��������ʼ��webService����Ϊspringbootʱ��Ƕtomcat��������Ҫ�Լ�����tomcat��ssm��ʱ����tomcat������������Ҳ�Ͳ���Ҫ�ˣ�

��ʮ��ע��listener��multicaster��

��ʮһ��������bean������DefaultSingletonBeanRegistry���棨����ƽʱ�õĵ��ж�������DefaultSingletonBeanRegistry��singletonObjects��map�У���ǰ��˵��BeanPostProcessҲ���������������õġ�������beanʱ���������initializeBean������

```
//AbstractAutowireCapableBeanFactory��initializeBean����
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
		//����ǰ��ע�ᵽBeanFactory��BeanPostProcess������postProcessBeforeInitialization����
		//���ﻹû��ִ�г�ʼ���������������ڳ�ʼ������֮ǰ���õġ�����ͨ��
		//CommonAnotationBeanPostProcessͨ��������ִ�����ǵ�@PostConstruct������ǰ��ͨ���������з�
		//��list�����þͳ��������������Ƕ����������ǰ��ģ�������ִ�У�˳�������ִ��˳��Ҳ�����ˡ���
		//���ǵ�������beanpostprocess��postProcessBeforeInitializationǰ�÷�����Ȼ�����bean�м�
		//��@PostConstructע�ⷽ����Ȼ�����beanָ����init-method������������
		//����postProcessAfterInitialization��
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

���һ��������ɣ��о�û��ɶ������ᷢ��һ��ContextRefreshedEvent�¼��������ɵڰ˲���SimpleApplicationEventMulticaster���㲥������Ҳ���Ƿ���ֻҪ�����˸��¼��ļ������������յ���Ϣ�����﷢����Ϣ�����û��task�����ǵ��̷߳�����ֱ�ӵ���listener��onApplicationEvent��

����spring�����ľ�ˢ�����ˡ�



# ����WebMvcConfigurationSupportʵ�ֶ�̬ע�����⡣

���˵���룬ǰ���û����Ҳ���Ǽ̳�WebMvcConfigurationSupport���������ν��Լ�beanNameע�뵽Ŀ�����factoryBeanName���ԡ���������׼��ʵ��DispacherServlet�еļ����ࣨHandlerMapping��HandlerMappingAdapter�ȣ��ֻܶ࣬Ҫ��WebMvcConfigurationSupport����������ע���bean��name���ǣ�������HandlerMappingAdapterΪ����ʱ��ͨ��ɨ�踸bean���Ƿ��м̳���WebMvcConfigurationSupport��bean������У��⽫��bean��nameע�뵽HandlerMappingAdapter��RootBeanDefinition��factoryBeanName�����С�����ʵ�������������2������ʵ�֣���ʹ����ĸ˳����ǰ��WebMvcConfigurationSupport��

�����ʱ����HandlerMappingAdapter��RootBeanDefinition������˵��������ͳһ��HandlerMappingAdapter�����RootBeanDefinition�����Ƿ���factoryBeanName���ԣ�����и����ԣ���ȥfactorybean�л�ȡ�ã�ʵ����WebMvcConfigurationSupport���ࣩbean��Class�������﷢���ˣ��������б����@Configurationע����࣬���඼��cglib��̬����ʵ�ֵģ��ǻ��ڼ̳еĴ�������Ȼ��ὫisStatic��������Ϊfalse������ͻ�ͨ����ʵ����WebMvcConfigurationSupport���ࣩ��class�Ƿ�ͨ��cglib��������ǣ��򷵻��丸��class��������ǣ��򷵻ص�ǰclass�������Ż�ȡ�������з�����Ȼ����߼����¾����ж�����Ҫ��ʼ����bean��name�Ƿ�͸ղ�����õ����з����е�Ψһһ��������������ͬ���Ҹ÷���������У�����Ϊ��bean����Ҫ�ɣ�ʵ����WebMvcConfigurationSupport���ࣩ�������ɣ��ں����ķ����У�Ҳ��ͨ���������������ɵģ����������ǵ�ArgumentResolversҲ��ͨ��������̬ע���ȥ�ġ��൱�������÷��䷽���������Ƕ����bean(���Ҫ����Щ������bean�ķ���������ֻ��Ҫ�����涯̬ע��ķ����ͺ��ˣ��ͱ���ע��ArgumentResolvers)��������Ҫ����

```
//AbstractAutowireCapableBeanFactory��createBeanInstance��Ҳ������׼������bean��ʱ�򣬻��жϸ�mbd��
//����factoryMethodName�Ƿ�Ϊ�գ������Ϊ�գ������FactoryMethodName������
if (mbd.getFactoryMethodName() != null) {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}
------------------------------------------------------------		
//ConstructorResolver��instantiateUsingFactoryMethod����
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
			******************************//ʡ���м����
			isStatic=true;
		}
		*****************************************************************
		if (factoryMethodToUse == null || argsToUse == null) {
			// Need to determine the factory method...
			// Try all methods with this name to see if they match the given arguments.
			factoryClass = ClassUtils.getUserClass(factoryClass);

			Method[] rawCandidates = getCandidateMethods(factoryClass, mbd);
			List<Method> candidateList = new ArrayList<>();
			//���漸�о����жϸ�bean��name�Ƿ���ֻ��һ����������ͬ�������˼����Ѱ�ҹ���������
			//�����ǵ�
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
					//����ҵ���������ù�����������beanwapper��ͨ������ȥ���ù���������
					bw.setBeanInstance(instantiate(beanName, mbd, factoryBean, uniqueCandidate, EMPTY_ARGS));
					return bw;
				}
			}
		//������ǹ��������в����ģ��������������ͺ�����ȥע��bean��̫�鷳�ˣ�������Ҫ�ٿ���
------------------------------------------------
```

�����߼�����������ġ�



# ����

�������⡣����factorybean��

������Ҫ��ȡfactorybean�����factorybean�е�getObjectTypeû�б���д�Ļ���spring��������ǰ���һ��&���ţ������жϻ��������������жϣ��������£��������Ͳ���ȥ��ȡgetObject�е�bean�ˡ�������˵��Factorybean���أ���һ��factorybean����Ϊ�����ȡ��ʵ���������ص���ʱ������ж�����factorybean������ǰ��û��&�Ļ������ͻ����factorybean����getObject����ע���ʵ�У�mybatis�Ĵ���ԭ��Ҳ����������

```

//AbstractAutowireCapableBeanFactory��getTypeForFactoryBean��������resultΪnullʱ��Ҳ��������û��
//��дgetObjectType����ʱ�������AbstractFactory.getTypeForFactoryBean����
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
//AbstractFactory.getTypeForFactoryBeanƬ�Σ��������ǻᱻ����&�����ǩ
try {
			FactoryBean<?> factoryBean = doGetBean(FACTORY_BEAN_PREFIX + beanName, FactoryBean.class, null, true);
			return getTypeForFactoryBean(factoryBean);
		}
---------------------------------------------------
//AbstractBeanFactory��getObjectForBeanInstance����������������������ɳɣ�����Ͳ���ֱ�ӷ���
//factorybean��
if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
			return beanInstance;
	}
```

