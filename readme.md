# Spring源码解读

在使用Spring的时候，我们应用及了解偏多的主要就是`IOC、DI、AOP、事务`等等操作，然而很多时候我们只是停留在使用层面上，对于Spring内部是如何实现这些功能的全然不知；通过阅读Spring的相关源码或者通过`Debug`跟踪Spring的调用过程，可以帮忙我们更深入的了解Spring的各种机制；

## IOC（控制反转）

`IOC`全称为 `Inversion of Control`，也就是控制反转的意思；在我们平时的编程中，所有的`POJO`都是需要我们通过各种各样的方式进行实例化的，例如`new`；而控制反转的意思是，我们不再需要自己手动将相关的对象进行实例化的工作，我们把这对象实例化的相关控制权反转交给了Spring，Spring会帮我们将相关的对象注册成`Bean`，并交给相关的容器管理；

那么Spring通过什么依据来帮我们生成并管理`Bean`对象呢？

熟悉Spring的人都知道，我们可以通过将我们的实体信息配置到相应的`XML`、`Properties`文件中，或者通过注解`Annotation`的方式来标记，而Spring会按照我们的相关配置，将相应的实体生成对应的`Bean`对象，并主持到Spring的`Ioc`容器中去；

简言之，Spring的`Ioc`包含以下步骤

* 定位：将对应的资源文件加载到输入流中，并获取对应的资源对象；以`XML`为例，获取到`Document`对象；
* 加载：将解析完的资源加载成BeanDefinition对象；
* 注册：将BeanDefinition注册到Ioc容器中去；

> 定位加载注册的阶段因人而异，可以认为定位**只是确定文件文件**，也可以认为是**解析成对应的资源对象**；加载可以理解为**加载资源文件，解析成资源对象**，也可以理解成**加载成BeanDefiniton对象**；注册可以理解成**映射成BeanDefinition并注册到Ioc容器**中，也可以理解成只是**注册到Ioc容器**中；
>
> 但是Ioc容器的控制反转操作基本上就是以上步骤；

### 定位

定位的意思就是确定需要生成`Bean`对象的实体配置存放的位置，并通过解读对应的资源文件，解析资源文件的信息；以`XML`配置文件为例，主要就是通过`XML解析`技术，将配置解析成相应的`Document`跟`Element`对象；

这里以`ClassPathXmlApplicationContext`来进行代码跟踪，不管`XmlBeanFactory`还是`ClassPathXmlApplicationContext`，其最终调用都是一致的，以`XmlBeanFactory`还能更加清晰直观的查看到调用；

一般在使用Spring的`ClassPathXmlApplicationContext`来作为`Ioc`容器的时候，我们一般都会以以下代码开始

```java
ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("application-context.xml");
context.getBean("test");
```

而Spring在进行`ClassPathXmlApplicationContext`实例化的时候，内部最终会调用

```java
public ClassPathXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
    throws BeansException {
    super(parent);
    /**	多个资源配置文件的处理*/
    setConfigLocations(configLocations);
    if (refresh) {
        refresh();
    }
}
```

> 虽然ClassPathXmlApplicationContext本身就是一个BeanFactory，但是实际上在进行操作Bean的时候，还是委派给创建出来的BeanFactory对象；原因是BeanFactory的很多实现调用最终都是通过DefaultListableBeanFactory调用的，而ClassPathXmlApplicationContext实际上并没有继承该类，所以会自己创建一个BeanFactory对象，而这个对象就是DefaultListableBeanFactory；

实际上在`refersh`中会进行一系列的操作，包括生成一个对应的`BeanFactory`对象；创建`BeanFactory`的关键代码为：

```java
# AbstractApplicationContext
ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

# 最终会调用到以下方法
AbstractRefreshableApplicationContext.refreshBeanFactory()
```

在生成对应的`BeanFactory`对象后，会将`BeanFactory`对象往下传递，最终传递到`XmlBeanDefinitionReader`对象中；

```java
# AbstractRefreshableApplicationContext
loadBeanDefinitions(beanFactory);

# 调用方法
AbstractXmlApplicationContext.loadBeanDefinitions();

# 调用方法
XmlBeanDefinitionReader.loadBeanDefinitions();
```

这里在`loadBeanDefinitions`中会生成一个`XmlBeanDefinitionReader`对象，在Spring中，这个对象可以理解成就是`Ioc`容器用来定位并解析`XML`资源文件内容的；在方法链条调用中会发现，最终会调用回

