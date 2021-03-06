---
title: "蚂蚁金服 Service Mesh 技术风险思考和实践"
author: "嘉祁"
description: " 本文根据蚂蚁金服中间件 SRE 技术专家黄家琦（嘉祁）于 Service Mesh Meetup#9 杭州站上的分享整理。"
categories: "Service mesh"
tags: ["Service mesh","Service Mesh Meetup"]
date: 2020-01-20T18:00:00+08:00
cover: "https://cdn.nlark.com/yuque/0/2020/jpeg/226702/1579512982408-3be5895a-662e-4ced-889f-8555a1ba312e.jpeg"
---

![Service Mesh Meetup#9 现场图](https://cdn.nlark.com/yuque/0/2020/jpeg/226702/1579076089247-e03e880a-2a78-44ba-9b0f-5ca3c92cf5a4.jpeg)

本文根据蚂蚁金服[中间件 SRE](https://job.alibaba.com/zhaopin/position_detail.htm?&positionId=80151)技术专家黄家琦（嘉祁）于 Service Mesh Meetup#9 杭州站上的分享整理。

## 背景

Service Mesh 在软件形态上，是将中间件的能力从框架中剥离成独立软件。而在具体部署上，保守的做法是以独立进程的方式与业务进程共同存在于业务容器内。蚂蚁金服的做法是从开始就选择了拥抱云原生。

大规模落地的过程中，我们在看到 Service Mesh 带来的巨大红利的同时，也面临过很多的挑战。为此，蚂蚁金服配备了“豪华”的技术团队阵容，除了熟悉的 SOFA 中间件团队，还有安全、无线网关以及专门配备了专属的 SRE 团队，使得 MOSN 能力更加全面更加可靠。

今天我们详细聊聊技术风险这个方面。对于一个新的中间件技术，在落地过程中总会面临稳定性、运维模式变化等等很多的问题与挑战，需要建设相应的技术风险能力，确保整个落地过程以及长期运行的稳定和可靠。

本次分享主要从以下三个方面展开：

1. 落地过程中对于部署模式和资源分配的思考；
1. 变更包括接入、升级的支持、问题与思考；
1. 整个生产生命周期中稳定性面临的挑战与技术风险保障；

### Sidecar 部署模式

业务容器内独立进程的好处在于与传统的部署模式兼容，易于快速上线；但独立进程强侵入业务容器，对于镜像化的容器更难于管理。而云原生化，则可以将 Service Mesh 本身的运维与业务容器解耦开来，实现中间件运维能力的下沉。在业务镜像内，仅仅保留长期稳定的 Service Mesh 相关 JVM 参数，从而仅通过少量环境变量完成与 Service Mesh 的联结。同时考虑到面向容器的运维模式的演进，接入 Service Mesh 还同时要求业务完成镜像化，为进一步的云原生演进打下基础。

|  | 优 | 劣 |
| --- | --- | --- |
| 独立进程 | 兼容传统的部署模式；改造成本低；快速上线 | 侵入业务容器； 镜像化难于运维 |
| Sidecar | 面向终态；运维解耦 |  依赖 K8s 基础设施；运维环境改造成本高；应用需要镜像化改造 |

在接入 Service Mesh 之后，一个典型的 POD 结构可能包含多个 Sidecar：

- MOSN：RPC Mesh, MSG Mesh, ...（扩展中）；
- 其它 Sidecar；

MOSN：[https://github.com/mosn/mosn](https://github.com/mosn/mosn)

![一个典型的 POD](https://cdn.nlark.com/yuque/0/2020/png/226702/1579076089223-be1dc853-7d5c-490a-8374-c7d2d90bd8b3.png)

这些 Sidecar 容器与业务容器共享相同的网络 Namespace，使得业务进程可以以本地端口访问 Service Mesh 提供的服务，保证了与保守做法一致的体验。

#### 基础设施云原生支撑

我们也在基础设施层面同步推进了面向云原生的改造，以支撑 Service Mesh 的落地。

#### 运维平台模型支撑

首先是运维平台需要能够理解 Sidecar，支撑 Sidecar 模型的新增元数据，包括基于 POD 的 Sidecar 的多种标签，以及 Sidecar 基线配置的支持。

#### 业务全面镜像化

其次我们在蚂蚁金服内部推进了全面的镜像化，我们完成了内部核心应用的全量容器的镜像化改造。改造点包括：

- 基础镜像层面增加对于 Service Mesh 的环境变量支撑；
- 应用 Dockerfile 对于 Service Mesh 的适配；
- 推进解决了存量前后端分离管理的静态文件的镜像化改造；
- 推进了大量使用前端区块分发的应用进行了推改拉的改造；
- 大批量的 VM 模式的容器升级与替换；

#### 容器 POD 化

除了业务镜像层面的改造，Sidecar 模式还需要业务容器全部跑在 POD 上，来适应多容器共享网络。由于直接升级的开发和试错成本很高，我们最终选择将接入 Service Mesh 的 数百个应用的数万个非 K8s 容器，通过大规模扩缩容的方式，全部更换成了 K8s PODs。

经过这两轮改造，我们在基础设施层面同步完成了面向云原生的改造。

### 资源的演进

Sidecar 模式的带来一个重要的问题，如何分配资源。

#### 理想比例的假设独占资源模型

最初的资源设计基于内存无法超卖的现实。我们做了一个假设：

- MOSN 的基本资源占用与业务选择的规格同比例这一假设。

CPU 和 Memory 申请与业务容器相应比例的额外资源。这一比例最后设定在了 CPU 1/4，Memory 1/16。

此时一个典型 Pod 的资源分配如下图示：

![一个典型 Pod 的资源分配](https://cdn.nlark.com/yuque/0/2020/png/226702/1579076089341-38cbe7b1-7da4-4f50-841e-035ad148bd8f.png)

#### 独占资源模型的问题

这一方式带来了两个问题：

1. 蚂蚁金服已经实现了业务资源的 Quota 管控，但 Sidecar 并不在业务容器内，Service Mesh 容器成为了一个资源泄漏点；
1. 业务很多样，部分高流量应用的 Service Mesh 容器出现了严重的内存不足和 OOM 情况；

#### 共享资源模型

讨论之后，我们追加了一个假设：

- Service Mesh 容器占用的资源实质是在接入 Service Mesh 之前业务已使用的资源。接入 Service Mesh 的过程，同时也是一次资源置换。

基于这个假设，推进了调度层面支持 POD 内的资源超卖，新的资源分配方案如下图，Service Mesh 容器的 CPU、MEM 都从 POD 中超卖出来，业务容器内仍然可以看到全部的资源。

![共享资源模型](https://cdn.nlark.com/yuque/0/2020/png/226702/1579076089260-70fd75ce-a543-47c1-ad27-fbcd48c60228.png)

考虑到内存超卖也引入了 POD OOM 的风险，因此对于 Sidecar 容器还调整了 OOM Score，保证在内存不足时，Service Mesh 进程能够发挥启动比 Java 业务进程更快的优势，降低影响。

新的分配方案解决了同时解决了以上两个问题，并且平稳支持了蚂蚁金服的双十一大促。

## 变更-Service Mesh 是如何在蚂蚁金服内部做变更的

Service Mesh 的变更包括了接入与升级，所有变更底层都是由 Operator 组件来接受上层写入到 POD annotation 上的标识，对相应 POD Spec 进行修改来完成，这是典型的云原生的方式。

### 接入-从无至有

标准的云原生的接入方式，是在创建时通过 sidecar-operator webhook 注入一个 Sidecar 容器。

![标准云原生的接入方式](https://cdn.nlark.com/yuque/0/2020/png/226702/1579076089309-bad0a13c-a6c1-4b45-a7cb-9c7e526d3329.png)

这个方式的固有缺陷在于：

- 滚动替换过程需要 Buffer 资源；
- 过程缓慢；
- 回滚慢，应急时间长；

#### 原地接入

原地接入是为了支撑大规模的快速接入与回滚，通过在存量的 POD 上操作修改 POD Spec。

![原地接入方式](https://cdn.nlark.com/yuque/0/2020/png/226702/1579076089276-fa4bba8b-da5b-4d6a-9837-b4dcbc177ad7.png)

尽管看起来不太云原生，但期望能绕过以上几个痛点，从而可以：

- 不需要重新分配资源；
- 可原地回滚；

### 升级

Service Mesh 是深度参与业务流量的，最初的 Sidecar 的升级方式也需要业务伴随重启。

![升级模型](https://cdn.nlark.com/yuque/0/2020/png/226702/1579076089278-98c7478a-4289-4a28-95af-ba7768fa8f81.png)

#### 带流量的平滑升级

![带流量的平滑升级](https://cdn.nlark.com/yuque/0/2020/png/226702/1579076089281-f75a1a70-40f3-44ee-9178-dc939303d5c6.png)

为了规避 Java 应用重启带来的重新预热等问题，MOSN 提供了更为灵活的平滑升级机制：由 Operator 控制启动第二个 MOSN Sidecar，完成连接迁移，再退出旧的 Sidecar。整个过程业务可以做到流量不中断，几近无感。

### 变更能力的取舍

虽然提供了原地接入与平滑升级，但从实践的效果来看，对存量 POD spec 的修改带来的问题，比收益要多得多。

| **创建时注入**/**普通升级** | **原地注入**/**平滑升级** |
| --- | --- |
| 不改变现存 POD spec | 改变现存 POD |
| 资源预分配，不影响调度； 逻辑简单，成功率高；与原有业务升级逻辑一致；变更简单完全可逆 |  资源分配需要重新计算，细节多易出错；逻辑复杂成功率与效率不佳；强依赖 operator，不易观测；逆变更需要反向重新计算，难度高 |
| 接入需要替换资源；升级需要中断业务 | 接入不需要额外资源；升级时业务无感 |

最终我们在生产环境放弃了相应的能力，而依赖于更灵活高效的调度层与变更流程来解决接入与升级中的问题。

### 变更流程支撑

我们建设了完整的 Service Mesh 变更流程来支撑全站规模的接入与升级的变更动作。整体变更按照环境-IDC 多维度分组，结合全程变更管控，确保变更过程平稳、可控。

![变更流程支撑](https://cdn.nlark.com/yuque/0/2020/png/226702/1579076089316-0bbe0999-45b1-47b2-8c6b-b1f964eaae34.png)

#### 全程变更管控

全程变更管控提供了不限于以下几方面的能力：

- 强制灰度/分批流程；
- 前后置业务自动巡检；
- 变更完整性校验；

其中业务自动巡检，提供了在每个分组变更前自动处理参数检查，冲突规避，容量检查等，和变更后基于算法与规则匹配的业务检查的能力，变更完整度检查等，保障规模变更稳定可靠。

![全程变更管控](https://cdn.nlark.com/yuque/0/2020/png/226702/1579076089301-f84f6a1e-36a7-4806-8806-0679bab3445d.png)

### 控制面的变更

控制面的实际上面临所有使用 Deployment、daemonset 等云原生部署方式相同的问题，即无法回答：

- 如何控制灰度范围；
- 当前的进度如何；

总体来说，我们在向类似 CafeDeployment 的解决方案靠近来解决这类通用问题。

## 技术风险建设-我们如何保障 Service Mesh 的生产稳定

### 监控与定位能力 

Service Mesh 为监控与定位提供的数据包括：

- Metrics：实时的业务统计数据；
- 错误码：Service Mesh 的运行时的分类错误日志；
- 业务监控：tracing与业务错误日志；

基于这些数据，我们期望回答三个问题：

- 是否是 Service Mesh 的问题；
- 哪个 Service Mesh 组件的问题；
- 如何处理出现的问题；

从这三个问题出发，我们的监控平台为 Service Mesh 提供了全局的 Service Mesh 大盘和业务视图的 Service Mesh 视图，在其中提供错误码聚合与 Metrics 数据，展示各指标的 Top 视角。

基于这样的数据聚合能力，在最上层提供基于 Tracing 定位与监控数据的 chat ops 定位能力，最终来回答这三个问题。

基于全局大盘的数据，我们得以在落地过程中发现并定位了某些业务场景下出现的连接问题。

#### 仍面临的问题

虽然已经有了很多的数据聚合，但在定位上，还面临一些挑战：

- 模块启用在不同应用存在差异，错误码聚合效果有偏差；
- 报警阈值对应于不同应用存在差异；
- 集群规模极大带来了极大的噪声；

这些问题说明单一维度的指标与低阶的阈值已经无法适应我们的业务与问题发现的要求了，定位能力需要向多维度，高阶标准方向进化。

### 预案与应急能力

Service Mesh 自身具备按需关闭部分功能的能力，包括但不限于：

- 日志分级降级；
- Tracelog 日志分级降级；
- 控制面(Pilot)依赖降级；
- 软负载均衡长轮询降级；

对于 Service Mesh 依赖的服务，为了防止潜在的抖动风险，也增加了相应的预案：

- 软负载均衡列表停止变更；
- 服务注册中心高峰期关闭推送；

Service Mesh 是非常基础的组件，目前的应急手段主要是重启：

- Sidecar 单独重启；
- POD 重启；

#### 自愈

基于以上的预案与应急手段，结合现存的监控与定位能力，我们在 Service Mesh 实现了多种场景下的自愈，极大加快了已知场景的处理响应。

![自愈](https://cdn.nlark.com/yuque/0/2020/png/226702/1579076089352-1df83feb-232b-447c-bfb0-2269770250f7.png)

### 攻击能力

攻击能力主要指两个层面：

- 对Service Mesh本身的攻击演练能力
- 在Service Mesh中提供细粒度的业务攻击服务

一方面是常态化验证 Service Mesh 自身的稳定性，另一方面则对于全站层面的技术风险能力的深入提供基础能力。

## 未来

Service Mesh 在快速落地的过程中，遇到并解决了一系列的问题，但同时也要看到还有更多的问题还亟待解决。做为下一代云原生化中间件的核心组件之一，Service Mesh 的技术风险能力还需要持续的建议与完善。未来需要在下面这些方面持续建设：

- 高度自愈能力；
- 更精准的变更防控能力与规模化高度无人值守变更能力；
- 精确智能化定位能力；
- 更高效，低噪声的高精度监控；

欢迎有志于中间件 Service Mesh 化与云原生稳定性的同学[加入我们](https://job.alibaba.com/zhaopin/position_detail.htm?&positionId=80151)，共同建设 Service Mesh 的未来。

## 关于 Service Mesh Meetup

![Service Mesh Meetup 合集](https://cdn.nlark.com/yuque/0/2020/png/226702/1579142930594-646e45cb-d69b-488a-b4c4-37e512f3c7bc.png)

本期是 Service Mesh Meetup 第9期杭州站，2018年4月杭州的第一期 Service Mesh Meetup 开启了 ServiceMesher 社区的全国技术交流之旅，至今已经快2年了，我们走过了北京、上海、深圳、广州、成都、杭州等六座城市，累计举办9期线下活动。

同时，感谢滴滴、华为、网易、酷家乐、中兴通讯、美团、唯品会、七牛、谐云科技、慧择网、虎牙、才云、京东、新浪微博、联邦车网、JEX、Apache SkyWalking、知群等公司的支持，参与技术分享、共同组织活动。

2020年，社区希望走到更多的城市，与更多对 Service Mesh 感兴趣的技术同学们交流、探讨想法。

We are ServiceMesher! May the Mesh be with you!

ServiceMesher：[https://www.servicemesher.com/](https://www.servicemesher.com/)
