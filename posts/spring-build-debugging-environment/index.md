# IDEA编译Spring源码


## 前言

相信大家都和我一样，想要阅读Spring源码。但是在编译Spring源码过程中，会遇到了很多坑，这个过程也实在让人心烦。下面我就把自己遇到的坑和解决办法详细的写出来，希望能帮助到大家。解决到阅读Spring源码的拦路虎。

先介绍下我的开发环境，不同版本的细节可能有所不同，仅供参考。

IDEA	: 2019旗舰版

Spring:  5.2.2RELEASE

下面介绍具体过程

## 1 前提准备

### 1.1 获取源码

从Github获取[Spring官方源码](https://github.com/spring-projects/spring-framework),可以直接使用git clone来下载源代码，也可以直接下载zip压缩包。我fork到自己的仓库，这样的好处就是可以自己写一些注释，有了自己的仓库，可以进行自由的提交。这个过程不在细说，不熟悉的小伙伴参考其他资料。

### 1.2 安装AspectJ

Spring实现了AOP（面向切面编程），它依赖AspectJ，因此需要下载AspectJ并安装。安装过程可以参考[AspectJ入门及在IDEA中的配置](https://juejin.im/post/5e481046f265da574111f9ee)

### 1.3 安装gradle（可选）

IDEA默认已经安装或者会自动安装gradle。gradle类似maven，可以构建项目，可以对包依赖进行管理。

网上资料很多，就不重点介绍了。重要的是要安装版本问题，对Spring5.2来说,不要安装6.*版本的gradle，否则会报错```The build scan plugin is not compatible with Gradle 6.0 and later. Please use the Gradle Enterprise plugin instead.```。 我就是被这个错误坑的好惨，后来换成4.9就没问题了，当时没有试5的版本，按说应该是可以的。

## 2.安装步骤

Spring源码目录下有一个import-into-idea.md这个文件，里边写得比较详细如何将Spring源码导入到IDEA。

![image-20200418112330561](/images/Spring-build-debugging-environment/image-20200418112330561.png)

### 2.1 编译spring-oxm:compileTestJava

进入Spring源码目录，控制台输入：```gradlew :spring-oxm:compileTestJava```，如果下载插件特别慢时，加上国内镜像。可以参考[gradle配置国内镜像](https://www.cnblogs.com/a8457013/p/8408196.html)。

### 2.2 使用IDEA导入项目

如下图，最好使用红色框中，使用默认的gradle。我之前使用的本地的gradle就出现了```spring error: cannot find symbol method get Archive File()```的问题。换成默认的就解决了。

![image-20200418113309577](/images/Spring-build-debugging-environment/image-20200418113309577.png)

导入后，gradle开始构建项目

**注意**：构建过程中会特别慢，所以要换成国内的源。打开源码目录下build.gradle文件，最好是搜索出含有repositories关键字的，加上阿里云的源，这样会省去很多麻烦。

阿里云源：

```gradle
repositories{
maven{ url 'http://maven.aliyun.com/nexus/content/groups/public/'}
maven { url 'https://maven.aliyun.com/repository/public/' }
maven { url 'https://maven.aliyun.com/repository/google/'}
maven { url 'https://maven.aliyun.com/repository/jcenter/'}
}
```

如：当时我只加了configure下面的repositories，kotlin下载时还是特别慢，后来发现原来没有更换kotlin下载的源，这一点千万要注意，当时被坑的很惨。

buildscript也要加上阿里云的源：

```
buildscript {
    ext.kotlin_version = '1.3.72'
    repositories {
        maven{ url 'http://maven.aliyun.com/nexus/content/groups/public/'}
        mavenCentral()
    }
    dependencies {
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
    }
}
```

## 3 编译问题

### 3.1  AspectJ编译问题

问题如下图：

![image-20200418115124966](/images/Spring-build-debugging-environment/image-20200418115124966.png)

原因：Idea默认使用的Javac编译器，而AspectJ关键字它不认识。这个时候需要我们前期准备的AspectJ编译器Ajc了。

具体步骤：

1. 将Idea的编译器设置为Ajc

   打开：IDEA--Preferences--Build,Execution,Deployment--Compiler--JavaCompiler,将Use compiler设置为Ajc。将Path to Ajc compiler设置为AspectJ安装目录下的lib文件夹中的aspectjtools.jar文件。勾选Delegate to Javac选项，它能够只编译AspectJ的Facets项目，而其他普通项目还是交由Javac来编译。 

   ![image-20200418115414791](/images/Spring-build-debugging-environment/image-20200418115414791.png)

2. 将spring-aop_main和spring-aspectjs_main两个模块添加AspectJ Facets

   打开：File--Project Structure--Facets，点击+号，选择AspectJ，选择spring-aop_main。添加完后，同样的操作，将spring-aspectjs_main模块也设置AspectJ。

   

   ![image-20200418115840396](/images/Spring-build-debugging-environment/image-20200418115840396.png)

再次执行build，已经没有错误了 

![image-20200418120002392](/images/Spring-build-debugging-environment/image-20200418120002392.png)

### 3.2 CoroutinesUtils找不到符号

错误如下图：

![image-20200418120234646](/images/Spring-build-debugging-environment/image-20200418120234646.png)

解决办法：将spring-framework/spring-core/build/libs/spring-core-5.2.6.BUILD-SNAPSHOT.jar包添加到spring-core的依赖里。注意不同版本jar包位置可能不一样。

打开：File--Project Structure--Modules

![image-20200418120857869](/images/Spring-build-debugging-environment/image-20200418120857869.png)

## 4. 运行示例

解析 XML 配置文件成对应的 BeanDefinition 们的流程

调试 org.springframework.beans.factory.xml.XmlBeanDefinitionReaderTests 的 withFreshInputStream() 方法

IDEA能够正确编译Spring源码了，开始正式学习Spring源码了！