```java
# XmlBeanDefinitionReader
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
	Assert.notNull(encodedResource, "EncodedResource must not be null");
	if (logger.isInfoEnabled()) {
		logger.info("Loading XML bean definitions from " + encodedResource.getResource());
	}

	Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
	if (currentResources == null) {
		currentResources = new HashSet<>(4);
		this.resourcesCurrentlyBeingLoaded.set(currentResources);
	}
	if (!currentResources.add(encodedResource)) {
		throw new BeanDefinitionStoreException(
				"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
	}
	try {
		InputStream inputStream = encodedResource.getResource().getInputStream();
		try {
			InputSource inputSource = new InputSource(inputStream);
			if (encodedResource.getEncoding() != null) {
				inputSource.setEncoding(encodedResource.getEncoding());
			}
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
```

通过获取资源文件的输入流信息，并包装成一个`InputSource`对象，最终调用一下方法完成解析操作

```java
return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
```

最终会调用`javax.xml.parsers`下的包来完成解析成`document`的操作

```java
# XmlBeanDefinitionReader
Document doc = doLoadDocument(inputSource, resource);

# 以下完成document解析操作
DefaultDocumentLoader.loadDocument();
DocumentBuilder.parse();
```

在完成`xml`文件解析动作后，会通过`BeanDefinitionDocumentReader`对象来将`Document`对象中的`root Element`元素传递下去，开启将配置信息加载映射成`BeanDefinition`对象;

```java
# XmlBeanDefinitionReader.registerBeanDefinitions
documentReader.registerBeanDefinitions(doc, createReaderContext(resource));

DefaultBeanDefinitionDocumentReader.registerBeanDefinitions()
DefaultBeanDefinitionDocumentReader.doRegisterBeanDefinitions()
```

```java
protected void doRegisterBeanDefinitions(Element root) {
	// Any nested <beans> elements will cause recursion in this method. In
	// order to propagate and preserve <beans> default-* attributes correctly,
	// keep track of the current (parent) delegate, which may be null. Create
	// the new (child) delegate with a reference to the parent for fallback purposes,
	// then ultimately reset this.delegate back to its original (parent) reference.
	// this behavior emulates a stack of delegates without actually necessitating one.
	BeanDefinitionParserDelegate parent = this.delegate;
	this.delegate = createDelegate(getReaderContext(), root, parent);

	if (this.delegate.isDefaultNamespace(root)) {
		String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
		if (StringUtils.hasText(profileSpec)) {
			String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
					profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
			if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
				if (logger.isInfoEnabled()) {
					logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
							"] not matching: " + getReaderContext().getResource());
				}
				return;
			}
		}
	}

    /**	处理xml前的操作*/
	preProcessXml(root);
    /**	将元素映射成BeanDefinition对象入口*/
	parseBeanDefinitions(root, this.delegate);
    /**	处理xml后的操作*/
	postProcessXml(root);

	this.delegate = parent;
}
```

以来，就完成了所有的动作，相应的时序图如下

