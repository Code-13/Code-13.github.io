---
title: Spring-framework源码分析（三）统一资源加载
abbrlink: 5eef8624
categories:
  - Spring-framework源码分析
date: 2020-07-22 19:54:10
tags:
  - 源码分析
  - Spring
  - Spring-framework
cover: 'https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/20/20200720195606.jpg'
coverWidth: 1200
coverHeight: 750
---

## 关于统一资源和资源加载策略

官网对于 org.springframework.core.io.Resource 的一段说明： https://docs.spring.io/spring-framework/docs/5.2.7.RELEASE/spring-framework-reference/core.html#resources-introduction

`Java’s standard java.net.URL class and standard handlers for various URL prefixes, unfortunately, are not quite adequate enough for all access to low-level resources. For example, there is no standardized URL implementation that may be used to access a resource that needs to be obtained from the classpath or relative to a ServletContext. While it is possible to register new handlers for specialized URL prefixes (similar to existing handlers for prefixes such as http:), this is generally quite complicated, and the URL interface still lacks some desirable functionality, such as a method to check for the existence of the resource being pointed to.`

　　Spring’s Resource interface is meant to be a more capable interface for abstracting access to low-level resources.

　　`在 Java 中，将不同来源的资源抽象成 java.net.URL ，即统一资源定位器（Uniform Resource Locator），然后通过注册不同的 handler ( URLStreamHandler )来处理不同来源的资源的读取逻辑，一般 handler 的类型使用不同前缀（协议， Protocol ）来识别，如“file:”“http:” “jar:”等，然而 URL 没有默认定义相对 Classpath 或 ServletContext 等资源的 handler ，虽然可以注册自己的 URLStreamHandler 来解析特定的 URL 前缀（协议）， 比如“classpath:”，然而这需要了解 Url 的实现机制，实现也比较复杂，而且 Url 也没有提供一些基本的方法，如检查当前资源是否存在、检查当前资源是否可读等方法。 因而 Spring 对其内部使用到的资源实现了自己的抽象结构 ： Resource 接口封装底层资源，（《Spring源码深度解析 第二版》，略微修改）然后通过 ResourceLoader 接口来实现 Resource 的加载策略，也即是提供了统一的资源定义和资源加载策略的抽象。通过不同策略进行的所有资源加载，都可以返回统一的抽象给客户端，客户端对资源可以进行的操作，则由 Resource 接口进行界定，具体如何处理，则交由不同来源的资源实现类来实现。`

- `Resource` ：提供统一的资源定义抽象，界定了对资源可以进行的处理操作。例如：文件资源（ FileSystemResource ） 、 Classpath 资源（ ClassPathResource ）、 URL 资源（ UrlResource ）、 InputStream 资源（ InputStreamResource ） 、Byte 数组（ ByteArrayResource ）等 。
- `ResourceLoader` ：提供统一的资源加载策略抽象，返回统一的 Resource 资源抽象给客户端。

<!--more-->

## 统一资源 Resource

### Resource 接口定义

`org.springframework.core.io.Resource` 为 Spring 框架用到的所有资源提供了一个统一的抽象接口，它继承了 `org.springframework.core.io.InputStreamSource` 接口，而 InputStreamSource 接口则用于将对应的资源封装为 Java 的标准 InputStream

Resource 接口是具体资源访问策略的抽象，也是所有资源访问类所实现的接口。 Resource 接口提供的通用方法如下（详情可参考 API 文档 or 源码注释）：


```java
public interface Resource extends InputStreamSource {

    /**
     * 检查资源是否物理形式实际存在
     */
    boolean exists();

    /**
     * 资源是否可读取
     */
    default boolean isReadable() {
        return exists();
    }

    /**
     * 资源文件是否打开状态，如果资源文件不能多次读取，每次读取结束应该显式关闭，以防止资源泄漏。
     */
    default boolean isOpen() {
        return false;
    }

    /**
     * 是否是文件系统中的文件 File
     */
    default boolean isFile() {
        return false;
    }

    /**
     * 返回资源对应的 URL 句柄
     */
    URL getURL() throws IOException;

    /**
     * 返回资源对应的 URI 句柄
     */
    URI getURI() throws IOException;

    /**
     * 返回资源对应的 File 句柄
     */
    File getFile() throws IOException;

    /**
     * 返回 ReadableByteChannel
     */
    default ReadableByteChannel readableChannel() throws IOException {
        return Channels.newChannel(getInputStream());
    }

    /**
     * 返回资源的内容长度
     */
    long contentLength() throws IOException;

    /**
     * 资源的最后修改时间
     */
    long lastModified() throws IOException;

    /**
     * 根据资源的相对路径创建对应的资源
     */
    Resource createRelative(String relativePath) throws IOException;

    /**
     * 资源的文件名，不带路径信息的文件名
     */
    @Nullable
    String getFilename();

    /**
     * 资源的描述信息，用来在错误处理中打印信息 。
     */
    String getDescription();

}
```

InputStreamSource 接口定义如下：

```java
public interface InputStreamSource {

   /**
    * 返回底层资源的标准输入流。每次调用都返回新的输入流，调用者必须负责关闭输入流。
    */
   InputStream getInputStream() throws IOException;

}
```

