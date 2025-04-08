# pingora-config



draft in chinese.

**一个 rust 初学者的练手项目**

----------

## 0. 动机

我们一直有 api gateway 的需求

1. load balance 负载均衡
2. api 依据业务条件进行代理 
3. 有一些业务上的关联集成

我们实现了 go 的 proxy, 业务上满足要求, 但性能上掉链子. 接着, 尝试了 rust 的 proxy, 性能上满足了, 业务集成上基本配置文件硬写, 动态调整上有缺失.



后来,  

[pingora](https://github.com/cloudflare/pingora) 开源,  反向代理与 api gateway 有了一个重量级的框架. 感谢 [cloudflare](https://github.com/cloudflare/) !!!


[pingsix](https://github.com/zhu327/pingsix) 给了我极大的惊喜. 这个项目由 [apisix](https://apisix.apache.org/) 启发,  作者[Timmy](https://github.com/zhu327) 的专业/专注, 让我赞叹. [Timmy的博客](https://zhu327.github.io/) 值得看一看.

再就是这个访谈
https://pythonhunter.org/episodes/ep47 , 其中谈到 pingora 的配置是一个有趣的方向

so, 我开始进行一个小尝试

写一个基于 [pingsix](https://github.com/zhu327/pingsix)  的配置工具

为什么基于 [pingsix](https://github.com/zhu327/pingsix) :

1.  [pingsix](https://github.com/zhu327/pingsix) 代码架构非常优秀, 即简单又易于扩展, 适合于我这样的 rust 初学者
2.  [pingsix](https://github.com/zhu327/pingsix) 基于 [pingora](https://github.com/cloudflare/pingora) , 紧随着 pingora 的框架/思想, 插件/扩展简单而不失灵活性
3.   [pingsix](https://github.com/zhu327/pingsix) 继承了 [apisix](https://apisix.apache.org/) 的核心功能, 包括使用 etcd 进行配置同步, 复用了  [apisix](https://apisix.apache.org/) 的 web 配置界面

但是,  我们有一些业务集成的要求, 需要用 grpc 

1. 需要 grpc 来配置 api gateway 
2. 需要 api gateway 同步一些信息到我们的业务系统, 方便触发一些关联操作
3. 我们的使用 grpc-web ( 结合 react ) 进行 web 管理

所以, 基于 [pingsix](https://github.com/zhu327/pingsix) 优秀的架构与代码, 我们得以修改很少的代码, 并结合 grpc 可以达到我们的目标

1. 有 grpc-web 的 web ( react ) 管理界面
2. 基于 grpc 可以扩展配置储存与同步, 不限于使用 etcd 进行配置同步

[pingcap](https://github.com/vicanso/pingap)  这个开源项目也很优秀. 只是这个大而全的开源项目, 只是配置上有点复杂, 扩展上, 也有点复杂.

## 1. 设计

### 1.1  概述

配置嘛, 设计比较简单

1. pingsix 配置, 有对应的 protobuf 定义
2. 有一个 grpc 服务, 嵌在 pingsix 中, 提供 grpc-web 及 grpc 服务进行配置管理
3. 有一个 grpc-web + react 的 typescript 前端界面

这里,  protobuf 定义要简单匹配 pingsix  /pingora 的概念.

### 1.2 原配置拆解

以下是 pingsix 的一个配置示例

```
pingora:
  version: 1
  threads: 32
  pid_file: /run/pingora.pid
  upgrade_sock: /tmp/pingora_upgrade.sock
  user: nobody
  group: webusers

pingsix:
  listeners:
    - address: 0.0.0.0:8084
      offer_h2c: false
      offer_h2: false
      
  etcd:
    host:
      - "http://192.168.2.141:2379"
    prefix: /apisix

  admin:
    address: "0.0.0.0:8082"
    api_key: pingsix

  prometheus:
    address: 0.0.0.0:8081

  sentry:
    dsn: https://1234567890@sentry.io/123456
    
routes:
  - id: web
    uris:
      - /api.{*p}
      - /
      - /static/{*p}
    upstream_id: grpc

  - id: 5
    uri: /echo
    host: www.163.com
    service_id: 2
    plugins:
      echo:
        body: "Hello world!"
        headers:
          X-TEST: demo

  - id: photo
    uri: /storage/{*p}
    upstream_id: preview


upstreams:
  - id: grpc
    nodes:
      "192.168.1.3:8083": 1
      "192.168.1.4:8083": 1
    type: roundrobin
    scheme: http

  - id: preview
    nodes:
      "192.168.2.1:8081": 1
    type: roundrobin
    scheme: http
    
 - id: 5
    uri: /echo
    host: www.163.com
    service_id: 2
    plugins:
      echo:
        body: "Hello world!"
        headers:
          X-TEST: demo

services:
  - id: 1
    hosts: ["www.qq.com"]
    upstream_id: 2
  - id: 2
    upstream:
      nodes:
        "www.163.com": 1
      type: roundrobin
      scheme: http
    plugins:
      limit-count:
        key_type: head
        key: Host
        time_window: 1
        count: 1
        rejected_code: 429
        rejected_msg: "Pleas slow down!"
        
global_rules:
  - id: global
    plugins:
      logger: {}

```



拆解后, 分为 5个部署 

1. pingora 运行配置
2. pingsix 配置, 可以理解为 proxy 的基本配置
3. routes 路由配置, 包括路由上的 plugin 及一些逻辑
4. upstreams 配置
5. services 配置
6. global_rules 配置, 基本上可以看作全局的, 适合所有 routes 的 plugin 

作为一个基础的 api gateway , 配置就是4个

```
route 路由( 就是 uri 匹配/参数条件) --> plugin 插件 --> upstream 上游服务器 ( nodes 服务器)
                                               --> services 服务
```



### 1.3 配置的 protobuf 定义

pingora 运行配置

```
pingora:
  version: 1
  threads: 32
  pid_file: /run/pingora.pid
  upgrade_sock: /tmp/pingora_upgrade.sock
  user: nobody
  group: webusers
```

对应的 protobuf

```
syntax = "proto3";

package api.px.v1;

import "api/px/v1/plugin.proto";

message ConfigPingora {
  string version = 1;
  uint32 threads = 2;
  string pid_file = 3;
  string upgrade_sock = 4;
  string user = 5;
  string group = 6;
}
```



路由配置

```
routes:
  - id: web
    uris:
      - /api.{*p}
      - /
      - /static/{*p}
    upstream_id: grpc

  - id: 5
    uri: /echo
    host: www.163.com
    service_id: 2
    plugins:
      echo:
        body: "Hello world!"
        headers:
          X-TEST: demo

  - id: photo
    uri: /storage/{*p}
    upstream_id: preview
```

可以看到 

```
    plugins:
      echo:
        body: "Hello world!"
        headers:
          X-TEST: demo
```

是 plugin 配置, 这里可以抽取 plugin 进行配置复用



对应的 protobuf 

```
message ConfigRoute {
  string version = 1;
  string id = 2;
  repeated string uris = 3;
  optional string upstream_id = 4;
  repeated string methods = 5;
  repeated string plugin_id_list = 6;  // 相同配置, 复用
  repeated Plugin plugin_list = 7;
  optional string service_id = 8;
  repeated string hosts = 9;
  optional string host = 10;
  optional ConfigTimeout timeout = 11;
  uint32 priority = 12;
}
```





upstream 配置

```
  - id: grpc
    nodes:
      "192.168.1.3:8083": 1
      "192.168.1.4:8083": 1
    type: roundrobin
    scheme: http
```

对应 protobuf 

```
// 超时
message ConfigTimeout {
  uint64 connect = 1;
  uint64 send = 2;
  uint64 read = 3;
}

// 节点
message UpstreamNode {
  string address = 1;
  uint32 priority = 2;
}

// upstream 
message ConfigUpstream {
  string version = 1;
  string id = 2;
  uint32 retries = 3;
  uint64 retry_timeout = 4;
  repeated UpstreamNode nodes = 5;
  string scheme = 6;
  optional ConfigTimeout timeout = 7;
}
```







## 2. 实现

实现上, 一步一步来吧, 

*  定义 protobuf 接口, 用 [bufbuild/buf ](https://github.com/bufbuild/buf) 进行生成
*   tonic grpc 服务
*   rust grpc client 示例 ( 兼作 cli 配置工具)
*   golang grpc client 示例
*   front 前端, 支持 grpc-web 实现 web 管理, 用 [nice-grpc-web](https://github.com/deeplay-io/nice-grpc)
*   fork  [pingsix](https://github.com/zhu327/pingsix)  , 调整配置部分, 与 grpc 配置进行对接



## 3. 限制

仅在  linux / macOS 上测试 

 



