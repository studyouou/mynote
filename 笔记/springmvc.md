1、srpingmvc dispacher初始化。每个dispatcherServlet 在初始化时都会拥有一个WebApplicationContext。
初始化时，首先会创建一个WebApplicationContext，然后再创建dispatcherServlet，并将WebApplicationContext作为构造参数
传给dispatcherServlet，再添加dispatcherServlet的初始context类。
并且在启动tomcat时，会自动扫描该项目下是否有 WebApplicationInitializer的实现类，如果有，会自动去加载并且初始化dispatcherServlet，
如果在web.xml中配置有dispatcherServlet，会覆盖掉自己实现的dispatcherServlet。

2、springmvc拦截器HandlerInterceptor
自定义拦截器需要实现HandlerInterceptor，并且需要实现preHandle，postHandle，afterCompletion三个方法。
preHandle是在执行请求之前执行，返回值为boolean，当返回true时，接着向下执行，当为false时，就直接不会执行实体方法和下面的方法。
postHandle：是在实体处理程序之后执行的方法，当实体执行程序上有@ResponseBody时，就可能出现先返回值的现象（就是打断点并不会
阻止返回值返回）
afterCompletion：是在完成请求后执行的方法。
实现完后，需要配置该拦截器
<mvc:interceptors>
        <mvc:interceptor>
            <mvc:mapping path="/**"></mvc:mapping>
            <!--<mvc:exclude-mapping path="***"/>-->
            <bean id="intercept" class="****"></bean>
        </mvc:interceptor>
    </mvc:interceptors>
其中mvc:mapping代表需要拦截的请求路径，<mvc:exclude-mapping>是额外不需拦截的路径，bean就是自己定义的拦截器了
数据验证，当hibernet-validate的版本过高时，会报错，目前不知到解决办法。

3、springmvc上传文件
需要pom添加io依赖
 <dependency>
      <groupId>commons-fileupload</groupId>
      <artifactId>commons-fileupload</artifactId>
      <version>1.2.1</version> <!-- makesure correct version here -->
    </dependency>
    <dependency>
      <groupId>commons-io</groupId>
      <artifactId>commons-io</artifactId>
      <version>1.3.2</version>
    </dependency>
然后需要添加文件解析器
 <bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
        <property name="defaultEncoding" value="utf-8"></property>
        <property name="maxUploadSizePerFile" value="1024"></property>
        <property name="maxUploadSize" value="10240"></property>
    </bean>
最后写方法
public String upload(@RequestParam("file") MultipartFile file){
         ......
}
并且声明List<MultipartFile>可以解析上传多个文件。

4、矩阵变量，例子
//  GET /start/2;q=32/xx;p=23
@RequestMapping("/start/{id}/{name}")
public String start(HttpServletRequest request,@MatrixVariable(name="q" ,pathVal="id") int q,@MatrixVariable(name="p",pathVal="name")int p){
        ****
}
其中name就是上面url对应的q，pathVal对应上面相应名称。

5注解
@PathVariable:方法参数设置对应的属性获取值，应用与restful接口
@RequestParam：请求参数
@RequestHeader：请求参数，用于获取请求头值。
@CookieValue：用于获取请求携带的cookie。不必须的话可以设置参数required=false
@ModelAttribute：使用在方法参数内用于数据绑定。如果数据绑定错误，会抛出BindException。如果需要控制器检查，则后面可以紧跟
BindingResult用于收集错误。还有是自己绑定内部数据，不需要外部绑定如：
 @ModelAttribute
    public User sets(){
        User user =  new User();
        user.setName("xxxxx");
        user.setAge(3333);
        return user;
    }

 public String bindtest(@ModelAttribute(value = "user",binding = false)User user, BindingResult bindingResult){

}
这样就是绑定自己内部的User

@InitBinder
这个是在该controller使用
1、将请求参数（即表单或查询数据）绑定到模型对象。
2、将基于字符串的请求值（例如请求参数，路径变量，标题，cookie等）转换为目标类型的控制器方法参数。
3、String在呈现HTML表单时将模型对象值格式化为值。
如绑定一个日期转化器
@InitBinder
 public void format(WebDataBinder webDataBinder){
        SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd");
        simpleDateFormat.setLenient(false);
        webDataBinder.registerCustomEditor(Date.class,new CustomDateEditor(simpleDateFormat,false));
    }
可以配合ControllerAdvice使用

