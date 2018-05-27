[TOC]



前面有四个小节关于Spring的基础知识，分别是：

* IoC容器
* JavaConfig
* 事件监听
* SpringFactoriesLoader详解



> Spring Boot 框架的**初衷**：**快速地启动Spring应用**
>
>
> Spring Boot应用本质上就是一个 **基于Spring框架** 的应用, 它是对 Spring" **约定优先于配置** "理念的最佳实践产物
>



Spring Boot最重要的**四大核心**：
* 自动配置
* 起步依赖
* Actuator
* 命令行界面



## 探索Spring IoC容器
### Spring IoC容器

#### BeanDefinition、BeanDefinitionRegistry & BeanFactory 概念和关系

<u>BeanDefinition</u>：Bean的具体定义

<u>BeanDefinitionRegistry和BeanFactory</u>存储了BeanDefinition：

* BeanDefinitionRegistry抽象出bean的**注册**逻辑，
* BeanFactory                  抽象出bean的**管理**逻辑，

而各个BeanFactory的实现类就具体承担了bean的注册以及管理工作。它们之间的关系就如下图：

![image](https://raw.githubusercontent.com/zy475459736/markdown-pics/master/SpringBoot_List/spring_1.png)

DefaultListableBeanFactory作为一个比较通用的BeanFactory实现，它同时也实现了BeanDefinitionRegistry接口，因此它就承担了Bean的注册管理工作。



#### 示例代码

```java
//这段代码仅为了说明BeanFactory底层的大致工作流程，实际情况会更加复杂，比如bean之间的依赖关系可能定义在外部配置文件(XML/Properties)中、也可能是注解方式

// 默认容器实现
DefaultListableBeanFactory beanRegistry = new DefaultListableBeanFactory();
// 根据业务对象构造相应的BeanDefinition
AbstractBeanDefinition definition = new RootBeanDefinition(Business.class,true);
// 将bean定义注册到容器中
beanRegistry.registerBeanDefinition("beanName",definition);
// 如果有多个bean，还可以指定各个bean之间的依赖关系
// ........

// 然后可以从容器中获取这个bean的实例
// 注意：这里的beanRegistry其实实现了BeanFactory接口，所以可以强转，
// 单纯的BeanDefinitionRegistry是无法强制转换到BeanFactory类型的
BeanFactory container = (BeanFactory)beanRegistry;
Business business = (Business)container.getBean("beanName");
```


#### IoC容器二阶段工作流程

Spring IoC容器的整个工作流程大致可以分为两个阶段。（类似于初始化和实例化）

##### 容器启动阶段----> 加载Configuration MetaData

BeanDefinitionReader会对加载的Configuration MetaData进行解析和分析，并将分析后的信息组装为相应的BeanDefinition，最后把这些保存了bean定义的BeanDefinition，注册到相应的BeanDefinitionRegistry，这样容器的启动工作就完成了。

这个阶段主要完成一些准备性工作，更侧重于**<u>bean对象管理信息**的收集</u>。

当然一些验证性或者辅助性的工作也在这一阶段完成。

下面的代码将模拟BeanFactory如何从xml配置文件中加载bean的定义以及依赖关系：
```java
// 通常为BeanDefinitionRegistry的实现类，这里以DeFaultListabeBeanFactory为例
BeanDefinitionRegistry beanRegistry = new DefaultListableBeanFactory(); 
// XmlBeanDefinitionReader实现了BeanDefinitionReader接口，用于解析XML文件
XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReaderImpl(beanRegistry);
// 加载配置文件
beanDefinitionReader.loadBeanDefinitions("classpath:spring-bean.xml");

// 从容器中获取bean实例
BeanFactory container = (BeanFactory)beanRegistry;
Business business = (Business)container.getBean("beanName");
```
##### Bean的实例化阶段

经过第一阶段，所有bean定义都通过BeanDefinition的方式注册到BeanDefinitionRegistry中。

当

1. 某个请求通过容器的getBean方法请求某个对象，

2. 或者因为依赖关系容器需要隐式的调用getBean，

   就会触发**第二阶段的活动**：

   容器会首先检查所请求的对象之前是否已经实例化完成。如果没有，则会根据注册的BeanDefinition所提供的信息实例化被请求对象，并为其注入依赖。当该对象装配完毕后，容器会立即将其返回给请求方法使用。



**BeanFactory**只是Spring IoC容器的一种实现，如果没有特殊指定，它采用采用**延迟初始化**策略：只有当访问容器中的某个对象时，才对该对象进行初始化和依赖注入操作。

另外一种使用更多的容器：**ApplicationContext**，它构建在BeanFactory之上，属于更高级的容器，除了具有BeanFactory的所有能力之外，还提供对事件监听机制以及国际化的支持等。它管理的bean，**在容器启动时全部完成初始化和依赖注入操作**。




### Spring容器扩展机制

IoC容器负责管理容器中所有bean的生命周期，而在bean生命周期的不同阶段，Spring提供了不同的扩展点来改变bean的命运。

* BeanFactoryPostProcessor

  在容器实例化相应对象之前，对beandefinition做一些额外的操作

* BeanPostProcessor

  在容器实例化相应对象之后，对BeanDefinition做一些额外的操作

简单的对比，**BeanFactoryPostProcessor处理bean的定义，而BeanPostProcessor则处理bean完成实例化后的对象**。

Bean的整个生命周期：
![image](https://raw.githubusercontent.com/zy475459736/markdown-pics/master/SpringBoot_List/spring_2.png)

#### BeanFactoryPostProcessor

在容器的启动阶段，**BeanFactoryPostProcessor**允许我们在**容器实例化相应对象**之前，对注册到容器的BeanDefinition所保存的信息做一些额外的操作，比如修改bean定义的某些属性或者增加其他信息等，这需要的步骤如下：

1、需要实现org.springframework.beans.factory.config.BeanFactoryPostProcessor接口，
2、需要实现org.springframework.core.Ordered接口，以保证BeanFactoryPostProcessor按照顺序执行

##### BeanFactoryPostProcessor示例----PropertyPlaceholderConfigurer

Spring提供了为数不多的BeanFactoryPostProcessor实现，我们以**PropertyPlaceholderConfigurer**来说明其大致的工作流程。

在Spring项目的XML配置文件中，经常可以看到许多配置项的值使用占位符，而将占位符所代表的值单独配置到独立的properties文件，这样可以将散落在不同XML文件中的配置集中管理，而且也方便运维根据不同的环境进行配置不同的值。这个非常实用的功能就是由PropertyPlaceholderConfigurer负责实现的。

根据前文，当BeanFactory在第一阶段加载完所有配置信息时，BeanFactory中保存的对象的属性还是以占位符方式存在的，比如${jdbc.mysql.url}。

当PropertyPlaceholderConfigurer作为BeanFactoryPostProcessor被应用时，它会使用properties配置文件中的值来替换相应的BeanDefinition中占位符所表示的属性值。当需要实例化bean时，bean定义中的属性值就已经被替换成我们配置的值。当然其实现比上面描述的要复杂一些，这里仅说明其大致工作原理，更详细的实现可以参考其源码。



#### BeanPostProcessor

与之相似的，还有**BeanPostProcessor**，其存在于对象实例化阶段。跟BeanFactoryPostProcessor类似，它会处理容器内所有符合条件并且已经实例化后的对象。

BeanPostProcessor定义了两个接口：
```java
public interface BeanPostProcessor {
    // 前置处理
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
    // 后置处理
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```
这两个方法中都传入了bean对象实例的引用，为扩展容器的对象实例化过程提供了很大便利，在这儿几乎可以对传入的实例执行任何操作。

<u>** 注解、AOP ** 等功能的实现均大量使用了BeanPostProcessor。</u>

##### BeanPostProcessor 示例一----自定义注解

比如有一个自定义注解，你完全可以实现BeanPostProcessor的接口，在其中判断bean对象的外部是否有该注解，如果有，你可以对这个bean实例执行任何操作。

##### BeanPostProcessor 示例二----Aware接口

再来看一个更常见的例子，在Spring中经常能够看到各种各样的**Aware接口**，其作用就是在对象实例化完成以后将Aware接口定义中规定的依赖注入到当前实例中。比如最常见的ApplicationContextAware接口，实现了这个接口的类都可以获取到一个ApplicationContext对象。

当容器中每个对象的实例化过程走到BeanPostProcessor前置处理这一步时，容器会检测到之前注册到容器的ApplicationContextAwareProcessor，然后就会调用其postProcessBeforeInitialization()方法，检查并设置Aware相关依赖。

```java
// 代码来自：org.springframework.context.support.ApplicationContextAwareProcessor
// 其postProcessBeforeInitialization方法调用了invokeAwareInterfaces方法
private void invokeAwareInterfaces(Object bean) {
    if (bean instanceof EnvironmentAware) {
        ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
    }
    if (bean instanceof ApplicationContextAware) {
        ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
    }
    // ......
}
```


## JavaConfig & 常见Annotation

### JavaConfig
下面是使用XML配置方式来描述bean的定义：
```xml
<bean id="bookService" class="cn.moondev.service.BookServiceImpl"></bean>
```
而基于JavaConfig的配置形式是这样的：
```java
@Configuration
public class MoonBookConfiguration {

    // 任何标志了@Bean的方法，其返回值将作为一个bean注册到Spring的IOC容器中
    // 方法名默认成为该bean定义的id
    @Bean
    public BookService bookService() {
        return new BookServiceImpl();
    }
}
```
如果两个bean之间有依赖关系的话，在XML配置中应该是这样：

```java
<bean id="bookService" class="cn.moondev.service.BookServiceImpl">
    <property name="dependencyService" ref="dependencyService"/>
</bean>

<bean id="otherService" class="cn.moondev.service.OtherServiceImpl">
    <property name="dependencyService" ref="dependencyService"/>
</bean>

<bean id="dependencyService" class="DependencyServiceImpl"/>
```
而在JavaConfig中则是这样：
```Java
@Configuration
public class MoonBookConfiguration {

    // 如果一个bean依赖另一个bean，则直接调用对应JavaConfig类中依赖bean的创建方法即可
    // 这里直接调用dependencyService()
    @Bean
    public BookService bookService() {
        return new BookServiceImpl(dependencyService());
    }

    @Bean
    public OtherService otherService() {
        return new OtherServiceImpl(dependencyService());
    }

    @Bean
    public DependencyService dependencyService() {
        return new DependencyServiceImpl();
    }
}
```


### @ComponentScan

### @Import
### @Conditional
### @ConfigurationProperties与@EnableConfigurationProperties
```properties
// jdbc config
jdbc.mysql.url=jdbc:mysql://localhost:3306/sampledb
jdbc.mysql.username=root
jdbc.mysql.password=123456
......
```
```java
// 配置数据源
@Configuration
public class HikariDataSourceConfiguration {

    @Value("jdbc.mysql.url")
    public String url;
    @Value("jdbc.mysql.username")
    public String user;
    @Value("jdbc.mysql.password")
    public String password;
    
    @Bean
    public HikariDataSource dataSource() {
        HikariConfig hikariConfig = new HikariConfig();
        hikariConfig.setJdbcUrl(url);
        hikariConfig.setUsername(user);
        hikariConfig.setPassword(password);
        // 省略部分代码
        return new HikariDataSource(hikariConfig);
    }
}
```
以下方式更为得优雅，
```java
@Component
//  还可以通过@PropertySource("classpath:jdbc.properties")来指定配置文件
@ConfigurationProperties("jdbc.mysql")
// 前缀=jdbc.mysql，会在配置文件中寻找jdbc.mysql.*的配置项
pulic class JdbcConfig {
    public String url;
    public String username;
    public String password;
}
```
```java
@Configuration
public class HikariDataSourceConfiguration {

    @AutoWired
    public JdbcConfig config;
    
    @Bean
    public HikariDataSource dataSource() {
        HikariConfig hikariConfig = new HikariConfig();
        hikariConfig.setJdbcUrl(config.url);
        hikariConfig.setUsername(config.username);
        hikariConfig.setPassword(config.password);
        // 省略部分代码
        return new HikariDataSource(hikariConfig);
    }
}
```
@ConfigurationProperties对于更为复杂的配置，处理起来也是得心应手，比如有如下配置文件：
```properties
#App
app.menus[0].title=Home
app.menus[0].name=Home
app.menus[0].path=/
app.menus[1].title=Login
app.menus[1].name=Login
app.menus[1].path=/login

app.compiler.timeout=5
app.compiler.output-folder=/temp/

app.error=/error/
```
可以定义如下配置类来接收这些属性
```JAVA
@Component
@ConfigurationProperties("app")
public class AppProperties {

    public String error;
    public List<Menu> menus = new ArrayList<>();
    public Compiler compiler = new Compiler();

    public static class Menu {
        public String name;
        public String path;
        public String title;
    }

    public static class Compiler {
        public String timeout;
        public String outputFolder;
    }
}
```





## SpringFactoriesLoader详解

### JVM三种类加载器

* BootStrapClassLoader  ——>   Java核心类库
* ExtClassLoader              ——>   Java扩展类库
* AppClassLoader            ——>   应用的类路径CLASSPATH下的类库

### 双亲委派模型简介

JVM通过**双亲委派模型**进行类的加载，我们也可以通过继承java.lang.classloader实现自己的类加载器。

```
当一个类加载器收到类加载任务时，会先交给自己的父加载器去完成，因此最终加载任务都会传递到最顶层的BootstrapClassLoader，只有当父加载器无法完成加载任务时，才会尝试自己来加载。
```
采用双亲委派模型的一个好处是**保证使用不同类加载器最终得到的都是同一个对象**。
```Java
protected Class<?> loadClass(String name, boolean resolve) {
    synchronized (getClassLoadingLock(name)) {
    // 首先，检查该类是否已经被加载，如果从JVM缓存中找到该类，则直接返回
    Class<?> c = findLoadedClass(name);
    if (c == null) {
        try {
            // 遵循双亲委派的模型，首先会通过递归从父加载器开始找，
            // 直到父类加载器是BootstrapClassLoader为止
            if (parent != null) {
                c = parent.loadClass(name, false);
            } else {
                c = findBootstrapClassOrNull(name);
            }
        } catch (ClassNotFoundException e) {}
        if (c == null) {
            // 如果还找不到，尝试通过findClass方法去寻找
            // findClass是留给开发者自己实现的，也就是说
            // 自定义类加载器时，重写此方法即可
           c = findClass(name);
        }
    }
    if (resolve) {
        resolveClass(c);
    }
    return c;
    }
}
```

### 双亲委派弊端

但**双亲委派模型并不能解决所有的类加载器问题**。

比如，Java 提供了很多服务提供者接口(Service Provider Interface，SPI)，允许第三方为这些接口提供实现。常见的 SPI 有 JDBC、JNDI、JAXP 等。

SPI的接口由核心类库提供，却由第三方实现，这样就存在一个问题：SPI 的接口是 Java 核心库的一部分，是由BootstrapClassLoader加载的；SPI实现的Java类一般是由AppClassLoader来加载的。BootstrapClassLoader是无法找到 SPI 的实现类的，因为它只加载Java的核心库。它也不能代理给AppClassLoader，因为它是最顶层的类加载器。也就是说，双亲委派模型并不能解决这个问题。

### 双亲委派弊端解决方案ContextClassLoader

**线程上下文类加载器(ContextClassLoader)**正好解决了这个问题。从名称上看，可能会误解为它是一种新的类加载器，实际上，它仅仅是Thread类的一个变量而已，可以通过setContextClassLoader(ClassLoader cl)和getContextClassLoader()来设置和获取该对象。

如果不做任何的设置，Java应用的线程的上下文类加载器默认就是AppClassLoader。在核心类库使用SPI接口时，传递的类加载器使用线程上下文类加载器，就可以成功的加载到SPI实现的类。线程上下文类加载器在很多SPI的实现中都会用到。但在JDBC中，你可能会看到一种更直接的实现方式，比如，JDBC驱动管理java.sql.Driver中的loadInitialDrivers()方法中，你可以直接看到JDK是如何加载驱动的：

```java
for (String aDriver : driversList) {
    try {
        // 直接使用AppClassLoader
        Class.forName(aDriver, true, ClassLoader.getSystemClassLoader());
    } catch (Exception ex) {
        println("DriverManager.Initialize: load failed: " + ex);
    }
}
```
其实讲解线程上下文类加载器，最主要是让大家在看到`Thread.currentThread().getClassLoader()`和`Thread.currentThread().getContextClassLoader()`时不会一脸懵逼，这两者除了在许多底层框架中取得的ClassLoader可能会有所不同外，其他大多数业务场景下都是一样的。

### 类加载器的另外一个重要功能：加载资源

类加载器除了加载class外，还有一个非常重要功能，就是加载资源，它可以从jar包中读取任何资源文件，比如，**ClassLoader.getResources(String name)方法就是用于读取jar包中的资源文件**，其代码如下：

```Java
public Enumeration<URL> getResources(String name) throws IOException {
    Enumeration<URL>[] tmp = (Enumeration<URL>[]) new Enumeration<?>[2];
    if (parent != null) {
        tmp[0] = parent.getResources(name);
    } else {
        tmp[0] = getBootstrapResources(name);
    }
    //findResources(name)方法会遍历其负责加载的所有jar包，找到jar包中名称为name的资源文件，这里的资源可以是任何文件，甚至是.class文件
    tmp[1] = findResources(name);
    return new CompoundEnumeration<>(tmp);
}
```
它的逻辑其实跟类加载的逻辑是一样的，首先判断父类加载器是否为空，不为空则委托父类加载器执行资源查找任务，直到BootstrapClassLoader，最后才轮到自己查找。而**不同的类加载器负责扫描不同路径下的jar包，就如同加载class一样，最后会扫描所有的jar包，找到符合条件的资源文件**。

下面的示例，用于查找Array.class文件：
```java
// 寻找Array.class文件
public static void main(String[] args) throws Exception{
    // Array.class的完整路径
    String name = "java/sql/Array.class";
    Enumeration<URL> urls = Thread.currentThread().getContextClassLoader().getResources(name);
    while (urls.hasMoreElements()) {
        URL url = urls.nextElement();
        System.out.println(url.toString());
    }
}
```
运行后可以得到如下结果：

```
$JAVA_HOME/jre/lib/rt.jar!/java/sql/Array.class
```


根据资源文件的URL，可以构造相应的文件来读取资源内容。

### 详解**SpringFactoriesLoader**源码

```java
public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
// spring.factories文件的格式为：key=value1,value2,value3
// 从所有的jar包中找到META-INF/spring.factories文件
// 然后从文件中解析出key=factoryClass类名称的所有value值
public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
    String factoryClassName = factoryClass.getName();
    // 取得资源文件的URL
    Enumeration<URL> urls = (classLoader != null ? 	 classLoader.getResources(FACTORIES_RESOURCE_LOCATION) : ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
    List<String> result = new ArrayList<String>();
    // 遍历所有的URL
    while (urls.hasMoreElements()) {
        URL url = urls.nextElement();
        // 根据资源文件URL解析properties文件
        Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
        String factoryClassNames = properties.getProperty(factoryClassName);
        // 组装数据，并返回
        		     result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
    }
    return result;
}
```
有了前面关于ClassLoader的知识，再来理解这段代码：从CLASSPATH下的每个Jar包中搜寻所有META-INF/spring.factories配置文件，然后将解析properties文件，找到指定名称的配置后返回。需要注意的是，其实这里不仅仅是会去ClassPath路径下查找，会扫描所有路径下的Jar包，只不过这个文件只会在Classpath下的jar包中。来简单看下spring.factories文件的内容吧：

```properties
// 来自 org.springframework.boot.autoconfigure下的META-INF/spring.factories
// EnableAutoConfiguration后文会讲到，它用于开启Spring Boot自动配置功能

org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration\
```
**执行loadFactoryNames(EnableAutoConfiguration.class, classLoader)后，得到对应的一组@Configuration类，我们就可以通过反射实例化这些类然后注入到IOC容器中，最后容器里就有了一系列标注了@Configuration的JavaConfig形式的配置类。**

这就是SpringFactoriesLoader，它本质上属于Spring框架私有的一种**扩展方案**，类似于SPI，Spring Boot在Spring基础上的很多核心功能都是基于此。
## Spring容器的事件监听机制
在服务器端，事件的监听机制更多的用于***异步通知***以及***监控和异常处理***。

Java提供了实现事件监听机制的两个基础类：
* java.util.EventObject  自定义事件类型
* java.util.EventListener  事件的监听器

同时还需要注意  EventPublisher 事件发布者


首先定义事件类型，通常的做法是扩展EventObject，随着事件的发生，相应的状态通常都封装在此类中：
```Java
public class MethodMonitorEvent extends EventObject {
    // 时间戳，用于记录方法开始执行的时间
    public long timestamp;

    public MethodMonitorEvent(Object source) {
        super(source);
    }
}
```
事件发布之后，相应的监听器即可对该类型的事件进行处理，我们可以在方法开始执行之前发布一个begin事件，在方法执行结束之后发布一个end事件，相应地，事件监听器需要提供方法对这两种情况下接收到的事件进行处理：
```java
// 1、定义事件监听接口
public interface MethodMonitorEventListener extends EventListener {
    // 处理方法执行之前发布的事件
    public void onMethodBegin(MethodMonitorEvent event);
    // 处理方法结束时发布的事件
    public void onMethodEnd(MethodMonitorEvent event);
}
```
```Java
// 2、事件监听接口的实现：如何处理
public class AbstractMethodMonitorEventListener implements MethodMonitorEventListener {

    @Override
    public void onMethodBegin(MethodMonitorEvent event) {
        // 记录方法开始执行时的时间
        event.timestamp = System.currentTimeMillis();
    }

    @Override
    public void onMethodEnd(MethodMonitorEvent event) {
        // 计算方法耗时
        long duration = System.currentTimeMillis() - event.timestamp;
        System.out.println("耗时：" + duration);
    }
}
```
**事件监听器**接口针对不同的事件发布实际提供相应的处理方法定义，最重要的是，其方法只接收MethodMonitorEvent参数，说明这个监听器类只负责监听器对应的事件并进行处理。

有了事件和监听器，剩下的就是发布事件，然后让相应的监听器监听并处理。通常情况，我们会有一个事件发布者，它本身作为事件源，在合适的时机，将相应的事件发布给对应的事件监听器：

```Java
public class MethodMonitorEventPublisher {

    private List<MethodMonitorEventListener> listeners = new ArrayList<MethodMonitorEventListener>();

    public void methodMonitor() {
        MethodMonitorEvent eventObject = new MethodMonitorEvent(this);
        publishEvent("begin",eventObject);
        // 模拟方法执行：休眠5秒钟
        TimeUnit.SECONDS.sleep(5);
        publishEvent("end",eventObject);

    }

    private void publishEvent(String status,MethodMonitorEvent event) {
        // 避免在事件处理期间，监听器被移除，这里为了安全做一个复制操作
        List<MethodMonitorEventListener> copyListeners = new ArrayList<MethodMonitorEventListener>(listeners);
        for (MethodMonitorEventListener listener : copyListeners) {
            if ("begin".equals(status)) {
                listener.onMethodBegin(event);
            } else {
                listener.onMethodEnd(event);
            }
        }
    }
    
    public static void main(String[] args) {
        MethodMonitorEventPublisher publisher = new MethodMonitorEventPublisher();
        publisher.addEventListener(new AbstractMethodMonitorEventListener());
        publisher.methodMonitor();
    }
    // 省略实现
    public void addEventListener(MethodMonitorEventListener listener) {}
    public void removeEventListener(MethodMonitorEventListener listener) {}
    public void removeAllListeners() {}

}
```
对于事件发布者（事件源）通常需要关注两点：

1、**在合适的时机发布事件**。此例中的methodMonitor()方法是事件发布的源头，其在方法执行之前和结束之后两个时间点发布MethodMonitorEvent事件，每个时间点发布的事件都会传给相应的监听器进行处理。在具体实现时需要注意的是，事件发布是顺序执行，为了不影响处理性能，事件监听器的处理逻辑应尽量简单。

2、**事件监听器的管理**。publisher类中提供了事件监听器的注册与移除方法，这样客户端可以根据实际情况决定是否需要注册新的监听器或者移除某个监听器。如果这里没有提供remove方法，那么注册的监听器示例将一直被MethodMonitorEventPublisher引用，即使已经废弃不用了，也依然在发布者的监听器列表中，这会导致隐性的内存泄漏。

### Spring容器内的事件监听机制
> Spring的ApplicationContext容器内部中（以下两者皆以bean的形式注册在容器中）：
>
> 所有事件类型均继承自**org.springframework.context.AppliationEvent**（继承自EventObject）
>
> 所有监听器都实现**org.springframework.context.ApplicationListener**（继承自EventListener）
>
> 一旦在容器内发布ApplicationEvent及其子类型的事件，注册到容器的ApplicationListener就会对这些事件进行处理。

Spring提供了ApplicationEvent接口的一些默认的实现，比如：

* ContextClosedEvent         表示容器在即将关闭时发布的事件类型
* ContextRefreshedEvent   表示容器在初始化或者刷新的时候发布的事件类型

> ApplicationContext容器在启动时，会自动识别并加载EventListener类型的bean，一旦容器内有事件发布，将通知这些注册到容器的EventListener。
>

**ApplicationContext接口继承了ApplicationEventPublisher接口，该接口提供了void publishEvent(ApplicationEvent event)方法定义，**不难看出，**ApplicationContext容器担当的就是事件发布者的角色。**



可以查看**AbstractApplicationContext.publishEvent(ApplicationEvent event)方法的源码**：ApplicationContext将事件的发布以及监听器的管理工作委托给ApplicationEventMulticaster接口的实现类。在容器启动时，会检查容器内是否存在名为applicationEventMulticaster的ApplicationEventMulticaster对象实例。如果有就使用其提供的实现，没有就默认初始化一个SimpleApplicationEventMulticaster作为实现。

最后，如果我们业务需要在容器内部发布事件，只需要为其注入ApplicationEventPublisher依赖即可：实现ApplicationEventPublisherAware接口或者ApplicationContextAware接口。

## 揭秘自动配置原理
```java
@SpringBootApplication
public class MoonApplication {
    public static void main(String[] args) {
        SpringApplication.run(MoonApplication.class, args);
    }
}
```
@SpringBootApplication开启**组件扫描**和**自动配置**
@SpringApplication.run负责启动引导应用程序

@SpringBootApplication是一个复合的注解，
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
        @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
        @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    // ......
}
```
以下三个注解起到关键作用：
* @SpringBootConfiguration       
    *  就是@Configuration，它是Spring框架的注解，标明该类是一个JavaConfig配置类
* @ComponentScan
    * 启用组件扫描
* @EnableAutoConfiguration
    * 就是开启Spring Boot自动配置功能，Spring Boot会根据应用的依赖、自定义的bean、classpath下有没有某个类 等等因素来猜测你需要的bean，然后注册到IoC容器中。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
//@Import注解用于导入类，并将EnableAutoConfigurationImportSelector作为一个bean的定义注册到容器中
//EnableAutoConfigurationImportSelector会将所有符合条件的@Configuration配置都加载到容器中
@Import(EnableAutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    // ......
}
```
```java
//EnableAutoConfigurationImportSelector.class
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    // 省略了大部分代码，保留一句核心代码
    // 注意：SpringBoot最近版本中，这句代码被封装在一个单独的方法中
    List<String> factories = new ArrayList<String>(new LinkedHashSet<String>(  
        SpringFactoriesLoader.loadFactoryNames(EnableAutoConfiguration.class, this.beanClassLoader)));
}
```
这个类会扫描所有的jar包，将所有符合条件的@Configuration配置类注入的容器中，何为符合条件，看看META-INF/spring.factories的文件内容：
```Properties
// 来自 org.springframework.boot.autoconfigure下的META-INF/spring.factories
// 配置的key = EnableAutoConfiguration，与代码中一致
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration\
.....
```

以DataSourceAutoConfiguration为例，看看Spring Boot是如何自动配置的：
```java
@Configuration
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({ Registrar.class, DataSourcePoolMetadataProvidersConfiguration.class })
public class DataSourceAutoConfiguration {
}
```

分别说一说：

* @ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })：当Classpath中存在DataSource或者EmbeddedDatabaseType类时才启用这个配置，否则这个配置将被忽略。
* @EnableConfigurationProperties(DataSourceProperties.class)：将DataSource的默认配置类注入到IOC容器中，DataSourceproperties定义为：
```java
// 提供对datasource配置信息的支持，所有的配置前缀为：spring.datasource
@ConfigurationProperties(prefix = "spring.datasource")
public class DataSourceProperties  {
    private ClassLoader classLoader;
    private Environment environment;
    private String name = "testdb";
    ......
}
```
* @Import({ Registrar.class, DataSourcePoolMetadataProvidersConfiguration.class })：导入其他额外的配置，就以DataSourcePoolMetadataProvidersConfiguration为例吧。
```java
@Configuration
public class DataSourcePoolMetadataProvidersConfiguration {

    @Configuration
    @ConditionalOnClass(org.apache.tomcat.jdbc.pool.DataSource.class)
    static class TomcatDataSourcePoolMetadataProviderConfiguration {
        @Bean
        public DataSourcePoolMetadataProvider tomcatPoolDataSourceMetadataProvider() {
            .....
        }
    }
  ......
}
```
DataSourcePoolMetadataProvidersConfiguration是数据库连接池提供者的一个配置类，即Classpath中存在org.apache.tomcat.jdbc.pool.DataSource.class，则使用tomcat-jdbc连接池，如果Classpath中存在HikariDataSource.class则使用Hikari连接池。

