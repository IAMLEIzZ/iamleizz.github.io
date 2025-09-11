---
title: Kotlin + SpringBoot 踩坑实录
date: 2025-08-09T19:35:27+08:00
lastmod: 2025-08-09T19:35:27+08:00
draft: false
tags:
  - Java
  - Kotlin
  - SpringBoot
categories: 
author: 小石堆
---
	最近在做 Kotlin 项目的时候，由于没有系统性学习过Kotlin，遇到了一个坑，当我的 Kotlin 项目的某个模块，使用 @Autowried 注解注入的时候，明明初始化过对象的成员变量，但是获取的时候，仍然为 Null。在排查完问题后，最终写下此篇，内容主要涉及到了 Kotlin 的 final 和 Spring 的代理机制。
## 问题复现
```Kotlin
@Component  
open class ApiGw {  
    private var endPoint:String? = null  
  
    @PostConstruct  
    open fun init() {  
        endPoint = "初始化 endPoint"  
        println(endPoint)  
        println("this class" + this.javaClass.toString())  
    }  
  
    @Cacheable(cacheNames = ["userCache"], key = "#id")  
    fun printEndpoint(id: Long) {  
        println(endPoint)  
        println("this class" + this.javaClass.toString())  
    }  
}
```

```Kotlin
@RestController  
class ControllerA {  
    @Autowired  
    private lateinit var api: ApiGw  
  
    @GetMapping("/hello")  
    fun sayHello(): String {  
        api.printEndpoint(123)  
        return "Hello from Kotlin Spring Boot!"  
    }  
}
```
上面的代码，调用 `sayHello()` 的时候，我预期 `endPoint` 的输出为 “初始化 endPoint”。因为在 `@PostConstruct` 的作用下，`endPoint` 已经被初始化过了。但是结果并非如此：
```
初始化 endPoint
this class:com.example.cglibtest.ApiGw
null
this class: class com.example.cglibtest.ApiGw$$EnhancerBySpringCGLIB$$e1b2c3
```
我们可以看到，虽然 `endPoint` 已经被初始化了，但是后续访问的过程中，仍然为 `Null`。是初始化失败了吗？
## 问题解析
这里我们现了解一下 Spring 的两种代理机制：
1. JDK 动态代理
2. CGLIB 动态代理
### JDK 动态代理
JDK 代理某个类要求这个类必须是至少是一个接口的实现，因为 JDK 代理的本质是模仿，代理类实现目标类所实现的接口，调用接口方法时通过 InvocationHandler 的 invoke 方法来处理
即：
- 目标类必须实现至少一个接口
- 代理对象只能作为接口类型使用，不能直接作为目标类的具体类型使用
### CGLIB 动态代理
CGLIB 代理某个类要求这个类必须是可以被继承的，代理类继承目标类，重写非 final 方法，在重写的方法中插入增强逻辑（比如事务、日志等）
即：
- 目标类不能是 `final` 类，因为子类继承时会编译错误
- 需要代理的方法不能是 `final`，否则无法被重写
### 问题定位
由于 Kotlin 默认所有的类都是 final 的，同时我的 `printEndpoint `方法加了 `@Cacheable` 注解，导致 Spring 开启了 CGLIB 代理。其实本身目标类的初始化是 OK 的，但是重写非 `open` 的 `printEndpoint ` 方法时失败了，所以等于没有成功代理这个类。因此当访问到这个方法的时候，并没有成功将方法转发到目标类，进而无法访问到初始化过的 `endPoint`。
改为 `open` 后:
```Kotlin
@Component  
open class ApiGw {  
    private var endPoint:String? = null  
  
    @PostConstruct  
    open fun init() {  
        endPoint = "初始化 endPoint"  
        println(endPoint)  
        println("this class" + this.javaClass.toString())  
    }  
  
    @Cacheable(cacheNames = ["userCache"], key = "#id")  
    open fun printEndpoint(id: Long) {  
        println(endPoint)  
        println("this class" + this.javaClass.toString())  
    }  
}
```

```
初始化 endPoint
this class:com.example.cglibtest.ApiGw
初始化 endPoint
this class:com.example.cglibtest.ApiGw
```
不难看出，当改为全 `open` 后，成功获取到了对应的对象。但是类的输出为什么是this class:com.example.cglibtest.ApiGw？难道说没走代理？
其实是走了，因为这个方法是由代理类转发给目标类执行的。
## 解决方案
1. 写代码的时候注意将需要 `open` 的类和方法都加上 `open` 关键字
2. 使用 jetbrains 提供的 `all-open` 插件，这个插件会在编译的时候将需要置为 opend 的类编译成 open 的。
```xml
<plugin>  
    <groupId>org.jetbrains.kotlin</groupId>  
    <artifactId>kotlin-maven-plugin</artifactId>  
    <version>${kotlin.version}</version>  
    <executions>  
        <execution>  
            <id>compile</id>  
            <phase>compile</phase>  
            <goals>  
                <goal>compile</goal>  
            </goals>  
        </execution>  
        <execution>  
            <id>test-compile</id>  
            <phase>test-compile</phase>  
            <goals>  
                <goal>test-compile</goal>  
            </goals>  
        </execution>  
    </executions>  
    <configuration>  
        <args>  
            <arg>-Xjsr305=strict</arg>  
        </args>  
        <compilerPlugins>  
            <plugin>spring</plugin>  
        </compilerPlugins>  
        <jvmTarget>11</jvmTarget>  
    </configuration>  
    <dependencies>  
        <dependency>  
            <groupId>org.jetbrains.kotlin</groupId>  
            <artifactId>kotlin-maven-allopen</artifactId>  
            <version>${kotlin.version}</version>  
        </dependency>  
    </dependencies>  
</plugin>
```