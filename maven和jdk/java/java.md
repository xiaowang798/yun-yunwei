### 1. jdk 的配置

本课程是使用 IDEA 进行开发，在IDEA 中配置 jdk 的方式很简单，打开`File->Project Structure`，如下图所：

![IDEA中配置jdk](D:\Desktop\bishe\maven和jdk\java\javamd图片\a20d2bc2003e9e9a4fd025f4a2246c94.png)

1. 选择 SDKs

2. 在 JDK home path 中选择本地 jdk 的安装目录

3. 在 Name 中为 jdk 自定义名字

   

maven配置

创建了 Spring Boot 项目之后，需要进行 maven 配置。打开`File->settings`，搜索 maven，配置一下本地的 maven 信息。如下：

![maven配置](D:\Desktop\bishe\maven和jdk\java\javamd图片\ba51186a64e4959040eedab85c1f38d5.png)

![cf8342e64ad32626551e383f2c86c0dd](D:\Desktop\bishe\maven和jdk\cf8342e64ad32626551e383f2c86c0dd.png)

### 2 编码配置

同样地，新建项目后，我们一般都需要配置编码，这点非常重要，很多初学者都会忘记这一步，所以要养成良好的习惯。

IDEA 中，仍然是打开`File->settings`，搜索 encoding，配置一下本地的编码信息。如下：

![编码配置](D:\Desktop\bishe\maven和jdk\java\javamd图片\0296023508c0ba5ecba29963b035cc19.png)

### 3. Spring Boot 项目工程结构

Spring Boot 项目总共有三个模块，如下图所示：

![Spring Boot项目工程结构](D:\Desktop\bishe\maven和jdk\java\javamd图片\277ae310b5a72b2e0e2513245107a61f.png)

```
src/main/java路径：主要编写业务程序
src/main/resources路径：存放静态文件和配置文件
src/test/java路径：主要编写测试程序
默认情况下，如上图所示会创建一个启动类 Course01Application，该类上面有个@SpringBootApplication注解，该启动类中有个 main 方法，没错，Spring Boot 启动只要运行该 main 方法即可，非常方便。另外，Spring Boot 内部集成了 tomcat，不需要我们人为手动去配置 tomcat，开发者只需要关注具体的业务逻辑即可。
在项目开发中，接口与接口之间，前后端之间数据的传输都使用 Json 格式

Spring Boot 中对依赖都做了很好的封装，可以看到很多 `spring-boot-starter-xxx` 系列的依赖，这是 Spring Boot 的特点之一，不需要人为去引入很多相关的依赖了
```

![image-20250117144320993](C:\Users\xiaowang798abc\AppData\Roaming\Typora\typora-user-images\image-20250117144320993.png)

###  4.Vue通过MVVM模式,能够实现视图与模型的双向绑定。

 简单来说，就是数据变化的时候, 页面会自动刷新, 页面变化的时候，数据也会自动变化

Vue是一个类似于Jquery的一个JS框架，所以，如果想使用Vue，则在当前页面导入Vue.js文件即可。

```
<!-- 在线导入 -->
<!-- 开发环境版本，包含了用帮助的命令行警告 -->
<script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
<!-- 生产环境版本，优化了尺寸和速度 -->
<script src="https://cdn.jsdelivr.net/npm/vue"></script>

<!-- 本地导入 -->
<script src="node_modules/vue/dist/vue.js"></script>

```

我们可以将redis当作mysql的缓存，应用(app)所有读的操作都负载到redis上，因为redis够快，如果直接从mysql上读会对它造成巨大的压力，之前的mysql主从复制同样也是为了解决这样的问题

### 5.java笔记

```
项目介绍文件有的是.md文件，这个需要记事本或者typora打开，word打开是乱码。
如果源码里有pom.xml文件，需要配置maven环境，也可以不配置，idea会自带maven环境。
将 C:\Program Files\Java\jdk-9.0.4 加入JAVA_HOME环境变量中，目的是告诉Java工具（如Maven等）JDK的根目录在哪里。
将 C:\Program Files\Java\jdk-9.0.4\bin 加入PATH环境变量中，目的是将JDK的二进制文件目录加入系统的PATH环境变量中。这使得你可以在命令行中直接使用Java的命令行工具，如 java、javac 等。

select version();去naicate看mysql的版本
使用 Maven 给我们带来的最直接的好处，就是统一管理jar 包，那么这些 jar 包存放在哪里呢？它们就在您的本地仓库中，默认地址位于 C:\Users\系统用户名.m2\repository 目录下，为了方便，我们就修改一下这个默认地址。

Maven.如何自己配置镜像，打开settings.xml文件，可以使用记事本或者其他文本编辑软件打开，这里我使用的是Notepad++软件打开
安装新版本 IDEA 之前，如果本机安装过老版本的 IDEA, 需要先彻底卸载，以免两者冲突，导致破解失败。

在安装的时候会提示卸载老版本 IDEA
```

