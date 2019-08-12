# curl命令

```
curl http://localhost:8080/student/{studentId}/teacher #默认发送get请求
curl -i http://localhost:8080/student/{studentId}/teacher #返回带http头的，I是只有头
curl -L http******************  #跟随链接重定向
curl -A  ****** http******************  #来自定义用户代理
curl -H  ****** http******************  #传递特定的header
curl -c  ****** http******************  #保存cookie，-c 后面跟上要保存的文件名
curl -b  ****** http******************  #读取cookie
curl -X post http******************     #指定请求方式
curl -F "key=value" -F "filename=@file.tar.gz" http://localhost/upload #上传文件
```

viewResolver学习

实现不用@ResponseBody返回json

```
@RequestMapping("/beanNameUrl")
    public ModelAndView getBean(){
        ModelAndView modelAndView = new ModelAndView();
        modelAndView.setViewName("asnyController");
        return modelAndView;
    }


@RestController
public class AsnyController extends MappingJackson2JsonView {
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
//        return (()-> {  //第一种方式是返回callable
//            for (int i = 0; i < 20; i++) {
//                Thread.sleep(500);
//                System.out.println(i);
//            }
//            Model model = ModelUtil.getModel("异步处理线程:" + Thread.currentThread().getId() + "处理完成", 1, "chuli");
//            return model;
//        });
    }
    private void doSometh(DeferredResult<Model> deferredResult){
        new Thread(()->{
            try {
                for (int i=0;i<20;i++){
                    Thread.sleep(500);
                    System.out.println(i);
                }
                Model model = ModelUtil.getModel("异步处理线程:"+Thread.currentThread().getId()+"处理完成",1,"chuli");
                deferredResult.setResult(model);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
    }
}
<!--
 <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver" id="resolver">
        <property name="prefix" value="/WEB-INF/"/>
        <property name="suffix" value=".jsp"/>
    </bean>
    -->
/**
这里通过这个解析类来解析view，返回什么其实是根据View来决定的，这个resolver只是用来决定你要根据什么来解析，想上面配置的解析器，就是通过解析web-inf下jsp，而这个指定是通过解析spring中的applicationcontext中的bean来查早对应的解析器，以上面的controller为列子，我们可以通过继承MappingJackson2JsonView，这样我们设置view时设置为该controller名字，就可以查找到该view（注意，查找的bean必须是实现了View接口的，不然并不会返回），这里还可以根据继承不同的view实现类来返回不同的格式，上面是返回的json，MappingJackson2XmlView是返回xml形式的view。所以我们这里自己配置的viewResolvers是BeanNameViewResolver，注意这里需要配置order，因为该类的优先级较低，在解析时，它是排在最后一个的，不设置会直接被其他解析器解析的。
*/
 <bean id="viewResolvers是" class="org.springframework.web.servlet.view.BeanNameViewResolver">
        <property name="order" value="1"></property>
    </bean>
```

