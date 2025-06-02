---
title: AI Agent需要什么样的数据库？
description: Databricks 在 2025 年 5 月 14 日正式宣布以 10 亿美元收购开源数据库初创公司 Neon，从中可看出亚秒级冷启动、Scale to Zero、Branching等特性，正成为 AI Agent 时代数据库所必备的能力之一。

date: 2025-06-02
categories:
  - AI
tags:
  - AI
  - 大数据
author: 元闰子
---

## 前言

Databricks 在 2025 年 5 月 14 日正式宣布以 10 亿美元收购开源数据库初创公司 Neon<sup>[1\][2]</sup>，它的产品是一款兼容 PostgreSQL 的 Serverless 数据库。Neon 的 slogan 是 "**Ship faster with Postgres**"，其最初的目的是为应用开发者提供一个低成本、开发高效的数据库底座。

回顾 Databricks 近两年的收购策略，Databricks 正围绕其 Data + AI 战略迅速补强 AI 短板，所收购的产品已几乎可组成一条完整的 AI 工作流水线。

![](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2025-06-02-082957.png)

从 Databricks Data Intelligence Platform 架构图<sup>[4]</sup>来看，Databricks 几乎已具备 `Data -> AI Model -> AI Application` 的 AI 全栈能力，但 `AI Application` 所依赖的数据库仍然选择第三方的 Amazon RDS 等。

![](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2025-06-02-083401.png)

所以，**对 Databricks 来说，收购一款高度适配 AI 应用的数据库变得势在必行**。

在各类 AI 应用中，**AI Agent 无疑是最活跃的发展主线**。2025 年被誉为是 AI Agent 元年，相关的框架、协议、应用都得到了快速的发展。据 Gartner预测，到2028年，至少 15% 的日常工作决策将通过 AI Agent 自主做出，33% 的企业软件应用程序也将包含 AI Agent。

Databricks 最终选择 Neon，**无疑是看中了 Neon 数据库的高度亲和 AI Agent 业务特征的关键能力**。

本文将从 AI Agent 业务特征讲起，探讨 AI Agent 时代需要什么样的数据库，以及 Databricks 选择 Neon 的原因。

## AI Agent 业务特征

AI 时代最受益的人群可能是是开发者，AI 辅助编程的出现，使得开发变得更加容易。当前主要有两类产品：

- **Agentic IDE**：更多的是作为开发工具，辅助以自然语言生成 App 的能力。典型的产品是 Cursor，以及字节的 Trae。
- **Text-to-Web App Generator**：通常提供从 App 生成到上线的全流程托管能力。典型的产品是 Bolt 和 Replit。

a16z 数据显示<sup>[4]</sup>，2024 年以来，这两类产品的月活量急剧增加。Bolt 更是宣称在 2025 年前两个月已实现 2000 万美元年化收入，以及 200 万注册用户。

![](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2025-06-02-111506.png)

AI 辅助编程背后的关键技术就是 AI Agent。以 Text-to-Web App Generator 为例，它的主要业务流程如下：

![](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2025-06-02-110151.png)

用户只需通过自然语言命令 AI Agent 生成一款 App，后面的流程全程交给 AI Agent 规划和执行即可，用户只需不断点击确认或否定它的策略即可。

在该业务流程中，数据库扮演者核心的角色。一方面，数据库几乎是 Web App 不可或缺的组件；另一方面 AI Agent 也需要它来存储记忆上下文，比如用户的历史对话、中间结果、计划等。

![](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2025-06-02-111130.png)

AI Agent 对数据库的使用有着如下的几个特征：

- **即时创建**。在收到用户的命令后，依据自身规划即时创建数据库、表结构。
- **大量小实例**。从前文分析可知，AI 辅助编程的用户数量暴涨，带动了通过 AI Agent 创建数据库实例的数量上升，而且这类实例通常是小实例。Neon 数据<sup>[1]</sup>显示，2025 年以来，由 AI Agent 创建的实例数量已经是开发者创建的 4 倍之多。
  ![](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2025-06-02-111412.png)
- **活跃时长不稳定**。一方面，AI 辅助编程的用户通常集中在白天使用；另一方面，开发调试阶段，数据库活跃时长通常很低，即使 App 发布后，新应用的流量也会是低且不稳定。
- **频繁调试**。AI 的不确定性较大，很可能首次创建的 App 并不符合用户预期，需要频繁进行策略更改、调试。调试过程大概率会涉及数据回滚。

