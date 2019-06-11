# 操作指南

> 贡献者：[@ImPerat0R\_](https://github.com/tssujt)

在[快速开始](zh/start.md)部分设置沙箱很容易; 建设一个生产级环境则需要更多的工作！

这些操作指南将指导您完成使用和配置 Airflow 环境的常见任务。

* [设置配置选项](zh/howto/set-config.md)
* [初始化数据库后端](zh/howto/initialize-database.md)
* [使用Operators](zh/howto/operator.md)
  * [BashOperator](zh/howto/operator.md)
  * [PythonOperator](zh/howto/operator.md)
  * [Google Cloud Platform Operators](zh/howto/operator.md)
* [管理连接](zh/howto/manage-connections.md)
  * [使用 UI 创建连接](zh/howto/manage-connections.md)
  * [使用 UI 编辑连接](zh/howto/manage-connections.md)
  * [使用环境变量创建连接](zh/howto/manage-connections.md)
  * [连接类型](zh/howto/manage-connections.md)
* [保护连接](zh/howto/secure-connections.md)
* [写日志](zh/howto/write-logs.md)
  * [在本地编写日志](zh/howto/write-logs.md)
  * [将日志写入 Amazon S3](zh/howto/write-logs.md)
  * [将日志写入 Azure Blob 存储](zh/howto/write-logs.md)
  * [将日志写入 Google 云端存储](zh/howto/write-logs.md)
* [用 Celery 扩大规模](zh/howto/executor/use-celery.md)
* [用 Dask 扩展](zh/howto/executor/use-dask.md)
* [使用 Mesos 扩展（社区贡献）](zh/15.md)
  * [任务直接在 mesos 从站上执行](zh/15.md)
  * [在 mesos 从站上的容器中执行的任务](zh/15.md)
* [使用 systemd 运行 Airflow](zh/howto/run-with-systemd.md)
* [用 upstart 运行 Airflow](zh/howto/run-with-upstart.md)
* [使用测试模式配置](zh/howto/use-test-config.md)
