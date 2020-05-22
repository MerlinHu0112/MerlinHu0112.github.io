---
title: Spring学习之IoC
comments: true
top: false
date: 2020-05-22 16:00:29
tags:
	- Spring
	- IoC
categories:
	- 学习笔记
---

Spring 是基于 POJO（Plain Old Java Object，简单 Java 对象）的轻量级开发框架，由多个模块组成。其中，Core 模块是 Spring 框架的基石，它提供 IoC 容器的实现，以及以依赖注入的方式管理对象之间的依赖关系。

在传统的开发中，对其它对象的引用或依赖关系的管理由具体的类负责，导致代码高度耦合、难以测试。通过 IoC容器（如 Spring 中的 IoC 容器）可以主动完成对象创建和依赖关系的注入，使得 Java 的类与类之间解耦合。

本文基于 Spring 4.2.5 版本，从以下五个方面介绍 Spring 中的 IoC 容器：

- Spring 提供的两种 IoC 容器（BeanFactory 和 ApplicationContext）
- IoC 容器启动与 bean 的实例化过程
- bean 的生命周期
- IoC 容器装配 bean 的方式
- IoC 容器依赖注入方式

<!--more-->

Spring 框架如下图所示：

![](Spring学习之IoC/Spring框架总体框架.jpg)



#### 一、控制反转和依赖注入

**IoC（Inversion of Control，控制反转）**：在 IoC 容器之前，对象的引用或依赖关系的管理由具体的类负责，主动权在该具体类。IoC 容器之后，这些功能由容器负责，不同的类之间只要专注于自身即可。由之前的主动行为变成被动行为，故称之为控制反转。

**DI（Dependency Injection，依赖注入）**：所谓依赖注入，就是在 IoC 容器运行期间，动态地将依赖关系注入到对象之中。依赖注入是实现 IoC 的一种方式。



#### 二、BeanFactory

Spring 实现了两种 IoC 容器，分别是基础的 BeanFactory 容器和基于 BeanFactory 的 ApplicationContext 容器。

Spring 提供两种 IoC 容器：

- BeanFactory：基础的 IoC 容器，默认懒加载策略，**使用 bean 时才完成初始化及依赖注入**，因此容器启动速度快、所需资源少。
- ApplicationContext：基于 BeanFactory，额外实现 MessageSource（国际化）、ApplicationEvenPublisher（事件发布）、ResourcePatternResolver 接口。**在容器启动后，完成所有 bean 的初始化和依赖注入**。因此容器启动时间较长、所需资源较多。

![](Spring学习之IoC/IoC容器的功能.jpg)



##### 2.1 BeanFactory 容器底层结构

实现 BeanFactory 容器，常见的接口关系图如下：

![](Spring学习之IoC/BeanFactory容器常用接口.jpg)

实现 BeanFactory 容器涉及四个重要接口：**BeanFactory**、**BeanDefinitionRegistry**、**BeanDefinition**、**BeanDefinitionReader**。

- BeanFactory 接口：定义管理 bean 的方法。
- BeanDefinitionRegistry 接口：定义注册 bean 的方法。
- BeanDefinition 接口：保存 bean 的信息，包括对象的 Class 类型、构造方法参数以及其它属性等。
- BeanDefinitionReader 接口：读取配置文件内容、将其映射至 BeanDefinition，然后将映射后的 BeanDefinition 注册到 BeanDefinitionRegistry 中，由后者完成 bean 的注册和加载。常见实现类有两种：
  - XmlBeanDefinitionReader：读取 XML 格式的配置文件；
  - PropertiesBeanDefinitionReader：读取 Properties 格式的配置文件。



*BeanFactory 接口的源码如下*

```java
// 用于访问Spring Bean容器的根接口。使用此接口及其子接口可以实现Spring的依赖注入功能
// 通常，BeanFactory将加载存储在配置源中的bean的定义，并使用beans包来配置bean
public interface BeanFactory {
    // 区别FactoryBean。前缀为'&'表示获取工厂本身，而非工厂返回的bean
	String FACTORY_BEAN_PREFIX = "&";
    // 根据bean的名称获取bean
    Object getBean(String name) throws BeansException;
    // 根据bean的名称和类型获取bean
    <T> T getBean(String name, Class<T> requiredType) throws BeansException;
    // 根据类型获取bean，此类型可以是实现的接口或父类
    <T> T getBean(Class<T> requiredType) throws BeansException;
    // 判断给定名称的bean是否存在
    boolean containsBean(String name);
    // 判断给定名称的bean是否是单例模式
    boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
    // 判断给定名称的bean是否是原型模式
    boolean isPrototype(String name) throws NoSuchBeanDefinitionException;
    // 返回给定bean名称的别名。若给定的是别名，则返回相应的规范bean名称和其它别名
    String[] getAliases(String name);
    ......
}
```