6、异步处理
springmvc开启的异步处理和springboot的有点不同
1、springmvc异步是，当请求进来时，servlet线程池会启动一个线程处理该请求，如果这里是异步的话，当处理到处理方法开启线程时，
线程会退出，并由后台新开启的去处理，而servlet线程接着去接受处理其他请求，response挂起，等待当方法处理完成后，再调用一个
servlet线程去返回response，可以用mvc的handlerinterceptor验证。代码
```
@RequestMapping("/asyn")
    public DeferredResult<Model> asyncTask(){
        DeferredResult<Model> deferredResult = new DeferredResult<Model>(15000L);
        System.out.println("请求线程:"+Thread.currentThread().getId());
        doSometh(deferredResult);
        deferredResult.onTimeout(new Runnable() {
            @Override
            public void run() {
                System.out.println("超时执行了,执行线程是:"+Thread.currentThread().getId());
            }
        });
        return deferredResult;
```
```
return (()-> {  //第一种方式是返回callable
            for (int i = 0; i < 20; i++) {
                Thread.sleep(500);
                System.out.println(i);
            }
            Model model = ModelUtil.getModel("异步处理线程:" + Thread.currentThread().getId() + "处理完成", 1, "chuli");
            return model;
        });
```
springboot中启动的异步是，在用户请求后，方法处理直接开启新线程，并且直接放回结果给用户，后台接着处理该业务。
直接在需要实现异步Controller上加@EnableAsync注解，在需要同步的方法上加@Async注解，代码如下
@RestController
@EnableAsync
public class CommonController {
@RequestMapping("/anysAdd")
    @ResponseBody
    public Model anysAdd(){
        logger.info("当前请求线程是:"+Thread.currentThread().getId());
        aysnService.asyncAdd();
        return ModelUtil.getModel("异步线程处理",1,Thread.currentThread().getId());
    }
}
-----------------------
@Service
public class AysnService {
    @Async
    public void asyncAdd(){
        for (int i=0;i<10;i++){
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("线程"+Thread.currentThread().getId()+"正在为您处理");
        }
    }
}



# 启动分析

1. 首先启动会根据web.xml中配置的ContextLoaderListener用来监听ServletContext初始化，当可以从ServletContextEvent事件中获取ServletContext。接着调用initWebApplicationContext

```
initWebApplicationContext(event.getServletContext());
```

在这个方法体内，会创建一个WebApplicationContext，再去初始化它

```
if (this.context == null) {
				//判断当前context是否为空，为空创建一个ApplicationContext
				this.context = createWebApplicationContext(servletContext);
			}
			//判断创建的context是否ConfigurableWebApplicationContext的实列
			if (this.context instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
				//判断cwac是否已经刷新
				if (!cwac.isActive()) {
					// The context has not yet been refreshed -> provide services such as
					// setting the parent context, setting the application context id, etc
					if (cwac.getParent() == null) {
						// The context instance was injected without an explicit parent ->
						// determine parent for root web application context, if any.
						ApplicationContext parent = loadParentContext(servletContext);
						cwac.setParent(parent);
					}
					//刷新上下文（初始化容器上下文）
					configureAndRefreshWebApplicationContext(cwac, servletContext);
				}
			}
```

configureAndRefreshWebApplicationContext的代码如下：

```
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
		//判断对象地址是否相同
		if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
			// The application context id is still set to its original default value
			// -> assign a more useful id based on available information
			String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
			if (idParam != null) {
				wac.setId(idParam);
			}
			else {
				// Generate default id...
				wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
						ObjectUtils.getDisplayString(sc.getContextPath()));
			}
		}

		wac.setServletContext(sc);
		//从web.xml中获取contextConfigLocation参数的值（配置文件位置）
		String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
		if (configLocationParam != null) {
			//不为null就设置为该ApplicationContext的配置地址
			wac.setConfigLocation(configLocationParam);
		}

		// The wac environment's #initPropertySources will be called in any case when the context
		// is refreshed; do it eagerly here to ensure servlet property sources are in place for
		// use in any post-processing or initialization that occurs below prior to #refresh
		ConfigurableEnvironment env = wac.getEnvironment();
		if (env instanceof ConfigurableWebEnvironment) {
			((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
		}
		//获取自定义的ApplicationContextInitializer
		customizeContext(sc, wac);
		wac.refresh();
	}
```

最后刷新上下文，并将该上下文添加到ServletContext中，该WebApplicationContext作为spring的上下文。

因为DispatcherServlet是继承了FrameworkServlet，HttpServletBeanHttpServletGenericServletServlet，

