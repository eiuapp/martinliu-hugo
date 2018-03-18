---
date: 2018-03-17T10:50:57+08:00
title: "基础架构持续集成和应用部署"
subtitle: "在DevOps的场景下如何对基础架构进行持续集成和分层的配置管理，基于统一配置管理工具的应用部署和传统部署有什么不同？"
description: "在DevOps的场景下如何对基础架构进行持续集成和分层的配置管理，基于统一配置管理工具的应用部署和传统部署有什么不同？"
categories: "DevOps"
tags: ["DevOps","CI","IaC"]
keywords: ["DevOps","CI","IaC","CD","Jenkins","Chef"]
bigimg: [{src: "https://res.cloudinary.com/martinliu/image/upload/carauari-brazil.png", desc: "DevOps"}]
---

通常我们谈的更多的是应用的持续集成，而基础架构的持续集成怎么做？先来回顾一下现在几乎所有人都使用的手工交付流程，源码经过编译打包以后，被存在内部的某个文件服务器上。Ops团队的某个组/人被分配到工单，根据工单的需求，它在测试和生产环境中开始准备：

* 用模板手工克隆虚拟机，或者在给虚拟化管理员发任务单后，等待回复
* 使用用户名和密码手工登录服务器，这里有些企业还要等待领导的审批，才能得到密码信封和所需要的访问密码
* 根据最初的工单和自己的经验对操作系统进行配置，往往可能还需和需求方又一次沟通，确认相关参数细节
* 手工的下载应用安装包，凭经验和工单信息部署应用，并做最基本的手工冒烟测试（可能就是看下页面有没有正常显示，或者服务起没起）
* 手工测试这些虚拟机的服务和状态，凭经验觉得OK了以后，回复工单，关闭工单。

以上工作场景，可能是一个Ops人员很常规的一天，或者是几天内的工作，他们在这个过程中也可能会有疑问，也可能对工作结果不确定；但是，日常的工作经验告诉他，差不多了，关闭任务单要紧，还有好多项目催活呢！就这样，配置并不精确的虚拟机被交给了下游的需求方。

以上工作过程的问题如下：

1. 工作周期长，速度慢。实际上工作周期拖延的越久，工作结果质量越差，并不是我们想想中的慢工出细活。
2. 所有步骤纯手工操作，费事费力，出错几率高，机会无法无痛回退。也可能有人说，我们不需要那么快，我们不是互联网公司；可是从精益的角度看，凡是以上工作没有增值，或者说对业务价值的交付为零；你是由于公司在给你发着公司，才错误的感觉，这个工作活动应该有价值。
3. 上游传递来的信息可能不全面，不准确，很有可能造成错误配置，因此会返工。
4. 传递给下游的虚拟机很可能在后续的部署过程中，由于应用需求的变化，而需要下游的人重新配置相同的一些列参数。


> 手工部署付出的时间和代价 = 应的数量 X 应用的版本数量 X 环境的数量

如果某一项任务的重复频率越高，那么对它进行优化，效率优化所产生的汇报也会越明显。从这个角度出发可以设置持续部署流程的改善目标：

* 减少总体人工工作时间和代价
* 提高速度、可靠性和频率
* 能处理应用部署，能处理数据库Schema的更新
* 能够实现自服务，让需要部署的人一键式部署

从以上目标出发，可以看出必须将手工劳动，变为自动化的过程。因此，基础架构即代码 IaC 就会用到。所下图所示：