以上足以说明**Spring Boot如何利用条件话配置来实现自动配置**的。

> @EnableAutoConfiguration中导入了EnableAutoConfigurationImportSelector类，而这个类的selectImports()通过SpringFactoriesLoader得到了大量的配置类，而每一个配置类则根据条件化配置来做出决策，以实现自动配置。
>

整个流程很清晰，但EnableAutoConfigurationImportSelector.selectImports()是何时执行的？
> 其实EnableAutoConfigurationImportSelector.selectImports()会在容器启动过程( AbstractApplicationContext.refresh() )中执行。

## Spring Boot 应用启动的秘密

SpringBoot整个启动流程分为两个步骤：
* SpringApplication对象初始化
* 执行该对象的run方法
```JAVA
@SpringBootApplication
public class test {

    public static void main(String[] args) {
        //SpringApplication.run()方法所返回的Context，可以从中取出一些Bean 来进行其它的一些操作
        //ConfigurableApplicationContext context = 
        //								SpringApplication.run(DtsServerApp.class, args);
        //NettyServerController controller = context.getBean(NettyServerController.class);
        //controller.start();				   
         SpringApplication.run(test.class,args);
    }
}
```

### SpringApplication初始化

看下SpringApplication的初始化流程，SpringApplication的构造方法中调用initialize(Object[] sources)方法，其代码如下：
```java
	public static ConfigurableApplicationContext run(Object source, String... args) {
        return run(new Object[]{source}, args);
    }
    
    public static ConfigurableApplicationContext run(Object[] sources, String[] args) {
        return (new SpringApplication(sources)).run(args);
    }
	/**
	 * Create a new {@link SpringApplication} instance. The application context will load
	 * beans from the specified sources (see {@link SpringApplication class-level}
	 * documentation for details. The instance can be customized before calling
	 * {@link #run(String...)}.
	 * @param sources the bean sources
	 * @see #run(Object, String[])
	 * @see #SpringApplication(ResourceLoader, Object...)
	 */
	public SpringApplication(Object... sources) {
		initialize(sources);
	}
	private void initialize(Object[] sources) {
	    // 上下文配置源保存在set中
		if (sources != null && sources.length > 0) {
			this.sources.addAll(Arrays.asList(sources));
		}
		// 推导是否是web应用
		this.webEnvironment = deduceWebEnvironment();
        
        /*
        * TWO IMPORTANT STEPS:
        */
		// 从 META-INF/spring.factories 中加载应用上下文初始化器类
		setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
		// 从 META-INF/spring.factories 中加载事件监听器
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		// 推导出主类，
		this.mainApplicationClass = deduceMainApplicationClass();
	}

```
初始化流程中最重要的就是通过SpringFactoriesLoader找到spring.factories文件中配置的以下两个接口的实现类名称，以便后期构造相应的实例

