# Golang工程化实践（一）

本专栏将分享使用Golang语言落地工程化项目的一些实践，作为首篇文章，本文将介绍Golang落地的工程项目结构演进过程及详细管理方式，并简单介绍Golang包管理的应用总结。

## 项目结构

如果只需要使用Golang写一个Hello world，那么一个main.go就能满足要求；但当Golang落地到具体工程项目，多人协同开发就需要有一个统一的工程目录布局。下面基于项目实践介绍从初期使用传统的MVC的目录结构到基于DDD的微服务架构目录结构的演进过程。

### MVC目录结构

```
.
├── cmd
├── conf
├── docker
├── docs # 项目所有文档
├── hack # 工具目录
├── Makefile
├── pkg
│   ├── api
│   ├── cron # 相当于cronjob
│   ├── middleware # 中间件
│   ├── service
│   │   ├── auth
│   │   │   ├── proto          
│   │   │   │   └── auth.proto
│   │   │   └── service.go
│   ├── store
│   │   ├── auth
│   │   │   ├── auth.go
│   │   │   └── interface.go
│   └── utils
├── scripts
├── test
│   └── testdata
├── tools
└── version
└── README.md
└── CHANGELOG.md
└── OWNERS.md
```

#### 目录解析

以下是部分关键目录的作用简介：

- /cmd：后端server的main函数，基于cobra实现
- /conf：server的所有配置项，一份配置文件配置多个服务，toml格式定义配置
- /pkg：所有业务代码所在的目录，包含以下子目录
  - api：使用[grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway)将HTTP请求转gRPC请求，认证也在这个目录下实现，在不需要独立网关的场景下使用这种方式是比较不错的选择
  - service：定义服务的目录，目录下有一个proto目录，里面存放详细的gRPC接口定义，service目录下是对gRPC接口的具体实现
  - store：持久化层目录，下面有不同服务的持久化数据定义以及数据读写的实现，其中：interface.go定义了数据库表结构以及抽象出一层接口，auth.go实现对数据库的读写

#### 软件模型

