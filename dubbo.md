# 实现原理



* 消费端接口代理

  javassist

* 服务端异步调用

  基于 NIO 的非阻塞实现并行调用，客户端不需要启动多线程即可完成并行调用多个远程服务，相对多线程开销较小。

* 提供端异步执行

  是否异步执行，不可靠异步，只是忽略返回值，不阻塞执行线程	async配置

  

  注意：Provider端异步执行和Consumer端异步调用是相互独立的，你可以任意正交组合两端配置

  - Consumer同步 - Provider同步
  - Consumer异步 - Provider同步
  - Consumer同步 - Provider异步
  - Consumer异步 - Provider异步

* 序列化

* 参数回调

  将基于长连接生成反向代理，这样就可以从服务器端调用客户端逻辑

* 服务降级

  优雅停机Dubbo 是通过 JDK 的 ShutdownHook 来完成优雅停机的，所以如果用户使用 `kill -9 PID` 等强制关闭指令，是不会执行优雅停机的，只有通过 `kill PID` 时，才会执行。

* 泛化接口GenericService

* mina 到 netty

* 注册中心 zookeeper、Apollo、nacos

* dubbo协议

  - 连接个数：单连接
  - 连接方式：长连接
  - 传输协议：TCP
  - 传输方式：NIO 异步传输
  - 序列化：Hessian 二进制序列化



* HTTP 长连接、短连接

  [跳转](https://www.cnblogs.com/gotodsp/p/6366163.html "博客园")

  

# Dubbo启动过程源码解析

```java
    public static void main(String[] args) throws Exception {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("spring/dubbo-provider.xml");
        context.start();
        System.in.read();
    }
```



```java
	public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
			throws BeansException {

		super(parent);
        //传入配置文件的路径
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();
		}
	}
```

AbstractApplicationContext

```java
	public void refresh() throws BeansException, IllegalStateException {
      	//获取同步锁
		synchronized (this.startupShutdownMonitor) { 
			
             //1、准备工作，获取系统的properties属性，验证必要参数等
			prepareRefresh();

			//2、刷新BeanFactory
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			//3、
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
			}catch (BeansException ex) {
				...
			}
		}
	}
```



方法入口处会首先获取同步锁，保证Spring容器的刷新操作在同一时刻不会并发执行。同步锁的实现很简单，就是利用内部的一个final对象当做“锁”，启动、销毁容器的操作都需要在获取这把锁之后才能进行。

```java
	/** Synchronization monitor for the "refresh" and "destroy" */
	private final Object startupShutdownMonitor = new Object();
```



```java
	protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		refreshBeanFactory();
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (logger.isDebugEnabled()) {
			logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
		}
		return beanFactory;
	}
```



AbstractRefreshableApplicationContext

```java
	protected final void refreshBeanFactory() throws BeansException {
        //如果有存在的工厂，先销毁
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
            //创建 DefaultListableBeanFactory
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);
            //加载定义的bean
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
```



AbstractXmlApplicationContext.loadBeanDefinitions

```java
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// Create a new XmlBeanDefinitionReader for the given BeanFactory.
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// Configure the bean definition reader with this context's
		// resource loading environment.
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
        //初始化一个xml解析器
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// Allow a subclass to provide custom initialization of the reader,
		// then proceed with actually loading the bean definitions.
		initBeanDefinitionReader(beanDefinitionReader);
		loadBeanDefinitions(beanDefinitionReader);
	}
```



```java
	protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			reader.loadBeanDefinitions(configResources);
		}
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
            //加载执行位置的配置文件
			reader.loadBeanDefinitions(configLocations);
		}
	}
```

XmlBeanDefinitionReader.doLoadBeanDefinitions

```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
			Document doc = doLoadDocument(inputSource, resource);
			return registerBeanDefinitions(doc, resource);
		}
		...
	}	


protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
		return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
				getValidationModeForResource(resource), isNamespaceAware());
	}
```

xsd、dtd都是用于校验配置文件的格式。

根据xsd，使用SAX模型将xml文件解析成 org.w3c.dom.Document对象。

下面就是通过Document来注册bean

```java
	public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
        //构造了一个DefaultBeanDefinitionDocumentReader实例
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		int countBefore = getRegistry().getBeanDefinitionCount();
        //构造XmlReaderContext上下文时，会创建DefaultNamespaceHandlerResolver
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
```

DefaultBeanDefinitionDocumentReader处理Document之前，会构造一个DefaultNamespaceHandlerResolver，用于解析不同的DOM节点。

```java
	private Map<String, Object> getHandlerMappings() {
		if (this.handlerMappings == null) {
			synchronized (this) {
				if (this.handlerMappings == null) {
					try {
						Properties mappings =
	//从所有包下搜索名称为"META-INF/spring.handlers"的文件		
                            PropertiesLoaderUtils.loadAllProperties(this.handlerMappingsLocation, this.classLoader);
						if (logger.isDebugEnabled()) {
							logger.debug("Loaded NamespaceHandler mappings: " + mappings);
						}
						Map<String, Object> handlerMappings = new ConcurrentHashMap<String, Object>(mappings.size());
						CollectionUtils.mergePropertiesIntoMap(mappings, handlerMappings);
						this.handlerMappings = handlerMappings;
					}
					catch (IOException ex) {
						throw new IllegalStateException(
								"Unable to load NamespaceHandler mappings from location [" + this.handlerMappingsLocation + "]", ex);
					}
				}
			}
		}
		return this.handlerMappings;
	}
```

该类会去加载所有的名为spring.handlers的properties文件，里面指定了命名空间到处理类的映射关系。

dubbo包下的：

```properties
http\://dubbo.apache.org/schema/dubbo=org.apache.dubbo.config.spring.schema.DubboNamespaceHandler
http\://code.alibabatech.com/schema/dubbo=org.apache.dubbo.config.spring.schema.DubboNamespaceHandler
```

DubboNamespaceHandler定义了`<dubbo:application>`、`<dubbo:registry>`等节点的解析器。

```java
public class DubboNamespaceHandler extends NamespaceHandlerSupport {

    static {
        //去classpath下检查是否有其他的同名class，否则会打错误日志
        Version.checkDuplicate(DubboNamespaceHandler.class);
    }

    @Override
    public void init() {
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("config-center", new DubboBeanDefinitionParser(ConfigCenterBean.class, true));
        registerBeanDefinitionParser("metadata-report", new DubboBeanDefinitionParser(MetadataReportConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
    }

}
```

https://blog.csdn.net/peace_hehe/article/details/79288053

然后再看如何解析节点：

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
    if (delegate.isDefaultNamespace(root)) {
        NodeList nl = root.getChildNodes();
        for (int i = 0; i < nl.getLength(); i++) {
            Node node = nl.item(i);
            if (node instanceof Element) {
                Element ele = (Element) node;
                if (delegate.isDefaultNamespace(ele)) {
                    //解析默认命名空间的元素
                    parseDefaultElement(ele, delegate);
                }
                else {
                    //解析自定义命名空间的元素
                    delegate.parseCustomElement(ele);
                }
            }
        }
    }
    else {
        delegate.parseCustomElement(root);
    }
}
```

方法入口处为DOM树的根节点，逐层遍历子节点，将每个子节点交给对应的解析器来处理。

以下面的配置文件为例：

```xml
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
       http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">

    <!-- provider's application name, used for tracing dependency relationship -->
    <dubbo:application name="demo-provider"/>

    <dubbo:registry address="multicast://224.5.6.7:1234" />

    <!-- use dubbo protocol to export service on port 20880 -->
    <dubbo:protocol name="dubbo"/>

    <!-- service implementation, as same as regular local bean -->
    <bean id="demoService" class="org.apache.dubbo.demo.provider.DemoServiceImpl"/>

    <!-- declare the service interface to be exported -->
    <dubbo:service interface="org.apache.dubbo.demo.DemoService" ref="demoService"/>

