# IOC XMLBeanDefinitionReader


## 前言



## 1. Resource资源定位

Spring的配置文件读取是通过ClassPathResource进行封装的，如```new ClassPathResource ("beanFactoryTest.xml")```。

JavaSE中有个标准类 `java.net.URL`，Spring为何选择自造轮子？

在JavaSE中有个标准类 `java.net.URL`，该类为资源定位器（Uniform Resource Locator）。具体过程为：将不同来源的资源抽象成URL，通过注册不同的handler（URLStreamHandler）来处理不同来源的资源的读取逻辑，一般handler的类型使用不同前缀（协议，Protocol）来识别，如“file:”“http:”“jar:”等。但是URL没有默认定义相对Classpath或ServletContext等资源的handler，虽然可以注册自己的URLStreamHandler来解析特定的URL前缀（协议），比如“classpath:”，但URL没有提供基本的方法。

`java.net.URL` 的局限性迫使 Spring 必须实现自己的资源加载策略：Resource接口封装底层资源。

Resource接口代码如下：

```java
public interface Resource extends InputStreamSource {
	
	//资源是否存在，存在性
	boolean exists();
    
    //资源是否可读，可读性
	default boolean isReadable() {
		return true;
	}

	//资源所代表的句柄是否被一个 stream 打开了，打开状态
	default boolean isOpen() {
		return false;
	}

	//是否为 File
	default boolean isFile() {
		return false;
	}

	//返回资源的 URL 的句柄
	URL getURL() throws IOException;

	//返回资源的 URI 的句柄
	URI getURI() throws IOException;

	//返回资源的 File 的句柄
	File getFile() throws IOException;

	//返回 ReadableByteChannel
	default ReadableByteChannel readableChannel() throws IOException {
		return java.nio.channels.Channels.newChannel(getInputStream());
	}

	//资源内容的长度
	long contentLength() throws IOException;

	//资源最后的修改时间
	long lastModified() throws IOException;

	//根据当前资源创建一个相对资源
	Resource createRelative(String relativePath) throws IOException;

	//资源的文件名
	@Nullable
	String getFilename();

	//资源的描述，可在错误处理中详细地打印出错的资源文件
	String getDescription();

}
```

资源文件相关类图如下：

![资源文件处理相关类图](/images/IOC-XMLBeanDefinitionReader/image-20200418213207280.png)

<center> 资源文件处理相关类图</center>

- FileSystemResource ：对 `java.io.File` 类型资源的封装，只要是跟 File 打交道的，基本上与 FileSystemResource 也可以打交道。支持文件和 URL 的形式，实现 WritableResource 接口，且从 Spring Framework 5.0 开始，FileSystemResource 使用 NIO2 API进行读/写交互。
- ClassPathResource ：class path 类型资源的实现。使用给定的 ClassLoader 或者给定的 Class 来加载资源。
- UrlResource ：对 `java.net.URL`类型资源的封装。内部委派 URL 进行具体的资源操作。
- InputStreamResource ：将给定的 InputStream 作为一种资源的 Resource 的实现类。
- ByteArrayResource ：对字节数组提供的数据的封装。如果通过 InputStream 形式访问该类型的资源，该实现会根据字节数组的数据构造一个相应的 ByteArrayInputStream。

### 1.2 举例

如加载文件使用以下代码：

```java
Resource resource=new ClassPathResource("beanFactoryTest.xml");
InputStream inputStream=resource.getInputStream();
```

Resource接口对所有的资源文件进行统一处理。

ClassPathResource中的getInputStream，是通过class或者classLoader提供的底层方法进行调用。

```java
if (this.clazz != null) {
			is = this.clazz.getResourceAsStream(this.path);
		}
		else if (this.classLoader != null) {
			is = this.classLoader.getResourceAsStream(this.path);
		}
		else {
			is = ClassLoader.getSystemResourceAsStream(this.path);
		}
```

先通过Resource相关类对配置文件进行封装和读取，之后交给XmlBeanDefinitionReader处理。

## 2 XmlBeanDefinitionReader

1．封装资源文件。当进入XmlBeanDefinitionReader后，首先对参数Resource使用EncodedResource类进行封装。

```java
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
		return loadBeanDefinitions(new EncodedResource(resource));
	}
```

2．获取输入流。从Resource中获取对应的InputStream并构造InputSource。

转入可复用方法`loadBeanDefinitions(EncodedResource encodedResource) `。重点为调用函数`doLoadBeanDefinitions(inputSource, encodedResource.getResource())`

