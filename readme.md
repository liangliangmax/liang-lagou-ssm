

# 自定义ssm框架总结

### 目录

1. 结构说明
2. 整合过程
3. 启动运行



### 正文

1. 结构说明

   本项目是在前两个项目（自定义mybatis和自定义spring容器）的基础上进行整合。

首先目录结构分为三大部分

- com.liang.mybatis.core		之前写的mybatis的核心功能，解决了持久层的初始化
- com.liang.spring.core           之前写的spring的核心容器，解决了bean依赖注入的功能
- com.liang.spring.webmvc    这次开发的mvc的核心组件
- com.liang.ssm_demo            用于测试的项目，模拟正式环境下用户的使用



2. 整合过程

   - 首先是spring和mybatis的整合

     

     mybatis启动时候会自己解析xml生成MappedStatement，这个是最终运行的sql对象，这里的关键就是如何将这些代理对象添加为spring的bean对象。

     

     由于这里是一个简版的spring容器，没有使用factoryBean来生成bean，（ 正常的mybatis和spring整合参考https://www.cnblogs.com/lanqingzhou/p/13592232.html这个文章）,  这个简单的容器缺失其他功能，所以需要另外想办法。

     

     既然mybatis的功能是完整的，意味着SqlSessionFactoryBuilder是好用的，也能创建出sqlSessionFactory，里面也有statement对象，那么只需要把SqlSessionFactory当做一个bean托管到spring容器进行生成，这样在spring启动时候，就会触发mybatis核心功能的加载，进而触发解析xml等动作。于是在ssm_demo项目中创建了自定义的bean。

     ​	
   
     ```java
     @Configuration
     public class SqlSessionFactoryConfig {
         @Value("${mybatis.mapper.configPath}")
         private String mapperScanPackage;
     
         @Autowired
         private DataSource dataSource;
         
         @Bean
         public SqlSessionFactory sqlSessionFactory() throws Exception {
             if(mapperScanPackage.startsWith("classpath:")){
                 mapperScanPackage = mapperScanPackage.replace("classpath:","");
             }
             InputStream resourceAsStream = Resources.getResourceAsStream(mapperScanPackage);
             SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
             sqlSessionFactoryBuilder.setDataSource(dataSource);
             SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBuilder.build(resourceAsStream);
             return sqlSessionFactory;
         }
    }
     ```



​				这样生成完成之后spring容器里面就会有sqlSessionFactory对象。



​				那怎样才能扫描到接口呢，这里只能是在spring容器初始化的时候最后生成指定路径下的mapper接口的代理类，因为如果一开始就生成代理类的话由于sqlSessionFactory还没创建出来，代理类中需要sqlSessionFactory，所以没办法一开始就生成接口代理类。

​     			

​				等到之前的bean都生成完毕，最后用cglib生成指定接口的代理类，填加到spring容器中。


```
     //生成mapper代理对象
     Set<Class<?>> classes = getClasses();
     
     for (Class<?> aClass : classes) {
     
         //如果是mapper的类，是不能被实例化出来的，因为没有实现类，需要直接创建代理类
         if(aClass.isAnnotationPresent(Mapper.class)){
             doCreateMapperProxy(GenerateBeanNameUtil.generateBeanName(aClass),aClass);
         }
    }
```

​				

​				代理对象生成的过程是

     public class MapperProxyFactory {
     
         private SqlSessionFactory sqlSessionFactory;
     
         public void setSqlSessionFactory(SqlSessionFactory sqlSessionFactory) {
             this.sqlSessionFactory = sqlSessionFactory;
         }
     
         /**
          * 使用cglib动态代理生成代理对象
          * @param obj 委托对象
          * @return
          */
         public Object getCglibProxy(Class obj) {
             return  Enhancer.create(obj, new MethodInterceptor() {
                 @Override
                 public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                     Object result = null;
                     Object mapper = sqlSessionFactory.openSession().getMapper(obj);
                     result = method.invoke(mapper,objects);
                     return result;
                 }
             });
         }
    }


​      			当service调用mapper时候，会通过sqlSessionFactory.openSession().getMapper(obj) 来获取具体的代理对象，这里的obj就是传入的接口类型，mybatis会根据接口的class去搜索对应的mappedStatement对象，然后调用mybatis的方法来执行sql。



   - 然后是spring和springmvc的整合

        整合springmvc最核心的就是dispatchServlet，需要在tomcat启动的时候就要加载这个类。整个spring容器的初始化也是有这个类调用的。

        

        第一步就是配置web.xml,使这个servlet在tomcat启动时候就初始化。

   ```xml
   <!DOCTYPE web-app PUBLIC
           "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
           "http://java.sun.com/dtd/web-app_2_3.dtd" >
   <web-app>
       <display-name>Archetype Created Web Application</display-name>
       <servlet>
           <servlet-name>liangmvc</servlet-name>
           <servlet-class>com.liang.spring.webmvc.servlet.DispatcherServlet</servlet-class>
           <init-param>
               <param-name>contextConfigLocation</param-name>
               <param-value>classpath:springmvc.properties</param-value>
           </init-param>
           <load-on-startup>1</load-on-startup>
       </servlet>
       <servlet-mapping>
           <servlet-name>liangmvc</servlet-name>
           <url-pattern>/*</url-pattern>
       </servlet-mapping>
</web-app>
   ```

   ​				

   ​		在DispatcherServlet中初始化调用init方法，里面会执行相关代码

   ```java
   @Override
   public void init(ServletConfig config) throws ServletException {
   
       //加载文件
       String contextConfigLocation = config.getInitParameter("contextConfigLocation");
   
       //初始化容器
       initContext(contextConfigLocation);
   
       //构造一个HandlerMapping处理器映射器，将配置好的url和Method建立映射关系
       initHandlerMapping();
   
       //初始化interceptor
       initInterceptor();
   
   }
   ```

