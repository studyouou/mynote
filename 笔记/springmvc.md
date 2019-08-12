1��srpingmvc dispacher��ʼ����ÿ��dispatcherServlet �ڳ�ʼ��ʱ����ӵ��һ��WebApplicationContext��
��ʼ��ʱ�����Ȼᴴ��һ��WebApplicationContext��Ȼ���ٴ���dispatcherServlet������WebApplicationContext��Ϊ�������
����dispatcherServlet�������dispatcherServlet�ĳ�ʼcontext�ࡣ
����������tomcatʱ�����Զ�ɨ�����Ŀ���Ƿ��� WebApplicationInitializer��ʵ���࣬����У����Զ�ȥ���ز��ҳ�ʼ��dispatcherServlet��
�����web.xml��������dispatcherServlet���Ḳ�ǵ��Լ�ʵ�ֵ�dispatcherServlet��

2��springmvc������HandlerInterceptor
�Զ�����������Ҫʵ��HandlerInterceptor��������Ҫʵ��preHandle��postHandle��afterCompletion����������
preHandle����ִ������֮ǰִ�У�����ֵΪboolean��������trueʱ����������ִ�У���Ϊfalseʱ����ֱ�Ӳ���ִ��ʵ�巽��������ķ�����
postHandle������ʵ�崦�����֮��ִ�еķ�������ʵ��ִ�г�������@ResponseBodyʱ���Ϳ��ܳ����ȷ���ֵ�����󣨾��Ǵ�ϵ㲢����
��ֹ����ֵ���أ�
afterCompletion��������������ִ�еķ�����
ʵ�������Ҫ���ø�������
<mvc:interceptors>
        <mvc:interceptor>
            <mvc:mapping path="/**"></mvc:mapping>
            <!--<mvc:exclude-mapping path="***"/>-->
            <bean id="intercept" class="****"></bean>
        </mvc:interceptor>
    </mvc:interceptors>
����mvc:mapping������Ҫ���ص�����·����<mvc:exclude-mapping>�Ƕ��ⲻ�����ص�·����bean�����Լ��������������
������֤����hibernet-validate�İ汾����ʱ���ᱨ��Ŀǰ��֪������취��

3��springmvc�ϴ��ļ�
��Ҫpom���io����
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
Ȼ����Ҫ����ļ�������
 <bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
        <property name="defaultEncoding" value="utf-8"></property>
        <property name="maxUploadSizePerFile" value="1024"></property>
        <property name="maxUploadSize" value="10240"></property>
    </bean>
���д����
public String upload(@RequestParam("file") MultipartFile file){
         ......
}
��������List<MultipartFile>���Խ����ϴ�����ļ���

4���������������
//  GET /start/2;q=32/xx;p=23
@RequestMapping("/start/{id}/{name}")
public String start(HttpServletRequest request,@MatrixVariable(name="q" ,pathVal="id") int q,@MatrixVariable(name="p",pathVal="name")int p){
        ****
}
����name��������url��Ӧ��q��pathVal��Ӧ������Ӧ���ơ�

5ע��
@PathVariable:�����������ö�Ӧ�����Ի�ȡֵ��Ӧ����restful�ӿ�
@RequestParam���������
@RequestHeader��������������ڻ�ȡ����ͷֵ��
@CookieValue�����ڻ�ȡ����Я����cookie��������Ļ��������ò���required=false
@ModelAttribute��ʹ���ڷ����������������ݰ󶨡�������ݰ󶨴��󣬻��׳�BindException�������Ҫ��������飬�������Խ���
BindingResult�����ռ����󡣻������Լ����ڲ����ݣ�����Ҫ�ⲿ���磺
 @ModelAttribute
    public User sets(){
        User user =  new User();
        user.setName("xxxxx");
        user.setAge(3333);
        return user;
    }

 public String bindtest(@ModelAttribute(value = "user",binding = false)User user, BindingResult bindingResult){

}
�������ǰ��Լ��ڲ���User

@InitBinder
������ڸ�controllerʹ��
1��������������������ѯ���ݣ��󶨵�ģ�Ͷ���
2���������ַ���������ֵ���������������·�����������⣬cookie�ȣ�ת��ΪĿ�����͵Ŀ���������������
3��String�ڳ���HTML��ʱ��ģ�Ͷ���ֵ��ʽ��Ϊֵ��
���һ������ת����
@InitBinder
 public void format(WebDataBinder webDataBinder){
        SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd");
        simpleDateFormat.setLenient(false);
        webDataBinder.registerCustomEditor(Date.class,new CustomDateEditor(simpleDateFormat,false));
    }
�������ControllerAdviceʹ��

