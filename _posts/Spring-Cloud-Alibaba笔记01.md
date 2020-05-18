## Spring-Cloud-Alibaba笔记01-关于远程调用Dubbo

[TOC]

---

> Nacos既然连上了，当然是用试试远程调用的东东了！！！

### 使用Nacos本身的服务调用

> 好像也挺好用的，参数方法直接调用就可以了，用于访问远程Rest接口的。
>
> 推荐参考文章：<https://www.jianshu.com/p/27a82c494413>

这个比较简单，就是发个请求，URL地址使用服务名。

```java
// ======================  启动类 ================================================
@EnableDiscoveryClient  // 服务发现
@SpringBootApplication
public class DemoApplication {

    //  这里这小段 就是需要的加上的东东了
	@LoadBalanced
	@Bean
	public RestTemplate restTemplate() {
		return new RestTemplate();
	}

	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}
}
 // ====================== 控制测试类 ============================================
	private final RestTemplate restTemplate;

	@Autowired
	public SampleController(RestTemplate restTemplate) {this.restTemplate = restTemplate;}

    // 提供的服务
	@RestController
	class EchoController {
		@RequestMapping(value = "/echo/{string}", method = RequestMethod.GET)
		public String echo(@PathVariable String string) {
			return "Hello ，this is xiaohang-demo information : (#^.^#) Nacos Discovery " + string + "Hello " + userName + " " + age + "!";
		}
	}

    // 远程调用rest接口，使用服务名称。nacos会自行路由等。
	@RequestMapping("/getXhUser")
	public String getXhUser() {
		return restTemplate.getForObject("http://xh-demo/echo/" + "xiaohangTest", String.class);
	}
}
```

**RestTemplate** 就是个发送http请求的工具，而这里的不同用处，就是使用的URL并不是具体地址，而是nacos的服务名称。而已，就这样了  。。。。。

------

### Dubbo

> 简单说就是，远程服务调用的。非Rest接口调用。
>
> 参考文章：<https://segmentfault.com/a/1190000019896723> 【超级详细的】

#### **节点角色说明**

| 节点      | 角色说明                               |
| --------- | -------------------------------------- |
| Provider  | 暴露服务的服务提供方                   |
| Consumer  | 调用远程服务的服务消费方               |
| Registry  | 服务注册与发现的注册中心               |
| Monitor   | 统计服务的调用次数和调用时间的监控中心 |
| Container | 服务运行容器                           |

#### 整体步骤

> 使用的步骤还是很简单的，管它的实现细节复杂呢。

* 部署启动Nacos服务；【上一篇有文档，很简单】
* 启动容器，加载，**运行服务提供者**。
* 服务提供者在启动时，在注册中心**发布注册**自己提供的**服务**。
* 服务消费者在启动时，在注册中心**订阅**自己所需的**服务**。
* 服务消费者把服务当做本地调用就可以了。dubbo会自动注入。

#### 具体操作Demo

##### 服务端-Provider

###### **配置：**

```xml
		<dependency>
			<groupId>com.alibaba.cloud</groupId>
			<artifactId>spring-cloud-starter-dubbo</artifactId>
		</dependency>
```

```properties
######################## dubbo config : ########################
# 微服务治理控制台(Dubbo): https://edas.console.aliyun.com/#/dubboManage/SPServiceSearchConfig
# dubbo 服务扫描基础包路径
dubbo.scan.base-packages= com.example.demo.dubbo.provider    
# Dubbo 服务暴露的协议配置，其中子属性 name 为协议名称，port 为协议端口（ -1 表示自增端口，从 20880 开始）
dubbo.protocol.name= dubbo
dubbo.protocol.port= -1
# 挂载到 Spring Cloud 注册中心
dubbo.registry.address= nacos://192.168.1.180:8848
# 用于服务消费方订阅服务提供方的应用名称的列表，若需订阅多应用，使用 "," 分割。 不推荐使用默认值为 "*"，它将订阅所有应用。
# 这里默认使用了当前应用名，请根据需要增加对应的应用名
#dubbo.cloud.subscribed-services= ${provider.application.name}
dubbo.cloud.subscribed-services= *
```

###### **定义接口**

> 这个接口的定义，在消费的时候也会用到，所有我估计着项目使用的话，这一部分需要作为公共包引入，或者使用自己的meaven仓库才行。

```java
public interface UserService {
    String test(String message);
}
```

###### **接口实现**

```java
package com.example.demo.dubbo.provider;
// 这个@Service 就是dubbo会扫描并注册发布的服务注解了 ！！！！！
@Service
public class UserServiceImpl implements UserService {
    @Override
    public String test(String message) {
        return "this is provider service's information:"+message;
    }
}
```

然后？ 就没有了  。。。。。

##### 消费方-Consumer

###### **配置：**

跟服务端一样的，略过。只是消费的话可以不配置扫描包路径的。

###### **<font color=red>引入服务接口包：重点</font>**

> 这个就是我之前一直想不通，为什么消费项目上可以使用提供方的service类进行注入了。

把前一个项目Provider，点一下maven的install。然后就有了一个demo-0.0.1-SNAPSHOT.jar 这么个的包。

加到项目的lib引用里面去，然后就可以了。

###### **服务调用：**

```java
// 就是这个UserService接口，是在服务Provier里面定义的接口，这里要用到才能注入的。
// 就是上面Provier项目的包demo-0.0.1-SNAPSHOT.jar里面的接口定义
import com.example.demo.dubbo.api.UserService;

@RestController
public class DubboTest {

    // 这个注解会自动注入dubbo远程的服务的  
    @Reference
    private UserService userService;    

    @RequestMapping("/getDubboTest")
    public String getXhUser() {
        // 这里的使用 ，就跟本地调用一样就可以了，没有差别！！！！！
        return userService.test("Consumer's Test!!");
    }

}
```

好的，操作结束了

#### 测试一下

访问<http://localhost:8081/getDubboTest>   这是Consumer的项目的方法了。

返回内容：this is provider service's information:Consumer's Test!!  

证明，调用成功了！！！！！  **收工了！！！！！！**

------

### 小结一下

主要还是Dubbo的使用，用起来还是很方便的，不管实现的话，只要配置好，当做本身项目的服务直接调用就好。

注入什么，都是配置好自动的了，不用管！

**纠结的：**

因为，消费Consumer需要使用到服务端Provider的接口定义类，所以项目需要应用Provider的包才行。这个比较麻烦！

**所有的对外或者对内调用的接口，需要单独一个包进行管理。**
**实现与使用（远程调用） 都基于这个接口包即可！！** 

------

------

2020-04-15  小杭  (⊙o⊙)…  学不动了啊。。。。。。。

------

------



