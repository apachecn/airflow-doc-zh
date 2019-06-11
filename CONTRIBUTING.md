# 贡献指南

> 请您勇敢地去翻译和改进翻译。虽然我们追求卓越，但我们并不要求您做到十全十美，因此请不要担心因为翻译上犯错——在大部分情况下，我们的服务器已经记录所有的翻译，因此您不必担心会因为您的失误遭到无法挽回的破坏。（改编自维基百科）

原文：

+ [在线阅读](https://airflow.apache.org/)

负责人：

* [@ImPerat0R\_](https://github.com/tssujt)

截止时间：领取任务之后的两个星期（每章）

## 目录

+ [项目](zh/project.md)
+ [协议](zh/license.md)
+ [快速开始](zh/start.md)
+ [安装](zh/installation.md)
+ [教程](zh/tutorial.md)
+ [操作指南](zh/howto/index.md)
+ [设置配置选项](zh/howto/set-config.md)
+ [初始化数据库后端](zh/howto/initialize-database.md)
+ [使用Operators](zh/howto/operator.md)
+ [管理连接](zh/howto/manage-connections.md)
+ [保护连接](zh/howto/secure-connections.md)
+ [写日志](zh/howto/write-logs.md)
+ [用Celery扩大规模](zh/howto/executor/use-celery.md)
+ [用Dask扩展](zh/howto/executor/use-dask.md)
+ [使用Mesos扩展（社区贡献）](zh/15.md)
+ [使用systemd运行Airflow](zh/howto/run-with-systemd.md)
+ [用upstart运行Airflow](zh/howto/run-with-upstart.md)
+ [使用测试模式配置](zh/howto/use-test-config.md)
+ [UI/截图](zh/ui.md)
+ [概念](zh/concepts.md)
+ [数据分析](zh/profiling.md)
+ [命令行界面](zh/cli.md)
+ [调度和触发器](zh/scheduler.md)
+ [插件](zh/plugins.md)
+ [安全](zh/security.md)
+ [时区](zh/timezone.md)
+ [实验性 Rest API](zh/api.md)
+ [集成](zh/integration.md)
+ [数据血缘](zh/lineage.md)
+ [常见问题](zh/faq.md)
+ [API 参考](zh/code.md)

## 流程

### 一、认领

首先查看[整体进度](https://github.com/apachecn/airflow-doc-zh/issues/1)，确认没有人认领了你想认领的章节。

然后回复 ISSUE 来认领章节，注明“章节 + QQ 号”。

### 二、校对

校对：

+ 语法
+ 术语使用
+ 代码格式

### 三、提交

+ `fork` Github 项目
+ 修改`zh`下面的文档
+ `push`
+ `pull request`

请见 [Github 入门指南](https://github.com/apachecn/kaggle/blob/master/docs/GitHub)。
