## Spring-Cloud-Alibaba笔记00 - Portainer-Nacos-Sentinel和服务发现注册Demo

[TOC]

---

> Spring-Cloud-Alibaba的说明？   不存在的。。。。。看官方介绍去

### 安装docker 管理器 Portainer

```shell
[root@localhost ~]# yum install docker

[root@localhost ~]# systemctl start docker

[root@localhost ~]# systemctl enable docker

[root@localhost ~]# docker volume create portainer_data
portainer_data
[root@localhost ~]# docker volume ls
DRIVER              VOLUME NAME
local               portainer_data


[root@localhost /]# find -name portainer_data
./var/lib/docker/volumes/portainer_data


[root@localhost volumes]# sudo docker run -d -p 9000:9000 -p 8000:8000 --name portainer --restart always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer
Unable to find image 'portainer/portainer:latest' locally
Trying to pull repository docker.io/portainer/portainer ...
latest: Pulling from docker.io/portainer/portainer
d1e017099d17: Pull complete
a7dca5b5a9e8: Pull complete
Digest: sha256:4ae7f14330b56ffc8728e63d355bc4bc7381417fa45ba0597e5dd32682901080
Status: Downloaded newer image for docker.io/portainer/portainer:latest
ed1c016bf70355328c8c82ed8c0c0d3ddc280339361775083b8539ef1c5b751d


[root@localhost /]# docker ps
CONTAINER ID        IMAGE                 COMMAND             CREATED             STATUS              PORTS                                            NAMES
ed1c016bf703        portainer/portainer   "/portainer"        2 minutes ago       Up 2 minutes        0.0.0.0:8000->8000/tcp, 0.0.0.0:9000->9000/tcp   portainer

#### 虽然不知道发生了什么 ，但是已经安装好了  ╮(╯_╰)╭
```

**然后就可以访问了**

访问：http://192.168.1.180:9000

admin / 12345678

#### **接着就报错了  ε=(´ο｀*)))唉**

错误信息：连接失败

```
Failure
Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get http://%2Fvar%2Frun%2Fdocker.sock/_ping: dial unix /var/run/docker.sock: connect: permission denied
```

#### **解决方案：**

> 百度了很多的东东，都是说什么当前用户没有docker用户组权限，呀的我用的是root啊。
>
> 最后发现是 linux 系统的问题  。。。。。。。。。
>
> 参考文章：<https://blog.51cto.com/bguncle/957315>

SELinux 主要作用就是最大限度地减小系统中服务进程可访问的资源（最小权限原则）。

```
查看SELinux状态：sestatus 命令进行查看

1、/usr/sbin/sestatus -v      ##如果SELinux status参数为enabled即为开启状态
SELinux status:                 enabled
2、getenforce                 ##也可以用这个命令检查
关闭SELinux：
1、临时关闭（不用重启机器）：
setenforce 0                  ##设置SELinux 成为permissive模式
                              ##setenforce 1 设置SELinux 成为enforcing模式

2、修改配置文件需要重启机器：
修改/etc/selinux/config 文件
将SELINUX=enforcing改为SELINUX=disabled
重启机器即可
```

**然后就可以正常连接本地的docker进行管理了。**

Portainer控制台：192.168.1.180:9000   admin / 123123123

------

------

### 安装Nacos-1.1.4服务器

> 官方文档：<https://nacos.io/zh-cn/docs/use-nacos-with-dubbo.html>

下载1.1.4版本的完整安装包。：<https://github.com/alibaba/nacos/releases/tag/1.1.4>

```shell
[root@localhost nacos]# tar -zxvf nacos-server-1.1.4.tar.gz

[root@localhost bin]# ls
shutdown.cmd  shutdown.sh  startup.cmd  startup.sh
[root@localhost bin]# sh startup.sh -m standalone
/usr/local/java/jdk/bin/java  -Xms512m -Xmx512m -Xmn256m -Dnacos.standalone=true -Djava.ext.dirs=/usr/local/java/jdk/jre/lib/ext:/usr/local/java/jdk/lib/ext:/u01/nacos/nacos/plugins/cmdb:/u01/nacos/nacos/plugins/mysql -Xloggc:/u01/nacos/nacos/logs/nacos_gc.log -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M -Dnacos.home=/u01/nacos/nacos -Dloader.path=/u01/nacos/nacos/plugins/health -jar /u01/nacos/nacos/target/nacos-server.jar  --spring.config.location=classpath:/,classpath:/config/,file:./,file:./config/,file:/u01/nacos/nacos/conf/ --logging.config=/u01/nacos/nacos/conf/nacos-logback.xml --server.max-http-header-size=524288
nacos is starting with standalone
nacos is starting，you can check the /u01/nacos/nacos/logs/start.out
```