### Resource 继承体系

![](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/22/20200722200745.png)

底层资源可能会有各种来源，像文件系统、Url、classpath，甚至是 servletcontext 等，因此，Resource 需要根据资源的不同类型提供不同的具体实现，如 文件（ FileSystemResource ） 、 Classpath 资源（ ClassPathResource ）、 URL 资源（ UrlResource ）、 InputStream 资源（ InputStreamResource ） 、Byte 数组（ ByteArrayResource ）等。

常见的 Resource 实现类：(摘自文档，简单的翻译一下)

- `ClassPathResource` ：class path 类型资源的封装实现类，内部使用给定的 ClassLoader 或者给定的 Class 进行资源加载。
  - Resource implementation for class path resources. Uses either a given ClassLoader or a given Class for loading resources.
  - Supports resolution as java.io.File if the class path resource resides in the file system, but not for resources in a JAR. Always supports resolution as URL.


- `FileSystemResource` ：java.io.File 和 java.nio.file.Path 类型资源的封装实现类，用于处理文件系统资源。支持 File 和 Url 的形式。从 Spring Framework 5.0 开始使用 NIO2 进行 读/写 交互。从 5.1 开始，还可能是通过 Path 句柄来进行构造，这种场景下它将通过 NIO2进行所有的文件系统交互；只有通过 getFile() 时才转转为 File。
  - Resource implementation for java.io.File and java.nio.file.Path handles with a file system target. Supports resolution as a File and also as a URL. Implements the extended WritableResource interface.
  - Note: As of Spring Framework 5.0, this Resource implementation uses NIO.2 API for read/write interactions. As of 5.1, it may be constructed with a Path handle in which case it will perform all file system interactions via NIO.2, only resorting to File on getFile().


- `ByteArrayResource` ：对 byte 数组的封装实现类，会根据给定的 byte 数组构造一个对应的 ByteArrayInputStream 作为 InputStream 类型的返回。
  - Resource implementation for a given byte array.
  - Creates a ByteArrayInputStream for the given byte array.
  - Useful for loading content from any given byte array, without having to resort to a single-use InputStreamResource. Particularly useful for creating mail attachments from local content, where JavaMail needs to be able to read the stream multiple times.


- `UrlResource` ：对 java.net.URL 类型资源的封装实现类。支持 URL 和 File（使用 file: 协议的时候）的形式
  - Resource implementation for java.net.URL locators. Supports resolution as a URL and also as a File in case of the "file:" protocol.


- `InputStreamResource` ：将给定的 InputStream 作为资源的封装实现类。只有当其他类型都无法使用的时候才会用到，尽量使用相匹配的类型进行处理。
  - Resource implementation for a given InputStream.
  - Should only be used if no other specific Resource implementation is applicable. In particular, prefer ByteArrayResource or any of the file-based Resource implementations where possible.
  - In contrast to other Resource implementations, this is a descriptor for an already opened resource - therefore returning true from isOpen(). Do not use an InputStreamResource if you need to keep the resource descriptor somewhere, or if you need to read from a stream multiple times.


### AbstractResource

**`AbstractResource` 为 `Resource` 接口的默认实现，它实现了 `Resource` 接口的大部分的公共实现，作为 `Resource` 接口中的重中之重，其定义如下**

