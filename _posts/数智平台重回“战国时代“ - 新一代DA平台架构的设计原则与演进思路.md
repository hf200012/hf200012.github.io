---
layout: post
title: "[云器Lakehouse系列] 关涛：数智平台重回“战国时代“ - 新一代DA平台架构的设计原则与演进思路（上）"
date: 2024-11-07 
description: "湖仓一体"
tag: 云器Lakehouse
---
# 关涛：数智平台重回“战国时代“ - 新一代DA平台架构的设计原则与演进思路（上）
## 数据平台的三次革命，以及背后的驱动力

数据平台的发展历程可以分为三个阶段：20世纪70年代，关系型模型和SQL语言的出现推动了数据库技术的发展，主要处理小规模的结构化数据；2000年，Google为了满足搜索业务需求，奠基了大数据和分布式系统技术，推动了大数据平台的发展；2022-2023年，GPU规模扩大和数据量增加，使得大模型具备了涌现能力和智能。展望未来，随着智能汽车等设备的普及，机器数据将成为主流，其数据量可能是人类数据的100倍，这将进一步推动数据平台的发展。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/YKKw3T8CyLygK8T47qkmcJUEM1252L2b1ChmuBndOMe0JKiaXnyJLVolLH27l7yFwa00997cpKjibCdbdMjVZKFQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 三次革命: 70年代的数据库技术兴起、2000年左右的大数据兴起和现在的大模型时代。
2. 技术驱动和规模驱动创新推动
3. 未来，机器产生的数据将成为主流，推动数据平台的进一步发展。

"当规模扩大100倍的时候,就是一个全新的问题,它会带来全新的挑战,也带来全新的价值。"

### **当前数据平台发展现状、挑战与改进**

关涛在演讲中列举了行业最佳实践的多个数据平台架构，并做了一个精炼的总结