6���첽����
springmvc�������첽�����springboot���е㲻ͬ
1��springmvc�첽�ǣ����������ʱ��servlet�̳߳ػ�����һ���̴߳������������������첽�Ļ��������������������߳�ʱ��
�̻߳��˳������ɺ�̨�¿�����ȥ������servlet�߳̽���ȥ���ܴ�����������response���𣬵ȴ�������������ɺ��ٵ���һ��
servlet�߳�ȥ����response��������mvc��handlerinterceptor��֤������
```
@RequestMapping("/asyn")
    public DeferredResult<Model> asyncTask(){
        DeferredResult<Model> deferredResult = new DeferredResult<Model>(15000L);
        System.out.println("�����߳�:"+Thread.currentThread().getId());
        doSometh(deferredResult);
        deferredResult.onTimeout(new Runnable() {
            @Override
            public void run() {
                System.out.println("��ʱִ����,ִ���߳���:"+Thread.currentThread().getId());
            }
        });
        return deferredResult;
```
```
return (()-> {  //��һ�ַ�ʽ�Ƿ���callable
            for (int i = 0; i < 20; i++) {
                Thread.sleep(500);
                System.out.println(i);
            }
            Model model = ModelUtil.getModel("�첽�����߳�:" + Thread.currentThread().getId() + "�������", 1, "chuli");
            return model;
        });
```
springboot���������첽�ǣ����û�����󣬷�������ֱ�ӿ������̣߳�����ֱ�ӷŻؽ�����û�����̨���Ŵ����ҵ��
ֱ������Ҫʵ���첽Controller�ϼ�@EnableAsyncע�⣬����Ҫͬ���ķ����ϼ�@Asyncע�⣬��������
@RestController
@EnableAsync
public class CommonController {
@RequestMapping("/anysAdd")
    @ResponseBody
    public Model anysAdd(){
        logger.info("��ǰ�����߳���:"+Thread.currentThread().getId());
        aysnService.asyncAdd();
        return ModelUtil.getModel("�첽�̴߳���",1,Thread.currentThread().getId());
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
            System.out.println("�߳�"+Thread.currentThread().getId()+"����Ϊ������");
        }
    }
}



# ��������

1. �������������web.xml�����õ�ContextLoaderListener��������ServletContext��ʼ���������Դ�ServletContextEvent�¼��л�ȡServletContext�����ŵ���initWebApplicationContext

```
initWebApplicationContext(event.getServletContext());
```

������������ڣ��ᴴ��һ��WebApplicationContext����ȥ��ʼ����

```
if (this.context == null) {
				//�жϵ�ǰcontext�Ƿ�Ϊ�գ�Ϊ�մ���һ��ApplicationContext
				this.context = createWebApplicationContext(servletContext);
			}
			//�жϴ�����context�Ƿ�ConfigurableWebApplicationContext��ʵ��
			if (this.context instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
				//�ж�cwac�Ƿ��Ѿ�ˢ��
				if (!cwac.isActive()) {
					// The context has not yet been refreshed -> provide services such as
					// setting the parent context, setting the application context id, etc
					if (cwac.getParent() == null) {
						// The context instance was injected without an explicit parent ->
						// determine parent for root web application context, if any.
						ApplicationContext parent = loadParentContext(servletContext);
						cwac.setParent(parent);
					}
					//ˢ�������ģ���ʼ�����������ģ�
					configureAndRefreshWebApplicationContext(cwac, servletContext);
				}
			}
```

configureAndRefreshWebApplicationContext�Ĵ������£�

```
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
		//�ж϶����ַ�Ƿ���ͬ
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
		//��web.xml�л�ȡcontextConfigLocation������ֵ�������ļ�λ�ã�
		String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
		if (configLocationParam != null) {
			//��Ϊnull������Ϊ��ApplicationContext�����õ�ַ
			wac.setConfigLocation(configLocationParam);
		}

		// The wac environment's #initPropertySources will be called in any case when the context
		// is refreshed; do it eagerly here to ensure servlet property sources are in place for
		// use in any post-processing or initialization that occurs below prior to #refresh
		ConfigurableEnvironment env = wac.getEnvironment();
		if (env instanceof ConfigurableWebEnvironment) {
			((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
		}
		//��ȡ�Զ����ApplicationContextInitializer
		customizeContext(sc, wac);
		wac.refresh();
	}
```

���ˢ�������ģ���������������ӵ�ServletContext�У���WebApplicationContext��Ϊspring�������ġ�

��ΪDispatcherServlet�Ǽ̳���FrameworkServlet��HttpServletBeanHttpServletGenericServletServlet��

�ڵ�һ�γ�ʼ��Sevlet��ʱ�����������init������

