最近项目启动使用dubbo+springboot启动，在启动时报错遇到Unable to start ServletWebServerApplicationContext due to missing ServletWebServerFactory

通过观看源码分析，springboot在启动时，会根据是否含有几个类选择对应的ApplicationContext，代码如下：
   
		//实现引用有org.springframework.web.reactive.DispatcherHandler
		if (ClassUtils.isPresent(REACTIVE_WEB_ENVIRONMENT_CLASS, null)
				&& !ClassUtils.isPresent(MVC_WEB_ENVIRONMENT_CLASS, null)
				&& !ClassUtils.isPresent(JERSEY_WEB_ENVIRONMENT_CLASS, null)) {
			return WebApplicationType.REACTIVE;
		}
		//是否同时引用有javax.servlet.Servlet，org.springframework.web.context.ConfigurableWebApplicationContext，如果都有
		则是下面WebApplicationType.SERVLET类型即ApplicationContext为AnnotationConfigServletWebServerApplicationContext，如果只有一个或是都没有，则为WebApplicationType.NONE即默认，为AnnotationConfigApplicationContext；
		for (String className : WEB_ENVIRONMENT_CLASSES) {
			if (!ClassUtils.isPresent(className, null)) {
				return WebApplicationType.NONE;
			}
		}
		return WebApplicationType.SERVLET;

接下来，在spring刷新上下文时，有一个onRefresh是有不同实现的，AnnotationConfigServletWebServerApplicationContext的实现为

	protected void onRefresh() {
		//这是就会去寻找ServletWebServerFactory
		super.onRefresh();
		try {
			createWebServer();
		}
		catch (Throwable ex) {
			throw new ApplicationContextException("Unable to start web server", ex);
		}
	}

但是我的项目是springboot加dubbo，所以不需要tomcat，所以未应用Tomcat类，而ServletWebServerFactory刷新到beanFactory是在ServletWebServerFactoryAutoConfiguration中

	@Configuration
	@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
	@ConditionalOnClass(ServletRequest.class)
	@ConditionalOnWebApplication(type = Type.SERVLET)
	@EnableConfigurationProperties(ServerProperties.class)
	@Import({ ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class,
			//这里就会加载EmbeddedTomcat
			ServletWebServerFactoryConfiguration.EmbeddedTomcat.class,
			ServletWebServerFactoryConfiguration.EmbeddedJetty.class,
			ServletWebServerFactoryConfiguration.EmbeddedUndertow.class })
	public class ServletWebServerFactoryAutoConfiguration {

	。。。。。。。。。。。。。。。。。。。。。。。
	
	}
看EmbeddedTomcat类
	@Configuration
	@ConditionalOnClass({ Servlet.class, Tomcat.class, UpgradeProtocol.class })
	@ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
	public static class EmbeddedTomcat {

		@Bean
		public TomcatServletWebServerFactory tomcatServletWebServerFactory() {
			return new TomcatServletWebServerFactory();
		}

	}
注意他的@ConditionalOnClass({ Servlet.class, Tomcat.class, UpgradeProtocol.class }),这个注解代表项目中需要有这3个类的引用，他们的全类名是javax.servle.Servlet,org.apache.catalina.startup.Tomcat,org.apache.coyote.UpgradeProtocol
我的问题就在这里，由于我pom中引用了spring-boot-starter-amqp，这个需要引用了spring-web，而这个包就有ConfigurableWebApplicationContext，所以第一步的ApplicationContext最终为AnnotationConfigServletWebServerApplicationContext，然而我又没有Tomcat等几个引用，这几个引用在spring-boot-starter-web（包含了pring-boot-starter-tomcat）或pring-boot-starter-tomcat中所引用的tomcat-embed-core，org.apache.tomcat中，所以最终报错，我的解决方案是删除了那个spring-boot-starter-amqp，因为他是需要web容器支持的，所以自己手写连接发送消息的rabbitmq，所以最后加载的是AnnotationConfigApplicationContext，就不需要ServletWebServerFactory。其他问题一般添加那几个包的引用应该也能解决，或者网上其他方案 。