> TWO IMPORTANT INTERFACE:
>
> * ApplicationContextInitializer
> * ApplicationListener

1、ApplicationContextInitializer的主要目的是在ConfigurableApplicationContext做refresh之前，对ConfigurableApplicationContext实例做进一步的设置或处理。

ConfigurableApplicationContext继承自ApplicationContext，其主要提供了对ApplicationContext进行设置的能力。

2、ApplicationListener是Spring框架对Java事件监听机制的一种框架实现。

> 如果想为Spring Boot应用添加监听器，该如何实现？
>
> Spring Boot提供两种方式来添加自定义监听器：
>
> * 通过SpringApplication.addListeners(ApplicationListener<?>... listeners)
> * SpringApplication.setListeners(Collection<? extends ApplicationListener<?>> listeners)两个方法来添加一个或者多个自定义监听器
> * @Listener(todo)
> * META-INF/spring.factories文件中新增配置
>

既然SpringApplication的初始化流程中已经从spring.factories中获取到ApplicationListener的实现类，那么我们直接在自己的jar包的META-INF/spring.factories文件中新增配置即可：
```properties
org.springframework.context.ApplicationListener=\
cn.moondev.listeners.xxxxListener\
```
关于SpringApplication的初始化，我们就说这么多。

### SpringApplication运行

