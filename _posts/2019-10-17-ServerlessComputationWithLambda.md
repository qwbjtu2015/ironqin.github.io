---
layout: post
title: "Serverless Computation With OpenLambda"
date: 2019-10-17 
description: "OpenLambda论文翻译"
tag: 无服务器计算

---  

> 文章提出了一个全新的开源平台OpenLambda，针对无服务器计算中越来越多的模型，用于构建下一代web服务和应用。文章介绍了无服务器计算的关键部分并列举了在此类系统设计和实现中必须要解决的一系列问题。为了更好地促进无服务器应用的构建，文章对于当前的web应用也做了简要研究。

## 1 引言

&emsp;&emsp;数据中心和软件平台的快速发展引发了我们对于如何构建、部署和管理线上应用和服务的思考。之前每个应用都运行在自己的物理机上，但购买和维护大量的机器成本非常高，同时这些机器实际上没有被充分利用，这些问题促成了一个跨越式的进步：虚拟化。虚拟化使大量服务整合在服务器上，极大地降低了成本，同时方便了管理。

&emsp;&emsp;然而基于硬件的虚拟化并不是万能的，为解决更加基础性的问题，更加轻量级的虚拟化技术被提出。这里最广泛的一个解决方案就是“容器”，它通过在namespace上进行虚拟化实现，类似一个Unix进程运行着应用。通过与Docker等分布式工具相结合，容器可以让开发者无需等待虚拟机漫长时间的启动过程，非常便捷和快速的启动一个新服务。

&emsp;&emsp;但无论是基于虚拟机还是基于容器，都是以服务器为中心。长时间以来，服务器都是作为线上服务的后台，但新的云计算平台的出现预示着传统服务器作为后端的结束。服务器的配置和管理非常复杂，而且服务器的启动时间严重限制了云计算时代应用快速扩展的能力。

