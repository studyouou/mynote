# pom进行profile选择

在pom文件进行profile配置
首先指定profiles

```
 <profiles>
        <profile>
            <id>dev</id>
            <properties>
                <profile.active>dev</profile.active>
            </properties>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
        </profile>
        <profile>
            <id>test</id>
            <properties>
                <profile.active>test</profile.active>
            </properties>
        </profile>
    </profiles>

```

解释：id为指定的配置id，properties的作用是在application.yml中指定spring.profiles.active指定哪个profile用的(用法@profile.active@)。activation的activeByDefault是在打包或运行时未指定哪个profile时默认执行当前profile。还需要配置resources

```
<resources>
           <resource>
               <directory>src/main/resources</directory>
               <filtering>true</filtering>
           </resource>
       </resources>

```

在配置resource时，有includs和excludes其中includes是包含那些文件(springboot中会默认包含resources根目录下所有配置文件，待确认，至少不配置是会加载的)
但是配置了excludes的文件则不会包含经配置文件。filtering的作用就是可以在application.yml中使用占位符可以解析，就如@profile.active@。

```
spring:
  profiles:
    active: @profile.active@
#这里server.port不起作用，因为dev中优先级高一点
server:
  port: 8080

```

# 配置学习

1、据了解ConfigurationPropertiesBindingPostProcessor会对@Configurationproperties注解的bean进行属性赋值
可以通过import注解来加载bean
在import里有3种可以引用的类型，第一种是实现了ImportSelector的类（直接实现ImportSelector类才会被调用selectImports方法，实现DeferredImportSelector
的类不会被调用selectImports方法，会调用另一个方法如一个map中再处理），第二种是实习了ImportBeanDefinitionRegistrar，会调用registerBeanDefinitions
方法。第三种是不属于前两种的类。
实现ImportSelector的类调用selectImports会返回需要注册类的name，然后注册到容器中。如

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
@Import(MyImportSelector.class)
public @interface EnableLogInfo {
    String name() default "onlyCar";
}

public class MyImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        if (annotationMetadata.getAnnotationAttributes(EnableLogInfo.class.getName()).toString().contains("onlyCar")){
            return new String[]{Car.class.getName()};
        }
        return new String[]{Person.class.getName(),Car.class.getName()};
    }
}
@EnableLogInfo
public class MyImportLoad {
}

```

只要能扫描到EnableLogInfo，就可以注入了，可以放在启动类上，也可以写一个spring.factories(springboot启动默认扫描里面的类)
在resources下建立一个META-INF，在里面创建spring.factories里面添加
org.ougen.springbootdemo.defaulload.MyImportLoad
MyImportLoad类只是一个标识性类。

通过发现@EnableConfigurationProperties上有@Import({EnableConfigurationPropertiesImportSelector.class})注解，而在默认启动类
ConfigurationPropertiesAutoConfiguration类上有@EnableConfigurationProperties，所以EnableConfigurationPropertiesImportSelector
会在ConfigurationClassParser的processImports方法执行selectImports，返回ConfigurationPropertiesBeanRegistrar类名和
ConfigurationPropertiesBindingPostProcessorRegistrar类名并加载进容器，并且他们都是ImportBeanDefinitionRegistrar的实现类，所以
会执行registerBeanDefinitions方法，前一个是加载系统配有@ConfigurationProperties的类，并且将数据封装进去，后一个加载
ConfigurationPropertiesBindingPostProcessor类和一个metafactory，而ConfigurationPropertiesBindingPostProcessor就是解析赋值
带有ConfigurationProperties注解的类。

基于同样的原理，我们也可以通过实现ImportBeanDefinitionRegistrar来加载自己的类，如：
MyImportSelector 类（实现细节可以变化）

```
public class MyImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        String s = annotationMetadata.getAnnotationAttributes(EnableLogInfo.class.getName()).toString();
        if (s.contains("onlyCar")){
            return new String[]{Car.class.getName()};
        }
        return new String[]{Person.class.getName(),Car.class.getName()};
    }
}
RegiestImport 注解
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(MyImportRegiest.class)
public @interface RegiestImport {
    String value() default "onlyCar";
}

```

MyImportLoad 类（该类和上面一样启动加载，为了就是加载解析@RegiestImport注解）

```
@EnableLogInfo
@RegiestImport
public class MyImportLoad {
}
```