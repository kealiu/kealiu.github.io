---
layout: post
title: AWS CodeGuru Reviewer 介绍
category: AWS
tags: AWS CodeGuru CI
---

# AWS CodeGuru Reviewer是什么

CodeGuru是AWS近期推出的，基于机器学习的代码分析及性能分析工具。其中Reviewer主要用于代码提交是的code review以及整个代码仓库的代码分析。CodeGuru现在只支持Java，未来会推出其他语言的支持。

## CodeGuru Reviewer 所解决的问题

我们常见的研发流程如下图。CodeGuru Reviewer 通过自动化的code review，解决现在 code review流程走形式、浪费大量人力、结果不理想的问题。

![Devlopping Cycle](https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2020/02/28/DevWorkflow.png)

## CodeGuru Reviewer的能力 
Reviewer主要包含了以下方面的代码分析：
- AWS最佳实践
- 程序并发
- 资源泄露
- 敏感信息泄露
- 编码规范
- 输入校验

### 支持的git仓库

现阶段，AWS CodeGuru Reviewer支持以下git仓库
- AWS CodeCommit
- Bitbucket
- GitHub
- GitHub Enterprise Cloud
- GitHub Enterprise Server

仓库类型在不断增加中，以实际情况为准。

## CodeGuru 最佳实践

在引入了CodeGuru后（含CodeGuru Profiler)，建议采用以下最佳实践流程

![Devloper workflow with CodeGuru](https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2020/02/28/bigPicture.png)

# CodeGuru Review 实战

## 登录进入 CodeGuru

登录到AWS Console后，搜索 CodeGuru，即可进入CodeGuru页面。我们可以通过左侧的导航栏或者右侧的下来框进入。

![CodeGuru Console](https://raw.githubusercontent.com/kealiu/kealiu.github.io/master/images/2020-08-22-CodeGuru/ConsoleMain.JPG)

## 关联 git 仓库

进入CodeReviewer后，我们首先需要关联git仓库。 
- 如果是AWS CodeCommit，直接在最下方选择仓库即可
- 如果是Bitbucket/GitHub等，需要先通过OAUTH进行授权，以GitHub为例
    - 点击“Connect to GitHub"
    - 在弹出的对话框中登录 Github
    - 成功登录后，确认授权给AWS
    - 等待关联成功

![Assocaiate Repo](https://raw.githubusercontent.com/kealiu/kealiu.github.io/master/images/2020-08-22-CodeGuru/associate_repo.JPG)


## 代码Review

CodeGuru有两种Review模式：
- 一种是增量模式，每次进行提交代码时进行。适合于日常开发。
- 一种是全量模式，可以自行决定扫描时间，非常适合定期代码全量检查。

### PR/增量 Review

在PR模式下，开发在开发分支进行代码编写。完成后，提交Pull Requests合并到生产分支。这个时候会促发CodeGuru进行代码Review。

![PullRequest](https://raw.githubusercontent.com/kealiu/kealiu.github.io/master/images/2020-08-22-CodeGuru/PullRequests.JPG)


以 [https://github.com/kealiu/amazon-codeguru-reviewer-sample-app](https://github.com/kealiu/amazon-codeguru-reviewer-sample-app) 为例，我在dev分支进行了代码修改，准备合并到master分支，所以，我创建了一个Pull Requests，如图

![CodeReview](https://raw.githubusercontent.com/kealiu/kealiu.github.io/master/images/2020-08-22-CodeGuru/CodeReview.JPG)

在提交PullRequests后，CodeGuru会自动检测到该PullRequest并自动进行CodeReview。Review的进度及结果可以在 AWS 控制台CodeGuru Reviewer下面的 Code Reviews中看到。

![CodeReviewResult](https://raw.githubusercontent.com/kealiu/kealiu.github.io/master/images/2020-08-22-CodeGuru/CodeReviewResult.JPG)

在CodeGuru 分析完成后，我们可以在Code Reviewer中看到，也可以在GitHub上看到对应的comments. 开发可以根据结果进行调整，再合并代码。
![Comments In GitHub](https://raw.githubusercontent.com/kealiu/kealiu.github.io/master/images/2020-08-22-CodeGuru/comments.JPG)

### 全量Review

在CodeGuru Review界面，我们可以进入 Repository analysis 界面，进行全量的代码扫描分析。点击Create Repository Analysis，进入下面的创建页面。

![RepoReview](https://raw.githubusercontent.com/kealiu/kealiu.github.io/master/images/2020-08-22-CodeGuru/code-repo.JPG)

选择已经关联的代码仓库，然后输入期望扫描的分支，创建代码扫描。

![ReviewSelect](https://raw.githubusercontent.com/kealiu/kealiu.github.io/master/images/2020-08-22-CodeGuru/create_repo_review.JPG)

等待一小会，扫描完成后点击对应的分析项，即可进入查看详情。内容与PR扫描类似。

# CodeGuru 价格

 / | 前1500000行 | 超出1500000行
---|--- |---
全量Repo分析 | $0.5/100行 | $0.4/100行
PR/增量 | $0.75/100行 | $0.75/100行

# 总结

GuruCode Review帮助客户提早发现代码问题，提升代码质量，减少线上故障。CodeGuru 目前支持 Java 应用程序，很快还会支持更多的语言。CodeGuru 可帮助您更快、更早地发现问题，以便您可以构建和运行更出色的软件。