```java
public abstract class AbstractResource implements Resource {

    /**
     * 检查文件是否存在，或者检查有对应的流 InputStream 存在，此时需要关闭流
     */
    @Override
    public boolean exists() {
        // Try file existence: can we find the file in the file system?
        // 先判断文件 File 是否存在：基于 File 的判断
        if (isFile()) {
            try {
                return getFile().exists();
            }
            catch (IOException ex) {
                Log logger = LogFactory.getLog(getClass());
                if (logger.isDebugEnabled()) {
                    logger.debug("Could not retrieve File for existence check of " + getDescription(), ex);
                }
            }
        }
        // Fall back to stream existence: can we open the stream?
        // 其次检查是否是可以打开的流 InputStream：基于 InputStream 的判断
        try {
            getInputStream().close();
            return true;
        }
        catch (Throwable ex) {
            Log logger = LogFactory.getLog(getClass());
            if (logger.isDebugEnabled()) {
                logger.debug("Could not retrieve InputStream for existence check of " + getDescription(), ex);
            }
            return false;
        }
    }

    /**
     * 对于存在的资源，此实现总是返回true(从5.1版修订)，通过 exists() 进行判断
     */
    @Override
    public boolean isReadable() {
        return exists();
    }

    /**
     * 直接返回 false，表示未打开
     */
    @Override
    public boolean isOpen() {
        return false;
    }

    /**
     * 直接返回 false，表示不为 File，需要子类重写判断
     */
    @Override
    public boolean isFile() {
        return false;
    }

    /**
     * 直接抛出 FileNotFoundException 异常，需要子类实现
     */
    @Override
    public URL getURL() throws IOException {
        throw new FileNotFoundException(getDescription() + " cannot be resolved to URL");
    }

    /**
     * 基于 getURL() 返回的 URL 构建 URI
     */
    @Override
    public URI getURI() throws IOException {
        URL url = getURL();
        try {
            return ResourceUtils.toURI(url);
        }
        catch (URISyntaxException ex) {
            throw new NestedIOException("Invalid URI [" + url + "]", ex);
        }
    }

    /**
     * 直接抛出 FileNotFoundException 异常，需要子类实现
     */
    @Override
    public File getFile() throws IOException {
        throw new FileNotFoundException(getDescription() + " cannot be resolved to absolute file path");
    }

    /**
     * 返回根据 getInputStream() 的结果构建的 ReadableByteChannel
     */
    @Override
    public ReadableByteChannel readableChannel() throws IOException {
        return Channels.newChannel(getInputStream());
    }

    /**
     * 获取资源的长度。这里是通过全部读取来计算资源的字节长度
     */
    @Override
    public long contentLength() throws IOException {
        InputStream is = getInputStream();
        try {
            long size = 0;
            byte[] buf = new byte[256];
            int read;
            while ((read = is.read(buf)) != -1) {
                size += read;
            }
            return size;
        }
        finally {
            try {
                is.close();
            }
            catch (IOException ex) {
                Log logger = LogFactory.getLog(getClass());
                if (logger.isDebugEnabled()) {
                    logger.debug("Could not close content-length InputStream for " + getDescription(), ex);
                }
            }
        }
    }

    /**
     * 资源文件的最后的修改时间
     */
    @Override
    public long lastModified() throws IOException {
        File fileToCheck = getFileForLastModifiedCheck();
        long lastModified = fileToCheck.lastModified();
        if (lastModified == 0L && !fileToCheck.exists()) {
            throw new FileNotFoundException(getDescription() +
                    " cannot be resolved in the file system for checking its last-modified timestamp");
        }
        return lastModified;
    }

    /**
     * 返回相应的资源文件用于检查最后修改时间，内部直接使用 getFile() 实现
     */
    protected File getFileForLastModifiedCheck() throws IOException {
        return getFile();
    }

    /**
     * 直接抛出 FileNotFoundException 异常，需要子类实现
     */
    @Override
    public Resource createRelative(String relativePath) throws IOException {
        throw new FileNotFoundException("Cannot create a relative resource for " + getDescription());
    }

    /**
     * 获取资源名称，这里直接返回 null ，需要子类实现
     */
    @Override
    @Nullable
    public String getFilename() {
        return null;
    }

    @Override
    public boolean equals(@Nullable Object other) {
        return (this == other || (other instanceof Resource &&
                ((Resource) other).getDescription().equals(getDescription())));
    }

    @Override
    public int hashCode() {
        return getDescription().hashCode();
    }

    /**
     * 这里获取资源的描述信息作为返回
     */
    @Override
    public String toString() {
        return getDescription();
    }

}
```

- Spring 的很多顶层设计接口，都会有相关的 Abstract / Default 子类来处理一些典型的预实现，如果需要自定义这些接口的继承类，比如这里需要自定义的 Resource ，则建议直接继承 AbstractResource ，然后根据具体的资源特性重写相关的方法，而不是直接继承顶层接口 Resource，重写全部方法。


## 统一资源加载 ResourceLoader

`org.springframework.core.io.ResourceLoader` 为 Spring 资源加载的统一抽象，具体的资源加载则由相应的实现类来完成，所以我们可以将 ResourceLoader 称作为 **统一资源定位器**

### ResourceLoader接口定义

```java
public interface ResourceLoader {
    /** Pseudo URL prefix for loading from the class path: "classpath:". */
    // 用于从类路径加载的伪URL前缀:“classpath:”
    String CLASSPATH_URL_PREFIX = ResourceUtils.CLASSPATH_URL_PREFIX;


    /**
     * 根据指定的资源路径返回对应的资源 Resource，该 Resource 句柄
     * 应该总是一个可重用的资源描述符，允许多个 Resource.getInputStream()调用。
     * 需要注意的是该 Resource 句柄 并不确保对应的资源一定存在，需要调用
     * Resource.exists() 来进行实际的判断。
     * 该方法支持以下这些模式的资源加载：
     *      · 全限定路径 URL 位置的资源，如："file:C:/test.dat".
     *      · classpath 类路径位置的资源，如 "classpath:test.dat".
     *      · 相对路径的资源，如 "WEB-INF/test.dat". 这种情况下会根据不同实现返回不同的 Resource 实例
     */
    Resource getResource(String location);

    /**
     * 返回当前 ResourceLoader 所用到的 ClassLoader ，
     * 需要直接访问 ClassLoader 的客户端，可以通过 ResourceLoader 以这种统一的方式来直接获取 ClassLoader 。
     * Resource 中的实现类 ClassPathResource 可以根据指定的 ClassLoader 进行资源加载
     */
    @Nullable
    ClassLoader getClassLoader();

}
```

### ResourceLoader 的继承体系结构图

![](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/22/20200722202142.png)

### DefaultResourceLoader

#### DefaultResourceLoader 的内部属性

```java
// 类加载器
@Nullable
private ClassLoader classLoader;

// 用户自定义的特定协议资源加载解析策略接口，用于自定义加载策略
private final Set<ProtocolResolver> protocolResolvers = new LinkedHashSet<>(4);

// 各种类型资源的缓存
private final Map<Class<?>, Map<Resource, ?>> resourceCaches = new ConcurrentHashMap<>(4);
```

