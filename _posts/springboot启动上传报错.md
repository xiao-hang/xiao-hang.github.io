## 日常问题：上传接口报错

项目启动后，一段时间没怎么用之后，系统所有上传接口都失败，报错为接口异常。

> 这个问题通常不会出现的，因为正常情况下使用，不可能会有7天不进行文件上传操作的。
>
> 因为财务上传报表是每个工作日的啊  ╮(╯_╰)╭  
>
> 结果碰到了国庆春节长假，还有今年疫情的超级长假。然后就炸了 ε=(´ο｀*)))唉

---

### 问题

```
Springboot上传文件错误：org.springframework.web.multipart.MultipartException
....
org.springframework.web.multipart.MultipartException: Failed to parse multipart servlet request; nested exception is java.io.IOException: The temporary upload location [/tmp/tomcat.8514542127953693245.8091/work/Tomcat/localhost/ROOT] is not valid
....
```

### 原因

linux系统会定期(10天)清除tmp目录下没有使用过的文件，springBoot启动的时候会在/tmp目录下生成一个tomcat.****.port(tomcat.8514542127953693245.8091)的文件目录，此目录要是清除后，就是出现上传文件错误！

> 具体清除脚本详情：
>
> 从/var/log/cron 日志中发现，服务器除了调用用户的计划任务外，还会执行系统自己的.
>
> 进入 /etc/cron.daily  , 可以看到一个tmpwatch
>
> 里面调用了/usr/sbin/tmpwatch 脚本 参数为240 =10*24 =10天 

### 解决

第一种临时方法，万能的重启。临时文件会重新生成。

第二种方法，重启，并且调整系统清理的时长，把上文原因中提到的脚本时间延长到100+天。【要是几百天都不用的系统，也就不用维护啦 罒ω罒】

第三种方法，项目配置Bean，提供临时文件目录

```java
    @Bean
    public MultipartConfigElement multipartConfigElement() {
        MultipartConfigFactory factory = new MultipartConfigFactory();
        //单个文件最大
        factory.setMaxFileSize("10240KB");
        /// 设置总上传数据总大小
        factory.setMaxRequestSize("102400KB");
        // 这种写法在linux下是没问题的，但是开发windows上会找不到目录
        // factory.setLocation("/var/tmp");
        // 由于默认/tmp 临时目录下 Linux 会清理7天无操作目录，所以换个上传目录试试
        String location = System.getProperty("user.home") + "/data/tmp";
        File tmpFile = new File(location);
        if (!tmpFile.exists()) {
            tmpFile.mkdirs();
        }
        factory.setLocation(location);
        return factory.createMultipartConfig();
    }
```

---

### 参考文档

就这个了，很详细的：https://blog.csdn.net/liuxiaoming1109/article/details/93467180

这个是linux下文件自动删除的文章：<https://www.tuicool.com/articles/6Jj6rq>

---

小杭 2020-02-19  _(:з」∠)_