​						

​				初始化容器就是直接调用spring-core里面的方法，传入扫描的包进行容器加载

```java
//初始化容器
private void initContext(String contextConfigLocation) {

    try {
        if(StringUtils.isNotBlank(contextConfigLocation)){
            if(contextConfigLocation.startsWith("classpath:")){
                contextConfigLocation = contextConfigLocation.replaceAll("classpath:","");
            }
        }

        InputStream resourceAsStream = Resources.getResourceAsStream(contextConfigLocation);
        Properties properties = new Properties();
        properties.load(resourceAsStream);
        new AnnotationApplicationContext(properties.getProperty("scanPackage"));
    }catch (Exception e){
        e.printStackTrace();
    }

}
```



​				等容器初始化完成之后，会将所有的@Controller的bean都取出来，生成handler，每一个handler里面包含一个路径，一个要执行的方法和执行的对象，参数列表，相当于用url进行handler匹配，匹配上了就处理

```
private void initHandlerMapping() {

    AnnotationApplicationContext applicationContext = (AnnotationApplicationContext) IocUtil.getApplicationContext();

    for (Map.Entry<String, Object> stringObjectEntry : applicationContext.getBeans().entrySet()) {

        Class<?> aClass = stringObjectEntry.getValue().getClass();

        if(!aClass.isAnnotationPresent(Controller.class)){
            continue;
        }

        String baseUrl = "";

        if(aClass.isAnnotationPresent(RequestMapping.class)){
            RequestMapping annotation = aClass.getAnnotation(RequestMapping.class);
            baseUrl = annotation.value();
        }

        Method[] declaredMethods = aClass.getDeclaredMethods();

        for (Method declaredMethod : declaredMethods) {
            if(!declaredMethod.isAnnotationPresent(RequestMapping.class)){
                continue;
            }

            RequestMapping annotation = declaredMethod.getAnnotation(RequestMapping.class);
            String subUrl = annotation.value();
            String fullUrl = baseUrl + subUrl;

            //把method所有信息封装为handler
            Handler handler = new Handler(stringObjectEntry.getValue(),declaredMethod, Pattern.compile(fullUrl));

            //计算方法参数位置
            Parameter[] parameters = declaredMethod.getParameters();
            for (int i = 0; i < parameters.length; i++) {

                Parameter parameter = parameters[i];

                if(parameter.getType() == HttpServletRequest.class || parameter.getType() == HttpServletResponse.class){
                    //如果是request和response对象，那么参数名称写httpServletRequest和httpServletResponse
                    handler.getParamIndexMapping().put(parameter.getType().getSimpleName(),i);
                }else {
                    handler.getParamIndexMapping().put(parameter.getName(),i);
                }
            }

            handlerMapping.add(handler);
        }

    }

}
```

​				最后是加载interceptor，可以在方法执行之前进行拦截动作。由容器进行初始化，在initInterceptor方法中，将所有的实现了IHandlerInterceptor接口的bean拿出来，添加到interceptorChain里面，

```
//启动过程中，将所有IHandlerInterceptor的子类都扫描到容器中，然后从容器中将处理器都添加到处理器集合中
private void initInterceptor() {
    AnnotationApplicationContext applicationContext = (AnnotationApplicationContext) IocUtil.getApplicationContext();
    for (Map.Entry<String, Object> beanMap : applicationContext.getBeans().entrySet()) {
        Object bean = beanMap.getValue();
        if(bean instanceof IHandlerInterceptor){
            handlerInterceptors.add((IHandlerInterceptor) bean);
        }
    }
}
```



​			在进行真正的controller调用之前，先进行拦截器链的调用

```java
try {
    //在方法调用前，执行拦截操作
    if(!handlerInterceptors.isEmpty()){
        for (IHandlerInterceptor handlerInterceptor : handlerInterceptors) {
            if(!handlerInterceptor.needFilter(handler)){
                continue;
            }
            boolean b = handlerInterceptor.preHandle(handler, req, resp);
            if(!b){
                return;
            }
        }
    }

    handler.getMethod().invoke(handler.getController(),paraValues);
} catch (IllegalAccessException e) {
    e.printStackTrace();
} catch (InvocationTargetException e) {
    e.printStackTrace();
}
```



​				如果通过了拦截器之后才会真正的调用方法，否则会被拦截。

​						由于作业里面需要拦截@Security注解，所以写了一个SecurityInterceptor，这样在拦截器里面就能拦截到请求参数，和其他功能解耦。在com.liang.ssm_demo.interceptor.SecurityInterceptor




3. 启动运行

   - 先运行account.sql创建数据库，表及数据

   - 修改resource下面的jdbc.properties，将数据库修改为正确地址

   - 运行tomcat的maven插件，启动tomcat

   - 打开浏览器输入http://localhost:8080/account/queryAll?username=zhangsan 即可看到数据库里的数据。

     ```
     [Account{id='111', name='zhangsan', account='11110101010'}]
     ```

   - 如果没有username或者username不是zhangsan，则页面会出现 401 Authentication failed

   

   