#### DefaultResourceLoader 的构造函数

```java
/**
 * 无参构造函数，这里内部实际上就是优先使用 Thread.currentThread().getContextClassLoader()
 */
public DefaultResourceLoader() {
    this.classLoader = ClassUtils.getDefaultClassLoader();
}

/**
 * 指定 ClassLoader 的带参构造函数
 */
public DefaultResourceLoader(@Nullable ClassLoader classLoader) {
    this.classLoader = classLoader;
}


/**
 * 设置指定的 ClassLoader，也可以使用默认的 Thread.currentThread().getContextClassLoader()
 */
public void setClassLoader(@Nullable ClassLoader classLoader) {
    this.classLoader = classLoader;
}

@Override
@Nullable
public ClassLoader getClassLoader() {
    return (this.classLoader != null ? this.classLoader : ClassUtils.getDefaultClassLoader());
}
```

DefaultResourceLoader 的构造函数也比较简单，一个不带参的空构造函数和一个指定 ClassLoader 的构造函数。

ClassLoader 还可以通过 setClassLoader() 来进行指定，这里可以使用 `ClassUtils.getDefaultClassLoader()`，内部也是优先使用 `Thread.currentThread().getContextClassLoader()` 来获取，也即是执行线程的 ClassLoader。 

#### ProtocolResolver 自定义的资源加载策略

`org.springframework.core.io.ProtocolResolver` 是特定协议的资源加载解析策略接口。

用作 `DefaultResourceLoader` 的SPI，允许在不继承 `DefaultResourceLoader`(或 `ApplicationContext` 实现类)的情况下处理自定义协议。

这样，如果需要自定义资源 `Resource` 和相应的加载策略，则可以通过继承 `AbstractResource` 来实现对应的资源，然后再增加相关的 `ProtocolResolver` 来实现对应的资源定位加载策略，这样就不需要再继承 `DefaultResourceLoader` 了。

ProtocolResolver接口定于如下

```java
/**
 * A resolution strategy for protocol-specific resource handles.
 *
 * <p>Used as an SPI for {@link DefaultResourceLoader}, allowing for
 * custom protocols to be handled without subclassing the loader
 * implementation (or application context implementation).
 *
 * <p>特定协议的资源加载解决策略接口。
 * 用作 DefaultResourceLoader 的SPI，允许在不继承 DefaultResourceLoader(或 ApplicationContext 实现类)的情况下处理自定义协议。
 *
 * @author Juergen Hoeller
 * @since 4.3
 * @see DefaultResourceLoader#addProtocolResolver
 */
@FunctionalInterface
public interface ProtocolResolver {

   /**
    * Resolve the given location against the given resource loader
    * if this implementation's protocol matches.
    * <p>根据指定的 ResourceLoader 来解析对应的资源路径，若成功则返回相应的 Resource
    * @param location the user-specified resource location 指定的资源路径
    * @param resourceLoader the associated resource loader 指定的 ResourceLoader
    * @return a corresponding {@code Resource} handle if the given location
    * matches this resolver's protocol, or {@code null} otherwise
    */
   @Nullable
   Resource resolve(String location, ResourceLoader resourceLoader);

}
```

Spring 并没有 `ProtocolResolver` 接口的任何实现类，这个完全需要用户自己进行定义实现，然后再通过 `DefaultResourceLoader.addProtocolResolver(ProtocolResolver resolver)` 注册到 Spring 中，该方法代码如下：

```java
/**
 * Register the given resolver with this resource loader, allowing for
 * additional protocols to be handled.
 * <p>Any such resolver will be invoked ahead of this loader's standard
 * resolution rules. It may therefore also override any default rules.
 * <p>注册自定义的特定协议资源加载解决器，允许处理其他协议的资源
 * 需要注意的是这些解析器在资源加载时会先执行，因此可能会覆盖其他默认的加载规则
 * @since 4.3
 * @see #getProtocolResolvers()
 */
public void addProtocolResolver(ProtocolResolver resolver) {
    Assert.notNull(resolver, "ProtocolResolver must not be null");
    this.protocolResolvers.add(resolver);
}
```

#### getResource() 方法

- getResource(String location) 是 DefaultResourceLoader 的核心方法


- 它会根据提供的 location 返回对应的 Resource


- DefaultResourceLoader 的子类 ClassRelativeResourceLoader 和 FileSystemResourceLoader 并没有覆盖这个方法，因此 ResourceLoader 的资源加载策略就是依靠 DefaultResourceLoader 来实现的


具体实现如下：