##### 2.2 启动 BeanFactory 容器

Spring IoC 容器的启动过程主要包括：

- 启动初始化与资源定位
- 载入并解析 BeanDefinition
- 注册 BeanDefinition



（1）BeanFactory 容器使用示例

```java
public class App {
    public static void main( String[] args ) {
        // 创建IoC配置文件的抽象资源
        ClassPathResource resource = new ClassPathResource("ApplicationContext.xml");
        // 获取bean实例的注册表
        DefaultListableBeanFactory beanRegistry = new DefaultListableBeanFactory();
        // 创建载入BeanDefinition的读取器
        XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(beanRegistry);
        // 加载配置文件
        reader.loadBeanDefinitions(resource);
        // 获取bean
        UserService userService = (UserService) beanRegistry.getBean("userService");
        // 使用bean
        userService.serviceFunction();
    }
}
```



（2）源码分析

1. 首先创建配置文件的抽象资源

```java
ClassPathResource resource = new ClassPathResource("ApplicationContext.xml");
```

ClassPathResource 类的继承图谱如下图所示，可看出其继承自 AbstractResource 类，而后者实现了资源描述符接口 Resource。

![](Spring学习之IoC/ClassPathResource类继承图谱.jpg)

调用 ClassPathResource 类的构造函数，获取对象。

```java
public class ClassPathResource extends AbstractFileResolvingResource {
    
    public ClassPathResource(String path) {
        this(path, (ClassLoader) null); // 调用重载的构造函数，类加载器为null
    }

    public ClassPathResource(String path, ClassLoader classLoader) {
        // 首先判断资源路径是否为空
        Assert.notNull(path, "Path must not be null");
        // 规范资源路径
        String pathToUse = StringUtils.cleanPath(path);
        if (pathToUse.startsWith("/")) {
            pathToUse = pathToUse.substring(1);
        }
        this.path = pathToUse;
        // 当类加载器为null时，使用默认的类加载器
        this.classLoader = (classLoader != null ? classLoader : 
                            ClassUtils.getDefaultClassLoader());
    }
}
```

进入 ClassUtils 类的 getDefaultClassLoader 方法，**先后尝试获取当前线程上下文类加载器、ClassUtils 类加载器、启动类加载器。**需要注意的是，尝试获取类加载器是按照一定顺序的。若已获得类加载器，就返回该类加载器，而不需要获取后续的类加载器。

```java
public static ClassLoader getDefaultClassLoader() {
    ClassLoader cl = null;
    try {
        // 尝试获取当前线程上下文类加载器。若有，则返回
        cl = Thread.currentThread().getContextClassLoader();
    }
    catch (Throwable ex) {}
    if (cl == null) {
        // 若线程上下文类加载器为空，则尝试获取ClassUtils的类加载器。若有，则返回
        cl = ClassUtils.class.getClassLoader();
        if (cl == null) {
            try {
                // 若前面均为空，则尝试获取启动类加载器并返回
                cl = ClassLoader.getSystemClassLoader();
            }
            catch (Throwable ex) {}
        }
    }
    return cl;
}
```



2. 随后获取 bean 实例的注册表

```java
DefaultListableBeanFactory beanRegistry = new DefaultListableBeanFactory();
```

![](Spring学习之IoC/DefaultListableBeanFactory类继承关系图.jpg)

DefaultListableBeanFactory 类是 Spring 的 ConfigurableListableBeanFactory 和 BeanDefinitionRegistry 接口的默认实现。它是基于 bean 定义元数据（bean definition metadata）的 bean 工厂，可通过后处理器进行扩展。

```java
public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory
	implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable {
	
	public DefaultListableBeanFactory() {
		super(); // 1.调用父类的空参构造函数
	}
	
	// 9.初始化如下字段
	// 允许使用相同的bean名称重新注册不同的bean，即以后一个bean覆盖前一个
	private boolean allowBeanDefinitionOverriding = true;
	// 允许预加载类，包括设置了懒加载的bean
	private boolean allowEagerClassLoading = true;
	// bean名称与BeanDefinition对象的映射表
	private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<String, BeanDefinition>(256);
	// 其它字段略
}
```

抽象类 AbstractAutowireCapableBeanFactory 提供 bean 创建（构造函数解析），属性填充，wiring（包括自动装配）和初始化等功能。