在第一次初始化Sevlet类时，都会调用其init方法。

首先会获取web.xml中servlet中init-param的值。然后调用FrameworkServlet实现的initServletBean方法，重要的代码如下：

```
this.webApplicationContext = initWebApplicationContext();
initFrameworkServlet();
```

这两段代码分别如下

```
protected WebApplicationContext initWebApplicationContext() {
//从ServletContext中取的rootContext，也就是上面初始化并添加的WebApplicationContext
WebApplicationContext rootContext =
		WebApplicationContextUtils.getWebApplicationContext(getServletContext());
		WebApplicationContext wac = null;
		//判断webApplicationContext属性是否为空，不为空就说明已经初始化了一个WebApplicationContext。
		if (this.webApplicationContext != null) {
			// A context instance was injected at construction time -> use it
			wac = this.webApplicationContext;
			if (wac instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
                //如果已经刷新过了，则不需要再刷新
				if (!cwac.isActive()) {
					// The context has not yet been refreshed -> provide services such as
					// setting the parent context, setting the application context id, etc
					//如果未刷新，则把Sping级别的ApplicationContext设置为父属性
					if (cwac.getParent() == null) {
						// The context instance was injected without an explicit parent -> set
						// the root application context (if any; may be null) as the parent
						cwac.setParent(rootContext);
					}
					//刷新
					configureAndRefreshWebApplicationContext(cwac);
				}
			}
		}
		if (wac == null) {
			// No context instance was injected at construction time -> see if one
			// has been registered in the servlet context. If one exists, it is assumed
			// that the parent context (if any) has already been set and that the
			// user has performed any initialization such as setting the context id
			//查看contextAttribute是否被赋值
			wac = findWebApplicationContext();
		}
		//webApplicationContext属性并未赋值时
		if (wac == null) {
			// No context instance is defined for this servlet -> create a local one
			//创建一个新的WebApplicationContext，并指定parent属性为rootContext
			wac = createWebApplicationContext(rootContext);
		}

		if (!this.refreshEventReceived) {
			// Either the context is not a ConfigurableApplicationContext with refresh
			// support or the context injected at construction time had already been
			// refreshed -> trigger initial onRefresh manually here.
			synchronized (this.onRefreshMonitor) {
				onRefresh(wac);
			}
		}

		if (this.publishContext) {
			// Publish the context as a servlet context attribute.
			String attrName = getServletContextAttributeName();
			getServletContext().setAttribute(attrName, wac);
		}

		return wac;
```

所以DispatcherServlet内部含有自己的WebApplicationContext上下文

//配置文件解析，待看

DispatcherServlet的解析过程

首先FrameworkServlet是覆写了父类的service方法，当有请求进来的时候，会通过service方法调用processRequest，processRequest会获取请求，并将requestAttributes和LocaleContext存储在ThreadLocal中其中获得LocaleContext的方法buildLocaleContext被DispatcherServlet覆写，通过localeResolver类型来执行resolveLocaleContext方法，就CookieLocaleResolver而言，在配置时会cookieName，也就是用来通过cookie来获取相应的国际化名（en_US等），所以在这里是没有设置cookie，只是一直在取cookie，所以当切换语言时，一直会失败，这时就需要配拦截器了，根据和cookie名一样的参数获取当前语言，并且通过LocaleResolver的setLocale方法将Locale设置到request内，cookie需要重新保存进客户端中（Locale和timeZone都是存进request的LOCALE_REQUEST_ATTRIBUTE_NAME，TIME_ZONE_REQUEST_ATTRIBUTE_NAME这两个属性中），springmvc实现了这样一个拦截器（不管session还是cookie实现）LocaleChangeInterceptor，会调用LocaleResolver的setLocale储存Locale。

```
public LocaleContext resolveLocaleContext(final HttpServletRequest request) {
		parseLocaleCookieIfNecessary(request);
		return new TimeZoneAwareLocaleContext() {
			@Override
			@Nullable
			public Locale getLocale() {
				return (Locale) request.getAttribute(LOCALE_REQUEST_ATTRIBUTE_NAME);
			}
			@Override
			@Nullable
			public TimeZone getTimeZone() {
				return (TimeZone) request.getAttribute(TIME_ZONE_REQUEST_ATTRIBUTE_NAME);
			}
		};
}
private void parseLocaleCookieIfNecessary(HttpServletRequest request) {
		if (request.getAttribute(LOCALE_REQUEST_ATTRIBUTE_NAME) == null) {
		*********
		String cookieName = getCookieName();
			if (cookieName != null) {
				Cookie cookie = WebUtils.getCookie(request, cookieName);
				if (cookie != null) {
					String value = cookie.getValue();
					String localePart = value;
					*********	
				}
				*********
        request.setAttribute(LOCALE_REQUEST_ATTRIBUTE_NAME,
        (locale != null ? locale : determineDefaultLocale(request)));
        request.setAttribute(TIME_ZONE_REQUEST_ATTRIBUTE_NAME,
        (timeZone != null ? timeZone : determineDefaultTimeZone(request)));  
        }
```