![image-20180721172702649](https://ws1.sinaimg.cn/large/006tNc79gy1fthm145i5dj31kw0ol0wd.jpg)

### 加载

加载阶段主要是将解析出来的`Document`对象中的所有`Element`元素进行解析操作，并构建成一个`BeanDefinition`对象。比如我们的`XML`为：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">
	<!-- 必须放在最前面 -->
    <description/>
    <alias name="" alias=""/>
    <import resource=""/>
    <bean/>
    <beans>
        <bean/>
    </beans>
    <!-- more bean definitions go here -->
</beans>
```

那么在`ClassPathXmlApplicationContext`容器启动的时候，会在调用到以下方法的时候开始进行解析并加载成`BeanDifinition`对象；

```java
# DefaultBeanDefinitionDocumentReader
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
	if (delegate.isDefaultNamespace(root)) {
		NodeList nl = root.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (node instanceof Element) {
				Element ele = (Element) node;
				if (delegate.isDefaultNamespace(ele)) {
					parseDefaultElement(ele, delegate);
				}
				else {
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

在方法内部会判断获取顶级元素的孩子节点，并依次判断该节点是不是默认命名空间`http://www.springframework.org/schema/beans`的元素，如果是则调用一下方法完成默认元素的解析

```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
    /**	对import节点进行解析*/
	if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
		importBeanDefinitionResource(ele);
	}
    /**	对alias节点进行解析*/
	else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
		processAliasRegistration(ele);
	}
    /**	对bean节点进行解析*/
	else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
		processBeanDefinition(ele, delegate);
	}
    /**	对beans节点进行解析*/
	else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
		// recurse
	    /**	beans节点会包含多个子节点，递归*/
		doRegisterBeanDefinitions(ele);
	}
}
```

从以上代码中可以看到，默认命名空间中只会处理`import`、`alias`、`bean`、`beans`节点，但是`description`节点并没有做解析操作，也就是说`description`只是一个描述性节点；

对`import`节点的处理就是通过分析节点下的`resource`属性，定位到具体的资源位置，按之前的操作将`resource`中指定的资源再次进行解析成`Document`并执行后续操作；

对`alias`节点自然就是记录配置的`beanName`所对应的`alias`别名信息，等到实际上进行获取`Bean`对象的时候使用；

对`bean`节点的解析的具体实现代码如下，该方法负责加载的操作主要是调用`delegate`实现的，对应实现类为`BeanDefinitionParserDelegate`；

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
	BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
	if (bdHolder != null) {
		bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
		try {
			// Register the final decorated instance.
			BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
		}
		catch (BeanDefinitionStoreException ex) {
			getReaderContext().error("Failed to register bean definition with name '" +
					bdHolder.getBeanName() + "'", ele, ex);
		}
		// Send registration event.
		getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
	}
}
```

`BeanDefinitionParserDelegate`在处理`bean`元素节点的时候，会分为两个步骤

* parseBeanDefinitionElement：解析`bean`节点的基本属性信息、子节点属性
* decorateBeanDefinitionIfRequired：解析`bean`节点的自定义属性信息、自定义子节点属性信息

解析`bean`的基本属性的代码如下，内部通过调用各个不同的方法`(parse开头)`来解析不同的属性

```java
@Nullable
public AbstractBeanDefinition parseBeanDefinitionElement(
		Element ele, String beanName, @Nullable BeanDefinition containingBean) {

	this.parseState.push(new BeanEntry(beanName));

	String className = null;
	if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
		className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
	}
	String parent = null;
	if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
		parent = ele.getAttribute(PARENT_ATTRIBUTE);
	}

	try {
		AbstractBeanDefinition bd = createBeanDefinition(className, parent);

        /**	解析bean上的属性*/
		parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
        /**	设置描述属性*/
		bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));
        /**	meta子节点*/
		parseMetaElements(ele, bd);
        /**	lookup-method子节点*/
		parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
        /**	replaced-method子节点*/
		parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
        /**	构造方法*/
		parseConstructorArgElements(ele, bd);
        /**	property子节点*/
		parsePropertyElements(ele, bd);
        /**	qualifier*/
		parseQualifierElements(ele, bd);

		bd.setResource(this.readerContext.getResource());
		bd.setSource(extractSource(ele));

		return bd;
	}
	catch (ClassNotFoundException ex) {
		error("Bean class [" + className + "] not found", ele, ex);
	}
	catch (NoClassDefFoundError err) {
		error("Class that bean class [" + className + "] depends on not found", ele, err);
	}
	catch (Throwable ex) {
		error("Unexpected failure during bean definition parsing", ele, ex);
	}
	finally {
		this.parseState.pop();
	}

	return null;
}
```

其中构造方法给`property`是比较常用的，内部都会调用同一个方法`parsePropertyValue(Element ele, BeanDefinition bd, @Nullable String propertyName)`，**方法的内部实现为检查是否还存在子节点、且是否设置成`ref`或者`value`属性，注意两者不能共存；**

解析`bean`自定义属性及自定义子节点属性的代码如下：

```java
public BeanDefinitionHolder decorateBeanDefinitionIfRequired(
		Element ele, BeanDefinitionHolder definitionHolder, @Nullable BeanDefinition containingBd) {

	BeanDefinitionHolder finalDefinition = definitionHolder;

	// Decorate based on custom attributes first.
    /**	如果是自定义属性*/
	NamedNodeMap attributes = ele.getAttributes();
	for (int i = 0; i < attributes.getLength(); i++) {
		Node node = attributes.item(i);
		finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
	}

	// Decorate based on custom nested elements.
    /**	获取自定义子节点元素*/
	NodeList children = ele.getChildNodes();
	for (int i = 0; i < children.getLength(); i++) {
		Node node = children.item(i);
		if (node.getNodeType() == Node.ELEMENT_NODE) {
			finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
		}
	}
	return finalDefinition;
}
```

当两个方法都处理完毕后，会返回一个`BeanDefinitionHolder`对象，这个对象就是对`BeanDefinition`信息的封装，至此，加载阶段也完成了所有工作，时序图如下：

![image-20180721172120316](https://ws1.sinaimg.cn/large/006tNc79gy1fthlv6mbdij315w16q797.jpg)

### 注册

注册阶段主要就是将生成的`BeanDefinitionHolder`对象注册到`Ioc`容器中，注册逻辑如下

```java
# 
public static void registerBeanDefinition(
		BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
		throws BeanDefinitionStoreException {

	// Register bean definition under primary name.
	String beanName = definitionHolder.getBeanName();
	registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

	// Register aliases for bean name, if any.
	String[] aliases = definitionHolder.getAliases();
	if (aliases != null) {
		for (String alias : aliases) {
			registry.registerAlias(beanName, alias);
		}
	}
}
```

在这里主要是将`BeanDefinitionHolder`中的`BeanDefinition`信息及`beanName`、`alias`信息等注册到`Ioc`容器中，在代码中容器是`BeanDefinitionRegistry`，而实际上为`DefaultListableBeanFactory`；

在`ClassPathXmlApplicationContext`中创建完`DefaultListableBeanFactory`对象后，实际上引用被放入到了`XmlBeanDefinitionReader`中的`registry`属性中，而`XmlBeanDefinitionReader`又被放置到`XmlReaderContext`的`reader`属性中。而上面的`BeanDefinitionRegistry`对象实际上就是`XmlReaderContext.reader`；

所以最终会调用`DefaultListableBeanFactory.registerBeanDefinition`，实现如下：

```java
@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
		throws BeanDefinitionStoreException {

	Assert.hasText(beanName, "Bean name must not be empty");
	Assert.notNull(beanDefinition, "BeanDefinition must not be null");

	if (beanDefinition instanceof AbstractBeanDefinition) {
		try {
			((AbstractBeanDefinition) beanDefinition).validate();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
					"Validation of bean definition failed", ex);
		}
	}

	BeanDefinition oldBeanDefinition;

	oldBeanDefinition = this.beanDefinitionMap.get(beanName);
	if (oldBeanDefinition != null) {
		if (!isAllowBeanDefinitionOverriding()) {
			throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
					"Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
					"': There is already [" + oldBeanDefinition + "] bound.");
		}
		else if (oldBeanDefinition.getRole() < beanDefinition.getRole()) {
			// e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
			if (this.logger.isWarnEnabled()) {
				this.logger.warn("Overriding user-defined bean definition for bean '" + beanName +
						"' with a framework-generated bean definition: replacing [" +
						oldBeanDefinition + "] with [" + beanDefinition + "]");
			}
		}
		else if (!beanDefinition.equals(oldBeanDefinition)) {
			if (this.logger.isInfoEnabled()) {
				this.logger.info("Overriding bean definition for bean '" + beanName +
						"' with a different definition: replacing [" + oldBeanDefinition +
						"] with [" + beanDefinition + "]");
			}
		}
		else {
			if (this.logger.isDebugEnabled()) {
				this.logger.debug("Overriding bean definition for bean '" + beanName +
						"' with an equivalent definition: replacing [" + oldBeanDefinition +
						"] with [" + beanDefinition + "]");
			}
		}
		this.beanDefinitionMap.put(beanName, beanDefinition);
	}
	else {
		if (hasBeanCreationStarted()) {
			// Cannot modify startup-time collection elements anymore (for stable iteration)
			synchronized (this.beanDefinitionMap) {
				this.beanDefinitionMap.put(beanName, beanDefinition);
				List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
				updatedDefinitions.addAll(this.beanDefinitionNames);
				updatedDefinitions.add(beanName);
				this.beanDefinitionNames = updatedDefinitions;
				if (this.manualSingletonNames.contains(beanName)) {
					Set<String> updatedSingletons = new LinkedHashSet<>(this.manualSingletonNames);
					updatedSingletons.remove(beanName);
					this.manualSingletonNames = updatedSingletons;
				}
			}
		}
		else {
			// Still in startup registration phase
			this.beanDefinitionMap.put(beanName, beanDefinition);
			this.beanDefinitionNames.add(beanName);
			this.manualSingletonNames.remove(beanName);
		}
		this.frozenBeanDefinitionNames = null;
	}

	if (oldBeanDefinition != null || containsSingleton(beanName)) {
		resetBeanDefinition(beanName);
	}
}
```

查看以上代码可以发现`Ioc`容器`DefaultListableBeanFactory`实际上就是一个`ConcurrentHashMap`对象；

## DI（依赖注入）

`DI`全称为`Dependency Injection`，也就是依赖注入；在Spring容器将所有`Bean`对象注册进`Map`结构后，在`Map`中保留的依旧是`BeanDefinition`对象，明显不是我们实际需要用到的对象。

而在Spring中，有两种不同的容器

* XmlBeanFactory：需要在我们显示获取`Bean`的时候才进行实例化
* ClassPathXmlApplicationContext：在启动阶段就会对`Bean`实例化

Spring容器在将`Bean`对象进行实例化时，会将对应`Bean`依赖的对象也进行实例化，最终注入进去到目标`Bean`中。这个主动将依赖`Bean`设置到目标`Bean`的操作就是依赖注入，而不用我们手动设置；

在`ClassPathXmlApplicationContext`中，在`refersh`的调用中，当`bean`注册完毕后，会通过调用以下方法来完成创建`bean`对象的操作；

```java
# AbstractApplicationContext
finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory);
```

实际上会调用到`DefaultListableBeanFactory.preInstantiateSingletons`中，代码如下

```java
public void preInstantiateSingletons() throws BeansException {
	if (this.logger.isDebugEnabled()) {
		this.logger.debug("Pre-instantiating singletons in " + this);
	}

	// Iterate over a copy to allow for init methods which in turn register new bean definitions.
	// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
	List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

	// Trigger initialization of all non-lazy singleton beans...
	for (String beanName : beanNames) {
		RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
		if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
			if (isFactoryBean(beanName)) {
				Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
				if (bean instanceof FactoryBean) {
					final FactoryBean<?> factory = (FactoryBean<?>) bean;
					boolean isEagerInit;
					if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
						isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
										((SmartFactoryBean<?>) factory)::isEagerInit,
								getAccessControlContext());
					}
					else {
						isEagerInit = (factory instanceof SmartFactoryBean &&
								((SmartFactoryBean<?>) factory).isEagerInit());
					}
					if (isEagerInit) {
						getBean(beanName);
					}
				}
			}
			else {
				getBean(beanName);
			}
		}
	}

	// Trigger post-initialization callback for all applicable beans...
	for (String beanName : beanNames) {
		Object singletonInstance = getSingleton(beanName);
		if (singletonInstance instanceof SmartInitializingSingleton) {
			final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
			if (System.getSecurityManager() != null) {
				AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
					smartSingleton.afterSingletonsInstantiated();
					return null;
				}, getAccessControlContext());
			}
			else {
				smartSingleton.afterSingletonsInstantiated();
			}
		}
	}
}
```

可以发现如果容器发现注册进来的`BeanDefinition`对象不是抽象、是单例且没有配置成懒加载的，那么就会通过`getBean`方法进行获取`bean`对象的操作，而在`Bean`对象没有进行实例化之前，调用这个方法会进行实例化且依赖注入的操作；

在调用时，`getBean`会调用`doGetBean`方法，一开始`bean`没有初始化过，所以相应的`Map`结构中肯定没有对应的对象，最终会走到`createBean`方法，而`createBean`实际上是调用`doCreateBean`实现的；`doCreateBean`中会通过`createBeanInstance`方法来创建一个`BeanWrapper`的`Bean`装饰器；

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
	// Make sure bean class is actually resolved at this point.
	Class<?> beanClass = resolveBeanClass(mbd, beanName);

	if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
		throw new BeanCreationException(mbd.getResourceDescription(), beanName,
				"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
	}

	Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
	if (instanceSupplier != null) {
		return obtainFromSupplier(instanceSupplier, beanName);
	}

	if (mbd.getFactoryMethodName() != null)  {
		return instantiateUsingFactoryMethod(beanName, mbd, args);
	}

	// Shortcut when re-creating the same bean...
	boolean resolved = false;
	boolean autowireNecessary = false;
	if (args == null) {
		synchronized (mbd.constructorArgumentLock) {
			if (mbd.resolvedConstructorOrFactoryMethod != null) {
				resolved = true;
				autowireNecessary = mbd.constructorArgumentsResolved;
			}
		}
	}
	if (resolved) {
		if (autowireNecessary) {
			return autowireConstructor(beanName, mbd, null, null);
		}
		else {
			return instantiateBean(beanName, mbd);
		}
	}

	// Need to determine the constructor...
	Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
	if (ctors != null ||
			mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
			mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
		return autowireConstructor(beanName, mbd, ctors, args);
	}

	// No special handling: simply use no-arg constructor.
	return instantiateBean(beanName, mbd);
}
```