> - 运行环境/参数设置 -> 打印Banner -> 创建ApplicationContext -> 上下文准备 -> 上下文刷新
> - ------------------------------------------ 运行监视器 -----------------------------------

Spring Boot应用的整个启动流程都封装在SpringApplication.run方法中，
其本质上就是**在Spring容器启动的基础上做了大量的扩展**，按照这个思路来看看源码：
```JAVA
public ConfigurableApplicationContext run(String... args) {
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        ConfigurableApplicationContext context = null;
        FailureAnalyzers analyzers = null;
        configureHeadlessProperty();
        // ①  获取SpringApplicationRunListeners
        SpringApplicationRunListeners listeners = getRunListeners(args);
        listeners.starting();
        try {
            // ②  运行参数保存在ApplicationArguments中
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            //    配置环境属性，同时会触发 ApplicationEnvironmentPreparedEvent 事件
            ConfigurableEnvironment environment = 		 
                						prepareEnvironment(listeners,applicationArguments);
            // ③
            Banner printedBanner = printBanner(environment);
            // ④  创建具体的 ApplicationContext，
            //	  根据是否为 web 环境分别为：AnnotationConfigEmbeddedWebApplicationContext、
            //                             AnnotationConfigApplicationContext
            context = createApplicationContext();
            // ⑤  失败分析，错误诊断
            analyzers = new FailureAnalyzers(context);
            // ⑥  上下文准备
            prepareContext(context, environment, listeners, applicationArguments,printedBanner);
            // ⑦  刷新上下文
            refreshContext(context);
            // ⑧
            afterRefresh(context, applicationArguments);
            // ⑨  启动完成
            listeners.finished(context, null);
            stopWatch.stop();
            return context;
        }
        catch (Throwable ex) {
            handleRunFailure(context, listeners, analyzers, ex);
            throw new IllegalStateException(ex);
        }
    }
```
① 通过SpringFactoriesLoader查找并加载所有的SpringApplicationRunListeners，通过调用starting()方法通知所有的SpringApplicationRunListeners：应用开始启动了。