```
//LocaleChangeInterceptor的preHandle方法中调用。
localeResolver.setLocale(request, response, parseLocaleValue(newLocale));
```

接下来进入doService方法，他会为每个request请求添加下面几个参数即对应类

```
//添加上下文
request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
//添加localeResolver
request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
//添加themeResolver（待分析）
request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
//（待分析）
request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());
```

再接着到doDispatch，首先会判断该请求是否为文件上传的请求request请求是否包含multipart/form-data头，然后首先会解析request的url，解析出uri。通过该uri找到对应的控制器的方法，封装在HandlerMethod（其内部封装了controller，对应的方法，需要的参数，以及方法的注解等）中。最后将HandlerMethod和拦截器封装成一个HandlerExecutionChain返回给DispatcherServlet。DispatcherServlet通过HandlerMethod返回一个HandlerAdapter。然后通过HandlerExecutionChain中的拦截器对请求拦截处理（这里直接是调用拦截器的preHandler方法，如果返回false，直接调用拦截器的afterCompletion方法，并且返回false给DispatcherServlet）。然后用HandlerAdapter去执行方法。在这过程中，他会去找到Controller中的InitBind绑定的转换器等，会绑定参数，以及是否异步等（有点复杂，没怎么看的懂，根据方法名这么判断）。最后属性绑定完，执行ServletInvocableHandlerMethod的invokeAndHandle方法，invokeAndHandle通过反射调用controller的方法，代码大致如下

```
//DispatcherServlet的HandlerAdapter适配器调用执行handle（最底层通过放射调用controller中的方法）
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
//最终调用invokeHandlerMethod方法
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
		//里面包含http中缓存属性。
		ServletWebRequest webRequest = new ServletWebRequest(request, response);
		try {
			//根据@InitBinder注解注册的转换器工厂（待验证）
			WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
			ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
			
			//执行主体，调用它的invokeAndHandle方法执行反射方法调用controller方法
			ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
			if (this.argumentResolvers != null) {
				//如果配置了，就设置参数解析器
				invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
			}
			if (this.returnValueHandlers != null) {
				//不太知道它的作用
				invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
			}
			//设置binderFactory参数
			invocableMethod.setDataBinderFactory(binderFactory);
			invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);
			
			//（猜测，因该是用来存储model模型和view的）
			ModelAndViewContainer mavContainer = new ModelAndViewContainer();
			mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
			//将session以及里的值都会存在这里面
			modelFactory.initModel(webRequest, mavContainer, invocableMethod);
			mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);
			//这下面几行都是关于并发的，后续待看
			AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
			asyncWebRequest.setTimeout(this.asyncRequestTimeout);

			WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
			asyncManager.setTaskExecutor(this.taskExecutor);
			asyncManager.setAsyncWebRequest(asyncWebRequest);
			asyncManager.registerCallableInterceptors(this.callableInterceptors);
			asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

			if (asyncManager.hasConcurrentResult()) {
				Object result = asyncManager.getConcurrentResult();
				mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
				asyncManager.clearConcurrentResult();
				LogFormatUtils.traceDebug(logger, traceOn -> {
					String formatted = LogFormatUtils.formatValue(result, !traceOn);
					return "Resume with async result [" + formatted + "]";
				});
				invocableMethod = invocableMethod.wrapConcurrentResult(result);
			}
			//通过这里，就是执行调用放射的入口了，注意当方法或类上标注@ResponseBody
			//时，会直接放回值，所以拦截器的proHandler和afterCompletion是无法拦截的
			invocableMethod.invokeAndHandle(webRequest, mavContainer);
			if (asyncManager.isConcurrentHandlingStarted()) {
				return null;
			}
			//最后返回ModelAndView给DispatcherServlet，如果方法或类上有ResponseBody
			//，则这里放回null。
			return getModelAndView(mavContainer, modelFactory, webRequest);
		}
		finally {
			webRequest.requestCompleted();
		}
	}
```

