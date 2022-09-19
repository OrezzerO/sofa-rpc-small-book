# 数据流转

## 前言
上篇文章我们介绍了 SOFARPC 中一些重要的组件,以及他们是怎么交互的. 本文主要介绍他们他们之间数据如何流转. 本文依然是概括性质的,没有代码级别的分析,希望读完本文的朋友能对 SOFARPC 的数据流有个初步的印象,如果能有"这流程走得通了"的感觉那就更好了.


## 客户端发起请求
我们依然从 HelloService 的例子开始讲. 用户使用 `ConumserConfig` 来创建 HelloService 引用. 这个 ConsumerConfg 保存了这个服务的元数据.它在整个调用过程中有着重要作用: Cluster/Router/Filter/Proxy 等组件, 都会持有对应服务的 ConsumerConfig 引用.

proxy

-> sofa request
+ InterfaceName
+ MethodName
+ Method
+ MethodArgs
+ MethodArgSigs


ClientProxyInvoker  （ConsumerConfig）

-> sofa request
+ targetServiceUniuqeName interface:uniqueId
+ invokeType (sync/callback/future)
+ serializeType （序列化类型（byte））
+ timeout (调用维度超时)
+ requestProp.app  
+ requestProp.protocol  （tri/bolt）


Cluster -> filter  -> ConsumerInvoker

+ targetAppName
