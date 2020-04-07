# springboot Auto类加载
通过前面AutoConfiguration加载过程的解析。我们知道如何自动加载Auto类，如果自己写，需要在classpath下创建META-INF文件夹，再创建一个spring.factories。键是org.springframework.boot.autoconfigure.EnableAutoConfiguration，值是Auto类的全类名，通过逗号隔开。

# dubbo springboot加载
首先需要引入pom文件

		<dependency>
            <groupId>com.alibaba.boot</groupId>
            <artifactId>dubbo-spring-boot-starter</artifactId>
            <version>0.2.0</version>
        </dependency>


导入jar包中，自然有spring.factories，该文件下有

		org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
		com.alibaba.boot.dubbo.autoconfigure.DubboAutoConfiguration
		
		
		org.springframework.context.ApplicationListener=\
		com.alibaba.boot.dubbo.context.event.OverrideDubboConfigApplicationListener,\
		com.alibaba.boot.dubbo.context.event.WelcomeLogoApplicationListener,\
		com.alibaba.boot.dubbo.context.event.AwaitingNonWebApplicationListener

1. 后面几个是listener，监听spring相关事件的，我们主要分析DubboAutoConfiguration类。在分析前，先说如何加载进去。有2种方法，第一个是在启动类上加入@EnableDubbo注解，该注解上有一个@DubboComponentScan注解，这个注解上有@Import(DubboComponentScanRegistrar.class)注解，这个DubboComponentScanRegistrar会注入下面2个类。如果在application.yml中配置了dubbo.scan.basePackages属性，会再次加载一个ServiceAnnotationBeanPostProcessor用来扫描。
第二种就是配置文件中加入dubbo.scan.basePackages属性，必须是basePackages，不能是base-packages，这样就可以加载ServiceAnnotationBeanPostProcessor。

2. 分析2个类ServiceAnnotationBeanPostProcessor和ReferenceAnnotationBeanPostProcessor这两个类，这两个类都是实现了BeanDefinitionRegistryPostProcessor，实现了该类的，会在spring刷新上下文是调用postProcessBeanDefinitionRegistry方法。先看ServiceAnnotationBeanPostProcessor，这个类是关于注册服务使用的，也就是用来扫描阿里的@Service注解。代码如下

		@Override
		public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
		//解析配置
	    Set<String> resolvedPackagesToScan = resolvePackagesToScan(packagesToScan);
	
	    if (!CollectionUtils.isEmpty(resolvedPackagesToScan)) {
			//开始扫描解析配置的路径上的包
	        registerServiceBeans(resolvedPackagesToScan, registry);
	    } else {
	        if (logger.isWarnEnabled()) {
	            logger.warn("packagesToScan is empty , ServiceBean registry will be ignored!");
	        }
	    }
	
		}
	
		-------------------------------------------------------------------------


		private void registerServiceBeans(Set<String> packagesToScan, BeanDefinitionRegistry registry) {
		//初始化scaner
        DubboClassPathBeanDefinitionScanner scanner =
                new DubboClassPathBeanDefinitionScanner(registry, environment, resourceLoader);
		//获取bean命名方案，如果无则以AnnotationBeanNameGenerator当作bean名字生成方案
        BeanNameGenerator beanNameGenerator = resolveBeanNameGenerator(registry);

        scanner.setBeanNameGenerator(beanNameGenerator);
		//添加需要扫描的注解，这里是阿里的Service
        scanner.addIncludeFilter(new AnnotationTypeFilter(Service.class));
		
        for (String packageToScan : packagesToScan) {

            // Registers @Service Bean first
            scanner.scan(packageToScan);

            // Finds all BeanDefinitionHolders of @Service whether @ComponentScan scans or not.
            Set<BeanDefinitionHolder> beanDefinitionHolders =
                    findServiceBeanDefinitionHolders(scanner, packageToScan, registry, beanNameGenerator);

            if (!CollectionUtils.isEmpty(beanDefinitionHolders)) {

                for (BeanDefinitionHolder beanDefinitionHolder : beanDefinitionHolders) {
                    registerServiceBean(beanDefinitionHolder, registry, scanner);
                }

                if (logger.isInfoEnabled()) {
                    logger.info(beanDefinitionHolders.size() + " annotated Dubbo's @Service Components { " +
                            beanDefinitionHolders +
                            " } were scanned under package[" + packageToScan + "]");
                }

            } else {

                if (logger.isWarnEnabled()) {
                    logger.warn("No Spring Bean annotating Dubbo's @Service was found under package["
                            + packageToScan + "]");
                }

            }

        }

    }