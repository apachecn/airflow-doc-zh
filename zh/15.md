# 使用 Mesos 扩展（社区贡献）

> 贡献者：[@ImPerat0R\_](https://github.com/tssujt)、[@ThinkingChen](https://github.com/cdmikechen)

有两种方法可以将 Airflow 作为 mesos 框架运行：

1. 想要直接在 mesos slaves 上运行 Airflow 任务，要求每个 mesos slave 安装和配置 Airflow。
2. 在安装了 Airflow 的 docker 容器内运行 Airflow 任务，该容器在 mesos slave 上运行。

## 任务直接在 mesos slave 上执行

`MesosExecutor`允许您在 Mesos 群集上调度 Airflow 任务。为此，您需要一个正在运行的 mesos 集群，并且必须执行以下步骤 -

1. 在可以运行 web server 和 scheduler 的 mesos slave 上安装 Airflow，让我们将其称为“Airflow server”。
2. 在 Airflow server 上，从[mesos 下载](http://open.mesosphere.com/downloads/mesos/)安装 mesos python eggs。
3. 在 Airflow server 上，使用可以被所有 mesos slave 访问的数据库（例如 mysql），并在`airflow.cfg`添加配置。
4. 将您的`airflow.cfg`里 executor 的参数指定为*MesosExecutor*，并提供相关的 Mesos 设置。
5. 在所有 mesos slave 上，安装 Airflow。 从 Airflow 服务器复制`airflow.cfg` （以便它使用相同的 sqlalchemy 连接）。
6. 在所有 mesos slave 上，运行以下服务日志：

```py
airflow serve_logs
```

7. 在 Airflow server 上，如果想要开始在 mesos 上处理/调度 DAG，请运行：

```py
airflow scheduler -p
```

注意：我们需要-p 参数来挑选 DAG。

您现在可以在 mesos UI 中查看 Airflow 框架和相应的任务。Airflow 任务的日志可以像往常一样在 Airflow UI 中查看。

有关 mesos 的更多信息，请参阅[mesos 文档](http://mesos.apache.org/documentation/latest/)。 有关 MesosExecutor 的任何疑问/错误，请联系[@kapil-malik](https://github.com/kapil-malik)。

## 在 mesos slave 上的容器中执行的任务

[此 gist](https://gist.github.com/sebradloff/f158874e615bda0005c6f4577b20036e)包含实现以下所需的所有文件和配置更改：

1. 使用安装了 mesos python eggs 的软件环境创建一个 dockerized 版本的 Airflow。

> 我们建议利用 docker 的多阶段构建来实现这一目标。我们有一个 Dockerfile 定义从源（Dockerfile-mesos）构建特定版本的 mesos，以便创建 python eggs。在 Airflow Dockerfile（Dockerfile-airflow）中，我们从 mesos 镜像中复制 python eggs。

2. 在`airflow.cfg`创建一个 mesos 配置块。

> 配置块保持与默认 Airflow 配置（default_airflow.cfg）相同，但添加了一个选项`docker_image_slave`。 这应该设置为您希望 mesos 在运行 Airflow 任务时使用的镜像的名称。确保您具有适用于您的 mesos 主服务器的 DNS 记录的正确配置以及任何类型的授权（如果存在）。

3. 更改`airflow.cfg`以将执行程序参数指向 MesosExecutor（executor = SequentialExecutor）。

4. 确保您的 mesos slave 可以访问您`docker_image_slave`的 docker 存储库。

> [mesos 文档中提供了相关说明。](https://mesos.readthedocs.io/en/latest/docker-containerizer/)

其余部分取决于您以及您希望如何使用 dockerized Airflow 配置。