根据这些负载特征，我们很容易得出 AI Agent 对数据库的能力要求：

| 负载特征                  | 能力要求                                   |
| ------------------------- | ------------------------------------------ |
| 即时创建                  | **极短的冷启动时长**，提供极佳的用户体验。 |
| 大量小实例/活跃时长不稳定 | **自动弹性伸缩**，降低平台成本。           |
| 频繁调试                  | **PITR**，提供任意时间点数据恢复能力。     |

看到这里，我们很容易联想到**云原生 Serverless 数据库**，快速冷启动、自动弹性伸缩、多租户隔离都是 Serverless 的基本能力。

Neon 就是一款兼容 PostgreSQL 的 Serverless 数据库，相比其他同类产品， Neon 在某些方面做到了更加极致，这也是它能打动 Databricks 的根本原因。

下面，我们将介绍 Neon 是如何匹配这些负载特征的，以及它与其他 Serverless 数据库相比的不同之处。

## 存算分离架构

**存算分离架构是云原生 Serverless 数据库能够实现快速冷启动、自动弹性伸缩的基础**。

在存算一体的架构下，传统数据库引擎的计算和存储都绑定在一个节点内，这会导致如下几个问题：

- **启动慢**。节点是有状态的，启动时需要对数据进行初始化、加载等流程。
- **扩容慢**。传统数据库通常通过主从或分布式架构应对高并发场景，扩容出新实例后，还需完成数据同步才能对外提供服务。
- **成本高**。单节点磁盘容量有限，存储扩容意味着 CPU 也要跟着扩容，成本陡增。

![](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2025-06-02-113326.png)

把计算和存储分离，是解决上述几个问题的有效方法。

存算分离后，存储层通常采用低成本的对象存储，无限扩容，不影响计算节点，且所有计算节点共享同一份数据，成本陡降；

计算节点无状态，启动时省去各种数据初始化流程，只需与存储层建立连接，启动时长变短；且新扩容实例不再需要进行数据同步，扩容时长也变短了。

![](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2025-06-02-114316.png)

Neon 采用了典型的存算分离架构，存储层除了远端对象存储外，还有负责 WAL 文件持久化的 **Safekeeper**，以及负责将 WAL 预写日志文件转化成 page 供计算层查询的 **Pageserver**。

![](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2025-06-02-120419.png)

## 极速冷启动

存算分离架构让秒级的计算实例冷启动成为可能，而 Neon 不满足于此。

Neon 计算实例的冷启动过程大致可分层实例创建、实例配置、网络配置：

![](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2025-06-02-122415.png)

为了更优的冷启动速度，Neon 做了如下几个优化：

**（一）减少不必要的实例重配置**

一般来说，数据库配置很少变更。因此，处理首次冷启动，实例配置并非必选项。只需持续监听配置变化情况，在变化时再在冷启动时实施实例配置操作。

![](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2025-06-02-123124.png)

**（二）Compute Pool**

提前创建好一批“空”的计算实例，待用户请求到来时，优先从 Compute Pool 中拉取，若配置不符合规格，则新创建。

![](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2025-06-02-123617.png)

**（三）IP 缓存**

提前创建好一批随时可用的 IP 地址，避免地址创建和内部 DNS 路由消耗。

![](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2025-06-02-123958.png)

**（四）关键路径代码优化**

针对冷启动关键路径代码进行深度优化。

通过这些优化，Neon 将冷**启动时长从 3~6s 缩短至 500ms**<sup>[8]</sup>！

## 极致自动扩缩容

说起云原生，我们通常会想到 K8S + 容器 + 水平扩缩容，但 Neon 选择了 K8S + 虚机 + 垂直扩缩容的方案。

![](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2025-06-02-131218.png)

原因主要有如下几个：

**（一）虚机更强大的隔离**

相比容器，虚机提供了更强大的隔离能力。

虚机提供的是 OS 级的隔离，每个虚机都运行一个独立于宿主机的 GuestOS；

而容器提供的是进程级的隔离，共享宿主机内核，通过 namespace 和 cgroup 实现。

原生 K8S 并不支持对虚机的管理，所以 Neon 通过 NeonVM 来实现基于 QEUM + KVM 的虚拟化方案。