![img](https://pic2.zhimg.com/80/v2-03b0db4e62b2e1f843dff4ae6a4c5e82_720w.png)

软件模型图中，各板块的功能定义如下表所示：

| 项目        | 定义                           | 功能                                                         |
| ----------- | ------------------------------ | ------------------------------------------------------------ |
| dto         | 数据传输对象                   | 泛指用于展示层/API 层与服务层（业务逻辑层）之间的数据传输对象，由protobuf文件生成定义在*pb.go文件 |
| do          | domain object，领域对象        | 从现实世界中抽象出来的有形或无形的业务实体，上述目录树/pkg/service目录下的service.go文件包含do定义 |
| model（po） | 持久化数据定义                 | 可以理解为数据库表字段，对应上述目录树/pkg/store目录下的interface.go文件 |
| dao         | 数据读写层                     | 数据库和缓存全部在这层统一处理，对应上述目录树/pkg/store目录下的auth.go文件 |
| service     | 组合各种数据访问来构建业务逻辑 | 上述目录树/pkg/service目录下的service.go文件包含服务实现     |

思考：为什么需要三套数据结构（model、do、dto）？

-  使用一套数据结构的问题：dto的有些数据不需要也不能相互展示；多层级的代码使用一套数据结构会导致一些不必要的依赖，如Rob Pike说过A little coping is better than a little dependency。
-  do和model的区别：do为了实现业务逻辑可能会衍生一些临时对象和属性，这些数据可能只会在内存，不会持久化。model的中间表model可能不需要对do表示。model可能会为了持久化的某些性能做优化，而do不需要考虑，比如为了实现“乐观锁”以及冷热数据隔离。

项目初期阶段，业务场景不复杂（少于30个story），老版本的架构是完全足够支撑的。

### monorepo目录结构

基于[kratos](https://github.com/go-kratos/kratos-layout)项目的微服务monorepo目录结构中，目录相比第一个版本复杂了一些（嵌套多层）。其原因是随着软件复杂度上升，我们需要使用分而治之的方式来降低开发人员的思维负担，每一个小团队只需要负责做好属于自己的一个或多个相关联的服务即可。

```
├── api
│   └── user
│       ├── admin
│       │   └── v1
│       │       ├── error.proto
│       │       └── user.proto
│       ├── interface
│       │   └── v2
│       │       ├── error.proto
│       │       └── user.proto
│       └── service
│           └── v1
│               ├── error.proto
│               └── user.proto
├── app
│   └── user
│       ├──README.md
│       ├──CHANGELOG.md
│       ├──OWNERS.md
│       ├── admin
│       ├── interface
│       └── service
│           ├── cmd
│           │   └── user
│           │       ├── main.go
│           │       ├── wire_gen.go
│           │       └── wire.go
│           ├── configs
│           │   └── config.yaml
│           ├── internal
│           │   ├── biz
│           │   │   ├── biz.go
│           │   │   └── user.go
│           │   ├── conf
│           │   │   └── conf.proto
│           │   ├── data
│           │   │   ├── data.go
│           │   │   └── user.go
│           │   ├── pkg
│           │   │   └── util
│           │   │       └── password.go
│           │   ├── server
│           │   │   ├── grpc.go
│           │   │   ├── http.go
│           │   │   └── server.go
│           │   └── service
│           │       ├── list.go
│           │       ├── list_test.go
│           │       ├── service.go
│           │       ├── test
│           │       │   ├── conf
│           │       │   │   └── config.yaml
│           │       │   └── docker-compose.yaml
│           │       ├── wire_gen.go
│           │       └── wire.go
│           └── Makefile
├── deploy
│   ├── docker-compose
│   │   └── user
│   │       ├── docker-compose.yaml
│   │       └── Dockerfile
│   └── k8s
│       └── user
│           ├── conf
│           │   └── config.yaml
│           └── server.yaml
├── docs
├── hack
├── Makefile
└── pkg
    ├── log
    │   └── zap.go
    ├── middleware
    └── plugin
        └── apisix
```

#### 目录解析

以下是部分关键目录的作用简介：

- /app：app目录下会有三种类型的微服务代码

  -  interface：[BFF](https://samnewman.io/patterns/architectural/bff/)服务对前端暴露的接口实现，使用HTTP协议，kratos提供protobuf插件生成HTTP代码。interface会根据业务场景聚合多个内部服务接口

  - admin：运营测服务，通常数据权限更高，隔离带来更好的代码级别安全。用户权限管理、算法运维平台都算admin类型服务

  - service：对内的微服务。只接受来自其他内部服务的请求，目前大部分服务都是service服务。服务之间通信使用[gRPC](https://github.com/grpc)，并且在设计服务的时候[**一定避免循环依赖**](https://xie.infoq.cn/article/fd415110fbefe11b12b1d127f)，所以在做service层时候尽量做到小而美。

    - cmd：服务的启动代码，main函数所在目录，会涉及到程序的lifecycle。使用[googlewire](https://github.com/google/wire)注入的目录,为什么使用wire？

      1. 方便测试
    
      2. 单次初始化即可多次复用，对象可以由注入框架来控制实例化，减少重复代码，简易清晰的依赖管理，代码示例如下：
    
            ```go
                  type Person struct {
                  }
                  
                  func (d *Data) SavePerson(ctx context.Context,p *Person) error {
                  	//	save逻辑
                  	return nil
                  }
                  
                  type Data struct {
                  	db *sql.DB
                  }
                  
                  // Init方法传配置进去初始化
                  func (d *Data) Init(user, passwd, db string) (err error) {
                  	d.db, err = sql.Open("mysql", fmt.Sprintf("%s:%s@/%s", user, passwd, db))
                  	return err
                  }
                  
                  // Init方法传构造好的db进去初始化 db是个接口 那么在测试的时候很方便可以进行mock
                  func (d *Data) Init(db *sql.DB) {
                  	d.db =db
                  }
            ```
    
    
    - configs：单个服务的配置信息，使用yaml格式定义，后面会详细分析配置实践
    - internal：这个是在MVC目录结构上新增加的目录名称。该目录的作用主要是屏蔽掉存在相同业务逻辑跨服务引用代码，详细设计理念请参考本文“kratos的设计理念”部分
    
  
- /api：定义服务、前后端、后端算法之间的接口协议，统一使用protobuf进行描述，其中包括生成的go文件。该目录使用git submodule生成，api目录即interface仓库。该项目已经涉及到多个团队的协作开发，整个项目组的研发，前后端、算法、测试都使用protobuf进行接口对齐，团队之间不使用swagger对接口。使用swagger的弊端：dto和swagger文档需要保证强一致，开发人员忘记更新swagger会导致灾难性BUG；需要维护swagger文档增加工作量

- /pkg：所有服务公用的目录，目前存放基于APISIX开发的一些插件以及一些通用服务中间件实现

- /deploy：整个项目服务基于docker-compose和k8s部署的yaml目录

#### kratos的设计理念

在monorepo目录结构中，internal目录下也是核心业务代码，我们基于kratos重新设计了目录结构，首先看看kratos的设计理念 ： 

![img](https://pic1.zhimg.com/80/v2-ce846facc0703ce8092d0d1c43f5d6b4_720w.png)

图中的浅色板块是[DDD中相关概念](https://domain-driven-design.org/zh/ddd-concept-reference.html)，深色板块是kratos中的目录设计概念。 以下是比对MVC目录架构的总结：

- 新引入依赖倒置在biz层实现，虽然说还是定义了三种类型的数据结构，但新架构根据每个Object的作用不同，把Object定义划分到了不同目录；
- dto依然使用protobuf定义，在service目录下do转dto；
- do定义在biz目录，并且在这儿使用依赖倒置；
- data层依赖biz层的接口定义，相关的业务逻辑在data层实现（充血模型）；
- 使用ent定义po（ORM框架推荐使用ent），do转po在这一层实现。
- 老版本的架构符合松散分层架构（Relaxed Layered System），新版本架构既符合松散分层架构（Relaxed Layered System）也符合继承分层架构（Layering Through Inheritance）即高层继承并实现底层接口，好处可想而知，data层的倒置依赖使得基础设施无论怎么改变核心业务代码也无需改变。

## 包管理

Go 依赖管理是通过 Git 仓库模式实现，该部分是对Golang包管理的一些总结。

### 发展历程

- GOPATH 模式：所有需要编译的 go 工程的依赖包都放在 GOPATH 目录下

```bash
  GOPATH
  ├── bin  # go编译生成的二进制文件
  ├── pkg  # go存放预编译的目标文件的目录，用来加速后续程序的编译
  ├── src  # 存放源码文件
```

- Vendor：go 1.6 之后开启了 vendor 目录，在vendor目录中存放对应版本的依赖包
- Go Modules：go 1.11支持Go Modules,go 1.13版本默认开启，由 go.mod 和 go.sum 组成，主要包括依赖模块路径定义，以及通过 checksum 方式进行保证包的安全性验证，并且可以在 GOPATH 外进行创建和编译项目。

### Go Modules

在项目目录下执行bash   go mod init会生成go.mod文件，项目随即成为Go Modules项目。

执行命令bash   go get google.golang.org/grpc@v1.42.0，会在go.mod生成

![img](https://pic2.zhimg.com/80/v2-03607cb406f00b8c9fcb9ffc3a6dd984_720w.png)

基本语法：

- module：定义当前项目的模块名称
- go：标识当前模块的 Go 语言版本
- require：说明 module 需要什么版本的依赖
- exclude：用于从使用中排除一个特定的模块版本
- replace：替换 require 中声明的依赖，使用另外的依赖及其版本号

Checksum：

记录hash，并比较hash是否一致

1. go.sum 文件中每行记录由 module 名、版本和哈希组成，并由空格分开
2. 在引入新的依赖时，通常会使用 go get 命令获取该依赖，将该依赖包（依赖包是一个后缀为 .zip 的压缩包）下载到本地缓存目录 $GOPATH/pkg/mod/cache/download，并把哈希运算同步到 go.sum 文件中
3. 在构建应用时，会从本地缓存中查找所有 go.mod 中记录的依赖包，并计算本地依赖包的哈希值 ，然后与 go.sum 中的记录进行对比，当校验失败，go 命令将拒绝构建

## 参考链接

- [DDD中的相关概念](https://domain-driven-design.org/zh/ddd-concept-reference.html)
- [Go Modules官网详细介绍](https://go.dev/ref/mod)