当返回的ModelAndView时，这里会看这个请求方法是否是异步的，如果是的话，直接结束，释放该请求线程，后台接着处理逻辑，当处理完后，再获取一个线程返回response。

```
if (asyncManager.isConcurrentHandlingStarted()) {
   return;
}
```

后面接着会执行拦截器中的proHanlder方法

```
//DispatcherServlet中的方法
mappedHandler.applyPostHandle(processedRequest, response, mv);
//HandlerExecutionChain的方法，调用拦截器的proHandler方法
void applyPostHandle(HttpServletRequest request, HttpServletResponse response, @Nullable ModelAndView mv)
			throws Exception {

		HandlerInterceptor[] interceptors = getInterceptors();
		if (!ObjectUtils.isEmpty(interceptors)) {
			for (int i = interceptors.length - 1; i >= 0; i--) {
				HandlerInterceptor interceptor = interceptors[i];
				interceptor.postHandle(request, response, this.handler, mv);
			}
		}
	}
```

最后根据ModelAndView解析出视图（也就是需要返回的页面）

```
//DispatcherServlet的processDispatchResult方法中调用的render
protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
		//*****这里获取il8n国际化的Locale****
		View view;
		String viewName = mv.getViewName();
		if (viewName != null) {
			// We need to resolve the view name.
			view = resolveViewName(viewName, mv.getModelInternal(), locale, request);
			if (view == null) {
				throw new ServletException("Could not resolve view with name '" + mv.getViewName() +
						"' in servlet with name '" + getServletName() + "'");
			}
		}
	   //********************	
	}
	//resolveViewName解析出View
	protected View resolveViewName(String viewName, @Nullable Map<String, Object> model,
			Locale locale, HttpServletRequest request) throws Exception {

		if (this.viewResolvers != null) {
			for (ViewResolver viewResolver : this.viewResolvers) {
				View view = viewResolver.resolveViewName(viewName, locale);
				if (view != null) {
					return view;
				}
			}
		}
		return null;
	}
```

整个过程就算是完成了。

从启动到结束整体流程大致为：

首先web.xml中的监听器会监听ServletContext初始化，当其初始化后，会调用该listener中的方法出初始化spring范围的WebApplicationContext（这里需要定义一个全局的<context-param>，不然的化初始化WebApplicationContext的时候会默认的去找WEB-INF下的application.xml）。

接着初始化定义在web.xml中的DispatcherServlet。初始化过程会将spring中上下文设置为自己的parent，

接着会加载配置的xml文件的类，配置到DispatcherServlet中（待看）。

一个请求的流程大致是：

当一个请求进来时，会调用FrameworkServlet的service方法，sevice方法根据get、post请求调用自己覆写的doGet，doPost请求（都是调用processRequest方法）。接着在doDispatch方法内。HandlerMapping（RequestMappingHandlerMapping）会根据url封装一个HandlerExecutionChain（里面包含HandlerMapping，和拦截器）返回给DispatcherServlet，并且会调用拦截器的preHandler，再根据HandlerMapping匹配到对应的HandlerAdapter，由HandlerAdapter（参数、类实列、方法等由HandlerMapping提供）调用反射执行controller方法然后返回ModelAndView给DispatcherServlet。再由DispatcherServlet调用ViewResolver去解析出View，然后再由Model渲染返回。

request域中处了我们自己定义的属性，还可以取得

```
org.springframework.web.context.request.async.WebAsyncManager.WEB_ASYNC_MANAGER
org.springframework.web.servlet.HandlerMapping.bestMatchingHandler
org.springframework.web.servlet.DispatcherServlet.CONTEXT
org.springframework.web.servlet.HandlerMapping.matrixVariables
org.springframework.web.servlet.DispatcherServlet.THEME_SOURCE
org.springframework.web.servlet.DispatcherServlet.LOCALE_RESOLVER
org.springframework.web.servlet.i18n.CookieLocaleResolver.LOCALE
org.springframework.web.servlet.HandlerMapping.bestMatchingPattern
org.springframework.web.servlet.DispatcherServlet.OUTPUT_FLASH_MAP
org.springframework.web.servlet.HandlerMapping.pathWithinHandlerMapping
org.springframework.web.servlet.DispatcherServlet.FLASH_MAP_MANAGER
org.springframework.web.servlet.HandlerMapping.uriTemplateVariables
org.springframework.web.servlet.DispatcherServlet.THEME_RESOLVER
org.springframework.core.convert.ConversionService
```

