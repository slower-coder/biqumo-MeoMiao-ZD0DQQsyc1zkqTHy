
### 1、先讲讲 protostuf


protostuf 一直是高性能序列化的代表之一。但是用起来，可难受了，你得先申明 protostuf 配置文件，并且要把这个配置文件转成类。所以必然要学习新语法、新工具。


可能真的太难受了！于是乎，（有不爽的人）搞了个有创意的框架 protostuff（多一个字母“f”）。它借用注解，替代了 protostuf 文件申明和生成类的过程，丰常的接地气。


### 2、再讲讲 rpc


一讲 rpc ，很多人会想到 dubbo （国产）和 grpc。估计还会联想到注册与发现服务；可能还会联想到微服务。可能就会觉得这个事儿“老重啦”，害怕！


其实很简单的，你请求一次 http 就是个 rpc 请求了（远程过程调用嘛）。最典型的就是 http \+ json 请求了。


### 3、现在讲 httputils \+ protostuff


这里我们会用到两个重要的 [solon 框架](https://github.com)的插件：一个是 httputils 工具插件，一个是 protostuff 序列化插件。



```
<dependency>
    <groupId>org.noeargroupId>
    <artifactId>solon-serialization-protostuffartifactId>
dependency>

<dependency>
    <groupId>org.noeargroupId>
    <artifactId>solon-net-httputilsartifactId>
dependency>

```

这里要感谢 solon 框架，它强调[三元合一（mvc 与 rpc 是自然一体的）](https://github.com):[FlowerCloud机场订阅官网](https://hanlianfangzhi.com)。下面，开始干活啦...


* 公用包（也可以在客户端，服务端分别定义实体类。只要 `@Tag` 顺序与类型对应上即可 ）


这里定义一个 protostuff 实体类。注意 `@Tag` 注解，它是替代 protostuf 配置文件的关键。



```
@Setter
@Getter
public class MessageDo {
    @Tag(1)    // Protostuff 注解，顺序位从 1 开始
    private long id;
    @Tag(2)
    private String title;
}

```

* 服务端（只支持 @Body 数据接收，只支持实体类）


在 solon web 项目里，添加一个控制器（注解可以用 `@Remoting` 或 `@Controller`）。使用 `@Remoting` 时，方法上不需要加 `@Mapping` 注解。



```
#添加插件
org.noear:solon-web
org.noear:solon-serialization-protostuff

```


```
@Mapping("/rpc/demo")
@Remoting
public class HelloServiceImpl {
    @Override
    public MessageDo hello(@Body MessageDo message) { //还可接收路径变量，与请求上下文
        return message;
    }
}

```

* 客户端应用 for HttpUtils（只支持 body 数据提交，只支持实体类）



```
#添加插件
org.noear:solon-net-httputils

```


```
//应用代码
@Component
public class DemoCom {
    public MessageDo hello() {
        MessageDo message = new MessageDo();
        message.setId(3);
        
        //指明请求数据为 PROTOBUF，接收数据要 PROTOBUF
        return HttpUtils.http("http://localhost:8080/rpc/demo/hello")
                .serializer(ProtostuffBytesSerializer.getInstance())
                .header(ContentTypes.HEADER_CONTENT_TYPE, ContentTypes.PROTOBUF_VALUE)
                .header(ContentTypes.HEADER_ACCEPT, ContentTypes.PROTOBUF_VALUE)
                .bodyOfBean(message)
                .postAs(MessageDo.class);
    }
}

```

### 4、总结


总体上，跟 json 没什么大的区别。主要是指定了：序列化器、内容类型、接收类型，让各端能识别类据类型。


### 5、也可以使用“注解式 http 客户端”框架


肯定也会有人觉得，一个接口还好，如果有很多接口就要写很多重复的http请求代码了。所以，“注解式 http 客户端” 很重要，这也是很多 rpc 框架流行的原因，就像调用本地接口一样，使用远程接口。


nami 是 solon 框架的 rpc 客户端（或者，注解式 http 客户端），支持各种序列化。（只要是“支持序列化定制”的注解式 http 客户端，用法都差不多）


* 添加两个依赖包



```
#添加插件
org.noear:nami-coder-protostuff # protostuff 编解码支持
org.noear:nami-channel-http     # http 请求通道支持，也可以是 socketd（支持 tcp, udp, ws）

```

* 代码应用（只支持 body 数据提交，只支持实体类）



```
@NamiClient(url = "http://localhost:8080/rpc/demo", headers = {ContentTypes.PROTOBUF, ContentTypes.PROTOBUF_ACCEPT})
public interface HelloService {
    MessageDo hello(@NamiBody MessageDo message);
    //方法2
    //方法3
    //方法4
    //方法5
    //方法6
}

@Component
public class DemoCom {
    @NamiClient //注入
    HelloService helloService;
  
    public MessageDo hello() {
         MessageDo message = new MessageDo();
         message.setId(3);
        
         rerturn helloService.hello(message);
    }
}

```