```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {
		
	public AbstractAutowireCapableBeanFactory() {
		super(); // 2.继续调用父类的空参构造函数
		// 9.自动装配时忽略下述三个接口
		ignoreDependencyInterface(BeanNameAware.class);
		ignoreDependencyInterface(BeanFactoryAware.class);
		ignoreDependencyInterface(BeanClassLoaderAware.class);
	}
	
	// 8.初始化如下字段
	// 创建bean实例的策略，默认使用CGLib动态生成子类
	private InstantiationStrategy instantiationStrategy = new CglibSubclassingInstantiationStrategy();
	// 是否自动尝试解析bean之间的循环引用
	private boolean allowCircularReferences = true;
	// 其它字段略
}
```



```java
public abstract class AbstractBeanFactory extends FactoryBeanRegistrySupport 
											implements ConfigurableBeanFactory {
											
	public AbstractBeanFactory() {} // 3.继续调用父类的
	// 7.初始化如下字段
	// 获取类加载器（具体过程见上文）
	private ClassLoader beanClassLoader = ClassUtils.getDefaultClassLoader();
	// true表示缓存bean元数据，false表示每次使用时获取
	private boolean cacheBeanMetadata = true;
	// 其它字段略
}

// 支持需要处理FactoryBean实例的单例bean注册表
public abstract class FactoryBeanRegistrySupport extends DefaultSingletonBeanRegistry {
	// 6.初始化如下字段
	// 由FactoryBeans创建的单例对象的缓存，映射关系为“FactoryBean名称-->对象”
	private final Map<String, Object> factoryBeanObjectCache = new ConcurrentHashMap<String, Object>(16);
}

public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {
    // 5.初始化如下字段
    protected final Log logger = LogFactory.getLog(getClass()); // 子类记录器
    // 单例bean缓存，映射关系为“bean名称-->bean实例”，底层结构为ConcurrentHashMap，初始容量为256
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);
	// 单例工厂缓存，映射关系为“bean名称-->工厂对象”，底层结构为HashMap，初始容量为16
    private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);
    // 早期的单例bean缓存
    private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);
    // 已注册的单例bean名称集合，按照注册顺序，底层结构为LinkedHashSet
    private final Set<String> registeredSingletons = new LinkedHashSet<String>(256);
    // 当前正在创建的bean的名称
    private final Set<String> singletonsCurrentlyInCreation =
			Collections.newSetFromMap(new ConcurrentHashMap<String, Boolean>(16));
    // 还有一些其它字段，略过
}

public class SimpleAliasRegistry implements AliasRegistry {
    // bean的规范命名与别名的映射表，底层结构为ConcurrentHashMap
    private final Map<String, String> aliasMap = new ConcurrentHashMap<String, String>(16); // 4.初始化bean的别名与规范名称映射表
}
```



3. 创建载入BeanDefinition的读取器

```java
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(beanRegistry);
```

![](Spring学习之IoC/XmlBeanDefinitionReader类继承关系图.jpg)

首先看 XmlBeanDefinitionReader 类源码。此类用于读取 XML 格式的 bean 配置文件， 将实际的 XML 文档读取委托给 BeanDefinitionDocumentReader 接口的实现。 文档阅读器将向给定的 bean 工厂注册每个 bean 定义。

```java
public class XmlBeanDefinitionReader extends AbstractBeanDefinitionReader {
    // 以给定的bean工厂创建阅读器对象
    // 给定的bean工厂以BeanDefinitionRegistry的形式注册bean
    public XmlBeanDefinitionReader(BeanDefinitionRegistry registry) {
		super(registry);
	}
    
    // 初始化字段（仅列举部分）
    // 使用Spring默认的DocumentLoader实现，它使用标准的JAXP配置的XML解析器加载文档
    private DocumentLoader documentLoader = new DefaultDocumentLoader();
    // 检测XML文档是否基于DTD或XSD的验证
    private final XmlValidationModeDetector validationModeDetector = new XmlValidationModeDetector();
}
```

接下来看 XmlBeanDefinitionReader 的抽象父类 AbstractBeanDefinitionReader 的源码。

AbstractBeanDefinitionReader 类为指定的 bean 工厂创建一个新的 AbstractBeanDefinitionReader 对象。

```java
public abstract class AbstractBeanDefinitionReader implements EnvironmentCapable, BeanDefinitionReader {
    protected AbstractBeanDefinitionReader(BeanDefinitionRegistry registry) {
		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
		this.registry = registry;

		// 如果registry对象除BeanDefinitionRegistry外还实现了ResourceLoader接口，
        // 则将传入的registry对象作为默认的资源加载器，如ApplicationContext容器
		if (this.registry instanceof ResourceLoader) {
			this.resourceLoader = (ResourceLoader) this.registry;
		}
		else {
            // 传入的registry对象未实现ResourceLoader接口，
            // 则默认使用PathMatchingResourcePatternResolver作为资源加载器
			this.resourceLoader = new PathMatchingResourcePatternResolver();
		}

		// Inherit Environment if possible
		if (this.registry instanceof EnvironmentCapable) {
			this.environment = ((EnvironmentCapable) this.registry).getEnvironment();
		}
		else {
			this.environment = new StandardEnvironment();
		}
	}
}
```



