---
layout: post
title: AWS CodeGuru Profiler 介绍
category: AWS
tags: AWS CodeGuru CI
---

# AWS CodeGuru Profiler 是什么

CodeGuru是AWS近期推出的，基于机器学习的代码分析及性能分析工具。其中Profiler类似于JProfiler，但CodeGuru Profiler将采集到的运行数据上传到AWS CodeGuru服务，再由CodeGuru服务进行汇总，通过机器学习做深入分析，用以发现应用性能瓶颈，帮助用户提升应用效率。CodeGuru现在只支持Java系列（所有跑在JVM中的语言），未来会推出其他语言的支持。

## CodeGuru Profiler 所解决的问题

我们常见的研发流程如下图。CodeGuru Profiler 通过无侵入收集应用运行数据，再结合大数据分析，帮助用户实时了解程序运行状态。

![Devlopping Cycle](https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2020/02/28/DevWorkflow.png)

## CodeGuru Profiler的能力 
CodeGuru Profiler提供了统一的Profiler管理分析中心及对应的看板，主要包含了以下方面的能力：
- 应用时延及CPU使用率问题分析
- 帮助提升基础设施使用率
- 及时发现应用性能问题

### 支持的接入方式

现阶段，AWS CodeGuru Profiler支持无侵入接入，也支持通过SDK添加代码接入。

| 场景                         | 命令行无侵入                      | SDK代码                       |
|--------------------------------|------------------------------------|----------------------------|
| 对已经存在的应用做Profile   | Yes                                | No (需要重新编译) |
| 自定义鉴权信息 | No                                 | Yes                        |
| 控制Profile时间及覆盖范围 | No (从开始到应用推出) | Yes                        |
| 功能自动更新             | No (需要下载最新的Agent)           | Yes (下次编译后) |

CodeGuru 是全球服务，用户应用可以运行在任何环境，只要CodeGuru Agent/SDK能将Profile数据上传到支持CodeGuru的Region即可（网络可达）。

## CodeGuru 最佳实践

在引入了CodeGuru后（含CodeGuru Reviewer)，建议采用以下最佳实践流程

![Devloper workflow with CodeGuru](https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2020/02/28/bigPicture.png)

# CodeGuru Profiler 实战