```java
@Override
public Resource getResource(String location) {
    Assert.notNull(location, "Location must not be null");

    // 首先，先使用自定义的加载解析策略 ProtocolResolver 来加载资源，解析得到就直接返回相应资源
    for (ProtocolResolver protocolResolver : getProtocolResolvers()) {
        Resource resource = protocolResolver.resolve(location, this);
        if (resource != null) {
            return resource;
        }
    }
    // 然后，以 / 开头的资源，则返回 ClassPathContextResource 类型的资源
    if (location.startsWith("/")) {
        return getResourceByPath(location);
    }
    // 之后，以 classpath: 开头的，返回 ClassPathResource 类型的资源
    else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
        return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
    }
    else {
        try {
            // 判断是否为文件 Url，是则返回 FileUrlResource 类型的资源；否则返回 UrlResource 类型的资源
            // Try to parse the location as a URL...
            URL url = new URL(location);
            return (ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
        }
        catch (MalformedURLException ex) {
            // 最后，则返回 ClassPathContextResource 类型的资源
            // No URL -> resolve as resource path.
            return getResourceByPath(location);
        }
    }
}
```

getResourceByPath(String path) 方法定义：

```java
/**
 * 默认直接返回 ClassPathContextResource，子类可以覆盖重写
 */
protected Resource getResourceByPath(String path) {
    return new ClassPathContextResource(path, getClassLoader());
}
```

#### 完整的加载定位策略

- 首先，使用自定义的加载解析策略 ProtocolResolver 来加载资源，解析得到就直接返回相应资源
- 然后，以 / 开头的资源，则调用 getResourceByPath(String path) 返回 ClassPathContextResource 类型的资源
- 之后，以 classpath: 开头的，返回 ClassPathResource 类型的资源
- 接着，判断是否为文件 Url，是则返回 FileUrlResource 类型的资源；否则返回 UrlResource 类型的资源
- 最后，若出现了 MalformedURLException 异常，则还是委派 getResourceByPath(String path) 返回 ClassPathContextResource 类型的资源。这样就基本完成了资源的定位加载。

### FileSystemResourceLoader

如果是文件系统的文件资源，`DefaultResourceLoader.getResourceByPath(String path)` 的处理是不恰当的。这个时候就需要使用 `org.springframework.core.io.FileSystemResourceLoader` 了，它重写了 `DefaultResourceLoader.getResourceByPath(String path)` 方法，返回 `FileSystemResource` 类型的资源以便可以从文件系统进行加载，最终得到我们需要的资源类型

定义：

```java
public class FileSystemResourceLoader extends DefaultResourceLoader {

    /**
     * 返回 FileSystemContextResource 类型的资源
     * 将资源路径解析为文件系统路径，以 / 开头的会被解析为 VM 当前工作目录的相对路径
     */
    @Override
    protected Resource getResourceByPath(String path) {
        // 截取开头的 /，会当作 context 上下文的额相对路径
          if (path.startsWith("/")) {
            path = path.substring(1);
        }
        return new FileSystemContextResource(path);
    }


    /**
     * 通过实现 ContextResource 接口显式地表示上下文相关路径的 FileSystemResource
     */
    private static class FileSystemContextResource extends FileSystemResource implements ContextResource {

        public FileSystemContextResource(String path) {
            super(path);
        }

        // 返回文件资源的路径
        @Override
        public String getPathWithinContext() {
            return getPath();
        }
    }

}
```

这里的 `FileSystemContextResource` 是 `FileSystemResourceLoader` 的内部类，它继承了 `FileSystemResource` 类，还实现了 `ContextResource` 接口，既可以表示实际文件系统的资源，也可以表示 context `环境相对路径的资源。FileSystemContextResource` 的构造函数也比较简单，直接调用父类 `FileSystemResourceLoader` 的构造函数来实现。

### ClassRelativeResourceLoader

```java
/**
 * ResourceLoader 的实现类，将普通的资源路径解析为给定 java.lang.Class 相关的资源，
 * 用于解析加载 Class 所在包或所在包的子包下的资源
 */
public class ClassRelativeResourceLoader extends DefaultResourceLoader {

    private final Class<?> clazz;


    /**
     * 根据指定的 class 创建 ClassRelativeResourceLoader 实例，
     * 内部会通过 class 来获取对应的 ClassLoader，这个 ClassLoader 可以用于
     * 自定义解析策略的加载，也可以用于 ClassPathResource 类型的资源（协议是 classpath:），
     * 参考 DefaultResourceLoader.getResource()
     */
    public ClassRelativeResourceLoader(Class<?> clazz) {
        Assert.notNull(clazz, "Class must not be null");
        this.clazz = clazz;
        setClassLoader(clazz.getClassLoader());
    }

    // 返回 ClassRelativeContextResource 类型的资源
    @Override
    protected Resource getResourceByPath(String path) {
        return new ClassRelativeContextResource(path, this.clazz);
    }


    /**
     * 通过实现 ContextResource 接口显式地表示上下文相关路径的 ClassRelativeContextResource
     */
    private static class ClassRelativeContextResource extends ClassPathResource implements ContextResource {

        private final Class<?> clazz;

        public ClassRelativeContextResource(String path, Class<?> clazz) {
            super(path, clazz);
            this.clazz = clazz;
        }

        // 返回文件资源的路径
        @Override
        public String getPathWithinContext() {
            return getPath();
        }

        // 根据资源的相对路径创建对应的资源，内部会将相对路径拼接进去
        @Override
        public Resource createRelative(String relativePath) {
            String pathToUse = StringUtils.applyRelativePath(getPath(), relativePath);
            return new ClassRelativeContextResource(pathToUse, this.clazz);
        }
    }

}
```

