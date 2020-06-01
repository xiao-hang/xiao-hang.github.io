---
layout: post
#标题配置
title:   Spring-Cloud-Alibaba笔记02-项目设想尝试
#时间配置
date:   2020-06-01 12:00:00 +0800
#大类配置
categories: Spring-Cloud-Alibaba
#小类配置
tag: 项目设想尝试
---

* content
{:toc}





> 对于Spring Cloud Alibaba的组件来说，也就辣么几个，Nacos+Dubbo+Sentinel 就差不多了。
>
> 然后，这里试着对简单的微服务做个设想。

### 首先对于设想

直接看图吧。大概就是这个样子   ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

![微服务拆分设想](https://img-blog.csdnimg.cn/20200529162724951.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1RpYW5YdWVXdQ==,size_16,color_FFFFFF,t_70)

这里面画出来的，还是缺少了一些东西的，比如网关，负载啥的！！！

> 欢迎提提建议，推荐些好玩的组件。

---

### 尝试

主要的核心就是Nacos+Dubbo了，Sentinel 在上一篇笔记中已经搞定了的。

还有就是项目中的Dubbo接口独立分包，进行统一管理，多项目实现-调用。

至于里面各个项目 admin ，api，ser1-n 只要能连上Nacos+Dubbo 就可以了，具体各个项目内使用的技术可以随便了。

#### 项目结构

![image-20200529135256095](https://img-blog.csdnimg.cn/20200529162751950.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1RpYW5YdWVXdQ==,size_16,color_FFFFFF,t_70)

查不多就是这个样子，作为一个demo试试。

其中，api和pubSer为前笔记的demo，这里直接复制过来作为接口的实现与消费。

而admin项目，为旧项目修改，接入的。

> 由于就的项目使用的spring boot 版本为2.0.2 ，所有吧对应的一些版本都调低了。
>
> 具体版本的问题查看文章：<https://blog.csdn.net/qq_37143673/article/details/99292705> 
>
> 或者官方说明的地址：[https://github.com/alibaba/spring-cloud-alibaba/wiki/%E7%89%88%E6%9C%AC%E8%AF%B4%E6%98%8E](https://github.com/alibaba/spring-cloud-alibaba/wiki/版本说明)

### 公共接口

这个只是个定义，正的只是个接口的定义！！！

```java
public interface UserService {
    String test(String message);
	......
}
```

### 接口实现

作为实现接口的项目，使用的前笔记的项目，这里就再贴一下，主要的代码吧！

```java
@Service(group = "userServer-xiaohang" ,retries = 0,timeout = 10000)
public class UserServiceImpl implements UserService {
    @Override
    public String test(String message) {
        return "this is provider service's information:"+message;
    }
.........
```

罒ω罒，其实就是自动生成的demo项目 。。。

### Dubbo调用

> 使用一整套，spring cloud alibaba 的项目，只要一个注解就可以调用。
>
> 【需要加载的包，和配置请查看上一篇笔记，或者直接去生成个demo项目也有的】

```java
@RestController
public class DubboTest {

    @Reference   // 就是这注解，注入远程的服务
    private UserService userService;

    @RequestMapping("/getDubboTest")
    public String getXhUser() {
        return userService.test("Consumer's Test!!");
    }
..........
```

### 旧项目admin 调整并入

> 新的项目的使用都没有啥问题，但是在把旧项目并入的时候就发现了问题
>
> Nacos 可以使用，但是Dubbo总是如法注册，调用服务报错空指针。

修改的内容：

#### **pom文件的调整**，

感觉里面还是多加载了部分包，但本着能用就行了，没有去删减掉。

```xml
		<dependency>
			<groupId>com.alibaba.cloud</groupId>
<!--			<groupId>org.springframework.cloud</groupId>-->
			<artifactId>spring-cloud-starter-dubbo</artifactId>
			<version>2.0.2.RELEASE</version>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-openfeign</artifactId>
			<version>2.0.2.RELEASE</version>
		</dependency>
		<dependency>
			<groupId>com.alibaba.spring</groupId>
			<artifactId>spring-context-support</artifactId>
			<version>[latest version]</version>
		</dependency>
.....
<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>com.alibaba.cloud</groupId>
				<artifactId>spring-cloud-alibaba-dependencies</artifactId>
				<version>2.0.2.RELEASE</version>
				<type>pom</type>
				<scope>import</scope>
							<exclusions>
									<exclusion>
										<groupId>com.alibaba.cloud</groupId>
										<artifactId>spring-cloud-starter-dubbo</artifactId>
									</exclusion>
								</exclusions>
			</dependency>
			<dependency>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-dependencies</artifactId>
				<version>2.0.2.RELEASE</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
</dependencyManagement>
```

#### **启动调整：**

```java
Reference@EnableDubbo    //新增的
@DubboComponentScan 	//新增的  就是这个注解没有，导致@Reference注解无效的 我也不知道为什么
@SpringBootApplication
@EnableDiscoveryClient  //服务发现的
@EnableCaching
public class CosonleApplication extends SpringBootServletInitializer  {
。。。。。。。
```

#### 配置：

```yml
项目使用的yml配置的
新增的内容：
spring:
  cloud:
    nacos:
      discovery:
        server-addr: http://192.168.1.180:8848
    sentinel:
      transport:
        port: 8992
        dashboard: 10.0.100.208:8080  # 控制台地址
      eager: true
dubbo:
  application:
    name: happyPay-sentinel-xiaohang-cloud-test
  registry:
    address: nacos://192.168.1.180:8848
  protocol:
    name: dubbo
    port: -1
```

#### 代码测试：

```java
@Service
public class dubboService {

    @Reference(group = "userServer-xiaohang")   //默认不指定group的，没有关系
    private UserService userService;

    public void dubbotest(){
        System.out.println("小杭测试 springcloud dubbo 的使用！！！！！");
        System.out.println(userService.test("Consumer's Test!!"));
 。。。。。。。。
```

此服务也可以在正常的控制层注入

```java
    @Autowired
    com.ibeetl.cms.service.dubboService dubboService;
```

---

### 小结

试一下就可了的。

尝试Nacos+Dubbo+Sentinel是可以的。这样就可以简单的吧一堆的旧项目搞一起了。

至于更多的整合，就需要在业务上进行拆分了，至少简单并入，依照以上是没问题的。

> 还有各种的组件，比如MQ，MyCAT，Redis等等等  项目够大采用的到
>
> 小公司的项目，需要吗 ╮(╯_╰)╭ 

图片中的其他业务拆分，是个人对常用项目的理解上拆分的。看看就好，无需纠结。

---

2020-05-29  小杭

---

