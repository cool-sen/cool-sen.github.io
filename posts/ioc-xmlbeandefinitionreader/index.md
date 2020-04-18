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

   