SpringApplicationRunListeners其本质上就是一个事件发布者，它在SpringBoot应用启动的不同时间点发布不同应用事件类型(ApplicationEvent)，如果有哪些事件监听者(ApplicationListener)对这些事件感兴趣，则可以接收并且处理。还记得初始化流程中，SpringApplication加载了一系列ApplicationListener吗？这个启动流程中没有发现有发布事件的代码，其实都已经在SpringApplicationRunListeners这儿实现了。

简单的分析一下其实现流程，首先看下SpringApplicationRunListener的源码：
```java
public interface SpringApplicationRunListener {

    // 运行run方法时立即调用此方法，可以用户非常早期的初始化工作
    void starting();
    
    // Environment准备好后，并且ApplicationContext创建之前调用
    void environmentPrepared(ConfigurableEnvironment environment);

    // ApplicationContext创建好后立即调用
    void contextPrepared(ConfigurableApplicationContext context);

    // ApplicationContext加载完成，在refresh之前调用
    void contextLoaded(ConfigurableApplicationContext context);

    // 当run方法结束之前调用
    void finished(ConfigurableApplicationContext context, Throwable exception);

}
```
SpringApplicationRunListener只有一个实现类：EventPublishingRunListener。①处的代码只会获取到一个EventPublishingRunListener的实例，我们来看看starting()方法的内容：
```java
public void starting() {
    // 发布一个ApplicationStartedEvent
    this.initialMulticaster.multicastEvent(
        					new ApplicationStartedEvent(this.application, this.args));
}
```
顺着这个逻辑，你可以在②处的prepareEnvironment()方法的源码中找到listeners.environmentPrepared(environment);即SpringApplicationRunListener接口的第二个方法，那不出你所料，environmentPrepared()又发布了另外一个事件ApplicationEnvironmentPreparedEvent。

