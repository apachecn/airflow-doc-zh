# 调度和触发器

> 贡献者：[@Ray](https://github.com/echo-ray)

Airflow 调度程序监视所有任务和所有 DAG，并触发已满足其依赖关系的任务实例。 在幕后，它监视并与其可能包含的所有 DAG 对象的文件夹保持同步，并定期（每分钟左右）检查活动任务以查看是否可以触发它们。

Airflow 调度程序旨在作为 Airflow 生产环境中的持久服务运行。 要开始，您需要做的就是执行`airflow scheduler` 。 它将使用`airflow.cfg`指定的配置。

请注意，如果您在一天的`schedule_interval`上运行 DAG，则会在`2016-01-01T23:59`之后不久触发标记为`2016-01-01`的运行。 换句话说，作业实例在其覆盖的时间段结束后启动。
请注意，如果您运行一个`schedule_interval`为 1 天的的`DAG`，`run`标记为`2016-01-01`，那么它会在`2016-01-01T23:59`之后马上触发。换句话说，一旦设定的时间周期结束后，工作实例将立马开始。

**让我们重复一遍**调度`schedule_interval`在开始日期之后，在句点结束时运行您的作业一个`schedule_interval` 。

调度程序启动`airflow.cfg`指定的执行程序的实例。 如果碰巧是`LocalExecutor` ，任务将作为子`LocalExecutor`执行; 在`CeleryExecutor`和`MesosExecutor`的情况下，任务是远程执行的。

要启动调度程序，只需运行以下命令：

```py
airflow scheduler
```

## DAG 运行

DAG Run 是一个表示 DAG 实例化的对象。

每个 DAG 可能有也可能没有时间表，通知如何创建`DAG Runs` 。 `schedule_interval`被定义为 DAG 参数，并且优选地接收作为`str`的[cron 表达式](https://en.wikipedia.org/wiki/Cron)或`datetime.timedelta`对象。 或者，您也可以使用其中一个 cron“预设”：

<colgroup><col width="15%"><col width="69%"><col width="16%"></colgroup>
| 预置 | 含义 | cron 的 |
| --- | --- | --- |
| `None` | 不要安排，专门用于“外部触发”的 DAG |  |
| `@once` | 安排一次，只安排一次 |  |
| `@hourly` | 在小时开始时每小时运行一次 | `0 * * * *` |
| `@daily` | 午夜一天运行一次 | `0 0 * * *` |
| `@weekly` | 周日早上每周午夜运行一次 | `0 0 * * 0` |
| `@monthly` | 每个月的第一天午夜运行一次 | `0 0 1 * *` |
| `@yearly` | 每年 1 月 1 日午夜运行一次 | `0 0 1 1 *` |

您的 DAG 将针对每个计划进行实例化，同时为每个计划创建`DAG Run`条目。

DAG 运行具有与它们相关联的状态（运行，失败，成功），并通知调度程序应该针对任务提交评估哪组调度。 如果没有 DAG 运行级别的元数据，Airflow 调度程序将需要做更多的工作才能确定应该触发哪些任务并进行爬行。 在更改 DAG 的形状时，也可能会添加新任务，从而创建不需要的处理。

## 回填和追赶

具有`start_date` （可能是`end_date` ）和`schedule_interval`的 Airflow DAG 定义了一系列间隔，调度程序将这些间隔转换为单独的 Dag 运行并执行。 Airflow 的一个关键功能是这些 DAG 运行是原子的幂等项，默认情况下，调度程序将检查 DAG 的生命周期（从开始到结束/现在，一次一个间隔）并启动 DAG 运行对于尚未运行（或已被清除）的任何间隔。 这个概念叫做 Catchup。

如果你的 DAG 被编写来处理它自己的追赶（IE 不仅限于间隔，而是改为“现在”。），那么你将需要关闭追赶（在 DAG 本身上使用`dag.catchup = False` ）或者默认情况下在配置文件级别使用`catchup_by_default = False` 。 这样做，是指示调度程序仅为 DAG 间隔序列的最新实例创建 DAG 运行。

```py
"""
Code that goes along with the Airflow tutorial located at:
https://github.com/airbnb/airflow/blob/master/airflow/example_dags/tutorial.py
"""
from airflow import DAG
from airflow.operators.bash_operator import BashOperator
from datetime import datetime, timedelta


default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'start_date': datetime(2015, 12, 1),
    'email': ['airflow@example.com'],
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
    'schedule_interval': '@hourly',
}

dag = DAG('tutorial', catchup=False, default_args=default_args)
```

在上面的示例中，如果调度程序守护程序在 2016-01-02 上午 6 点（或从命令行）拾取 DAG，则将创建单个 DAG 运行，其`execution_date`为 2016-01-01 ，下一个将在 2016-01-03 上午午夜后创建，执行日期为 2016-01-02。

如果`dag.catchup`值为 True，则调度程序将为 2015-12-01 和 2016-01-02 之间的每个完成时间间隔创建一个 DAG Run（但不是 2016-01-02 中的一个，因为该时间间隔）尚未完成）并且调度程序将按顺序执行它们。 对于可以轻松拆分为句点的原子数据集，此行为非常有用。 如果您的 DAG 运行在内部执行回填，则关闭追赶是很好的。

## 外部触发器

请注意，在运行`airflow trigger_dag`命令时，也可以通过 CLI 手动创建`DAG Runs` ，您可以在其中定义特定的`run_id` 。 在调度程序外部创建的`DAG Runs`与触发器的时间戳相关联，并将与预定的`DAG runs`一起显示在 UI 中。

此外，您还可以使用 Web UI 手动触发`DAG Run` （选项卡“DAG” - &gt;列“链接” - &gt;按钮“触发器 Dag”）。

## 要牢记

* 第一个`DAG Run`是基于 DAG 中任务的最小`start_date`创建的。
* 后续`DAG Runs`由调度程序进程根据您的 DAG 的`schedule_interval`顺序创建。
* 当清除一组任务的状态以期让它们重新运行时，重要的是要记住`DAG Run`的状态，因为它定义了调度程序是否应该查看该运行的触发任务。

以下是一些可以**取消阻止任务的方法** ：

* 在 UI 中，您可以从任务实例对话框中**清除** （如删除状态）各个任务实例，同时定义是否要包括过去/未来和上游/下游依赖项。 请注意，接下来会出现一个确认窗口，您可以看到要清除的设置。 您还可以清除与 dag 关联的所有任务实例。
* CLI 命令`airflow clear -h`在清除任务实例状态时有很多选项，包括指定日期范围，通过指定正则表达式定位 task_ids，包含上游和下游亲属的标志，以及特定状态下的目标任务实例（ `failed`或`success` ）
* 清除任务实例将不再删除任务实例记录。 相反，它更新 max_tries 并将当前任务实例状态设置为 None。
* 将任务实例标记为失败可以通过 UI 完成。 这可用于停止运行任务实例。
* 将任务实例标记为成功可以通过 UI 完成。 这主要是为了修复漏报，或者例如在 Airflow 之外应用修复时。
* `airflow backfill` CLI 子命令具有`--mark_success`标志，允许选择 DAG 的子部分以及指定日期范围。