在创建`BeanWrapper`的时候要分为两种情况：

* 如果配置文件配置了构造方法注入，那么实际上会通过`autowireConstructor(beanName, mbd, ctors, args);`来创建对象，且在创建之前会先实例化依赖对象；
* 反之，会通过`instantiateBean(beanName, mbd);`来创建对象，再在后面的`populateBean`方法中设置依赖对象属性；

> 实际上通过autowireConstructor或者instantiateBean都是一样通过反射调用构造器来完成实例化操作，只是autowireConstructor可能会存在依赖其他bean对象，所以需要进行对依赖对象的实例化再完成当前对象的实例化；

如果是通过`autowireConstructor`来实现依赖注入的，那么实际上是调用

```java
# ConstructorResolver
autowireConstructor(final String beanName, final RootBeanDefinition mbd,
			@Nullable Constructor<?>[] chosenCtors, @Nullable final Object[] explicitArgs)
```
在方法内部会通过反射获取到类对应的所有构造方法及根据`RootBeanDefinition`中的参数对象，来找到对应的构造方法，并通过反射进行调用；而在进行相应的操作前，需要先定位到构造方法中指定的参数对象，通过以下代码来完成；

```java
# ConstructorResolver
minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
```
内部实际会调用`valueResolver.resolveValueIfNecessary`方法，方法的内部会处理各种不同类型的参数对象
```java
public Object resolveValueIfNecessary(Object argName, @Nullable Object value) {
	// We must check each value to see whether it requires a runtime reference
	// to another bean to be resolved.
	if (value instanceof RuntimeBeanReference) {
		RuntimeBeanReference ref = (RuntimeBeanReference) value;
		return resolveReference(argName, ref);
	}
	else if (value instanceof RuntimeBeanNameReference) {
		String refName = ((RuntimeBeanNameReference) value).getBeanName();
		refName = String.valueOf(doEvaluate(refName));
		if (!this.beanFactory.containsBean(refName)) {
			throw new BeanDefinitionStoreException(
					"Invalid bean name '" + refName + "' in bean reference for " + argName);
		}
		return refName;
	}
	else if (value instanceof BeanDefinitionHolder) {
		// Resolve BeanDefinitionHolder: contains BeanDefinition with name and aliases.
		BeanDefinitionHolder bdHolder = (BeanDefinitionHolder) value;
		return resolveInnerBean(argName, bdHolder.getBeanName(), bdHolder.getBeanDefinition());
	}
	else if (value instanceof BeanDefinition) {
		// Resolve plain BeanDefinition, without contained name: use dummy name.
		BeanDefinition bd = (BeanDefinition) value;
		String innerBeanName = "(inner bean)" + BeanFactoryUtils.GENERATED_BEAN_NAME_SEPARATOR +
				ObjectUtils.getIdentityHexString(bd);
		return resolveInnerBean(argName, innerBeanName, bd);
	}
	else if (value instanceof ManagedArray) {
		// May need to resolve contained runtime references.
		ManagedArray array = (ManagedArray) value;
		Class<?> elementType = array.resolvedElementType;
		if (elementType == null) {
			String elementTypeName = array.getElementTypeName();
			if (StringUtils.hasText(elementTypeName)) {
				try {
					elementType = ClassUtils.forName(elementTypeName, this.beanFactory.getBeanClassLoader());
					array.resolvedElementType = elementType;
				}
				catch (Throwable ex) {
					// Improve the message by showing the context.
					throw new BeanCreationException(
							this.beanDefinition.getResourceDescription(), this.beanName,
							"Error resolving array type for " + argName, ex);
				}
			}
			else {
				elementType = Object.class;
			}
		}
		return resolveManagedArray(argName, (List<?>) value, elementType);
	}
	else if (value instanceof ManagedList) {
		// May need to resolve contained runtime references.
		return resolveManagedList(argName, (List<?>) value);
	}
	else if (value instanceof ManagedSet) {
		// May need to resolve contained runtime references.
		return resolveManagedSet(argName, (Set<?>) value);
	}
	else if (value instanceof ManagedMap) {
		// May need to resolve contained runtime references.
		return resolveManagedMap(argName, (Map<?, ?>) value);
	}
	else if (value instanceof ManagedProperties) {
		Properties original = (Properties) value;
		Properties copy = new Properties();
		original.forEach((propKey, propValue) -> {
			if (propKey instanceof TypedStringValue) {
				propKey = evaluate((TypedStringValue) propKey);
			}
			if (propValue instanceof TypedStringValue) {
				propValue = evaluate((TypedStringValue) propValue);
			}
			if (propKey == null || propValue == null) {
				throw new BeanCreationException(
						this.beanDefinition.getResourceDescription(), this.beanName,
						"Error converting Properties key/value pair for " + argName + ": resolved to null");
			}
			copy.put(propKey, propValue);
		});
		return copy;
	}
	else if (value instanceof TypedStringValue) {
		// Convert value to target type here.
		TypedStringValue typedStringValue = (TypedStringValue) value;
		Object valueObject = evaluate(typedStringValue);
		try {
			Class<?> resolvedTargetType = resolveTargetType(typedStringValue);
			if (resolvedTargetType != null) {
				return this.typeConverter.convertIfNecessary(valueObject, resolvedTargetType);
			}
			else {
				return valueObject;
			}
		}
		catch (Throwable ex) {
			// Improve the message by showing the context.
			throw new BeanCreationException(
					this.beanDefinition.getResourceDescription(), this.beanName,
					"Error converting typed String value for " + argName, ex);
		}
	}
	else if (value instanceof NullBean) {
		return null;
	}
	else {
		return evaluate(value);
	}
}
```
如果我们在构造方法中配置`ref`依赖属性，那么最终会调用以下方法进行获取依赖`bean`对象

