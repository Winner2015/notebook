EntityResolver   SAX

DelegatingEntityResolver  Spring中的委派类，内部持有两个实现类

PluggableSchemaResolver负责解析xsd

BeansDtdResolver负责解析dtd

ResourceEntityResolver是DelegatingEntityResolver  子类

```java
public InputSource resolveEntity(String publicId, String systemId) throws SAXException, IOException {
    	//依靠子类加载对应的xsd文件或者dtd文件
		InputSource source = super.resolveEntity(publicId, systemId);
		if (source == null && systemId != null) { //找不到会从网上下载
			String resourcePath = null;
			try {
				String decodedSystemId = URLDecoder.decode(systemId, "UTF-8");
				String givenUrl = new URL(decodedSystemId).toString();
				String systemRootUrl = new File("").toURI().toURL().toString();
				// Try relative to resource base if currently in system root.
				if (givenUrl.startsWith(systemRootUrl)) {
					resourcePath = givenUrl.substring(systemRootUrl.length());
				}
			}
			catch (Exception ex) {
				// Typically a MalformedURLException or AccessControlException.
				if (logger.isDebugEnabled()) {
					logger.debug("Could not resolve XML entity [" + systemId + "] against system root URL", ex);
				}
				// No URL (or no resolvable URL) -> try relative to resource base.
				resourcePath = systemId;
			}
			if (resourcePath != null) {
				if (logger.isTraceEnabled()) {
					logger.trace("Trying to locate XML entity [" + systemId + "] as resource [" + resourcePath + "]");
				}
				Resource resource = this.resourceLoader.getResource(resourcePath);
				source = new InputSource(resource.getInputStream());
				source.setPublicId(publicId);
				source.setSystemId(systemId);
				if (logger.isDebugEnabled()) {
					logger.debug("Found XML entity [" + systemId + "]: " + resource);
				}
			}
		}
		return source;
	}
```

DelegatingEntityResolver  

```java
public InputSource resolveEntity(String publicId, String systemId) throws IOException {
	if (logger.isTraceEnabled()) {
		logger.trace("Trying to resolve XML entity with public id [" + publicId +
				"] and system id [" + systemId + "]");
	}

	if (systemId != null) {
		String resourceLocation = getSchemaMappings().get(systemId);
		if (resourceLocation != null) {
			Resource resource = new ClassPathResource(resourceLocation, this.classLoader);
			try {
                //加载对应的xsd文件
				InputSource source = new InputSource(resource.getInputStream());
				source.setPublicId(publicId);
				source.setSystemId(systemId);
				if (logger.isDebugEnabled()) {
					logger.debug("Found XML schema [" + systemId + "] in classpath: " + resourceLocation);
				}
				return source;
			}
			catch (FileNotFoundException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Couldn't find XML schema [" + systemId + "]: " + resource, ex);
				}
			}
		}
	}
	return null;
}
```
systemId = http://www.springframework.org/schema/beans/spring-beans-4.3.xsd

```java
	private Map<String, String> getSchemaMappings() {
		if (this.schemaMappings == null) {
			synchronized (this) {
				if (this.schemaMappings == null) {
					if (logger.isDebugEnabled()) {
						logger.debug("Loading schema mappings from [" + this.schemaMappingsLocation + "]");
					}
					try {
						Properties mappings =
						//schemaMappingsLocation = META-INF/spring.schemas
                            PropertiesLoaderUtils.loadAllProperties(this.schemaMappingsLocation, this.classLoader);
						if (logger.isDebugEnabled()) {
							logger.debug("Loaded schema mappings: " + mappings);
						}
						Map<String, String> schemaMappings = new ConcurrentHashMap<String, String>(mappings.size());
						CollectionUtils.mergePropertiesIntoMap(mappings, schemaMappings);
						this.schemaMappings = schemaMappings;
					}
					catch (IOException ex) {
						throw new IllegalStateException(
								"Unable to load schema mappings from location [" + this.schemaMappingsLocation + "]", ex);
					}
				}
			}
		}
		return this.schemaMappings;
	}
```

PropertiesLoaderUtils

```java
	public static Properties loadAllProperties(String resourceName, ClassLoader classLoader) throws IOException {
		Assert.notNull(resourceName, "Resource name must not be null");
		ClassLoader classLoaderToUse = classLoader;
		if (classLoaderToUse == null) {
			classLoaderToUse = ClassUtils.getDefaultClassLoader();
		}
        //从所有包下搜索名称为"META-INF/spring.schemas"
		Enumeration<URL> urls = (classLoaderToUse != null ? classLoaderToUse.getResources(resourceName) :
				ClassLoader.getSystemResources(resourceName));
		Properties props = new Properties();
		while (urls.hasMoreElements()) {
			URL url = urls.nextElement();
			URLConnection con = url.openConnection();
			ResourceUtils.useCachesIfNecessary(con);
			InputStream is = con.getInputStream();
			try {
				if (resourceName.endsWith(XML_FILE_EXTENSION)) {
					props.loadFromXML(is);
				}
				else {
					props.load(is);
				}
			}
			finally {
				is.close();
			}
		}
		return props;
	}
```



META-INF/spring.schemas实际上就是一个properties文件，里面指定了systemId 到xsd文件路径的映射关系。

例如，spring-bean.jar下可以找到spring bead相关的文件：

```properties
http\://www.springframework.org/schema/beans/spring-beans-2.0.xsd=org/springframework/beans/factory/xml/spring-beans-2.0.xsd
http\://www.springframework.org/schema/beans/spring-beans-2.5.xsd=org/springframework/beans/factory/xml/spring-beans-2.5.xsd
...
http\://www.springframework.org/schema/util/spring-util.xsd=org/springframework/beans/factory/xml/spring-util-4.3.xsd

```

而dubbo包下可以找到dubbo相关的文件：

```properties
http\://dubbo.apache.org/schema/dubbo/dubbo.xsd=META-INF/dubbo.xsd
http\://code.alibabatech.com/schema/dubbo/dubbo.xsd=META-INF/compat/dubbo.xsd
```

将这些properties存入DelegatingEntityResolver  的schemaMappings

通过这个map，就可以根据systemId找到对应的xsd文件路径。