### ResourcePatternResolver

`ResourceLoader` 的 `Resource getResource(String location)` 方法每次只返回一个 Resource，无法加载多个资源，所以一般会是对应到具体的某个资源文件。如果需要加载多个资源的，则需要用 `ResourceLoader` 的另一个扩展类 `org.springframework.core.io.support.ResourcePatternResolver`，它支持根据指定的资源路径一次返回多个 `Resource` 实例对象

```java
public interface ResourcePatternResolver extends ResourceLoader {

    String CLASSPATH_ALL_URL_PREFIX = "classpath*:";

    /**
     * 返回指定路径的 Resource 数组实例对象
     */
    Resource[] getResources(String locationPattern) throws IOException;

}
```

### PathMatchingResourcePatternResolver

`PathMatchingResourcePatternResolver` 为 `ResourcePatternResolver` 最常用的子类，它除了支持 `ResourceLoader` 和 `ResourcePatternResolver` 新增的 `classpath*:` 前缀外，还支持 Ant 风格的路径匹配模式（类似于 **/*.xml）。

#### 构造函数和属性

PathMatchingResourcePatternResolver 提供了3个构造函数，都与 ResourceLoader 有关：

- 实例化时需要指定 ResourceLoader，默认是 DefaultResourceLoader。
- PathMatcher pathMatcher 属性，默认是 AntPathMatcher，Ant 类型的路径匹配实现类 

```java
/**
 * 内置的 ResourceLoader 资源加载定位器
 */
private final ResourceLoader resourceLoader;
/**
 * Ant 路径匹配器，用于支持 Ant 类型的路径匹配
 */
private PathMatcher pathMatcher = new AntPathMatcher();

/**
 * 没有指定的话就是用默认的 DefaultResourceLoader
 */
public PathMatchingResourcePatternResolver() {
    this.resourceLoader = new DefaultResourceLoader();
}

public PathMatchingResourcePatternResolver(ResourceLoader resourceLoader) {
    Assert.notNull(resourceLoader, "ResourceLoader must not be null");
    this.resourceLoader = resourceLoader;
}

public PathMatchingResourcePatternResolver(@Nullable ClassLoader classLoader) {
    this.resourceLoader = new DefaultResourceLoader(classLoader);
}
```

#### getResource 方法

`getResource` 直接通过委托给 `ResourceLoader` 来进行加载，也即是默认的 `DefaultResourceLoader` 进行加载： 

```java
@Override
@Nullable
public ClassLoader getClassLoader() {
    return getResourceLoader().getClassLoader();
}

// 使用内置的 ResourceLoader 进行加载，如果不是 ant 风格的路径，应该都是单一的 Resource
@Override
public Resource getResource(String location) {
    return getResourceLoader().getResource(location);
}
```

#### getResources 方法

```java
@Override
public Resource[] getResources(String locationPattern) throws IOException {
    Assert.notNull(locationPattern, "Location pattern must not be null");
    // 以 "classpath*:" 开头的路径
    if (locationPattern.startsWith(CLASSPATH_ALL_URL_PREFIX)) {
        // 路径包含通配符
        // a class path resource (multiple resources for same name possible)
        if (getPathMatcher().isPattern(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()))) {
            // a class path resource pattern
            return findPathMatchingResources(locationPattern);
        }// 路径不含通配符
        else {
            // all class path resources with the given name
            return findAllClassPathResources(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()));
        }
    }// 不以 "classpath*:" 开头的路径
    else {
        // Generally only look for a pattern after a prefix here,
        // and on Tomcat only after the "*/" separator for its "war:" protocol.
        // 通常只在这里查找前缀后的模式，而在Tomcat中仅在“*/”分隔符后查找其“war:”协议。
        int prefixEnd = (locationPattern.startsWith("war:") ? locationPattern.indexOf("*/") + 1 :
                locationPattern.indexOf(':') + 1);
        // 路径包含通配符
        if (getPathMatcher().isPattern(locationPattern.substring(prefixEnd))) {
            // a file pattern
            return findPathMatchingResources(locationPattern);
        }// 不包含通配符，表示单一的资源
        else {
            // a single resource with the given name
            return new Resource[] {getResourceLoader().getResource(locationPattern)};
        }
    }
}
```

逻辑流程如下：

- 路径包含通配符的，不管有没有 "classpath*:" 开头的，都调用 findPathMatchingResources(String locationPattern) 进行多个资源的加载；
- 以 "classpath*:" 开头，但路径不包含通配符的，则调用 findAllClassPathResources(String location) 进行加载
- 不是 "classpath*:" 开头，也不包含通配符，那么理论上应该是某个资源路径，直接使用 ResourceLoader 来进行加载。

![](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/22/20200722212413.jpg)

#### findAllClassPathResources() 方法

以 "classpath*:" 开头，但路径不包含通配符的，则调用 findAllClassPathResources(String location) 进行加载，该方法返回 classes 路径下和所有 jar 包中与路径匹配的所有资源。

这种情况下比较常见的是不同的 jar 包但是相同路径的配置文件，例如，当前项目有个 /spring/app-service.xml，需要引用内部团队其他关联依赖 B.jar ，而且 B.jar 中也有同名的配置文件 /spring/app-service.xml，如果需要都加载，那就可以使用 classpath*:/spring/app-service.xml 来进行配置和加载。

```java
/**
 * 通过类加载器找到 classes 路径和所有 jar 包中与给定路径相匹配的资源，
 * 委托给 doFindAllClassPathResources(String) 进行实际的处理
 */