```java
private Object resolveReference(Object argName, RuntimeBeanReference ref) {
	try {
		Object bean;
		String refName = ref.getBeanName();
		refName = String.valueOf(doEvaluate(refName));
		if (ref.isToParent()) {
			if (this.beanFactory.getParentBeanFactory() == null) {
				throw new BeanCreationException(
						this.beanDefinition.getResourceDescription(), this.beanName,
						"Can't resolve reference to bean '" + refName +
						"' in parent factory: no parent factory available");
			}
			bean = this.beanFactory.getParentBeanFactory().getBean(refName);
		}
		else {
			bean = this.beanFactory.getBean(refName);
			this.beanFactory.registerDependentBean(refName, this.beanName);
		}
		if (bean instanceof NullBean) {
			bean = null;
		}
		return bean;
	}
	catch (BeansException ex) {
		throw new BeanCreationException(
				this.beanDefinition.getResourceDescription(), this.beanName,
				"Cannot resolve reference to bean '" + ref.getBeanName() + "' while setting " + argName, ex);
	}
}
```

**所以在初始化的时候，如果依赖的对应还没有进行实例化操作，那么会对依赖对象进行实例化操作；**

在获取到目标对象之后，最终会调用到`SimpleInstantiationStrategy.instantiate`方法，最终调用到`BeanUtils.instantiateClass`中通过反射进行初始化操作；