&emsp;&emsp;因此，“无服务器计算”模型被提出以适应当前可扩展类应用。开发者不用再把应用看做一组服务器的集合，仅需要定义一系列访问共同数据存储的函数。关于此类微服务的例子可以在亚马逊的[Lambda](https://aws.amazon.com/cn/lambda/)上找到，因此我们把这类服务构建叫做Lambda模型。

&emsp;&emsp;与传统的基于服务器的方法相比，Lambda模型具有许多优势。来自不同用户的Lambda handler(handler是Lambda中的函数接口）可以共享云服务提供商管理的服务器池，因此开发者无需关心服务器管理。Handler一般使用JavaScript或Python语言编写；通过在不同函数间共享运行时环境，针对特定应用的代码就会比较小，所以将这些handler代码发往集群中的其他worker节点成本也是很小的。最后，应用程序可以实现在不启动新服务器的前提下快速扩展。Lambda模型代表了应用程序间共享的最高层次，从硬件共享（虚拟机）到操作系统共享（容器）最终到运行时环境共享（Lambda）。

![Fig1 共享层级的革新](/images/posts/paper/openLambda-fig1.png)

&emsp;&emsp;论文中，我们提出了Lambda模型并讨论了相关的研究挑战。Lambda执行引擎必须安全有效的隔离handler。Handler是无状态的，因此在Lambda和数据库服务的集成方向有许多机会。Lambda负载均衡器必须根据session和代码以及数据位置选择出低延迟的方案。此外文章也进一步探索了一些新挑战，如即时编译（JIT）、软件包管理、web session、经济成本以及可移植性。

&emsp;&emsp;不幸的是，大多数现存的无服务器计算的系统实现多数都是闭源的。为了促进在Lambda架构上的研究，我们目前正在构建OpenLambda，研究人员可以在这个平台之上进行新的无服务器计算方法的评估，本篇论文也是实现这个OpenLambda平台的第一步。

## 2 Lambda背景介绍

&emsp;&emsp;为了聚焦于一个Lambda环境的具体实现，我们选择AWS的Lambda云平台作为研究对象。我们描述了AWS Lambda的编程模型（2.1节）和相比于基于服务器的模型的优势（2.2节）。

### 2.1 编程模型

Lambda模型允许开发者指定对应于不同事件的函数。我们这里讨论的事件是一个来自web应用的RPC调用且上传的函数是一个RPC handler的情况。开发者选择一个运行时环境（如Python7），然后上传相关代码，并指定它要处理的事件的函数名称。开发者就可以将一个使用独立的AWS网关服务的URL与Lambda关联起来。客户端代码则可以通过向这个URL发起请求来发起RPC调用（例如JavaScript可以通过AJAX发起POST请求）。

&emsp;&emsp;Handler可以在任一worker节点上运行，在AWS上，一个新的worker节点的启动时间为1-2秒。一旦有节点宕机，负载均衡器可以马上在一个新worker节点上启动Lambda handler来处理RPC调用的请求，而不会产生过多延迟。然而，对特定Lambda的请求一般会发往相同的worker节点来避免沙箱重新初始化所产生的时间消耗。

&emsp;&emsp;开发者可以对一个handler使用的资源设置限额（如设置内存和时间上限）。在AWS上，一个调用的费用内存上限（不是实际使用的内存）对应的费用乘以实际的执行时间，执行时间四舍五入到100ms的倍数。

&emsp;&emsp;Lambda函数必须是无状态的，如果相同的handler在相同的worker节点上倍同时调用，共同的状态必须是对任一个调用可见的，但是系统没有对此提供保证。因此，Lambda应用经常与云数据库同时使用。

### 2.2 Lambda优势

Lambda模型的一个主要优势在于当负载突然增加时可以快速自动扩展worker节点数量的能力。为了证实这一点，我们比较了AWS Lambda和基于容器的服务器平台[AWS Elastic Beanstalk](https://aws.amazon.com/cn/elasticbeanstalk/?nc2=type_a)（下文简称Elastic BS）。两个平台上，我们同时运行相同的标准一分钟：工作负载维持100个未完成的RPC请求，并且每个RPC handler持续200ms。

![Fig2 *响应时间* 该CDF显示从模拟负载突发到Elastic BS应用程序和AWS Lambda应用程序的响应时间度量](/images/posts/paper/openLambda-fig2.png)

&emsp;&emsp;上图2显示：使用AWS Lambda的RPC调用平均响应时间仅1.6秒，然而Elastic BS经常需要花费20秒左右。探究其中的原因，我们发现AWS可以在1.6秒之内启动100个worker实例来同时处理100个请求，但是Elastic BS则都是通过一个实例来处理这些请求，每个请求都需要前面的请求处理结束后才能进行处理，也就是100\*200ms=20s。

&emsp;&emsp;AWS Lambda同时具有不需要请求扩展所需要的配置，相反，Elastic BS的配置复杂，单单扩展操作就涉及20个不同的设置。即使我们尽可能快的调整Elastic BS扩展（不考虑经济成本），仍然不能在几分钟的时间内启动新的worker。

## 3 Lambda工作负载

&emsp;&emsp;到此为止，我们还没接触到Lambda的工作负载，它作为web服务（如Gmail或Facebook）的主要部分，目前都是在无服务器模型出现之前构建的。但通过分析这些现存的服务我们可以李阿娇未来的工作负载如何在Lambda环境上施加压力。具体地，我们分析现存的基于RPC的C/S模式的应用：Google Gmail。Gmail使用客户端的JavaScript通过ROC获取动态内容。JS RPC库（如AJAX）基于XHR接口，通过HTTP协议向后端发送GET或POST请求，参数和返回值被编码在URL或消息体（如JSON）中。我们使用Chrome扩展工具跟踪这些RPC调用。我们的工作负载包括刷新Gmail收件箱页面(浏览器缓存应该是存在的)。

![Fig3 *Google Gmail* 黑线代表RPC消息，灰线代表其他消息，线结束代表着请求和响应时间。上图按照GET和POST请求分为了上下两组](/images/posts/paper/openLambda-fig3.png)

## 4 研究内容

&emsp;&emsp;现在我们探索在无服务器计算领域的一些研究热点。

### 4.1 执行引擎

### 4.2 编译型语言

### 4.3 软件包支持

### 4.4 Cookies和Sessions

### 4.5 数据库

### 4.6 数据聚合器

### 4.7 负载均衡

### 4.8 成本分析

### 4.9 系统分解

## 5 面向OpenLambda

## 6 致谢
