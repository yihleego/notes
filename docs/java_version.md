# Java 版本变化

## Java 1.0 Oak 1996-01-23

- 诞生

## Java 1.1 Sparkler 1997-02-19

- 新增JDBC
- 支持反射
- 支持内部类
- 支持远程方法调用

## Java 1.2 Playground 1998-12-08

- 新增集合框架
- 支持JIT
- 对字符串常量做内存映射

## Java 1.3 Kestrel 2000-05-08

- Hotspot 作为默认虚拟机
- 新增代理类

## Java 1.4 Merlin 2004-02-06

- 支持XML的基本处理
- 支持断言
- 支持正则表达式
- 支持链式异常处理
- 支持IPV6
- 新增 Logging API
- 新增 Preferences API
- 新增 Image I/O API
- 新增 NIO API
- 增强 JDBC 3.0 API

## Java 5 Tiger 2004-09-30

- 支持泛型
- 新增枚举
- 自动装箱拆箱
- 支持可变参数
- 新增 annotation 和 instrument
- 支持 foreach 循环
- 支持静态导入
- 并发数据结构 (JUC)
- 新增 Arrays 和 StringBuilder

## Java 6 Mustang 2006-12-11

- 优化 String.intern()
- 优化 synchronized 引入了偏向锁和轻量级锁

## Java 7 Dolphin 2011-07-28

- switch 支持 String 类型
- 数字字面量的改进支持下划线
- 增强异常处理 try-with-resources
- 增强泛型推断
- JSR 203 More New I/O APIs for the JavaTMPlatform ("NIO.2")
- JSR 292 提供了 invokedynamic 字节码的 API 支持。
- 新增 Fork/Join framework

## Java 8 Spider 2014-03-18

- 支持 Lambda 表达式
- 支持方法引用
- 增强 annotation 新增 @Repeatable
- 新增 Optional 类
- 新增 Stream 类
- 新增 Base64 类
- JSR 310 Date/Time API 新增 java.time 包
- JavaScript引擎Nashorn
- 优化 HashMap 的 hash 方法并引入红黑树
- JEP 122 JVM使用 Metaspace 代替 PermGen 空间。

## Java 9 2017-09-22

- 模块系统：模块是一个包的容器，Java 9 最大的变化之一是引入了模块系统（Jigsaw 项目）。
- REPL (JShell)：交互式编程环境。
- HTTP 2 客户端：HTTP/2标准是HTTP协议的最新版本，新的 HTTPClient API 支持 WebSocket 和 HTTP2 流以及服务器推送特性。
- 改进的 Javadoc：Javadoc 现在支持在 API 文档中的进行搜索。另外，Javadoc 的输出现在符合兼容 HTML5 标准。
- 多版本兼容 JAR 包：多版本兼容 JAR 功能能让你创建仅在特定版本的 Java 环境中运行库程序时选择使用的 class 版本。
- 集合工厂方法：List，Set 和 Map 接口中，新的静态工厂方法可以创建这些集合的不可变实例。
- 私有接口方法：在接口中使用private私有方法。我们可以使用 private 访问修饰符在接口中编写私有方法。
- 进程 API: 改进的 API 来控制和管理操作系统进程。引进 java.lang.ProcessHandle 及其嵌套接口 Info 来让开发者逃离时常因为要获取一个本地进程的 PID 而不得不使用本地代码的窘境。
- 改进的 Stream API：改进的 Stream API 添加了一些便利的方法，使流处理更容易，并使用收集器编写复杂的查询。
- 改进 try-with-resources：如果你已经有一个资源是 final 或等效于 final 变量,您可以在 try-with-resources 语句中使用该变量，而无需在 try-with-resources 语句中声明一个新变量。
- 改进的弃用注解 @Deprecated：注解 @Deprecated 可以标记 Java API 状态，可以表示被标记的 API 将会被移除，或者已经破坏。
- 改进钻石操作符(Diamond Operator) ：匿名类可以使用钻石操作符(Diamond Operator)。
- 改进 Optional 类：java.util.Optional 添加了很多新的有用方法，Optional 可以直接转为 stream。
- 多分辨率图像 API：定义多分辨率图像API，开发者可以很容易的操作和展示不同分辨率的图像了。
- 改进的 CompletableFuture API ： CompletableFuture 类的异步机制可以在 ProcessHandle.onExit 方法退出时执行操作。
- 轻量级的 JSON API：内置了一个轻量级的JSON API
- 响应式流（Reactive Streams) API: Java 9中引入了新的响应式流 API 来支持 Java 9 中的响应式编程。
- 将G1设为默认的垃圾回收器实现

## Java 10 2018-03-21

- 局部变量的类型推断 var 关键字
- GC改进和内存管理 并行全垃圾回收器 G1
- 垃圾回收器接口
- 线程-局部变量管控
- 合并 JDK 多个代码仓库到一个单独的储存库中
- 新增API：ByteArrayOutputStream
- 新增API：List、Map、Set
- 新增API：java.util.Properties
- 新增API：Collectors收集器

## Java 11 2018-09-25

- 增强局部变量的类型推断
- 增强字符串
- 增强集合
- 增强Stream
- 增强Optional
- 增强InputStream
- 新增HTTP Client API

## Java 12 2019-03-19

- 增强字符串
- 支持Unicode 11
- JEP 354 Switch 表达式 (Preview)

## Java 13 2019-09-17

- JEP 350 Dynamic CDS Archives
- JEP 351 ZGC Uncommit Unused Memory
- JEP 353 重新实现 Socket API
- JEP 354 Switch 表达式 (Preview)
- JEP 355 文本块 (Preview)

## Java 14

## Java 15

## Java 16

## Java 17