在设置完通过构造方法指定的参数后，还会通过以下方法来装配其他通过`property`指定的属性

```java
# AbstractAutowireCapableBeanFactory
populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw);
```

最终会通过`applyPropertyValues`来完成属性的设置；方法内部最终也还是通过调用`valueResolver.resolveValueIfNecessary(pv, originalValue);`来完成依赖属性的获取；最终调用`BeanWrapperImpl.setValue`通过反射找到对应的实体的`setXxx`方法完成属性的设置；

通过`populateBean`完成属性设置后，会通过`initializeBean`方法调用`Bean`的初始化操作，包括在`bean`上配置的`init-method`方法，及调用通过继承`InitializingBean`并实现的`afterPropertiesSet`方法；

时序图

![image-20180721235655940](https://ws4.sinaimg.cn/large/006tNc79gy1fthxat5t3rj31kw153af3.jpg)



>  Spring的容器可以将IOC跟DI分别成为容器的启动跟实例化阶段；而在对象实例化之前，Spring提供了`BeanFactoryPostProcessor`的接口类来支持对`BeanDefinition`进行修改，如配置文件中的占位符，就是通过`PropertyPlaceholderConfigurer`来实现的；



## AOP（面向切面编程）

`AOP`全称为`Aspect Oriented Programming`，即**面向切面编程**；AOP主要是通过代理机制来实现的。

* JDK动态代理：JDK自带的代理机制，目标类对象必须实现接口；通过`Proxy.newProxyInstance`来实现；
* CGLIB：目标类对象没有实现接口，通过字节码增强，创建目标对象的子类，完成代码织入；

Spring在`Bean`对象的实例化时候，会通过调用`BeanPostProcessor`的`postProcessBeforeInitialization`及`postProcessAfterInitialization`来完成前置及后置处理，而AOP的功能就是通过后置处理来实现的；

```java
AbstractAutowireCapableBeanFactory#initializeBean --> AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsAfterInitialization -->
AnnotationAwareAspectJAutoProxyCreator#postProcessAfterInitialization
```

具体实现代码逻辑：

````java
# AbstractAutoProxyCreator
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) throws BeansException {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (!this.earlyProxyReferences.contains(cacheKey)) {
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}

protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
	if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
		return bean;
	}
	if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
		return bean;
	}
	if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}
	// Create proxy if we have advice.
	Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
	if (specificInterceptors != DO_NOT_PROXY) {
		this.advisedBeans.put(cacheKey, Boolean.TRUE);
		Object proxy = createProxy(
				bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
		this.proxyTypes.put(cacheKey, proxy.getClass());
		return proxy;
	}

	this.advisedBeans.put(cacheKey, Boolean.FALSE);
	return bean;
}
````

