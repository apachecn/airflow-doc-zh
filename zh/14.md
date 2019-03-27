# 使用 Dask 扩展

> 贡献者：[@ImPerat0R\_](https://github.com/tssujt)、[@ThinkingChen](https://github.com/cdmikechen)

`DaskExecutor`允许您在 Dask 分布式集群中运行 Airflow 任务。

Dask 集群可以在单个机器上运行，也可以在远程网络上运行。有关完整详细信息，请参阅[分布式文档](https://distributed.readthedocs.io/) 。

要创建集群，首先启动一个 Scheduler：

```py
# 一个本地集群的默认设置
DASK_HOST=127.0.0.1
DASK_PORT=8786

dask-scheduler --host $DASK_HOST --port $DASK_PORT
```

接下来，在任何可以连接到主机的计算机上启动至少一个 Worker：

```py
dask-worker $DASK_HOST:$DASK_PORT
```

编辑`airflow.cfg`以将执行程序设置为`DaskExecutor`并在`[dask]`部分中提供 Dask Scheduler 地址。

请注意：

* 每个 Dask worker 必须能够导入 Airflow 和您需要的任何依赖项。
* Dask 不支持队列。如果使用队列创建了 Airflow 任务，则会引发警告，但该任务会被提交给集群。