Nacos控制台：http://192.168.1.180:8848/nacos  nacos / nacos

------

------

### 安装Sentinel-1.7.2控制台

> 中文文档：<https://github.com/alibaba/Sentinel/wiki/%E6%8E%A7%E5%88%B6%E5%8F%B0>

下载项目的jar包，<https://github.com/alibaba/Sentinel/releases>，然后丢到docker去运行就可以了

```shell
docker run -d -p 8080:8080 --name sentinel --restart always -v /u01/sentinel:/u01/sentinel  hub.c.163.com/library/java:8-jre java -Dserver.port=8080 -Dcsp.sentinel.dashboard.server=localhost:8080 -Dproject.name=sentinel-dashboard -jar /u01/sentinel/sentinel-dashboard-1.7.2.jar
```

Sentinel控制台：http://192.168.1.180:8080   sentinel / sentinel

------

------

### 测试Demo

> 需要一个Demo，连接 Nacos , Sentinel 并且使用Dubbo 进行接口调用测试。
>
> 至于其他的网关等  ， 之后再说了  ╮(╯_╰)╭   

#### 生成DEMO

首先 ，使用 <https://start.aliyun.com/>  阿里DEMO项目生成器【好东东】

需要的组件为：**Spring Web,Nacos Service Discovery,Nacos Configuration,Spring Cloud Alibaba Sentinel .**

就这些，然后就是打开项目，删掉多余用不到的配置和代码了。【这个可以参考nacos官方文档，最简单版本的就行了】

#### 配置：

```properties
spring.application.name=xiaohang-demo
server.port=8080
# 设置配置中心服务端地址
spring.cloud.nacos.config.server-addr=http://192.168.1.180:8848

# Nacos 服务发现与注册配置，其中子属性 server-addr 指定 Nacos 服务器主机和端口
spring.cloud.nacos.discovery.server-addr=http://192.168.1.180:8848

######################## sentinel config : ########################
management.health.sentinel.enabled=false

spring.cloud.sentinel.transport.dashboard=http://192.168.1.180:8080
spring.cloud.sentinel.eager=true
```

#### 源码：

```java
@EnableDiscoveryClient  // 服务发现
@SpringBootApplication
public class DemoApplication {

	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}

}
```

```java
@RestController
@RefreshScope
class SampleController {

	@Value("${user.name}")
	String userName;

	@Value("${user.age:25}")
	int age;

	@RequestMapping("/user")
	public String simple() {
		return "Hello Nacos Config!" + "Hello " + userName + " " + age + "!";
	}

	@RestController
	class EchoController {
		@RequestMapping(value = "/echo/{string}", method = RequestMethod.GET)
		public String echo(@PathVariable String string) {
			return "Hello ，this is xiaohang-demo information : (#^.^#) Nacos Discovery " + string + "Hello " + userName + " " + age + "!";
		}
	}
}
```

**搞定，结束了。**

#### 测试一下 

访问 ：<http://localhost:8080/user>   

配置nacos  ：  Data id ： xiaoahng-demo  值 user.name = xiaohang-test .....【你随意】

返回内容：Hello Nacos Config!Hello xiaohang-test 111!

访问：Sentinel 控制台  ，也可以看到刚刚的服务接口流量了。。。。。  

【由于好像系统时间对不上，实时监控数据没有了 ε=(´ο｀*)))唉 】

#### 收工了

基础的就是这个样子了，之后再试试，原生服务调用 和 **Dubbo** 了。

------

------

2020-04-14 小杭

---

### 使用java1.8镜像说明

```
编辑：vi /etc/docker/daemon.json
{
"registry-mirrors": ["http://hub-mirror.c.163.com"]
}

重启 service docker restart

docker pull hub.c.163.com/library/java:8-jre
```

------

---

--spring.cloud.sentinel.transport.port=8992 --spring.cloud.sentinel.transport.dashboard=10.0.100.208:8080 --spring.cloud.sentinel.eager=true --spring.cloud.sentinel.enabled=true



---

### 遇到的问题

#### 控制台正确的规则推送后看不到

客户端成功接入控制台后，控制台配置规则准确无误，但是客户端收不到规则或者报错？比如配置的资源名正确但 `sentinel-record.log` 日志里面却报 resourceName 为空的错误？

**A:** 排查客户端是否使用了低版本的 fastjson，低版本的 fastjson 可能会有此问题，建议使用和 Sentinel 相关组件一致版本的 fastjson。

<font color=yellow>原来的版本是1.2.47 是不行的 ，改成1.2.58最新的版本就可以了！！！</font>

参考地址：

* 官方错误解决文档：<https://github.com/alibaba/Sentinel/wiki/FAQ>
* 参考的博客文章：<https://blog.csdn.net/qq_27641935/article/details/103196045> 

---