`wrapIfNecessary`用于判断是否有必要执行织入的逻辑，会判断当前`Bean`是否为`Advice、Pointcut、Advisor、AopInfrastructureBean`中的某一个接口或者接口实现类，如果是的话跳过，否则通过`getAdvicesAndAdvisorsForBean`进行判断当前`Bean`是否符合配置的切入逻辑，如果符合，那么通过`createProxy`方法来生成代理对象；

`getAdvicesAndAdvisorsForBean`的方法调用链条为：

```java
AbstractAdvisorAutoProxyCreator#getAdvicesAndAdvisorsForBean --> AbstractAdvisorAutoProxyCreator#findEligibleAdvisors 
```

```java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
    List<Advisor> candidateAdvisors = findCandidateAdvisors();
    List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
    extendAdvisors(eligibleAdvisors);
    if (!eligibleAdvisors.isEmpty()) {
        eligibleAdvisors = sortAdvisors(eligibleAdvisors);
    }
    return eligibleAdvisors;
}
```

`findCandidateAdvisors`的关键调用链条为：

```java
AbstractAdvisorAutoProxyCreator#findCandidateAdvisors --> 
AspectJAwareAdvisorAutoProxyCreator#findCandidateAdvisors -->
BeanFactoryAspectJAdvisorsBuilder#buildAspectJAdvisors
```

