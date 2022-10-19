# Triple 协议介绍

## 什么是 Triple 
Triple 是新一代RPC协议协议，它是基于 HTTP/2 上构建的 RPC 协议，完全兼容 gRPC，并在此基础上扩展出了更丰富的语义。Triple 底层使用了 gRPC 框架，SOFA RPC 将原生 gRPC 集成到自己的体系中，并通过底层 API 扩展了 gRPC 的功能。扩展了三个功能： 
1. 支持使用普通 Java interface 作为调用接口,普通 Pojo 作为接口参数 （gRPC只支持通过 proto 文件生成的接口和对象作为调用接口和参数）
2. 支持多种序列化方式，并可以横向扩展出更多序列化方式
3. 扩展 Header ，与SOFAR RPC、 Dubbo 适配

## 快读开始
<!-- todo 补充完整 -->

### HTTP/2 + protobuf (gRPC 
1. 创建 proto 文件
2. 生成 Java 代码
3. 编写引用和实现

### HTTP/2 + hessian over protobuf 
1. 直接编写引用和实现， 注明使用 triple 协议 

## Triple 工作原理

### 原生 gRPC 
<!-- todo 补充完整 -->
gRPC 使用 HTTP/2 协议。 通过 域名+端口+Path 的形式调用一个方法。其中 Path 形如： ‘/interfaceName/methodName’。这里的 interfaceName 和 methodName 均为 proto 文件中定义的。 <!-- todo 举个🌰 -->

比如我们写了如下一个 proto 文件： 
```proto
package helloworld;

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
  message DateTime {
    string date = 1;
    string time = 2;
  }
  DateTime dateTime = 2;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
  HelloRequest.DateTime dateTime = 2;
}
```

通过这个文件， 我们可以生成一系列 Java 类, 通过 Quick start 中的配置方式，我们可以配置客户端和服务端。在服务端中，我们会向 HTTP/2 Server 注册一个Path： `helloworld.Greeter/SayHello` 。凡是请求这个url， HTTP/2 Server 就会将请求转发给我们的实现类。

客户端实现也比较简单，通过proto生成出来的代码， 我们可以创建一个 stub。 （gRPC stub 相当于 SOFARPC 的引用）

在 stub 内部持有一个 MethodDescriptor 数据结构。 MethodDescriptor 包含了请求的 Path、参数序列化方法、 返回值序列化方法。stub 通过 MethodDescriptor 来序列化、反序列化，并发送请求到 `helloworld.Greeter/SayHello`。


### 扩展协议
我们定义了如下一个通用 proto 文件： 
```proto
syntax = "proto3";

option java_multiple_files = true;
option java_package = "triple";
option java_outer_classname = "GenericProto";

service GenericService {
    rpc generic (Request) returns (Response) {}
}

message Request {
    string serializeType = 1;
    repeated bytes  args = 2;
    repeated string argTypes = 3;
}

message Response {
    string serializeType = 1;
    bytes  data = 2;
    string type = 3;
}
```
> 下文通过 triple.Request 和 triple.Response 来指代上面proto中定义的两个类。

在客户端，我们将普通接口+Pojo 适配到 GenericService 接口和 triple.Request 参数上，然后通过 gRPC 调用服务端的 GenericService 接口， 服务端再将请求适配到普通接口的实现类上。 


具体来说，我们有一个接口：
```java
package com.alipay.sofa.rpc.triple;

public interface TripleHessianInterface {
    String echo(String name);
}
```
它有一个实现类： 

```java
public class TripleHessianInterfaceImpl implements TripleHessianInterface {

    @Override
    public String echo(String name) {
        return name;
    }
}
```



在服务端，当我们将 TripleHessianInterfaceImpl 声明成一个 Triple 服务时，我们会向 HTTP/2 Server 注册 Path `com.alipay.sofa.rpc.triple.TripleHessianInterface/echo`,和一个处理类，处理类是一个实现了 com.alipay.sofa.rpc.server.triple.GenericServiceImpl 的适配器，它会将请求适配到 TripleHessianInterfaceImpl 上。


在客户端，我们将 TripleHessianInterface 声明成一个 Triple Reference，SOFARPC 不再使用 gRPC stub 作为发送请求的 Client 而是通过 `ClientCalls.blockingUnaryCall` 静态方法调用服务端，这个方法需要如下参数： 

* **channel：** channel 代表一个链接，由 SOFARPC 进行维护
* **methodDescriptor：** 这是一个特殊的 MethodDescriptor： 它以 `com.alipay.sofa.rpc.triple.TripleHessianInterface/echo` 作为Path，以 Request 作为请求序列化目标，Response 作为返回序列化目标。 
* **options：** 请求选项，添加了一些请求配置比如超时时间等
* **Request：** 请求参数，这个参数是通过通用 proto 文件定义出来的triple.Request对象。 它内部存储了原始 Java 方法的参数序列化后的二进制值。

整体请求过程如下： 
1. 用户调用Reference 
2. SOFA RPC 通过自定义序列化方式将请求序列化，并构造 triple.Request
3. 通过blockingUnaryCall 将 triple.Request 发送到服务端的 `com.alipay.sofa.rpc.triple.TripleHessianInterface/echo` Path 上
4. 服务端将请求交给 `GenericServiceImpl` 处理，`GenericServiceImpl` 通过自定以反序列化方式反序列化请求， 并将请求给具体的 `TripleHessianInterfaceImpl` 实现类处理
5. 返回值亦有上述处理流程。



> SOFA RPC 在 Transport 层添加了对 Triple 协议的客户端适配（实现在 [TripleClientTransport.java](https://github.com/sofastack/sofa-rpc/blob/master/remoting/remoting-triple/src/main/java/com/alipay/sofa/rpc/transport/triple/TripleClientTransport.java)）。<br/>
> 服务端适配代码在 [GenericServiceImpl.java](https://github.com/sofastack/sofa-rpc/blob/master/remoting/remoting-triple/src/main/java/com/alipay/sofa/rpc/server/triple/GenericServiceImpl.java)。


