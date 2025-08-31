
+++

date = '2025-08-28T21:01:22+08:00'
draft = false
title = 'Lineman Mart项目 关注点梳理'
summary = '加入新团队后，陌生的技术栈和工具是否影响日常研发效率呢？我正在经历这种苦恼。我在想，能否梳理一套核心的待确认点，专业且高效地一步问到位...'


+++


作为「加入新团队后，我们最应该关注哪些点？」的实践篇，我想结合现在L团队的Mart Router服务和Mart Ranker服务来做一次详细了解。


## 从工具角度，优先跑通的研发流程

### 本地开发流程
IDE及插件：团队成员使用idea开发为主，ai场景会使用cursor。但公司内部不允许使用盗版idea软件，so，目前新入职员工以cursor作为开发ide。
研发辅助工具：
1. LENS：它是一款为k8s管理工具。能操作k8s资源，查看Pod运行状态。
2. Postman：能够执行HTTP/gRPC请求，且可以在团队之间便捷共享。

### 代码协作流程
代码分支模型：首先，开发阶段在个人分支上开发；接下来，本地自测和大连beta环境的测试，都是在个人分支上，tag是测试标签；然后是泰国侧的beta/beta2/rc环境测试，也是在个人分支上，tag是正式版本标签；最后，在部署上线前，需要把个人分支合并入master分支，并生成最终版本的镜像，部署上线。

提交规范：无严格规范，声明出本次变更内容以及关联的Jira issueID即可。

### 代码审查流程
代码平台：是Gitlab套壳。进入后大连团队的项目都在dalian-dev/ai-serving下，开发过程也会更改部分泰国仓库，比如proto/Ceylon仓库，具体仓库地址可以找leader。

审核人及审核流程：在部署到大连beta后，相关的MR要发出来，给团队成员审核；在部署泰国侧beta/beta2/rc环境前，需要将Ceylon MR发出来给负责人和泰国侧SRE审核；在部署prod前，proto仓库/业务仓库代码需要合并入master，需要给负责人进行审批。

### 测试流程
环节确认：本地dev自测 -> 大连Beta环境 -> 泰国侧beta/beta2 -> rc环境(即staging) -> prod环境
测试工具：需要实现单测，且覆盖率要达到70%；压测有脚本，需要ssh到压测机，到k6路径下，编写脚本并执行；接口测试暂无；Diff测试暂无。

### CI/CD与发布流程
gitlab-ci.yml详解：
1. 全部采用gitlab-ci来管理CICD流程，可以查看根目录下的gitlab-ci.yml，下面全部以Mart Router举例。
2. 定义流水线阶段：test阶段、build阶段、aws_push_image阶段、aws-login阶段、mirror-image阶段、deploy阶段。
- test阶段：一是进行代码质量检查，使用golangci-lint进行静态代码分析；二是执行Go单元测试，生成JUnit测试报告，计算代码覆盖率。
- build阶段：构建镜像，当有标签时才会出发，会构建Docker镜像并推送到大连的镜像仓库。
- aws镜像推送阶段：当标签是正式版本时触发，比如1.0.1，构建镜像并推向aws的镜像仓库。
- aws login阶段：当标签是正式版本时触发，会获取AWS ECR的临时登陆密码。
- 镜像同步阶段：当标签时正式版本时触发，会从AWS ECR复制镜像到Harbor。
- deploy阶段：这一步会判断标签。当标签不为空时，会往大连beta环境部署；当标签是正式版本时，会往rc、prod-clone、prod环境部署（实际上大连侧不能部署这些环境）

关于gitlab-ci的详细说明与使用，我会再单独梳理一篇blog。

### 生产部署流程
proto配置：需要合并入主干分支，并打出正式包；同时更改之前引用快照包的地方，都改为引用正式包。
Ceylon配置：需要合并入主干分支，由SRE操作；同时需要申请Vault、Unleash等，具体要看Ceylon中配置了啥。
打正式镜像：个人分支合并入主干分支，然后打正式版本tag，并生成正式镜像。
服务部署：由泰国侧更改argoCD配置，并执行sync。

### 生产回滚流程
待确认，当前团队回滚操作很少。

### 稳定性保障流程
监控平台：采用Grafana进行监控，目前有大连beta/rc/prod三个环境，泰国侧beta/beta2是看不到的。
日志平台：采用OpenSearch来查看日志信息，【todo】
故障演练平台：暂时没发现

### 项目管理流程
迭代周期：两周一轮迭代；第一周周四（与泰国侧整体会议），第二周周五（大连团队周会）；没有专门的需求评审会；每周周一、三、五会进行展会，分享进展
工作项管理：暂时没有严格的长篇故事-故事-任务的分层定义，大家一般会根据工作体量创建对应工作项；目前工时必须填写，一天8小时需要严格遵守。


## 从服务角度，优先摸透的关键实现

### 启动与初始化流程
启动类入口：在main.go，它采用pillars框架作为启动器，可以理解为Gin框架，关于启动框架，会单独梳理一篇blog。

依赖加载：在启动过程中，依赖的加载流程如下：
1. 环境变量加载：`github.com/joho/godotenv/autoload`会自动加载.env中的环境变量。他会按照当前工作目录.env、父工作目录.env、环境指定的配置文件.env.{ENV}的顺序查找并加载。
2. 配置加载：会执行config.LoadConfig，这里会初始化DFC配置、OSRM配置、并发控制配置、EP配置、模型服务配置。

依赖注入初始化：使用Wire进行依赖注入，有几个核心服务组件：StoreService（Redis存储服务）、ExpService（实验服务）、DfcService（DFC服务）、EPClient（实验平台客户端）、RankerClient（Ranker客户端）、RedisClient（Redis客户端）

gRPC服务启动：服务注册+拦截器配置。大概流程是这样的：
1. wire.InitializeApplication() 启动
2. 创建各种组件实例（storeService, expService, dfcService, rankerClient）
3. 调用 ProviderMartRouterServer() 创建服务实例。
4. 创建Handlers结构体，包装上面的服务实现（因为Router服务里就一个gRPC服务实现）
5. 创建 ServerCustomizer，并持有Handlers引用。
6. 在Server.go 中的 Configure 方法中，使用handlers.Router来注册拦截器和gRPC服务。