4. 加载bean的配置资源

```java
reader.loadBeanDefinitions(resource);
```

调用 XmlBeanDefinitionReader 类的 loadBeanDefinitions 方法加载资源

```java
public class XmlBeanDefinitionReader extends AbstractBeanDefinitionReader {
    // 1.调用loadBeanDefinitions方法
    @Override
	public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
		return loadBeanDefinitions(new EncodedResource(resource));
	}
    
    // 2.允许使用指定的编码格式解析资源文件
    public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isInfoEnabled()) {
			logger.info("Loading XML bean definitions from " + encodedResource.getResource());
		}
		// XmlBeanDefinitionReader内部维护的resourcesCurrentlyBeingLoaded对象保存
        // 最近被加载的资源
		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
		// 若尚未加载资源，进行初始化操作
        if (currentResources == null) {
			currentResources = new HashSet<EncodedResource>(4);
			this.resourcesCurrentlyBeingLoaded.set(currentResources);
		}
        // 添加EncodedResource对象
		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}
		try {
            // 获取资源的输入流对象
			InputStream inputStream = encodedResource.getResource().getInputStream();
			try {
                // 将输入流对象封装成org.xml.sax包的InputSource对象
				InputSource inputSource = new InputSource(inputStream);
				if (encodedResource.getEncoding() != null) {
                    // 设置编码方式
					inputSource.setEncoding(encodedResource.getEncoding());
				}
                // 3.调用doLoadBeanDefinitions方法加载输入流
				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
			}
			finally {
				inputStream.close();
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"IOException parsing XML document from " + encodedResource.getResource(), ex);
		}
		finally {
			currentResources.remove(encodedResource);
			if (currentResources.isEmpty()) {
				this.resourcesCurrentlyBeingLoaded.remove();
			}
		}
	}
    
    // 3.调用doLoadBeanDefinitions方法加载输入流
    protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
            // 4.使用默认的DocumentLoader加载资源文档
			Document doc = doLoadDocument(inputSource, resource);
            // 5.注册BeanDefinition
			return registerBeanDefinitions(doc, resource);
		}
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		catch (SAXParseException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
		}
		catch (SAXException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"XML document from " + resource + " is invalid", ex);
		}
		catch (ParserConfigurationException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Parser configuration exception parsing XML from " + resource, ex);
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"IOException parsing XML document from " + resource, ex);
		}
		catch (Throwable ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Unexpected exception parsing XML document from " + resource, ex);
		}
	}
    
    // 4.使用默认的DocumentLoader加载XML文档，返回获取的Document对象
    protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
		return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
				getValidationModeForResource(resource), isNamespaceAware());
	}
    
    // 5.注册BeanDefinition
    public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
		// 默认获取DefaultBeanDefinitionDocumentReader实例
        BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		// 获取原注册表记录数
        int countBefore = getRegistry().getBeanDefinitionCount();
        // 注册BeanDefinition
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		// 返回新注册的记录数
        return getRegistry().DefinitionCount() - countBefore;
	}
}
```

在第 4 步中，调用 DefaultDocumentLoader 类的 loadDocument 方法加载 XML 文档。

```java
public class DefaultDocumentLoader implements DocumentLoader {
    /**
	 * @param inputSource 输入源
	 * @param entityResolver 实体解析器
	 * @param errorHandler 错误处理器
	 * @param validationMode XML文档验证类型，DTD或XSD
	 * @param namespaceAware 是否支持XML命名空间
   	 */
   @Override
	public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
			ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {
		// 创建工厂实例，此工厂可根据XML文档获取解析器并生成DOM对象树
		DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
		if (logger.isDebugEnabled()) {
			logger.debug("Using JAXP provider [" + factory.getClass().getName() + "]");
		}
        // 创建一个JAXP DocumentBuilder对象，用于解析XML文档
		DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
		// 解析输入域，获取并返回Document对象
        return builder.parse(inputSource);
	}
}
```

至此，完成了 IoC 容器 BeanFactory 的启动工作，由 Spring 管理的 bean 被注册进容器中。需要注意的是，此时bean 尚未实例化。在使用 bean 时才完成初始化及依赖注入。