![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/YKKw3T8CyLygK8T47qkmcJUEM1252L2b00ha2H8JNQ4lNgUEOIwXKliaBAlyq3y92gibTs3jdFMEHeZs4LcDQ8WQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### 常见数据基础设施架构

正是由于每个组件都有自己独立的数据存储和元数据管理系统,导致了数据的冗余、元数据的不一致、数据流转的复杂性等问题。这种架构难以满足实时性、灵活性和低成本的要求,因为数据在不同的存储之间移动和同步会增加延迟和复杂度,而独立的元数据管理也使得数据治理和一致性维护变得困难。

总结一个典型的数据平台架构，包括以下几个关键组成部分：

1. 底层存储：可以是数据湖或数据仓库。
2. 计算层：分为流式计算、批式计算和交互式分析三个部分。
   - 流式计算：如使用 Flink 进行流处理。
   - 批式计算：如使用 Hadoop 进行离线处理。
   - 交互式分析：如架构图中的实时数仓查询系统。
3. 数据采集：从数据库和应用程序中采集数据。
4. 应用层：包括报表系统和业务系统等。

“这套架构就是我们大数据系统上常说的 Lambda 的架构。”

#### **Lambda架构的问题**

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/YKKw3T8CyLygK8T47qkmcJUEM1252L2bCCUSUvfic53LqARMVuqicicgSGaxtNgOuCSL9EOMbMvpv2o02iaQ773BDA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 标有蓝色三角的是有独立存储的模块。例如流计算Streaming Processing和交互分析Real-Time Analytics有自己的数据存储，导致了数据的冗余。
- 标有小数据库标识的是代表有独立的元数据管理系统，用于描述数据的结构、来源、转换逻辑等信息。

对照这张lambda架构的概念图，关涛总结了Lambda架构的三类问题，并详细展开分析了其中的原因。

#### **问题一：系统架构的组装问题**

关涛谈到，当前Lambda架构由批处理、流计算、交互分析3个“隐形的”模块化的系统拼合，由于彼此独立产生了以下问题：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/YKKw3T8CyLygK8T47qkmcJUEM1252L2bCLib3gakruodPYeUA5y579m9UnsVKq93npLSUDgMSiaJwSm9ooicxB4bw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



#### **问题二：高成本、高TCO**

关涛谈到，除了架构设计的问题外，当前主流数据平台的另一个突出问题是高昂的使用成本。从创新驱动到通用平台，"降低成本"已经成为当前用户最普遍的关切。随着大数据应用的不断深入，越来越多的企业开始将数据平台作为支撑业务运营和决策的基础设施。然而，**在AI时代，当更大规模的数据、更低密度的机器产生数据成为主流时，成本矛盾会变得更加突出**。

一个误区是，往往人们觉得硬件成本才是成本，但实际上企业应真正需要整体考察的是TCO：

***Total Cost of Ownership（TCO） =  硬件+软件成本 +（开发人员成本）+ 维护人力成本 + 治理优化成本***

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/YKKw3T8CyLygK8T47qkmcJUEM1252L2bBfVtfWN8rMzibg8X2aZbROdZJeMIWq7lfO37SO4GP9OC49SwZjhOOyQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



#### **问题三：数据治理和优化低效问题**

除了高昂的使用成本外，当前主流数据平台在数据治理和优化方面的低效问题也日益凸显。随着数据规模的爆发式增长和业务复杂度的不断提升，传统的人工治理和优化方式已经难以应对大数据时代的挑战。

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/YKKw3T8CyLygK8T47qkmcJUEM1252L2bc8sFBtR53SxHciaibPicDKbPsvwZhaicS1Msm8m2MfOh0nbLiaTsuFaZ78A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/YKKw3T8CyLygK8T47qkmcJUEM1252L2b15n3TK7U7HcM4MvuMlobVNFB0hoXiaYFNOR7yjNaIDjdPvTdXJ6MdyQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



## **总结改进目标：下一代数据平台应该具备哪些特性？**

从业务视角看,下一代面向数据分析的数据平台有几个关键的设计目标：

第一,我们希望有一个统一的接口,统一的语法和语义,以及一个统一且开放的数据表达方式。这样,我们的开发人员就不必面对多个引擎进行开发,大大简化了开发复杂度。

第二,我们希望能在数据查询性能、数据新鲜度和成本之间找到一个平衡点。因为这三者很难同时被满足,我们不可能既要高性能、实时更新,又要低成本。但我们可以在这三者之间寻求一个最佳的平衡。

第三,我们希望能够在这个平衡点之间灵活地进行调节。举个例子,如果明天就要进行一个大促活动,我们可能需要紧急地将一个数据处理流程调整为实时计算,因为大促期间的数据分析可能需要按小时甚至分钟来进行。我们的平台必须能够支持这种灵活的调节。

第四,作为一个新一代的平台,它的性能指标至少要达到当前最优的水平,甚至要超越当前水平。这是一个很高的要求,需要我们在架构设计上做出创新。

第五,我们需要在合适的领域引入AI技术。这一点其实与大数据领域的"不可能三角"类似。"不可能三角"告诉我们,数据的新鲜度、资源消耗和性能三者不可能同时被满足,我们只能在其中选择两个,很难兼顾第三个。我们的目标是尽可能扩大这个三角形的面积,但我们必须认识到,三者不可能同时被满足。所以,我们要追求的是在三者之间实现灵活的调节,同时保持接口的一致性。



![img](https://mmbiz.qpic.cn/sz_mmbiz_png/YKKw3T8CyLygK8T47qkmcJUEM1252L2bQlqm1rib6nDic3BeQQRXqHLiaqVibylA00RXCiaiaDhDNSuHpjx1qIOWzzaQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



**下一代数据平台架构应该是什么样的呢？应该由以下三个关键模块组成:**

第一,以湖仓一体作为数据底座。湖仓一体化的趋势目前已经成为业界共识,所有的企业都在向这个方向转型。这个转型过程大概需要5年的时间。之所以需要湖仓一体,是因为当大量半结构化和非结构化数据涌入时,传统的数据仓库已经无法承接。

第二,在数据仓库和数据分析部分,我们正在从Lambda架构走向Kappa架构,走向一个一体化的引擎。如果大家关注业界最先进的数据库和大数据平台,如Snowflake,就会发现它们都采用了这种一体化的架构。

第三,面向AI,我们需要一个可扩展的设计。这种可扩展性体现在多个方面,如湖仓一体的开放性设计,引擎的可插拔设计等。因为面向未来,同一份数据可能会被多个引擎或系统所消费,所以整个平台的开放性和可扩展性将成为关键。



![img](https://mmbiz.qpic.cn/sz_mmbiz_png/YKKw3T8CyLygK8T47qkmcJUEM1252L2b8E4qTfgc39Ye1OlxqfyzCOPeoEsHISFKUKyKiaGiaM34pzCaFuCvbFUA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

总之,下一代面向数据分析的数据平台需要在架构上进行重大的革新。它需要能够处理海量的半结构化和非结构化数据,需要提供统一的接口和语义,需要在性能、实时性和成本之间找到最佳平衡,需要能够灵活调节,需要引入AI技术。只有满足了这些要求,才能够适应未来数据分析的需求,帮助企业从海量复杂数据中持续不断地提炼出商业价值。

- 以湖仓一体作为数据底盘，承接大量半非结构化数据。
- 数据分析部分收敛为“一体化引擎”架构（关键改变），从Lambda架构走向Kappa架构。
- 面向AI做可扩展设计，包括湖仓一体的开放性设计和引擎的可插拔设计。
- 一份数据要被多个引擎或系统消费，开放性和可插拔性是设计关键。

**什么是数据的不可能三角，为什么上一代引擎难以实现一体化**

"数据的不可能三角"，形象地描述了数据处理过程中三个关键要素之间的矛盾关系，即数据新鲜度、资源消耗和性能之间的trade-off。我们很难同时兼顾这三个方面，往往需要在其中做出权衡和取舍。

数据的不可能三角：

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/YKKw3T8CyLygK8T47qkmcJUEM1252L2boDV1leq1biaCibx7fOAUPxuxicFEJ8aOARtoVekxOL4sf0YF6WTcib2Uyw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

专项优化的引擎难以做到一体化

- 优化方向不同：流计算面向数据新鲜度、批处理面向成本、交互分析面向查询性能
- 计算形态不同：批处理是主动计算过程，流计算是被动计算过程，数据驱动计算



![img](https://mmbiz.qpic.cn/sz_mmbiz_png/YKKw3T8CyLygK8T47qkmcJUEM1252L2bbY1B7Q3GBFoug3s7vGqbTwlQfZialZ1U7I0Ba9RD4CyvnxuURu8Duuw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**通用增量计算，解决一体化难题的新方式**

面对流、批、交互计算模式优化方向和计算形态不同的问题，传统的方式难以实现真正意义上的一体化。关涛分享到，“通用增量计算为解决这一难题提供了新的思路。通过引入增量计算，Lakehouse可以统一流、批、交互三种模式，通过调整增量计算的时间间隔，我们可以在数据新鲜度和成本之间灵活平衡；通过采用增量存储和Shared Everything架构，我们可以在存储和计算层面支撑一体化的实现。”

下面我们来详细了解增量计算的工作原理以及基于增量计算的系统设计方案。

**增量计算的概念如下**

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/YKKw3T8CyLygK8T47qkmcJUEM1252L2biccC1bwKQ4N4yfh2sDbdqMxsdsS5KGMuP9icwGnib8OJo28NJxWviaBrfA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

增量计算示意图

- 在T0时间点开始计算，得到T1时间点的结果集（Result set T1）
- 下一次计算从T1结果集和T1到T2的增量数据（Delta）进行计算，而不是从T0重新计算
- 通过Merge操作，将T1结果集和增量计算结果合并，得到T2时间点的结果集

增量计算的价值在于其灵活性，可以在批处理、流计算间根据需求切换，不用改变数据链路。当T0到T1的时间间隔足够长时，就是批处理；当时间间隔缩短到1秒钟时，就是流计算；通过调整时间间隔，可以在数据新鲜度和低成本之间找到平衡点。

使用增量计算统一所有计算模式，公式为：Result set T0 + Delta T0 = Result set T1；

在湖仓之上加入增量存储，作为统一的存储形态，表达增量的能力；采用Shared Everything的系统架构，融合实时数仓的MPP架构和批处理的BSP架构；灵活支持不同的调度模式。最终形成单引擎（Single-engine）设计，一个引擎或Kappa架构的引擎，支撑流、批、交互，通过增量计算、增量存储和Shared Everything架构实现一体化。

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/YKKw3T8CyLygK8T47qkmcJUEM1252L2b0upuCmW91elc2Oj01JpgfaHcasB4ndGopBQARx0iayj5JX7YsicsxMbw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**最终形成单引擎（Single-engine）设计：**

增量计算的引入，解决了流、批、交互计算模式优化方向和计算形态不同的问题，通过时间间隔的调整，在数据新鲜度和成本之间找到平衡。同时，增量存储和Shared Everything架构的采用，进一步支撑了单引擎设计，实现了真正意义上的一体化。

面对流、批、交互计算模式优化方向和计算形态不同的问题，传统的方式难以实现真正意义上的一体化。

**Single-Engine引擎在离线和实时场景的性能**

**批处理性能验证：**

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/YKKw3T8CyLygK8T47qkmcJUEM1252L2b5YGuGAlojRiaKNicib3zyql0OXvvjSQ4vNMcyxUIicpVG7AlpZmazRoXag/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 使用TPC-DS 10TB数据集进行ETL性能测试
- 使用TPC-H 100G数据集进行即席（Ad-hoc）查询性能测试
- 使用SSB flat数据集进行在线分析场景性能测试
- 与Spark、Clickhouse和Snowflake进行性能对比
  - 不同场景下，比Spark快7倍-40倍
  - 最多比Snowflake快3倍
  - 在Clickhouse擅长的两个场景中，比Clickhouse快20%到30%

**流计算性能验证：**

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/YKKw3T8CyLygK8T47qkmcJUEM1252L2bKrx3ibXdC9KtM5ubC5a2u5ARIAWhaqAGbQ1zQkhjJqHq0UIhgmb4fAA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 选取四个场景：单行处理、单流Join、单表聚合和双流聚合
- 与Flink进行资源消耗对比（越低表示资源消耗越少）
- 一体化引擎是完全原生（Native）和向量化的引擎，而Flink是10年前的Java引擎，仅针对流进行优化
- 使用增量计算，面向低成本进行优化
- 在不同场景下，相比Flink，一体化引擎在资源节省方面可能有10到1000倍的提升

性能测试验证了一体化引擎在处理大规模数据时的出色表现,同时其流、批、交互一体化的架构设计能够有效简化数据处理流程。包括我们服务的，以及更多企业开始认识到一体化引擎在提升数据分析效率方面的价值。

**AI如何为数据管理提效？**

在前面介绍了一体化引擎的优异性能后，我们仍然面临着一个问题：如何在不断变化的业务需求和工作负载下，持续保持数据建模的优化和平衡点的把控？传统的方式需要投入大量的人力和时间成本，而且难以应对动态变化的需求。

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/YKKw3T8CyLygK8T47qkmcJUEM1252L2bmh9kKjUQnu0FrOmvk1RNFticUUZmIIVlakibPMWQAo1BzUdu8uPIZTtg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

为了解决这个问题，我们引入了AI for Data的理念，即通过AI的方式来优化数据系统。这个概念在业界也得到了广泛的关注和实践，如Databricks和Amazon Redshift都在这一领域投入了重要的工作。

具体来说，我们的做法是在一体化引擎平台之外，采集各种运行状态的数据，如资源消耗、利用率、数据倾斜等。

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/YKKw3T8CyLygK8T47qkmcJUEM1252L2bWA1OQW9IhkSS4hn4VFMo20TicrSXSM9uVPOFAjBZtQ7LrjAic5h0LneQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

系统架构示意图，数据采集、分析、优化、反馈的闭环

然后经过分析和建模，得到一套优化建议，并将其反馈到系统中，形成一个自学习的闭环。通过这种方式，我们可以让系统持续处于一个相对更优的状态。

举个简单的例子，当我们对某张表加上索引时，虽然可以提升查询性能，但同时也会带来额外的成本。对于少量的表，人工分析和权衡是可行的；但当表的数量达到数万甚至更多时，人力优化几乎是不可能的。而通过AI for Data的方式，我们可以自动尝试添加索引，评估其带来的收益和成本，从而判断是否满足要求，进而决定保留还是删除索引。

我们在真实客户的数据集上进行了测试，发现通过对ODS层数据的计算冗余进行抽象和优化，可以节省40%的资源消耗，同时还能提升30%的查询性能。

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/YKKw3T8CyLygK8T47qkmcJUEM1252L2bs1wiaD8NUQ0FosTAZ32QIYSMqftG02AUV0tF8Zib4WeXmwVajZic55NicA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

优化前后的资源消耗和查询性能对比图

总的来说，AI for Data是一个非常有前景的方向，它可以帮助我们在不断变化的需求中，持续保持数据系统的最优状态，实现一举多得的效果。而这，也是我们在构建下一代数据平台时，必须重点考虑和实践的一个方向。
