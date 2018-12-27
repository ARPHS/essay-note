# SpringCoudBus自动刷新配置
bus ： 这里的意思是 总线 的意思

## bus 理论/原理
在 配置中心的时候没有贴这个图，使用文字图案描述的。

当时是没有bus来进行通知已经配置已经修改过了；


![](/assets/image/imooc/spring_cloud/snipaste_20180818_112415.png)

* config-server 与 config-client 之间使用消息队列通信

  当有新配置的时候，发送消息到client的队列中，然后client完成配置拉取和更新操作
* bus-refresh 是通知config-server git上的配置已经更新，可以让客户端进行更新了

  可以利用git的webooks功能自动触发 bus-refres的端点

## 初体验-configServer配置

加入依赖
```
compile('org.springframework.cloud:spring-cloud-starter-bus-amqp')
```

视频中引入该依赖之后，就能启动，然后mq队列中就生成了一个新的队列，而实际mq不能自动连接。

原因可能是视频中的mq安装在本地，而自己把mq安装在虚拟机上了，所以这里查看官网文档，怎么配置mq的连接信息
> 官网地址： http://cloud.spring.io/spring-cloud-static/spring-cloud-bus/2.0.0.RELEASE/single/spring-cloud-bus.html

配置自己的rabbitmq信息，再次启动就能连接上mq了
```
spring:
  rabbitmq:
    host: 192.168.106.128
    port: 5672
    username: guest
    password: guest
```

## 初体验-configClient配置
客户端也同样需要加依赖和配置mq连接信息；启动客户端后，即可看到 mq队列里面生成了一个新的队列；

如果客户端不能启动，启动报错，报错信息如果是：包含 localhost:888 的信息，那么还记得之前把配置放在git上。

eureka的连接信息要放入如bootstrap.yml中吗？不然找不到服务注册中心的连接

测试是否能热更新配置信息：
* 启动configServer 和 configClient
* 访问client的地址：http://localhost:9001/env/print 打印现在的 env属性信息
* 更改order-dev.yml中的env属性，然后提交到git
* 再次访问client的地址：http://localhost:9001/env/print 打印现在的 env属性信息

这个时候会发现没有效果；

## 总线刷新端点
> http://cloud.spring.io/spring-cloud-static/spring-cloud-bus/2.0.0.RELEASE/single/spring-cloud-bus.html#_bus_refresh_endpoint

原因是bus有一个刷新端点：/actuator/bus-refresh
要暴露该端点需要以下配置
```yaml
management:
  endpoints:
    web:
      exposure:
        include: bus-refresh
```
该端点是在配置中心开启的：git仓库中的配置更新之后，需要访问该端点，发送mq消息到各个客户端，通知客户端进行配置的更新；

重启项目后，会发现mapping中出现了 `/actuator/bus-refresh`地址；

使用post方法访问配置中心：http://localhost:9050/actuator/bus-refresh

而客户端需要配合`@RefreshScope`的注解来让属性所在的类进行刷新
```java
@RestController
@RefreshScope   // 刷新注解
public class EnvController {
    @Value("${env}")
    private String env;

    @GetMapping("/env/print")
    public String env() {
        return env;
    }
}
```

热更新已经实现，但是怎么自动化呢？

## 使用git的webHooks功能自动触发`/actuator/bus-refresh`
在git上有一个 webHooks 的功能，当仓库的指定事件发生后，会调用你指定的一个地址；

在开发中需要一个域名来让gitosc调用到我们的配置中心；

可以使用`natapp.cn`软件进行内网穿透映射到配置中心；（花生壳也可以，不过花生壳现在需要身份信息认证才能使用）


家里网络太差，在webHooks回调的时候，发生了参数解析异常。到时候再看看是怎么回事；

码云的webhooks发送的参数，不能被正常解析，视频中说使用github来演示

> 实战后补

在 gitlab 中也出现了参数解析异常的，发送过来的 json 串多了最后一个字符"]"，导致 json 解析失败。

解决方案：重新写一个 Controller 转调

```java
@RestController
@Slf4j
public class EventPushForwardController {
    // 全文搜索 bus-refresh 找到目标类
    @Autowired
    private RefreshBusEndpoint refreshBusEndpoint;

    @RequestMapping("/actuatorx/bus-refresh")
    public void busRefresh() {
        refreshBusEndpoint.busRefresh();
        log.info("配置更新推送");
    }
}

```

## natapp 使用教程
> 官网：https://natapp.cn
> 新手一分钟教程：https://natapp.cn/article/natapp_newbie

使用步骤：
1. 下载程序
2. 在官网注册账户，
3. 登录账户，购买隧道-免费隧道；（需要身份证信息认证）
4. 我的隧道页面中，能看到
5. 在下载的natapp.exe 同级目录创建 config.ini 文件
  ```
  [default]
  authtoken=你的authtoken
  ```
6. 点击natapp.exe 即可完成内网穿透；每次启动会被重新分配一个地址

## bus相关的坑

视频中说的monitor在新版中不存在了。config-monitor 项目也不存在了