② 创建并配置当前应用将要使用的Environment.

> Environment用于描述应用程序当前的运行环境，其抽象了两个方面的内容：
> * 配置文件(profile)
> * 属性(properties)
>

不同的环境(eg：生产环境、预发布环境)可以使用不同的配置文件，而属性则可以从<u>配置文件、环境变量、命令行参数</u>等来源获取。

因此，当Environment准备好后，在整个应用的任何时候，都可以从Environment中获取资源。

总结起来，②处的两句代码，主要完成以下几件事：

* 判断Environment是否存在，不存在就创建（如果是web项目就创建StandardServletEnvironment，否则创建StandardEnvironment）
* 配置Environment：配置profile以及properties
* 调用SpringApplicationRunListener的environmentPrepared()方法，通知事件监听者：应用的Environment已经准备好


③、SpringBoot应用在启动时会输出这样的东西：
```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v1.5.6.RELEASE)

```


④、根据是否是web项目，来创建不同的ApplicationContext容器。

⑤、创建一系列FailureAnalyzer，创建流程依然是通过SpringFactoriesLoader获取到所有实现FailureAnalyzer接口的class，然后在创建对应的实例。FailureAnalyzer用于分析故障并提供相关诊断信息。

⑥、初始化ApplicationContext，主要完成以下工作：