���Ȼ��ȡweb.xml��servlet��init-param��ֵ��Ȼ�����FrameworkServletʵ�ֵ�initServletBean��������Ҫ�Ĵ������£�

```
this.webApplicationContext = initWebApplicationContext();
initFrameworkServlet();
```

�����δ���ֱ�����

```
protected WebApplicationContext initWebApplicationContext() {
//��ServletContext��ȡ��rootContext��Ҳ���������ʼ������ӵ�WebApplicationContext
WebApplicationContext rootContext =
		WebApplicationContextUtils.getWebApplicationContext(getServletContext());
		WebApplicationContext wac = null;
		//�ж�webApplicationContext�����Ƿ�Ϊ�գ���Ϊ�վ�˵���Ѿ���ʼ����һ��WebApplicationContext��
		if (this.webApplicationContext != null) {
			// A context instance was injected at construction time -> use it
			wac = this.webApplicationContext;
			if (wac instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
                //����Ѿ�ˢ�¹��ˣ�����Ҫ��ˢ��
				if (!cwac.isActive()) {
					// The context has not yet been refreshed -> provide services such as
					// setting the parent context, setting the application context id, etc
					//���δˢ�£����Sping�����ApplicationContext����Ϊ������
					if (cwac.getParent() == null) {
						// The context instance was injected without an explicit parent -> set
						// the root application context (if any; may be null) as the parent
						cwac.setParent(rootContext);
					}
					//ˢ��
					configureAndRefreshWebApplicationContext(cwac);
				}
			}
		}
		if (wac == null) {
			// No context instance was injected at construction time -> see if one
			// has been registered in the servlet context. If one exists, it is assumed
			// that the parent context (if any) has already been set and that the
			// user has performed any initialization such as setting the context id
			//�鿴contextAttribute�Ƿ񱻸�ֵ
			wac = findWebApplicationContext();
		}
		//webApplicationContext���Բ�δ��ֵʱ
		if (wac == null) {
			// No context instance is defined for this servlet -> create a local one
			//����һ���µ�WebApplicationContext����ָ��parent����ΪrootContext
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

����DispatcherServlet�ڲ������Լ���WebApplicationContext������

//�����ļ�����������

DispatcherServlet�Ľ�������

����FrameworkServlet�Ǹ�д�˸����service�������������������ʱ�򣬻�ͨ��service��������processRequest��processRequest���ȡ���󣬲���requestAttributes��LocaleContext�洢��ThreadLocal�����л��LocaleContext�ķ���buildLocaleContext��DispatcherServlet��д��ͨ��localeResolver������ִ��resolveLocaleContext��������CookieLocaleResolver���ԣ�������ʱ��cookieName��Ҳ��������ͨ��cookie����ȡ��Ӧ�Ĺ��ʻ�����en_US�ȣ���������������û������cookie��ֻ��һֱ��ȡcookie�����Ե��л�����ʱ��һֱ��ʧ�ܣ���ʱ����Ҫ���������ˣ����ݺ�cookie��һ���Ĳ�����ȡ��ǰ���ԣ�����ͨ��LocaleResolver��setLocale������Locale���õ�request�ڣ�cookie��Ҫ���±�����ͻ����У�Locale��timeZone���Ǵ��request��LOCALE_REQUEST_ATTRIBUTE_NAME��TIME_ZONE_REQUEST_ATTRIBUTE_NAME�����������У���springmvcʵ��������һ��������������session����cookieʵ�֣�LocaleChangeInterceptor�������LocaleResolver��setLocale����Locale��

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
//LocaleChangeInterceptor��preHandle�����е��á�
localeResolver.setLocale(request, response, parseLocaleValue(newLocale));
```

����������doService����������Ϊÿ��request����������漸����������Ӧ��

```
//���������
request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
//���localeResolver
request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
//���themeResolver����������
request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
//����������
request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());
```

�ٽ��ŵ�doDispatch�����Ȼ��жϸ������Ƿ�Ϊ�ļ��ϴ�������request�����Ƿ����multipart/form-dataͷ��Ȼ�����Ȼ����request��url��������uri��ͨ����uri�ҵ���Ӧ�Ŀ������ķ�������װ��HandlerMethod�����ڲ���װ��controller����Ӧ�ķ�������Ҫ�Ĳ������Լ�������ע��ȣ��С����HandlerMethod����������װ��һ��HandlerExecutionChain���ظ�DispatcherServlet��DispatcherServletͨ��HandlerMethod����һ��HandlerAdapter��Ȼ��ͨ��HandlerExecutionChain�е����������������ش�������ֱ���ǵ�����������preHandler�������������false��ֱ�ӵ�����������afterCompletion���������ҷ���false��DispatcherServlet����Ȼ����HandlerAdapterȥִ�з�������������У�����ȥ�ҵ�Controller�е�InitBind�󶨵�ת�����ȣ���󶨲������Լ��Ƿ��첽�ȣ��е㸴�ӣ�û��ô���Ķ������ݷ�������ô�жϣ���������԰��ִ꣬��ServletInvocableHandlerMethod��invokeAndHandle������invokeAndHandleͨ���������controller�ķ����������������

```
//DispatcherServlet��HandlerAdapter����������ִ��handle����ײ�ͨ���������controller�еķ�����
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
//���յ���invokeHandlerMethod����
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
		//�������http�л������ԡ�
		ServletWebRequest webRequest = new ServletWebRequest(request, response);
		try {
			//����@InitBinderע��ע���ת��������������֤��
			WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
			ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
			
			//ִ�����壬��������invokeAndHandle����ִ�з��䷽������controller����
			ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
			if (this.argumentResolvers != null) {
				//��������ˣ������ò���������
				invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
			}
			if (this.returnValueHandlers != null) {
				//��̫֪����������
				invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
			}
			//����binderFactory����
			invocableMethod.setDataBinderFactory(binderFactory);
			invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);
			
			//���²⣬����������洢modelģ�ͺ�view�ģ�
			ModelAndViewContainer mavContainer = new ModelAndViewContainer();
			mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
			//��session�Լ����ֵ�������������
			modelFactory.initModel(webRequest, mavContainer, invocableMethod);
			mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);
			//�����漸�ж��ǹ��ڲ����ģ���������
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
			//ͨ���������ִ�е��÷��������ˣ�ע�⵱���������ϱ�ע@ResponseBody
			//ʱ����ֱ�ӷŻ�ֵ��������������proHandler��afterCompletion���޷����ص�
			invocableMethod.invokeAndHandle(webRequest, mavContainer);
			if (asyncManager.isConcurrentHandlingStarted()) {
				return null;
			}
			//��󷵻�ModelAndView��DispatcherServlet�����������������ResponseBody
			//��������Ż�null��
			return getModelAndView(mavContainer, modelFactory, webRequest);
		}
		finally {
			webRequest.requestCompleted();
		}
	}
```

�����ص�ModelAndViewʱ������ῴ������󷽷��Ƿ����첽�ģ�����ǵĻ���ֱ�ӽ������ͷŸ������̣߳���̨���Ŵ����߼�������������ٻ�ȡһ���̷߳���response��

```
if (asyncManager.isConcurrentHandlingStarted()) {
   return;
}
```

������Ż�ִ���������е�proHanlder����

```
//DispatcherServlet�еķ���
mappedHandler.applyPostHandle(processedRequest, response, mv);
//HandlerExecutionChain�ķ�����������������proHandler����
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

������ModelAndView��������ͼ��Ҳ������Ҫ���ص�ҳ�棩

```
//DispatcherServlet��processDispatchResult�����е��õ�render
protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
		//*****�����ȡil8n���ʻ���Locale****
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
	//resolveViewName������View
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

�������̾���������ˡ�

�������������������̴���Ϊ��

����web.xml�еļ����������ServletContext��ʼ���������ʼ���󣬻���ø�listener�еķ�������ʼ��spring��Χ��WebApplicationContext��������Ҫ����һ��ȫ�ֵ�<context-param>����Ȼ�Ļ���ʼ��WebApplicationContext��ʱ���Ĭ�ϵ�ȥ��WEB-INF�µ�application.xml����

���ų�ʼ��������web.xml�е�DispatcherServlet����ʼ�����̻Ὣspring������������Ϊ�Լ���parent��

���Ż�������õ�xml�ļ����࣬���õ�DispatcherServlet�У���������

һ����������̴����ǣ�

��һ���������ʱ�������FrameworkServlet��service������sevice��������get��post��������Լ���д��doGet��doPost���󣨶��ǵ���processRequest��������������doDispatch�����ڡ�HandlerMapping��RequestMappingHandlerMapping�������url��װһ��HandlerExecutionChain���������HandlerMapping���������������ظ�DispatcherServlet�����һ������������preHandler���ٸ���HandlerMappingƥ�䵽��Ӧ��HandlerAdapter����HandlerAdapter����������ʵ�С���������HandlerMapping�ṩ�����÷���ִ��controller����Ȼ�󷵻�ModelAndView��DispatcherServlet������DispatcherServlet����ViewResolverȥ������View��Ȼ������Model��Ⱦ���ء�

request���д��������Լ���������ԣ�������ȡ��

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