**（二）垂直扩容无需冷启动**

在K8S 中，水平扩容通过新创建 Pod 实现，这也意味着需要冷启动一次。

而垂直扩容直接在原来的 Pod 上动态扩容 CPU 和内存资源，免去冷启动流程。

在 Neon 中，扩缩容以 CU（Compute Unit，计算单元）为单位，每个 CU 固定为 1 vCPU 和 4 G 内存。

当然，垂直扩容也有它的问题，当节点资源不足时，扩容就会失败，这时就需要将计算实例迁移到其他节点上。

**（三）虚机的快速热迁移**

原生 K8S + 容器 的迁移主要通过在新节点上重新创建 Pod 实现，涉及冷启动流程。

而 Neon 基于虚机热迁移，能实现无缝实时迁移，业务中断时长可以控制在 100ms 以内。

在自动扩缩容上，Neon 也做到了极致，支持 Scale to Zero。当计算实例在一段时间，比如 5 分钟后，实例将自动缩容至零。

**极速的冷启动使得 Neon 有底气这么做，从而为用户提供极致的低成本**！

## 极速 PITR

**PITR**（Point-in-Time Recovery），时间点恢复指的是**数据库可以恢复至任意特定点的历史版本数据**，通常基于快照 + WAL 回放的方案实现。

![](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2025-06-02-134208.png)

传统数据库，包括大部分云原生数据库，都会将**备份恢复**视为一项昂贵、偶尔使用的操作。它们通常会将备份数据与当前数据区别对待，分开存储。

基于 ROW（Redirect-On-Write）技术可以做到秒级备份，但恢复时长则需要分钟级，甚至更久。

以下为阿里云 PolarDB MySQL 版本的恢复流程，主要包含三个流程（以 2 核 8 GB 的计算资源为例）：

1. 创建一个临时节点，并将快照数据恢复至该节点，耗时约 3~10 分钟。
2. 回放 WAL 日志增量数据，恢复速度约 1.5 GB / 分钟。
3. 将数据恢复至原集群，恢复速度约 2.16 GB / 分钟。

![](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2025-06-02-135825.png)

*分钟级的恢复速度显然无法满足 AI Agent 频繁调试的业务特征*。

提起快速数据回滚，我们可能首先会想到 Git，它基于分支管理代码，支持恢复至任意分支的任意的 Commit 点，速度极快。

Neon 正是以此为灵感，创新性地实现了 **Branching** 和 **Time Travel** 特性，**像 Git 管理分支一样管理数据**。

![](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2025-06-02-140239.png)

Neon 采用 **Non-overwriting Storage** 架构，将历史数据库与当前数据存放在一起，一视同仁，这使得**历史数据不再需要先恢复后访问，做到了实时历史数据访问**。

传统关系型数据库几乎都是基于 Overwriting Storage 架构，处理 Update 请求时，直接将源数据更新为新数据。采用 Non-overwriting Storage 架构大多是 Key-Value 数据库，比如 LevelDB、RocksDB 等，它们通常基于 LSM-Tree，不断追加新数据。

Neon 采用了类似 LSM-Tree 的 KV 存储架构，数据按层组织：

- **Image layer** 包含某个 LSN 下，某个 Key Range 的所有 page 数据快照。它由 Neon 后台自动创建，用于加速历史数据访问。
- **Delta layer** 包含某个 LSN Range 和 Key Range 下的 WAL Record，只存储带来数据变更的部分。

> LSN（Log Sequence Number）是 WAL 的唯一标识，递增，可当时间戳来用。

Neon 的 PITR 实现流程：找到离恢复点最近的 Image layer，回放此 Image layer 之上的 Delta layer 的 WAL Record。其中的关键，就是*如何快速找到对应的 Image layer 和 Delta layer*。

Neon 查询数据的接口为 `GetPage(Key, LSN)`，把 LSN 作为纵坐标，Key 作为横坐标，在查询 (Key, LSN) 开始向下划条线，首先遇到的 Image layer 及其之上的 Delta layer 即为要找的 layer。

![](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2025-06-02-142118.png)

这不是个简单的问题。最简单的方法是按照追加数组的形式组织 WAL，查询时间复杂度是 `O(N)`。但在大数据量场景下，查询性能将无法被接受。