本次Demo基于官方示例代码进行。代码包含两部分：生产者产生任务，消费者消费任务。使用SQS进行任务传递。任务内容为图片处理（CPU密集 & IO密集）。可以参考我fork的代码：[https://github.com/kealiu/aws-codeguru-profiler-demo-application](https://github.com/kealiu/aws-codeguru-profiler-demo-application)

## 登录进入 CodeGuru

登录到AWS Console后，搜索 CodeGuru，即可进入CodeGuru页面。我们可以通过左侧的导航栏或者右侧的下来框进入。

![CodeGuru Console](https://raw.githubusercontent.com/kealiu/kealiu.github.io/master/images/2020-08-22-CodeGuru/ConsoleMain.JPG)

选择Profiler，即可进入CodeGuru Profiler界面，如图所示：
![CodeGuru Profiler Console](https://raw.githubusercontent.com/kealiu/kealiu.github.io/master/images/2020-08-22-CodeGuruProfiler/CodeGuruProfilerConsole.png)

## 创建Profiling Group

一个Profiling Group是会被CodeGuru当作一个整体来进行分析。一个Profiling Group可以有多个应用信息，建议将相关联的应用放在一个Profiling Group。

![Create CodeGuru Profiler Group](https://raw.githubusercontent.com/kealiu/kealiu.github.io/master/images/2020-08-22-CodeGuruProfiler/CodeGuruNewProfileGroup.png)

## 下载sample代码

为了保证示例代码于本文一致，我把官方代码fork了一份。大家可以clone一下代码到本地。

```bash
git clone git@github.com:kealiu/amazon-codeguru-reviewer-sample-app.git
```

## 准备环境

因为需要配置SQS以及S3，本Demo使用AWS CLI（当然，你也可以使用控制台创建）

按照AWS CLI详见 [https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html)

```bash
# 配置AWS 认证鉴权信息
aws configure

# 创建CodeGroup Profiling Group
aws codeguruprofiler create-profiling-group --profiling-group-name DemoApplication-WithIssues

# 创建另外一个CodeGroup Profiling Group
aws codeguruprofiler create-profiling-group --profiling-group-name DemoApplication-WithoutIssues

# 创建程序依赖的S3 bucket
aws s3 mb s3://demo-application-test-bucket-1092734-REPLACE-ME

# 创建程序依赖的 SQS
aws sqs create-queue --queue-name DemoApplicationQueue
```

## 管理Profiling Group 权限

再运行程序之前，我们必须确认，程序有相应的CodeGuru Profiling Group的权限。如果是运行于AWS上的EC2，我们可以通过绑定Role到EC2，然后赋予对应Role Profiling Group权限。否则，根据程序配置的用户信息，赋予对应IAM用户权限。

在CodeGuru Profiling Console，选择Profiling Group后，点击Action进入权限管理，如图所示：

![Entering CodeGuru Profiling Groupo Permission Management](https://raw.githubusercontent.com/kealiu/kealiu.github.io/master/images/2020-08-22-CodeGuruProfiler/EnterProflngGroupPermission.png)

在权限管理中， 添加程序对于的Role或者IAM 用户：
![CodeGuru Profiling Groupo Permission Management](https://raw.githubusercontent.com/kealiu/kealiu.github.io/master/images/2020-08-22-CodeGuruProfiler/ProfilerManagerPermission.png)

## 运行示例程序

配置好环境后，可以分别运行有问题（会有错误日志）、正常版本进行对比。

### 环境准备

根据你的实际情况，替换下面脚本中的`YOUR-ACCOUNT-ID`, `demo-application-test-bucket-1092734-YOUR-BUCKET-REPLACE-ME`, `YOUR-AWS-REGION`

```
# 配置SQS地址环境变量
export DEMO_APP_SQS_URL=https://sqs.YOUR-AWS-REGION.queue.amazonaws.com/YOUR-ACCOUNT-ID/DemoApplicationQueue

# 配置S3 Bucket环境变量
export DEMO_APP_BUCKET_NAME=demo-application-test-bucket-1092734-YOUR-BUCKET-REPLACE-ME

# 配置CodeGuru Profiling Group所在的区域
export AWS_CODEGURU_PROFILER_TARGET_REGION=YOUR-AWS-REGION

# 配置CodeGuru Proling Group名称
export AWS_CODEGURU_PROFILER_GROUP_NAME=DemoApplication-WithIssues

# 编译示例程序 `DemoApplication-1.0-jar-with-dependencies.jar`
mvn clean install
```

### 运行程序的问题版本
```
java -javaagent:codeguru-profiler-java-agent-standalone-1.0.0.jar \
  -jar target/DemoApplication-1.0-jar-with-dependencies.jar with-issues
```

允许过程中，会看到`Expensive exception`, `Pointless work` 等error信息。这是预期的效果。

### 运行程序的正常版本
```
java -javaagent:codeguru-profiler-java-agent-standalone-1.0.0.jar \
  -jar target/DemoApplication-1.0-jar-with-dependencies.jar without-issues
```

### 等待分析结果

如果一切顺利，程序会将Profiling 数据上传到CodeGuru，大概5到15分钟后，在收集到足够数据后，CodeGuru会产生分析结果。

数据结果以火焰图的方式展示，火焰图的详细介绍可以参考 [https://www.ruanyifeng.com/blog/2017/09/flame-graph.html](https://www.ruanyifeng.com/blog/2017/09/flame-graph.html) 。

分析火焰图的展示方式有Overview/Hotspots与CPU/Latency两个维度：


 /  | CPU | Latency
---|---|---
概览 | CPU标准火焰图 | Latency标准火焰图
热点 | CPU热点图，与标准火焰图相反 |  Latency热点图，与标准火焰图相反

火焰图中，根据颜色，对`My Code`（自己写的代码），`Other code`（库代码）进行了区分，方便快速定位性能问题类型。

标准火焰图中，顶部出现平坦，意味着性能问题。理想状态下应该都是尖峰。通过逐步消除顶部平坦，进行性能优化。

#### CPU性能分析

![CPU Profling](https://raw.githubusercontent.com/kealiu/kealiu.github.io/master/images/2020-08-22-CodeGuruProfiler/ProfilerCPU.png)

#### Latency性能分析

![Latency Profling](https://raw.githubusercontent.com/kealiu/kealiu.github.io/master/images/2020-08-22-CodeGuruProfiler/ProfilerLatency.png)

### 改进措施推荐

分析结果还包含了整改措施，给用户提出一些整改意见，帮助用户消除程序潜在性能问题。

![Recommendations](https://raw.githubusercontent.com/kealiu/kealiu.github.io/master/images/2020-08-22-CodeGuruProfiler/ProfilerRecommendations.png)

# CodeGuru Profiler价格

按每实例每小时收费。一个实例就是一个运行CodeGuru Client的运行中的程序。

 / | 36,000小时内 | 超过36,000小时
---|--- |---
Lambda	 | 前500小时免费，之后$0.005/小时 | 该月免费
非Lambda（EC2，EKS，etc）| $0.005/小时 | 该月免费

# 总结

GuruCode Guru通过集中收集程序运行数据，帮助用户提升运行时性能，减少性能瓶颈，最终帮助提升用户体验，消除潜在风险，减少资源浪费。CodeGuru 目前支持 Java 应用程序，很快还会支持更多的语言。