* a) 将准备好的Environment设置给ApplicationContext

  b)  遍历调用所有的ApplicationContextInitializer的initialize()方法来对已经创建好的ApplicationContext进行进一步的处理

* 调用SpringApplicationRunListener的contextPrepared()方法，通知所有的监听者：ApplicationContext已经准备完毕

* 将所有的bean加载到容器中
  调用SpringApplicationRunListener的contextLoaded()方法，通知所有的监听者：ApplicationContext已经装载完毕

  ```java
  private void prepareContext(ConfigurableApplicationContext context,
  			ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
  			ApplicationArguments applicationArguments, Banner printedBanner) {
  		//a)	
      	context.setEnvironment(environment);
  		// ApplicationContext 创建后的处理扩展点：可以设置 BeanNameGenerator 和 ResourceLoader
  		postProcessApplicationContext(context);
  		//b) 
      	applyInitializers(context);
  		// “应用上下文已准备好” 事件，默认实现空
  		listeners.contextPrepared(context);
  		if (this.logStartupInfo) {
  			logStartupInfo(context.getParent() == null);
  			logStartupProfileInfo(context);
  		}

  		// Add boot specific singleton beans
  		//注册Spring boot 特有的单例 bean
  		context.getBeanFactory().registerSingleton("springApplicationArguments",
  				applicationArguments);
  		if (printedBanner != null) {
  			context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);
  		}

  		// Load the sources
  		Set<Object> sources = getSources();
  		Assert.notEmpty(sources, "Sources must not be empty");
  		load(context, sources.toArray(new Object[sources.size()]));
  		
          // 上下文加载完成，触发 ApplicationPreparedEvent 事件
  		listeners.contextLoaded(context);
  	}
  ```