protected Resource[] findAllClassPathResources(String location) throws IOException {
    String path = location;
    // 去掉开头的 / ，因此路径中有没有前导 / 都没有影响，都是相对路径
    if (path.startsWith("/")) {
        path = path.substring(1);
    }
    // 进行实际的资源定位处理，真正执行加载所有 classpath 资源
    Set<Resource> result = doFindAllClassPathResources(path);
    if (logger.isTraceEnabled()) {
        logger.trace("Resolved classpath location [" + location + "] to resources " + result);
    }
    // 转换为 Resource[] 返回
    return result.toArray(new Resource[0]);
}

/**
 * <p>通过类加载器找到 classes 路径和所有 jar 包中与给定路径相匹配的资源
 */
protected Set<Resource> doFindAllClassPathResources(String path) throws IOException {
    Set<Resource> result = new LinkedHashSet<>(16);
    ClassLoader cl = getClassLoader();
    // 1. 通过 ClassLoader 加载指定路径下的所有资源
    Enumeration<URL> resourceUrls = (cl != null ? cl.getResources(path) : ClassLoader.getSystemResources(path));
    // 2.
    while (resourceUrls.hasMoreElements()) {
        URL url = resourceUrls.nextElement();
        // 将 URL 转换成对应的 UrlResource
        result.add(convertClassLoaderURL(url));
    }
    // 3. 加载路径下所有的 jar 包
    if ("".equals(path)) {
        // The above result is likely to be incomplete, i.e. only containing file system references.
        // We need to have pointers to each of the jar files on the classpath as well...
        addAllClassLoaderJarRoots(cl, result);
    }
    return result;
}

/**
 * 将 ClassLoader 返回的给定 URL 转换成 Resource，默认是 UrlResource
 */
protected Resource convertClassLoaderURL(URL url) {
    return new UrlResource(url);
}
```

实际上执行加载的是 doFindAllClassPathResources(String path)，处理过程如下：

通过 ClassLoader 进行资源的加载，如果实例化 PathMatchingResourcePatternResolver 时已指定了 ResourceLoader，则使用这个ResourceLoader 的 ClassLoader 作为委托对象，通过调用其 getResources(String name) 方法来进行处理，否则通过 ClassLoader.getSystemResources(String name) 来进行处理。

ClassLoader.getResources()如下:

```java
public Enumeration<URL> getResources(String name) throws IOException {
    @SuppressWarnings("unchecked")
    Enumeration<URL>[] tmp = (Enumeration<URL>[]) new Enumeration<?>[2];
    if (parent != null) {
        tmp[0] = parent.getResources(name);
    } else {
        tmp[0] = getBootstrapResources(name);
    }
    tmp[1] = findResources(name);

    return new CompoundEnumeration<>(tmp);
}
```

这里比较明显，如果 ClassLoader 有父类加载器，则通过父类加载器迭代加载获取资源，直至最后则是调用 getBootstrapResources(name) 进行资源获取。

2. 迭代遍历 1 中获取到的 URL 集合资源，通过 convertClassLoaderURL(URL url) 将 URL 转换成 UrlResource 实例对象。
3. 如果路径 path 为""（classpath*: 或者 classpath*:/），则通过 addAllClassLoaderJarRoots(@Nullable ClassLoader classLoader, Set<Resource> result) 方法加载路径下所有 jar 包的资源

#### findPathMatchingResources() 方法

路径中包含通配符的，都是使用这个方法进行资源的加载：

```java
/**
 * <通过 Ant 风格的 PathMatcher 来查找匹配给定路径的所有资源，
 * 支持 在 jar 包、zip 文件、和文件系统中查找相关资源
 */