</beans>
```

定义了两个命名空间：

- `xxmlns:dubbo="http://dubbo.apache.org/schema/dubbo"`，自定义命名空间，包括`<dubbo:application ...>`  、`<dubbo:registry ...>`等元素
- `xmlns="http://www.springframework.org/schema/beans"`，默认命名空间，包括`<bean id="demoService" ...>`（相当于`<beans:bean id="demoService" ...>`）等元素

例如，解析到了第一个节点：

` <dubbo:application name="demo-provider"/>`

属于自定义命名空间的范畴，DefaultNamespaceHandlerResolver里面可以查询到，该命名空间对应的处理器:

`http\://dubbo.apache.org/schema/dubbo=org.apache.dubbo.config.spring.schema.DubboNamespaceHandler`

而DubboNamespaceHandler里面定义了"application"对应的解析器：

`registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));`

具体实现在DubboBeanDefinitionParser：

```java
 private static BeanDefinition parse(Element element, ParserContext parserContext, Class<?> beanClass, boolean required) {
      //将配置文件的一个节点，转换成Spring的bean对象
        RootBeanDefinition beanDefinition = new RootBeanDefinition();
        beanDefinition.setBeanClass(beanClass);
        beanDefinition.setLazyInit(false);
        String id = element.getAttribute("id");
        ...
        if (id != null && id.length() > 0) {
            if (parserContext.getRegistry().containsBeanDefinition(id)) {
                throw new IllegalStateException("Duplicate spring bean id " + id);
            }
            //将配置bean注册到DefaultListableBeanFactory
            parserContext.getRegistry().registerBeanDefinition(id, beanDefinition);
            beanDefinition.getPropertyValues().addPropertyValue("id", id);
        }
      //后面还有各种属性的解析
        ...
        return beanDefinition;
    }
```

顺便吐槽一下，这个方法写的实在不够优雅，尤其属性解析的部分，生拉硬拽，有失风范。

从中可以看到，dubbo的配置也会被看做一种特殊的Spring Bean对象，并注册到BeanFactory当中。

**注意：**此处只是注册了bean的定义，可以看做一份图纸，但是这个bean实际上还没有被实例化。