```java
	/**
	 * Apply any {@link ApplicationContextInitializer}s to the context before it is
	 * refreshed.
	 * @param context the configured ApplicationContext (not refreshed yet)
	 * @see ConfigurableApplicationContext#refresh()
	 */
	@SuppressWarnings({ "rawtypes", "unchecked" })
	protected void applyInitializers(ConfigurableApplicationContext context) {
		for (ApplicationContextInitializer initializer : getInitializers()) {
			Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(
					initializer.getClass(), ApplicationContextInitializer.class);
			Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
			initializer.initialize(context);
		}
	}
```

⑦、这个过程和普通 Spring 应用是一样的（ **org.springframework.context.support.AbstractApplicationContext#refresh**），实现对 bean 的加载初始化。

```java
	private void refreshContext(ConfigurableApplicationContext context) {
		refresh(context);
		if (this.registerShutdownHook) {
			try {
				context.registerShutdownHook();
			}
			catch (AccessControlException ex) {
				// Not allowed in some environments.
			}
		}
	}
	
	/**
	 * Refresh the underlying {@link ApplicationContext}.
	 * @param applicationContext the application context to refresh
	 */
	protected void refresh(ApplicationContext applicationContext) {
		Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
		((AbstractApplicationContext) applicationContext).refresh();
	}
	
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
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



调用ApplicationContext的refresh()方法，完成IoC容器可用的最后一道工序。从名字上理解为刷新容器，那何为刷新？就是插手容器的启动，联系一下第一小节的内容。那如何刷新呢？且看下面代码：

 ```java
// 摘自refresh()方法中一句代码
invokeBeanFactoryPostProcessors(beanFactory);
 ```

看看这个方法的实现：
```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
    ......
}
```
获取到所有的BeanFactoryPostProcessor来对容器做一些额外的操作。BeanFactoryPostProcessor允许我们在容器实例化相应对象之前，对注册到容器的BeanDefinition所保存的信息做一些额外的操作。

这里的getBeanFactoryPostProcessors()方法可以获取到3个Processor：

* ConfigurationWarningsApplicationContextInitializer$ConfigurationWarningsPostProcessor
* SharedMetadataReaderFactoryContextInitializer$CachingMetadataReaderFactoryPostProcessor
* ConfigFileApplicationListener$PropertySourceOrderingPostProcessor

不是有那么多BeanFactoryPostProcessor的实现类，为什么这儿只有这3个？因为在初始化流程获取到的各种ApplicationContextInitializer和ApplicationListener中，只有上文3个做了类似于如下操作：
```
public void initialize(ConfigurableApplicationContext context) {
    context.addBeanFactoryPostProcessor(new ConfigurationWarningsPostProcessor(getChecks()));
}
```
然后进入到PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors()方法

这个方法除了会遍历上面的3个BeanFactoryPostProcessor处理外，还会获取类型为BeanDefinitionRegistryPostProcessor的bean：org.springframework.context.annotation.internalConfigurationAnnotationProcessor，对应的Class为ConfigurationClassPostProcessor。

ConfigurationClassPostProcessor用于解析处理各种注解，包括：@Configuration、@ComponentScan、@Import、@PropertySource、@ImportResource、@Bean。当处理@import注解的时候，就会调用<自动配置>这一小节中的EnableAutoConfigurationImportSelector.selectImports()来完成自动配置功能。

⑧、查找当前context中是否注册有CommandLineRunner和ApplicationRunner，如果有则遍历执行它们。

SpringApplication启动后，如果我们想实现自己的逻辑，便可以通过实现 ApplicationRunner 或 CommandLineRunner 来实现（二者的唯一区别是参数形态）。这两种 Runner 的生效点便是上下文 refresh 完成后。

```Java
	/**
	 * Called after the context has been refreshed.
	 * @param context the application context
	 * @param args the application arguments
	 */
	protected void afterRefresh(ConfigurableApplicationContext context,
			ApplicationArguments args) {
		callRunners(context, args);
	}

	private void callRunners(ApplicationContext context, ApplicationArguments args) {
		List<Object> runners = new ArrayList<Object>();
		// 二者的顺序
		runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
		runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
		//根据order排序
		AnnotationAwareOrderComparator.sort(runners);
		for (Object runner : new LinkedHashSet<Object>(runners)) {
			if (runner instanceof ApplicationRunner) {
				callRunner((ApplicationRunner) runner, args);
			}
			if (runner instanceof CommandLineRunner) {
				callRunner((CommandLineRunner) runner, args);
			}
		}
	}
```

⑨、执行所有SpringApplicationRunListener的finished()方法。

```java
	public void finished(ConfigurableApplicationContext context, Throwable exception) {
		for (SpringApplicationRunListener listener : this.listeners) {
			callFinishedListener(listener, context, exception);
		}
	}

	private void callFinishedListener(SpringApplicationRunListener listener,
			ConfigurableApplicationContext context, Throwable exception) {
		try {
			listener.finished(context, exception);
		}
		catch (Throwable ex) {
			if (exception == null) {
				ReflectionUtils.rethrowRuntimeException(ex);
			}
			if (this.log.isDebugEnabled()) {
				this.log.error("Error handling failed", ex);
			}
			else {
				String message = ex.getMessage();
				message = (message == null ? "no error message" : message);
				this.log.warn("Error handling failed (" + message + ")");
			}
		}
	}
```

这就是Spring Boot的整个启动流程，其核心就是**<u>在Spring容器初始化并启动的基础上加入各种扩展点</u>**，这些扩展点包括：

* ApplicationContextInitializer
* ApplicationListener
* 各种BeanFactoryPostProcessor

只要理解这些扩展点是在何时如何工作的，能让它们为你所用即可。

言而总之，Spring才是核心，理解清楚Spring容器的启动流程，那Spring Boot启动流程就不在话下了。







* [原文](http://www.bijishequ.com/detail/522170?p=)
* [spring boot实战：自动配置原理分析 ](http://blog.csdn.net/liaokailin/article/details/49559951)
* [spring boot实战：Spring boot Bean加载源码分析](http://blog.csdn.net/liaokailin/article/details/49107209)