`buildAspectJAdvisors`方法的作用为获取所有的切面织入配置；

在获取到切面织入配置后，会通过`findAdvisorsThatCanApply`方法来查找与当前的`Bean`信息匹配的切面配置信息，通过`AopUtils.findAdvisorsThatCanApply`实现；

`findAdvisorsThatCanApply`的关键调用链条为：

```java
AbstractAdvisorAutoProxyCreator#findAdvisorsThatCanApply -->
AopUtils#findAdvisorsThatCanApply -->
AopUtils#canApply
```

如果最终发现有符合的切面配置信息，那么Spring会通过`ProxyFactory`创建代理对象，而`ProxyFactory`会通过创建`JdkDynamicAopProxy`或者`CglibAopProxy`来完成创建代理对象的动作；

当代理对象创建完成后，当我们调用具体的方法的时候，实际上调用的就是代理类，以`JdkDynamicAopProxy`为例，调用某个方法会直接调用`JdkDynamicAopProxy.invoke`方法；代码实现如下：

```java
# JdkDynamicAopProxy
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	MethodInvocation invocation;
	Object oldProxy = null;
	boolean setProxyContext = false;

	TargetSource targetSource = this.advised.targetSource;
	Object target = null;

	try {
		if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
			// The target does not implement the equals(Object) method itself.
			return equals(args[0]);
		}
		else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
			// The target does not implement the hashCode() method itself.
			return hashCode();
		}
		else if (method.getDeclaringClass() == DecoratingProxy.class) {
			// There is only getDecoratedClass() declared -> dispatch to proxy config.
			return AopProxyUtils.ultimateTargetClass(this.advised);
		}
		else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
				method.getDeclaringClass().isAssignableFrom(Advised.class)) {
			// Service invocations on ProxyConfig with the proxy config...
			return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
		}

		Object retVal;

		if (this.advised.exposeProxy) {
			// Make invocation available if necessary.
			oldProxy = AopContext.setCurrentProxy(proxy);
			setProxyContext = true;
		}

		// Get as late as possible to minimize the time we "own" the target,
		// in case it comes from a pool.
		target = targetSource.getTarget();
		Class<?> targetClass = (target != null ? target.getClass() : null);

		// Get the interception chain for this method.
		List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

		// Check whether we have any advice. If we don't, we can fallback on direct
		// reflective invocation of the target, and avoid creating a MethodInvocation.
		if (chain.isEmpty()) {
			// We can skip creating a MethodInvocation: just invoke the target directly
			// Note that the final invoker must be an InvokerInterceptor so we know it does
			// nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
			Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
			retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
		}
		else {
			// We need to create a method invocation...
			invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
			// Proceed to the joinpoint through the interceptor chain.
			retVal = invocation.proceed();
		}

		// Massage return value if necessary.
		Class<?> returnType = method.getReturnType();
		if (retVal != null && retVal == target &&
				returnType != Object.class && returnType.isInstance(proxy) &&
				!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
			// Special case: it returned "this" and the return type of the method
			// is type-compatible. Note that we can't help if the target sets
			// a reference to itself in another returned object.
			retVal = proxy;
		}
		else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
			throw new AopInvocationException(
					"Null return value from advice does not match primitive return type for: " + method);
		}
		return retVal;
	}
	finally {
		if (target != null && !targetSource.isStatic()) {
			// Must have come from TargetSource.
			targetSource.releaseTarget(target);
		}
		if (setProxyContext) {
			// Restore old proxy.
			AopContext.setCurrentProxy(oldProxy);
		}
	}
}
```

在`JdkDynamicAopProxy`类初始化的时候，已经将切面的信息通过构造方法传递给`advised`属性了，所以通过字节获取`advised`属性，可以拿到代理类代理的真实对象、要切入的方法的信息等等，最终会通过构造成一个`List`结构的对象，以类似于过滤器链的方式，依次完成方法调用；

```java
// 获取方法配置的切面信息,拼凑成一个chain链条
List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method,targetClass);

// 构造invocation对象
invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass,chain);

// 处理链条中的切入方法
// Proceed to the joinpoint through the interceptor chain.
retVal = invocation.proceed();
```

