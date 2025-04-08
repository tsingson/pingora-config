# pingora-config



draft in chinese.

## 0. 动机

我们一直有 api gateway 的需求

1. load balance 负载均衡
2. api 依据业务条件进行代理 
3. 有一些业务上的关联集成

我们实现了 go / rust 的不同版本, 在使用过程中, 在性能与业务定制上, 都有些限制

后来,  

[pingora](https://github.com/cloudflare/pingora) 开源,  反向代理与 api gateway 有了一个重量级的框架. 感谢 [https://github.com/cloudflare](https://github.com/cloudflare/)!!!


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

设计比较简单

1. pingsix 配置, 有对应的 protobuf 定义
2. 有一个 grpc 服务, 嵌在 pingsix 中, 提供 grpc-web 及 grpc 服务进行配置管理
3. 有一个 grpc-web + react 的 typescript 前端界面

这里,  protobuf 定义要简单匹配 pingsix  /pingora 的概念.

........ 


## 2. 实现

实现上, 一步一步来吧, 

1. fork  [pingsix](https://github.com/zhu327/pingsix)  , 调整配置部分, 与 grpc 配置进行对接
2. 加一个 tonic grpc 服务
3. 加一个 rust grpc client 示例 ( 兼作 cli 配置工具)
4. 加一个 golang grpc client 示例
5. 加一个 front 前端, 支持 grpc-web 实现 web 管理

 