![delivery-Pipeline](https://res.cloudinary.com/martinliu/image/upload/delivery-Pipeline.png)

上图是持续交付流水线模型，它的几个关键的：

* 代码的变更被Jenkins自动化的构建，打包后存储在Artifactory里，Artifactory里面还可以存储应用包的其它相关元数据，如测试结果，是否可以用于下一步的部署等。
* Jenkins从自动化的搭建所需要的环境，也就是制备虚拟机资源，然后调用Chef完成对虚拟机的配置，Chef完成应用包所需要的各个层次的配置。
* 环境配置完成后，可是使用Chef相关的测试工具对部署环境做验收测试，Chef是具支持测试驱动的相关工具。

## 基础架构持续集成的基础

以上持续交付流水线的基础概念包括：分层的系统管理、基础架构即代码 IaC、配置管理、Chef等。

### 分层的系统管理

这里的系统管理的层次涉及到OS相关的三个层次。下面自下而上地简单描述一下。

1. 制备管理：涉及到虚拟化层，这一层是资源表达层，目前所有主流的虚拟化都支持标准的Rest API，包括VMWare、EC2和Nuanix等。大多数主流配置管理工具都具备用于虚拟机生命周期管理（从生成、到开机、到删除等）的API功能，能按需的获得任何数量、规模、网络和操作系统类型的部署环境。
2. 配置管理：在任何类型的操作系统里自动化的安装和配置软件包，将所有配置参数配置好以后，持续保持这些配置点的状态。对于简单应用，来说按配置参数启动服务即任务完成。
3. 应用编排管理：对于复杂的分布式系统，由于各个自服务之间存在着依赖关系，所有自服务之前需要互通一些配置参数才能实现，应用程序整体的正常运行，配置应用服务器的odbc数据库连接，配置web前端的ldap认证服务器等等。目前微服务所涉及的服务发现和路由，是应用编排必备的配套设施。

### 基础架构即代码IaC

这个概念最早被Chef这类工具提出并实现，它的基本想法就是让Ops人员象开发人员一样，以基础架构代码为工作对象。而不是数十个图形和文字终端界面。使用类似于开发应用程序的方式，开发管理软件基础架构，在这里基础架构的API访问是基础，可惜的是在服务器虚拟化环境中，资源池的API功能几乎没有被用到。

像开发应用代码一样，基础架构的开发和管理也需要遵循相同的原则，例如：

* 一切从源代码开始:并对其进行严格的版本管理，要对对基础架构变更，就需要对相应的代码进行变更变更。从而力求做到服务器的无人登录运维。
* 模块化设计:不同应用底层所使用的基础架构有着大量的相同之处，模块化的设计不仅意味着标准化，也意味着更少的重新代码。我所用过的Terraform、Chef和Puppet这三种工具，都具有高度模块特性。
* 抽象能力：能够使用不同的模块和参数对任何特征的应用环境进行建模和表达，基础架构的代码开发也就是借助这种抽象能力，将所有工具中的抽象概念对象具体化成应用服务的模型。而且编写出来的基础架构代码不仅包含了所有对应用配置描述性的语义，而且还是能够被执行的代码，执行之后，就得到所期望的虚拟机、应用配置和应用服务。
* 可测试性：这是一个我曾经忽略的能力，而在了解之后，我才彻底说服了自己，IaC也是变成语言，IaC程序就是对基础架构编程，而且程序代码本身和它的运行结果都是可以测试的。在执行前对其语义语法测试，在运行以后对其运行结果测试。Chef在这方面表现的尤为突出。


### 配置管理

我可能是最早的一批进行ITIL配置管理实践，CMDB实践的这批人；我以前和甲方客户有着大量的关于配置管理和CMDB的对话，所经历过的项目也非常煎熬。而在DevOps场景下，感觉对此还是驾轻就熟的。

> Process for establishing and maintaining consistency

以上是基于维基百科的定义，它所表达的含义还是值得借鉴的；而如今国内对DevOps的认识，还有很多人是建立在配置管理相关的工具上的。这些配置管理多是基于主机（OS）的管理工具，包括：CFEngine、Puppet、Chef、Salt和Ansible等。它们都具有基础架构即代码的相关原则和特征。都能实现：定义服务器的目标期望状态的能力，在每一次执行周期里，它们都进行状态检查，回报当前状态和目标状态的偏差，在必要的时候也可以自动的执行必要的状态修复变更操作。

就Chef而言，这种配置管理工具，它使用Ruby语言实现了DSL，使用者只需要用Chef代码表达”What“即可，而不需要明白”How“；”What“既是对目标配置状态的描述，使用者只需要将需求转换为Chef代码，然后用Chef客户端工具运行它即可。Chef的代码清晰，表达目标的能力强大。在编码的时候遵循DSL规则，如果有必要的话也可以调用Ruby。

Chef是客户端服务器的架构，一个安装了**Chef-client**程序的节点可以注册到一个**Chef管理服务器**里。

Chef的开发者，在安装了用于和Chef服务器交互的名为**knife**的工具，称之为**工作站**的系统上工作。Chef使用一些列DSL资源（例如：package，service，file，directory等）用于目标节点的配置建模，这些代码可以映射到内部的用于执行代码的**提供者**。

代码实例如下所示，对于Linux操作系统里Apache服务器的描述。


```ruby

package 'httpd' do
  action :install
end

service 'httpd' do
  action [ :enable, :start ]
end

```

代码实例，Linux操作系统里的目录 /a/b/c


```ruby
directory '/a/b/c' do
  owner 'admin'
  group 'admin'
  mode '0755'
  action :create
  recursive true
end
```
以上的代码所保证在系统里这个目录的状态如下：


```sh
$ls ‐ld   /a/b/c
drwxr-­‐xr-­‐x. 5   admin   admin   4096    Feb 14 11:22    /a/b/c
```

Chef其它的重要术语：

* recipe    ：包含了一个或者对个资源描述定义
* cookbook  ：包含了一个或多个配方
* data bag  ：包含了一个或多个配置数据点(data bag item)，是JSON格式，一个菜谱可以包含一个或者多个数据袋
* run list  ：包含了一个或者多个食谱，可将其部署在被管理的节点上
* role      ：一组特定内容的运行清单构成了一个角色
* environments：同我们现在对环境的定义，与之一一对应

## 部署流程设计

将以上手工过程转换为自动化执行的、一键式触发、或者自动触发的过程。有几个关键点。

使用Chef部署自开发的应用程序，包括配置所依赖的操作系统配置和软件，以及自身所需的应用配置。使用Liquibase进行数据库的schema的部署和更新。用Jenkins协调和组织所有工序的执行。使用Jenkins管理部署流程的感觉和用它执行CI是类似。

从简单开始，尽量将一组彼此相关的、版本化的可部署物组织在一起发布，例如一个发布集合可以包含：UI、REST服务器、消息服务和数据库。使用一条命令构建，使用一条命令部署。

### Cookbook设计类型

**Library Cookbook** 库食谱：这种类型的食谱涵盖了通用的、可重用的逻辑。例如所有配置基线，也可以是安全基线。例如：dns、ntp、主机登录提示、用户和组、禁用服务清单等等。开发扩展的自定义chef资源，用来安装自开发应用。

**Application Cookbook** 应用系统食谱：在上面库食谱的基础上，一个应用系统对应一个食谱，每个应用可是是一个配方，配方使用自定义开发的Chef资源。这样就形成了非常轻量的代码库。

**Data Bag**数据袋：包含了各种应用配置，例如：服务端口、JAVA_OPTS等等。一个应用系统食谱一个数据袋，里面包含了该应用在每一套环境里的相关配置点。

上线一个新版本应用意味着配置的部署，大致的流程：编辑Chef代码、推送到Chef管理服务器、节点上运行Chef执行部署动作。Chef服务器的版本始终和版本控制库里的Master主干保持一致，这同样意味着环境配置和Master主干代码保持一致。


在开发Chef资源实例代码如下，这段代码表示了一个Java应用war包的部署。


![carbon](https://res.cloudinary.com/martinliu/image/upload/carbon.png)


基于类似于以上的自定义资源类型，在必要的时候开发Action（chef资源的操作），可能的操作定义有：

* 从Artifactory服务器下载Java、Tomcat和WAR包。
* 在标准的路径安装Java和Tomcat。
* 创建和配置Tomcat容器
* 在特定的容器里安装WAR包
* 在主机上开防火墙端口
* 生成应用属性文件
* 启动Tomcat容器


数据袋的实例代码结构如下：


```ruby
"version":
```

以上是data_bags/my_app/DEV.json的定义，还可以有其它环境的定义data_bags/my_app/TEST.json和data_bags/my_app/PROD.json等。

## 人员角色

**部署人员** 更新数据袋和环境定义文件，发起部署的动作，例如调度chef-client客户端的运行，或者推送新版本的更新。

**技术负责人** 维护应用系统食谱。

**框架开发人员** 维护库食谱，维护框架，持续改进流程。

以上这三种角色，从上到下是从Ops到Dev的过渡。对于传统IT组织的架构，部署人员是Ops团队的，框架开发人员是Dev团队的。不过目前也有Dev团队在其内部招聘运维研发的角色。这三种角色是基础架构即代码的层次结构和人员团队架构的对应，在实际工作中可以灵活应用；一方面覆盖所有技术层次，另外一方面引入所有必要的人员，是团队形成合力。



## 持续构建 -Cookbook build

在开发了各种Cookbook之后，我们就需要对它进行持续测试，因此就需要使用Cookbook的持续构建流程。
![cookbook-build](https://res.cloudinary.com/martinliu/image/upload/cookbook-build.png)

Cookbook的开发人员在Workstations工作站上开发Chef代码，将代码提交到GitHub上的Chef代码仓库，Jenkins的Master服务器会触发CI Job，调用Ruby Slave对Cookbook代码进行集成和测试，然后触发EC2临时实例的创建，将Cookbook在EC2实例中进行测试，使用Artifactory中存储的应用软件包部署应用。如果测试都通过了，就触发Release Job，它将Cookbook代码上传到Chef服务器，供所各种环境中的被管理节点使用。

对于Jenkins构建服务器而言，每一个应用系统对应的Cookbook组/集合的测试和发布都会在同一个构建服务器上发生，一般情况下这个服务器也是这些应用的CI服务器；这个Jenkins服务器也是相关Cookbook的CI作业和发布作业的运行地点。这个服务器上会安装所Ruby相关需要的gem包，应该能访问到与Chef服务器链接所需要的秘钥；应该可以使用到创建EC2测试节点虚拟机的秘钥。

### Cookbook CI Job

Cookbook CI作业的触发条件是在有新Chef代码被合并的时候。它会进静态代码扫码：

* 使用json和gem的相关工具分析JSON的句法
* 使用Tailor做Ruby的句法和风格扫描
* 使用Knife做Chef代码的句法分析
* 使用Foodctritic做Che代码的句法分析和正确性分析

还会包含Chef代码的集成测试，集成测试工具使用Test Kitchen，这个工具有一系列和虚拟化/云环境对接的插件，如 kitchen-ec2插件等。按需临时的创建用于集成测试的虚拟机，测试完毕得到测试结果之后，就删除这个临时虚拟机。

在集成测试的生命周期过程中也可能创建多个EC2实例，这个过程使对应用系统里的所有组件进行仿真的实际部署，单节点或者多节点的全量部署。在每个节点上执行Chef代码，在Chef对应用系统的配置和部署完成之后，对应用进行验证测试，测试包括测试相关的服务端口是否能访问，返回结果是否正常等等，Chef是可以进行测试驱动开发的，因此可以写出较细致的测试代码，从而分析集成测试通过与否。在测试结束了以后（10分钟左右），删除所有测试的虚拟机资源。

应该尽可能的优化集成测试，尽量缩短它的执行时间。可以创建专用的EC2 AMI操作系统镜像，预装所需要的Ruby环境和Chef工具。使用Chef Solo工具执行Chef代码的测试，以免将临时节点也添加到了Chef服务器，同时也消除了Chef的客户端和服务器架构的消耗，这个场景里没有使用Chef服务器的必要。使用一个名为CHEFDEV的伪环境来测试代码，而JSON文件里定义的真实环境则被保留用于正式生产环境。在创建EC2虚拟机的时候，给它们打上特定的标签，从而保持一定的可追踪性和环境的可维护性。

### Cookbook Release Job

这个作业的运行内容和CI Job基本一致的，而它是靠人为手工触发的，从Chef角度看，可以说：以上本文的所有工作属于Chef风格的基础架构即代码程序的持续交付。本作业将测试成功的代码在GitHub里打上标签，并且上传Cookbook的新版本到正式的Chef生产服务器上。



## 应用部署流程

Cookbook的开发和集成完毕了以后，它的结果产物是一些列新版本的Cookbook代码，它们最终上传到Chef服务器。支持生产环境应用部署的Chef服务器与各种环境保持连接，包括测试、预发布和生产等等。

![app-deploy-process](https://res.cloudinary.com/martinliu/image/upload/app-deploy-process.png)


在发布过程中所使用到的制品是从Artifactory中拉取的。下面简单说下这个架构中的关键点。

**Jenkins 部署服务器** 这是专门用于各种部署工作的Jenkins Master服务器。它的Slave应该满足这些需求：安装了所需要Ruby环境和gem包。安装了Chef工具，并且具有能更新Chef服务器的秘钥。具有能访问各种虚拟化环境中节点的SSH秘钥。

**部署作业的类型** 可以对于每一个应用组（一套应用系统）设置两个部署作业：开发环境中的开发人员所使用的DEV部署作业；另外是运维人员所使用的Non-Dev部署作业。在实践过程中也能发展出其他类型。

部署人员的工作流程：

1. 变更Chef相关代码和配置，包括：编辑应用的data bag配置数据点，有必要的话编辑环境文件，合并代码。
2. 然后在Jenkins部署服务器上执行作业。

通过以上的流程和工具，开发、测试和运维的相关人员，如果需要部署应用了，就可以用一键式的、自助式的部署模式，将任何应用应用系统通过一键式的方式自动化的部署的各种应用环境中。
![push-button-deploy](https://res.cloudinary.com/martinliu/image/upload/push-button-deploy.png)

这样我们将大量各种角色人员都从事的、没有附加值的应用部署工作，彻底的消灭掉了，节省的时间可以用来做更多有意义的工作，Dev人员有更多时间编码、测试人员不需要等待时间、运维人员也降低的工作压力。

## 总结

还有一些值得参考的原则：

* 尽可能的标准化：包括技术、设计和流程等方面；要能够支持环境的规模化扩展，意外是可以的，但是要尽量避免。
* 所有工具最好有API：避免在某个工具链上任何环节的脱节。
* 使用多种形式的沟通路径：全员大会的主题分享、每个团队的启动会议、与开发人员的随时沟通，使用文档进行知识传播和沟通。
* 保持乐观，尽量发现和找到那些志同道合的早期响应者，让他们和你站在一条战线上。



本文参考的演讲和视频包括：[https://www.youtube.com/watch?v=PQ6KTRgAeMU
](https://www.youtube.com/watch?v=PQ6KTRgAeMU
)



-------

