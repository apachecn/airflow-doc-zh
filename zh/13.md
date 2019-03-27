# 使用 Celery 扩大规模

> 贡献者：[@ImPerat0R\_](https://github.com/tssujt)、[@ThinkingChen](https://github.com/cdmikechen)

`CeleryExecutor`是您扩展 worker 数量的方法之一。为此，您需要设置 Celery 后端（**RabbitMQ**，**Redis**，...）并更改`airflow.cfg`以将执行程序参数指向`CeleryExecutor`并提供相关的 Celery 设置。

有关设置 Celery broker 的更多信息，请参阅[有关该主题的详细的 Celery 文档](http://docs.celeryproject.org/en/latest/getting-started/brokers/index.html)。

以下是您的 workers 的一些必要要求：

* 需要安装`airflow`，CLI 需要在路径中
* 整个集群中的 Airflow 配置设置应该是同构的
* 在 worker 上执行的 Operators（执行器）需要在该上下文中满足其依赖项。例如，如果您使用`HiveOperator`，则需要在该框上安装 hive CLI，或者如果您使用`MySqlOperator`，则必须以某种方式在`PYTHONPATH`提供所需的 Python 库
* workers 需要访问其`DAGS_FOLDER`的权限，您需要通过自己的方式同步文件系统。常见的设置是将 DAGS_FOLDER 存储在 Git 存储库中，并使用 Chef，Puppet，Ansible 或用于配置环境中的计算机的任何内容在计算机之间进行同步。如果您的所有盒子都有一个共同的挂载点，那么共享您的管道文件也应该可以正常工作

要启动 worker，您需要设置 Airflow 并启动 worker 子命令

```py
airflow worker
```

您的 worker 一旦启动就应该开始接收任务。

请注意，您还可以运行“Celery Flower”，这是一个建立在 Celery 之上的 Web UI，用于监控您的 worker。 您可以使用快捷命令`airflow flower`启动 Flower Web 服务器。

一些警告：

* 确保使用数据库来作为 result backend（Celery result_backend，celery 的后台存储数据库）的后台存储
* 确保在[celery_broker_transport_options]中设置超过最长运行任务的 ETA 的可见性超时
* 任务会消耗资源，请确保您的 worker 有足够的资源来运行 worker_concurrency 任务