```java
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isTraceEnabled()) {
			logger.trace("Loading XML bean definitions from " + encodedResource);
		}
		//通过属性记录已经加载的资源
		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();

		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}
		//从encodedResource中获取已经封装的Resource对象,并再次从Resource中获取其中的inputStream
		try (InputStream inputStream = encodedResource.getResource().getInputStream()) {
			//注意：InputSource这个类并不来自于Spring，它的全路径是org.xml.sax.InputSource
			InputSource inputSource = new InputSource(inputStream);
			if (encodedResource.getEncoding() != null) {
				inputSource.setEncoding(encodedResource.getEncoding());
			}
			//**真正的核心部分**
			return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
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

3. doLoadBeanDefinitions 主要做了三件事。

   * 获取对XML文件的验证模式。
   * 加载XML文件，并得到对应的Document。
   * 根据返回的Document注册Bean信息。

   代码如下：

   ```java
   protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
   			throws BeanDefinitionStoreException {
   
   		try {
   			//1. doLoadDocument 中有getValidationModeForResource获取XML文件的验证模式
   			//2. doLoadDocument 得到对应的Document
   			Document doc = doLoadDocument(inputSource, resource);
   			//3. 根据 Document 实例，注册 Bean 信息
   			int count = registerBeanDefinitions(doc, resource);
   			if (logger.isDebugEnabled()) {
   				logger.debug("Loaded " + count + " bean definitions from " + resource);
   			}
   			return count;
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
   ```

   ## 3 注册 BeanDefinition 流程

   分析上一节的`registerBeanDefinitions`方法

   ```java
   //XmlBeanDefinitionReader.java
   
   public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
   		// 创建 BeanDefinitionDocumentReader 对象
   		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
   		// 获取已注册的 BeanDefinition 数量
   		int countBefore = getRegistry().getBeanDefinitionCount();
   		// 加载及注册bean,过程为：先创建 XmlReaderContext 对象，再注册 BeanDefinition
   		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
   		// 计算新注册的 BeanDefinition 数量
   		return getRegistry().getBeanDefinitionCount() - countBefore;
   	}
   ```

   * createBeanDefinitionDocumentReader()方法，创建了DefaultBeanDefinitionDocumentReader实例。注意BeanDefinitionDocumentReader是一个接口，DefaultBeanDefinitionDocumentReader实现了这个接口。

   * 调用该 BeanDefinitionDocumentReader 的 `registerBeanDefinitions(Document doc, XmlReaderContext readerContext)` 方法，开启解析过程。

     解析过程如下：

     ```java
     // DefaultBeanDefinitionDocumentReader.java
     
     @Override
     public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
         this.readerContext = readerContext;
         // 获得 XML Document Root Element
         // 执行注册 BeanDefinition
         doRegisterBeanDefinitions(doc.getDocumentElement());
     }
     ```

   ### 3.1 doRegisterBeanDefinitions方法分析

   ```java
   // DefaultBeanDefinitionDocumentReader.java
   
   protected void doRegisterBeanDefinitions(Element root) {
   		BeanDefinitionParserDelegate parent = this.delegate;
   		this.delegate = createDelegate(getReaderContext(), root, parent);
   		//非核心代码
   		if (this.delegate.isDefaultNamespace(root)) {
   			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
   			if (StringUtils.hasText(profileSpec)) {
   				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
   						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
   				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
   					if (logger.isDebugEnabled()) {
   						logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
   								"] not matching: " + getReaderContext().getResource());
   					}
   					return;
   				}
   			}
   		}
   		//核心代码
       	//解析前处理，留给子类实现
   		preProcessXml(root);
       	// 解析
   		parseBeanDefinitions(root, this.delegate);
       	//解析后处理，留给子类实现
   		postProcessXml(root);
   
   		this.delegate = parent;
   	}
   ```

   * `preProcessXml(Element root)`、`postProcessXml(Element root)` 为前置、后置增强处理，目前 Spring 中都是空实现。这是模板方法设计模式的体现。如果需要在Bean解析前后做处理的话，只需要继承`DefaultBeanDefinitionDocumentReader`,重写这两个方法即可。
   * `parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate)` 是对根元素 root 的解析注册过程。

### 3.2 parseBeanDefinitions方法分析

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		//// 如果根节点使用默认命名空间，执行默认解析
		if (delegate.isDefaultNamespace(root)) {
			// 遍历子节点
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					// 如果该节点使用默认命名空间，执行默认解析
					if (delegate.isDefaultNamespace(ele)) {
						parseDefaultElement(ele, delegate);
					}
					// 如果该节点非默认命名空间，执行自定义解析
					else {
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		// 如果根节点非默认命名空间，执行自定义解析
		else {
			delegate.parseCustomElement(root);
		}
	}
```

* 若节点为默认命名空间，调用 `parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate)` 方法，开启默认标签的解析注册过程
* 若节点为自定义命名空间，调用`parseCustomElement(Element ele)` 方法，开启自定义标签的解析注册过程。