Neon 的解决方案是使用 [RPDS (Rust Persistent Data Structures)](https://github.com/orium/rpds) 数据结构来构建 WAL 索引，简单来讲，就是把当前数据和历史数据统一按照树的形式组织，基于 Copy-on-Write 实现数据更新。如下为简单的原理示意图：

![](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2025-06-02-143441.png)

对比传统按追加数组形式组织的 WAL Record，RPDS 实现了随机读时间复杂度从 O(N) 降至 O(logN)。

当然，**Non-overwriting Storage 架构的代价是消耗更多的存储空间，特别是在频繁更新的工作负载**。

所以，它必须要有 **Compaction** 和 **Garbage Collecting** 机制，避免数据无止境增长。另外，还能将冷数据存储到低成本的对象存储上，进一步提升了该方案可行性。

基于这个方案，**Neon 实现了秒级的 PITR，再加上高易用的类 git 命令，使得 Branching 和 Time Travel 成为它的 killer feature之一**。

## 最后

Neon 在 2021 年开源，最初主要**面向的用户是开发者，面向的场景是大量小实例**。它的很多特性都是以提供极致的开发体验为目的，比如 Scale to Zero 为开发者提供极低的成本、Branching 使得开发者能够基于生产数据创建副本调试而不影响生产环境。

从某种程度上看，AI Agent 就是一个“智能开发者”，这使得 Neon 非常匹配 AI Agent 的业务特征，这也是他被 Databricks 高价收购的重要原因。

类似地，阿里云 AnalyticDB PostgreSQL 版基于 Neon 架构推出了 Serverless 版本，也提供了 Scale to Zero、Branching 等特性。

总结起来，***亚秒级冷启动、Scale to Zero、Branching等特性，正成为 AI Agent 时代数据库所必备的能力之一***。

### 文章配图

可以在 [用Keynote画出手绘风格的配图](https://mp.weixin.qq.com/s/-sYW-oa6KzTR9LNdMWCSnQ) 中找到文章的绘图方法。

> #### 参考
>
> [1] [Databricks and Neon](https://www.databricks.com/blog/databricks-neon), Databricks
>
> [2] [Neon and Databricks](https://neon.tech/blog/neon-and-databricks), Neon
>
> [3] [Databricks architecture overview](https://docs.databricks.com/aws/en/lakehouse-architecture/reference), Databricks
>
> [4] [The Top 100 Gen AI Consumer Apps - 4th Edition](https://a16z.com/100-gen-ai-apps-4/), a16z
>
> [5] [From Prompt to Product: The Rise of AI-Powered Web App Builders](https://a16z.com/ai-web-app-builders/), a16z
>
> [6] [Neon - Serverless PostgreSQL - ASDS Chapter 3](https://jack-vanlightly.com/analyses/2023/11/15/neon-serverless-postgresql-asds-chapter-3), Jack Vanlightly
>
> [7] [Architecture decisions in Neon](https://neon.com/blog/architecture-decisions-in-neon), Neon
>
> [8] [Cold starts just got hot](https://neon.com/blog/cold-starts-just-got-hot), Neon
>
> [9] [1 Year of Autoscaling Postgres: How it’s going, and what’s next](https://neon.com/blog/1-year-of-autoscaling-postgres-at-neon), Neon
>
> [10] [Deep dive into Neon storage engine](https://neon.com/blog/get-page-at-lsn), Neon
>
> [11] [A Deep Dive Into Neon’s Instant PITR](https://neon.com/blog/pitr-deep-dive), Neon
>
> [12] [云原生数据库 PolarDB MySQL 版库表恢复整体流程和预估时间](https://help.aliyun.com/zh/polardb/polardb-for-mysql/overall-process-and-estimated-time?scm=20140722.S_help@@文档@@215405._.ID_help@@文档@@215405-RL_40TB-LOC_doc~UND~ab-OR_ser-PAR1_212a5d4017488310693608432deb05-V_4-RE_new5-P0_1-P1_0&spm=a2c4g.11186623.help-search.i1), PolarDB
>
> [13] [4年10亿美金，Neon用Serverless PG证明：AI需要的不是“大”，而是“隐形”](https://mp.weixin.qq.com/s/qFdrWJ6N1reTBIgdQ2rhSw), 阿里云AnalyticDB
>
> 更多文章请关注微信公众号：**元闰子的邀请**