protected Resource[] findPathMatchingResources(String locationPattern) throws IOException {
    // 确定根路径和子路径
    String rootDirPath = determineRootDir(locationPattern);
    String subPattern = locationPattern.substring(rootDirPath.length());
    // 获取根路径下的资源
    Resource[] rootDirResources = getResources(rootDirPath);
    Set<Resource> result = new LinkedHashSet<>(16);
    // 迭代遍历
    for (Resource rootDirResource : rootDirResources) {
        rootDirResource = resolveRootDirResource(rootDirResource);
        URL rootDirUrl = rootDirResource.getURL();
        // bundle 类型的资源
        if (equinoxResolveMethod != null && rootDirUrl.getProtocol().startsWith("bundle")) {
            URL resolvedUrl = (URL) ReflectionUtils.invokeMethod(equinoxResolveMethod, null, rootDirUrl);
            if (resolvedUrl != null) {
                rootDirUrl = resolvedUrl;
            }
            rootDirResource = new UrlResource(rootDirUrl);
        }
        // vfs 类型的资源
        if (rootDirUrl.getProtocol().startsWith(ResourceUtils.URL_PROTOCOL_VFS)) {
            result.addAll(VfsResourceMatchingDelegate.findMatchingResources(rootDirUrl, subPattern, getPathMatcher()));
        }
        // jar 类型的资源
        else if (ResourceUtils.isJarURL(rootDirUrl) || isJarResource(rootDirResource)) {
            result.addAll(doFindPathMatchingJarResources(rootDirResource, rootDirUrl, subPattern));
        }
        // 其他资源类型
        else {
            result.addAll(doFindPathMatchingFileResources(rootDirResource, subPattern));
        }
    }
    if (logger.isTraceEnabled()) {
        logger.trace("Resolved location pattern [" + locationPattern + "] to resources " + result);
    }
    // 转成 Resource 数组
    return result.toArray(new Resource[0]);
}
```

方法分两部

- 确定目录，获取该目录下得所有资源
- 在所获得的所有资源中进行迭代匹配获取我们想要的资源。

在这个方法里面我们要关注两个方法，一个是 `determineRootDir()`,一个是 `doFindPathMatchingFileResources()`。

#### determineRootDir()方法

determineRootDir 方法主要用于确认根路径： 

```java
/**
 * <p>确定指定路径的根路径
 */
protected String determineRootDir(String location) {
    // 冒号之后一位，即协议后面的位置
    int prefixEnd = location.indexOf(':') + 1;
    // 目录结束位置
    int rootDirEnd = location.length();
    // 循环判断是否有分隔符，如果有，则截断最后一个 / 之后的部分
    // 例如 classpath*:spring/*-service.xml，循环判断后剩下 spring/ 已经不包含通配符了
    // 如果是 classpath*:*-service.xml，则会出现 rootDirEnd == 0 的情况
    while (rootDirEnd > prefixEnd && getPathMatcher().isPattern(location.substring(prefixEnd, rootDirEnd))) {
        // 通配符至少占 1 个，再加上分隔符，所以 -2；最后 +1 表示返回结果要包含分隔符
        rootDirEnd = location.lastIndexOf('/', rootDirEnd - 2) + 1;
    }
    // 将 prefixEnd 的值赋给 rootDirEnd，也即是冒号后一位
    if (rootDirEnd == 0) {
        rootDirEnd = prefixEnd;
    }
    // 截取根目录
    return location.substring(0, rootDirEnd);
}
```

确定根路径如下:

| 原路径                             | 根路径               |
| ---------------------------------- | -------------------- |
| classpath*:spring/a*/*-service.xml | classpath*:spring/   |
| classpath*:spring/a/*-service.xml  | classpath*:spring/a/ |
| classpath*:*-service.xml           | classpath*:          |

#### doFindPathMatchingXxxResources() 方法

这个指的是特定类型资源的全部加载，只要是 findPathMatchingResources 方法中用到的这3个


- doFindPathMatchingJarResources(Resource rootDirResource, URL rootDirURL, String subPattern)
- doFindPathMatchingFileResources(Resource rootDirResource, String subPattern)
- VfsResourceMatchingDelegate.findMatchingResources(URL rootDirURL, String locationPattern, PathMatcher pathMatcher)


对于这几个方法的分析，可以参考以下文章：


[Spring源码情操陶冶-PathMatchingResourcePatternResolver路径资源匹配溶解器](https://www.cnblogs.com/question-sky/p/6959493.html)
[《深入 Spring IoC 源码之 ResourceLoader》](http://www.blogjava.net/DLevin/archive/2012/12/01/392337.html) 
[《Spring 源码学习 —— 含有通配符路径解析（上）》](http://www.coderli.com/spring-wildpath-parse/)

## 总结

- Spring 提供了 Resource 和 ResourceLoader 来统一抽象整个资源及其定位。使得资源与资源的定位有了一个更加清晰的界限，并且提供了合适的 Default 类，使得自定义实现更加方便和清晰。
- AbstractResource 为 Resource 的默认实现，它对 Resource 接口做了一个统一的实现，子类继承该类后只需要覆盖相应的方法即可，同时对于自定义的 Resource 我们也是继承该类。
- DefaultResourceLoader 同样也是 ResourceLoader 的默认实现，在自定 ResourceLoader 的时候我们除了可以继承该类外还可以实现 ProtocolResolver 接口来实现自定资源加载协议。
- DefaultResourceLoader 每次只能返回单一的资源，所以 Spring 针对这个提供了另外一个接口 ResourcePatternResolver ，该接口提供了根据指定的 locationPattern 返回多个资源的策略。其子类 PathMatchingResourcePatternResolver 是一个集大成者的 ResourceLoader ，因为它即实现了 Resource getResource(String location) 也实现了 Resource[] getResources(String locationPattern)。
- Resource 和 ResourceLoader 相关接口和实现类都是在 spring-core 包里面，是比较核心的基础功能。

## 参考

- [Spring-framework 5.2.7.RELEASE 官方文档](https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/index.html)
- [Spring5源码分析(005)——IoC篇之统一资源加载Resource和ResourceLoader](https://www.cnblogs.com/wpbxin/p/13061470.htm)
- [【死磕 Spring】—– IOC 之 Spring 统一资源加载策略](http://cmsblogs.com/?p=2656)
