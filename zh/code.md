# API 参考

## 运营商

运算符允许生成某些类型的任务，这些任务在实例化时成为 DAG 中的节点。 所有运算符都派生自`BaseOperator` ，并以这种方式继承许多属性和方法。 有关更多详细信息，请参阅[BaseOperator](31)文档。

有三种主要类型的运营商：

*   执行操作的操作员，或告诉其他系统执行操作的操作员
*   **传输**操作员将数据从一个系统移动到另一个系
*   **传感器**是某种类型的运算符，它将一直运行直到满足某个标准。 示例包括在 HDFS 或 S3 中登陆的特定文件，在 Hive 中显示的分区或当天的特定时间。 传感器派生自`BaseSensorOperator`并在指定的`poke_interval`运行 poke 方法，直到它返回`True` 。

### BaseOperator

所有运算符都派生自`BaseOperator`并通过继承获得许多功能。 由于这是引擎的核心，因此值得花时间了解`BaseOperator`的参数，以了解可在 DAG 中使用的原始功能。

```py
class airflow.models.BaseOperator(task_id, owner='Airflow', email=None, email_on_retry=True, email_on_failure=True, retries=0, retry_delay=datetime.timedelta(0, 300), retry_exponential_backoff=False, max_retry_delay=None, start_date=None, end_date=None, schedule_interval=None, depends_on_past=False, wait_for_downstream=False, dag=None, params=None, default_args=None, adhoc=False, priority_weight=1, weight_rule=u'downstream', queue='default', pool=None, sla=None, execution_timeout=None, on_failure_callback=None, on_success_callback=None, on_retry_callback=None, trigger_rule=u'all_success', resources=None, run_as_user=None, task_concurrency=None, executor_config=None, inlets=None, outlets=None, *args, **kwargs) 
```

基类： `airflow.utils.log.logging_mixin.LoggingMixin`

所有运营商的抽象基类。 由于运算符创建的对象成为 dag 中的节点，因此 BaseOperator 包含许多用于 dag 爬行行为的递归方法。 要派生此类，您需要覆盖构造函数以及“execute”方法。

从此类派生的运算符应同步执行或触发某些任务（等待完成）。 运算符的示例可以是运行 Pig 作业（PigOperator）的运算符，等待分区在 Hive（HiveSensorOperator）中着陆的传感器运算符，或者是将数据从 Hive 移动到 MySQL（Hive2MySqlOperator）的运算符。 这些运算符（任务）的实例针对特定操作，运行特定脚本，函数或数据传输。

此类是抽象的，不应实例化。 实例化从这个派生的类导致创建任务对象，该对象最终成为 DAG 对象中的节点。 应使用 set_upstream 和/或 set_downstream 方法设置任务依赖性。

参数：

*   `task_id( string )` - 任务的唯一，有意义的 id
*   `owner( string )` - 使用 unix 用户名的任务的所有者
*   `retries( int )` - 在失败任务之前应执行的重试次数
*   `retry_delay( timedelta )` - 重试之间的延迟
*   `retry_exponential_backoff( bool )` - 允许在重试之间使用指数退避算法在重试之间进行更长时间的等待（延迟将转换为秒）
*   `max_retry_delay( timedelta )` - 重试之间的最大延迟间隔
*   `start_date( datetime )` - 任务的`start_date` ，确定第一个任务实例的`execution_date` 。 最佳做法是将 start_date 四舍五入到 DAG 的`schedule_interval` 。 每天的工作在 00:00:00 有一天的 start_date，每小时的工作在特定时间的 00:00 有 start_date。 请注意，Airflow 只查看最新的`execution_date`并添加`schedule_interval`以确定下一个`execution_date` 。 同样非常重要的是要注意不同任务的依赖关系需要及时排列。 如果任务 A 依赖于任务 B 并且它们的 start_date 以其 execution_date 不排列的方式偏移，则永远不会满足 A 的依赖性。 如果您希望延迟任务，例如在凌晨 2 点运行每日任务，请查看`TimeSensor`和`TimeDeltaSensor` 。 我们建议不要使用动态`start_date`并建议使用固定的。 有关更多信息，请阅读有关 start_date 的 FAQ 条目。
*   `end_date( datetime )` - 如果指定，则调度程序不会超出此日期
*   `depends_on_past( bool )` - 当设置为 true 时，任务实例将依次运行，同时依赖上一个任务的计划成功。 允许 start_date 的任务实例运行。
*   `wait_for_downstream( bool )` - 当设置为 true 时，任务 X 的实例将等待紧接上一个任务 X 实例下游的任务在运行之前成功完成。 如果任务 X 的不同实例更改同一资产，并且任务 X 的下游任务使用此资产，则此操作非常有用。请注意，在使用 wait_for_downstream 的任何地方，depends_on_past 都将强制为 True。
*   `queue( str )` - 运行此作业时要定位到哪个队列。 并非所有执行程序都实现队列管理，CeleryExecutor 确实支持定位特定队列。
*   `dag( DAG )` - 对任务所附的 dag 的引用（如果有的话）
*   `priority_weight( int )` - 此任务相对于其他任务的优先级权重。 这允许执行程序在事情得到备份时在其他任务之前触发更高优先级的任务。
*   `weight_rule( str )` - 用于任务的有效总优先级权重的加权方法。 选项包括： `{ downstream &#124; upstream &#124; absolute }` `{ downstream &#124; upstream &#124; absolute }` `{ downstream &#124; upstream &#124; absolute }`默认为`downstream`当设置为`downstream`时，任务的有效权重是所有下游后代的总和。 因此，上游任务将具有更高的权重，并且在使用正权重值时将更积极地安排。 当您有多个 dag 运行实例并希望在每个 dag 可以继续处理下游任务之前完成所有运行的所有上游任务时，这非常有用。 设置为`upstream` ，有效权重是所有上游祖先的总和。 这与 downtream 任务具有更高权重并且在使用正权重值时将更积极地安排相反。 当你有多个 dag 运行实例并希望在启动其他 dag 的上游任务之前完成每个 dag 时，这非常有用。 设置为`absolute` ，有效权重是指定的确切`priority_weight`无需额外加权。 当您确切知道每个任务应具有的优先级权重时，您可能希望这样做。 此外，当设置为`absolute` ，对于非常大的 DAGS，存在显着加速任务创建过程的额外效果。 可以将选项设置为字符串或使用静态类`airflow.utils.WeightRule`定义的常量
*   `pool( str )` - 此任务应运行的插槽池，插槽池是限制某些任务的并发性的一种方法
*   `sla( datetime.timedelta )` - 作业预期成功的时间。 请注意，这表示期间结束后的`timedelta` 。 例如，如果您将 SLA 设置为 1 小时，则如果`2016-01-01`实例尚未成功，则调度程序将在`2016-01-02`凌晨 1:00 之后发送电子邮件。 调度程序特别关注具有 SLA 的作业，并发送警报电子邮件以防止未命中。 SLA 未命中也记录在数据库中以供将来参考。 共享相同 SLA 时间的所有任务都捆绑在一封电子邮件中，在此之后很快就会发送。 SLA 通知仅为每个任务实例发送一次且仅一次。
*   `execution_timeout( datetime.timedelta )` - 执行此任务实例所允许的最长时间，如果超出它将引发并失败。
*   `on_failure_callback( callable )` - 当此任务的任务实例失败时要调用的函数。 上下文字典作为单个参数传递给此函数。 Context 包含对任务实例的相关对象的引用，并记录在 API 的宏部分下。
*   `on_retry_callback` - 与`on_failure_callback`非常相似，只是在重试发生时执行。
*   `on_success_callback( callable )` - 与`on_failure_callback`非常相似，只是在任务成功时执行。
*   `trigger_rule( str )` - 定义应用依赖项以触发任务的规则。 选项包括： `{ all_success &#124; all_failed &#124; all_done &#124; one_success &#124; one_failed &#124; dummy}` `{ all_success &#124; all_failed &#124; all_done &#124; one_success &#124; one_failed &#124; dummy}` `{ all_success &#124; all_failed &#124; all_done &#124; one_success &#124; one_failed &#124; dummy}`默认为`all_success` 。 可以将选项设置为字符串，也可以使用静态类`airflow.utils.TriggerRule`定义的常量
*   `resources( dict )` - 资源参数名称（Resources 构造函数的参数名称）与其值的映射。
*   `run_as_user( str )` - 在运行任务时使用 unix 用户名进行模拟
*   `task_concurrency( int )` - 设置后，任务将能够限制 execution_dates 之间的并发运行
*   `executor_config( dict )` -

    由特定执行程序解释的其他任务级配置参数。 参数由 executor 的名称命名。 [``](31)示例：通过 KubernetesExecutor MyOperator（...，在特定的 docker 容器中运行此任务）

    &gt; executor_config = {“KubernetesExecutor”：
    &gt; 
    &gt; &gt; {“image”：“myCustomDockerImage”}}

    ）``

```py
clear(**kwargs) 
```

根据指定的参数清除与任务关联的任务实例的状态。

```py
dag 
```

如果设置则返回运算符的 DAG，否则引发错误

```py
deps 
```

返回运算符的依赖项列表。 这些与执行上下文依赖性的不同之处在于它们特定于任务，并且可以由子类扩展/覆盖。

```py
downstream_list 
```

@property：直接下游的任务列表

```py
execute(context) 
```

这是在创建运算符时派生的主要方法。 Context 与渲染 jinja 模板时使用的字典相同。

有关更多上下文，请参阅 get_template_context。

```py
get_direct_relative_ids(upstream=False) 
```

获取当前任务（上游或下游）的直接相对 ID。

```py
get_direct_relatives(upstream=False) 
```

获取当前任务的直接亲属，上游或下游。

```py
get_flat_relative_ids(upstream=False, found_descendants=None) 
```

获取上游或下游的亲属 ID 列表。

```py
get_flat_relatives(upstream=False) 
```

获取上游或下游亲属的详细列表。

```py
get_task_instances(session, start_date=None, end_date=None) 
```

获取与特定日期范围的此任务相关的一组任务实例。

```py
has_dag() 
```

如果已将运算符分配给 DAG，则返回 True。

```py
on_kill() 
```

当任务实例被杀死时，重写此方法以清除子进程。 在操作员中使用线程，子过程或多处理模块的任何使用都需要清理，否则会留下重影过程。

```py
post_execute(context, *args, **kwargs) 
```

调用 self.execute（）后立即触发此挂接。 它传递执行上下文和运算符返回的任何结果。

```py
pre_execute(context, *args, **kwargs) 
```

在调用 self.execute（）之前触发此挂钩。

```py
prepare_template() 
```

模板化字段被其内容替换后触发的挂钩。 如果您需要操作员在呈现模板之前更改文件的内容，则应覆盖此方法以执行此操作。

```py
render_template(attr, content, context) 
```

从文件或直接在字段中呈现模板，并返回呈现的结果。

```py
render_template_from_field(attr, content, context, jinja_env) 
```

从字段中呈现模板。 如果字段是字符串，它将只是呈现字符串并返回结果。 如果它是集合或嵌套的集合集，它将遍历结构并呈现其中的所有字符串。

```py
run(start_date=None, end_date=None, ignore_first_depends_on_past=False, ignore_ti_state=False, mark_success=False) 
```

为日期范围运行一组任务实例。

```py
schedule_interval 
```

DAG 的计划间隔始终胜过单个任务，因此 DAG 中的任务始终排列。 该任务仍然需要 schedule_interval，因为它可能未附加到 DAG。

```py
set_downstream(task_or_task_list) 
```

将任务或任务列表设置为直接位于当前任务的下游。

```py
set_upstream(task_or_task_list) 
```

将任务或任务列表设置为直接位于当前任务的上游。

```py
upstream_list 
```

@property：直接上游的任务列表

```py
xcom_pull(context, task_ids=None, dag_id=None, key=u'return_value', include_prior_dates=None) 
```

请参见 TaskInstance.xcom_pull（）

```py
xcom_push(context, key, value, execution_date=None) 
```

请参见 TaskInstance.xcom_push（）

### BaseSensorOperator

所有传感器均来自`BaseSensorOperator` 。 所有传感器都在`BaseOperator`属性之上继承`timeout`和`poke_interval` 。

```py
class airflow.sensors.base_sensor_operator.BaseSensorOperator(poke_interval=60, timeout=604800, soft_fail=False, *args, **kwargs) 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator") ， `airflow.models.SkipMixin`

传感器操作符派生自此类继承这些属性。

```py
 Sensor operators keep executing at a time interval and succeed when 
```

如果标准超时，则会达到并且失败。

参数：

*   `soft_fail( bool )` - 设置为 true 以在失败时将任务标记为 SKIPPED
*   `poke_interval( int )` - 作业在每次尝试之间应等待的时间（以秒为单位）
*   `timeout( int )` - 任务超时和失败前的时间（以秒为单位）。

```py
poke(context) 
```

传感器在派生此类时定义的功能应该覆盖。

### 核心运营商

#### 运营商

```py
class airflow.operators.bash_operator.BashOperator(bash_command, xcom_push=False, env=None, output_encoding='utf-8', *args, **kwargs) 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

执行 Bash 脚本，命令或命令集。

参数：

*   `bash_command( string )` - 要执行的命令，命令集或对 bash 脚本（必须为'.sh'）的引用。 （模板）
*   `xcom_push( bool )` - 如果 xcom_push 为 True，写入 stdout 的最后一行也将在 bash 命令完成时被推送到 XCom。
*   `env( dict )` - 如果 env 不是 None，它必须是一个定义新进程的环境变量的映射; 这些用于代替继承当前进程环境，这是默认行为。 （模板）

```py
execute(context) 
```

在临时目录中执行 bash 命令，之后将对其进行清理

```py
class airflow.operators.python_operator.BranchPythonOperator(python_callable, op_args=None, op_kwargs=None, provide_context=False, templates_dict=None, templates_exts=None, *args, **kwargs) 
```

基类： [`airflow.operators.python_operator.PythonOperator`](31 "airflow.operators.python_operator.PythonOperator") ， `airflow.models.SkipMixin`

允许工作流在执行此任务后“分支”或遵循单个路径。

它派生 PythonOperator 并期望一个返回 task_id 的 Python 函数。 返回的 task_id 应指向{self}下游的任务。 所有其他“分支”或直接下游任务都标记为`skipped`状态，以便这些路径不能向前移动。 `skipped`状态在下游被提出以允许 DAG 状态填满并且推断 DAG 运行的状态。

请注意，在`depends_on_past=True`中使用`depends_on_past=True`下游的任务在逻辑上是不合理的，因为`skipped`状态将总是导致依赖于过去成功的块任务。 `skipped`状态在所有直接上游任务被`skipped`地方传播。

```py
class airflow.operators.check_operator.CheckOperator(sql, conn_id=None, *args, **kwargs) 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

对 db 执行检查。 `CheckOperator`需要一个返回单行的 SQL 查询。 第一行的每个值都使用 python `bool` cast 进行计算。 如果任何值返回`False`则检查失败并输出错误。

请注意，Python bool cast 会将以下内容视为`False` ：

*   `False`
*   `0`
*   空字符串（ `""` ）
*   空列表（ `[]` ）
*   空字典或集（ `{}` ）

给定`SELECT COUNT(*) FROM foo`类的查询，只有当 count `== 0`它才会失败。 您可以制作更复杂的查询，例如，可以检查表与上游源表的行数相同，或者今天的分区计数大于昨天的分区，或者一组指标是否更少 7 天平均值超过 3 个标准差。

此运算符可用作管道中的数据质量检查，并且根据您在 DAG 中的位置，您可以选择停止关键路径，防止发布可疑数据，或者侧面接收电子邮件警报阻止 DAG 的进展。

请注意，这是一个抽象类，需要定义 get_db_hook。 而 get_db_hook 是钩子，它从外部源获取单个记录。

参数：`sql( string )` - 要执行的 sql。 （模板）

```py
class airflow.operators.docker_operator.DockerOperator(image, api_version=None, command=None, cpus=1.0, docker_url='unix://var/run/docker.sock', environment=None, force_pull=False, mem_limit=None, network_mode=None, tls_ca_cert=None, tls_client_cert=None, tls_client_key=None, tls_hostname=None, tls_ssl_version=None, tmp_dir='/tmp/airflow', user=None, volumes=None, working_dir=None, xcom_push=False, xcom_all=False, docker_conn_id=None, *args, **kwargs) 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在 docker 容器中执行命令。

在主机上创建临时目录并将其装入容器，以允许在容器中存储超过默认磁盘大小 10GB 的文件。 可以通过环境变量`AIRFLOW_TMP_DIR`访问已安装目录的路径。

如果在提取映像之前需要登录私有注册表，则需要在 Airflow 中配置 Docker 连接，并使用参数`docker_conn_id`提供连接 ID。

参数：

*   `image( str )` - 用于创建容器的 Docker 镜像。
*   `api_version( str )` - 远程 API 版本。 设置为`auto`以自动检测服务器的版本。
*   `command( str 或 list )` - 要在容器中运行的命令。 （模板）
*   `cpus( float )` - 分配给容器的 CPU 数。 此值乘以 1024.请参阅[https://docs.docker.com/engine/reference/run/#cpu-share-constraint](https://docs.docker.com/engine/reference/run/)
*   `docker_url( str )` - 运行**docker**守护程序的主机的 URL。 默认为 unix：//var/run/docker.sock
*   `environment( dict )` - 要在容器中设置的环境变量。 （模板）
*   `force_pull( bool )` - 每次运行时拉动泊坞窗图像。 默认值为 false。
*   `mem_limit( float 或 str )` - 容器可以使用的最大内存量。 浮点值（表示以字节为单位的限制）或字符串（如`128m`或`1g` 。
*   `network_mode( str )` - 容器的网络模式。
*   `tls_ca_cert( str )` - 用于保护 docker 连接的 PEM 编码证书颁发机构的路径。
*   `tls_client_cert( str )` - 用于验证 docker 客户端的 PEM 编码证书的路径。
*   `tls_client_key( str )` - 用于验证 docker 客户端的 PEM 编码密钥的路径。
*   `tls_hostname( str 或 bool )` - 与**docker**服务器证书匹配的主机名或 False 以禁用检查。
*   `tls_ssl_version( str )` - 与 docker 守护程序通信时使用的 SSL 版本。
*   `tmp_dir( str )` - 将容器内的挂载点挂载到操作员在主机上创建的临时目录。 该路径也可通过容器内的环境变量`AIRFLOW_TMP_DIR`获得。
*   `user( int 或 str )` - docker 容器内的默认用户。
*   `卷` - 要装入容器的卷列表，例如`['/host/path:/container/path', '/host/path2:/container/path2:ro']` 。
*   `working_dir( str )` - 在容器上设置的工作目录（相当于-w 切换 docker 客户端）
*   `xcom_push( bool )` - 是否会使用 XCom 将 stdout 推送到下一步。 默认值为 False。
*   `xcom_all( bool )` - 推送所有标准输出或最后一行。 默认值为 False（最后一行）。
*   `docker_conn_id( str )` - 要使用的 Airflow 连接的 ID

```py
class airflow.operators.dummy_operator.DummyOperator(*args, **kwargs) 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

操作员确实没什么。 它可用于对 DAG 中的任务进行分组。

```py
class airflow.operators.druid_check_operator.DruidCheckOperator(sql, druid_broker_conn_id='druid_broker_default', *args, **kwargs) 
```

基类： [`airflow.operators.check_operator.CheckOperator`](31 "airflow.operators.check_operator.CheckOperator")

对德鲁伊进行检查。 `DruidCheckOperator`需要一个返回单行的 SQL 查询。 第一行的每个值都使用 python `bool` cast 进行计算。 如果任何值返回`False`则检查失败并输出错误。

请注意，Python bool cast 会将以下内容视为`False` ：

*   `False`
*   `0`
*   空字符串（ `""` ）
*   空列表（ `[]` ）
*   空字典或集（ `{}` ）

给定`SELECT COUNT(*) FROM foo`类的查询，只有当 count `== 0`它才会失败。 您可以制作更复杂的查询，例如，可以检查表与上游源表的行数相同，或者今天的分区计数大于昨天的分区，或者一组指标是否更少 7 天平均值超过 3 个标准差。 此运算符可用作管道中的数据质量检查，并且根据您在 DAG 中的位置，您可以选择停止关键路径，防止发布可疑数据，或者在旁边接收电子邮件替代品阻止 DAG 的进展。

参数：

*   `sql( string )` - 要执行的 sql
*   `druid_broker_conn_id( string )` - 对德鲁伊经纪人的提及

```py
get_db_hook() 
```

返回德鲁伊 db api 钩子。

```py
get_first(sql) 
```

执行德鲁伊 sql 到德鲁伊经纪人并返回第一个结果行。

参数：`sql( str )` - 要执行的 sql 语句（str）

```py
class airflow.operators.email_operator.EmailOperator(to, subject, html_content, files=None, cc=None, bcc=None, mime_subtype='mixed', mime_charset='us_ascii', *args, **kwargs) 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

发送电子邮件。

参数：

*   `to( list 或 string （ 逗号 或 分号分隔 ） )` - 要发送电子邮件的电子邮件列表。 （模板）
*   `subject( string )` - 电子邮件的主题行。 （模板）
*   `html_content( string )` - 电子邮件的内容，允许使用 html 标记。 （模板）
*   `files( list )` - 要在电子邮件中附加的文件名
*   `cc( list 或 string _（ 逗号 或 分号分隔 ） )` - 要在 CC 字段中添加的收件人列表
*   `密件抄送( list 或 string （ 逗号 或 分号分隔 ） )` - 要在 BCC 字段中添加的收件人列表
*   `mime_subtype( string )` - MIME 子内容类型
*   `mime_charset( string )` - 添加到 Content-Type 标头的字符集参数。

```py
class airflow.operators.generic_transfer.GenericTransfer(sql, destination_table, source_conn_id, destination_conn_id, preoperator=None, *args, **kwargs) 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

将数据从连接移动到另一个连接，假设它们都在各自的钩子中提供所需的方法。 源钩子需要公开`get_records`方法，目标是`insert_rows`方法。

这适用于适合内存的小型数据集。

参数：

*   `sql( str )` - 针对源数据库执行的 SQL 查询。 （模板）
*   `destination_table( str )` - 目标表。 （模板）
*   `source_conn_id( str )` - 源连接
*   `destination_conn_id( str )` - 源连接
*   `preoperator( str 或 list[str] )` - sql 语句或在加载数据之前要执行的语句列表。 （模板）

```py
class airflow.operators.hive_to_druid.HiveToDruidTransfer(sql, druid_datasource, ts_dim, metric_spec=None, hive_cli_conn_id='hive_cli_default', druid_ingest_conn_id='druid_ingest_default', metastore_conn_id='metastore_default', hadoop_dependency_coordinates=None, intervals=None, num_shards=-1, target_partition_size=-1, query_granularity='NONE', segment_granularity='DAY', hive_tblproperties=None, *args, **kwargs) 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

将数据从 Hive 移动到 Druid，[del]注意到现在数据在被推送到 Druid 之前被加载到内存中，因此该运算符应该用于少量数据。[/ del]

参数：

*   `sql( str )` - 针对 Druid 数据库执行的 SQL 查询。 （模板）
*   `druid_datasource( str )` - 您想要在德鲁伊中摄取的数据源
*   `ts_dim( str )` - 时间戳维度
*   `metric_spec( list )` - 您要为数据定义的指标
*   `hive_cli_conn_id( str )` - 配置单元连接 ID
*   `druid_ingest_conn_id( str )` - 德鲁伊摄取连接 ID
*   `metastore_conn_id( str )` - Metastore 连接 ID
*   `hadoop_dependency_coordinates( list[str] )` - 用于挤压摄取 json 的坐标列表
*   `interval( list )` - 定义段的时间间隔列表，按原样传递给 json 对象。 （模板）
*   `hive_tblproperties( dict )` - 用于登台表的 hive 中 tblproperties 的其他属性

```py
construct_ingest_query(static_path, columns) 
```

构建 HDFS TSV 负载的接收查询。

参数：

*   `static_path( str )` - 数据所在的 hdfs 上的路径
*   `columns( list )` - 所有可用列的列表

```py
class airflow.operators.hive_to_mysql.HiveToMySqlTransfer(sql, mysql_table, hiveserver2_conn_id='hiveserver2_default', mysql_conn_id='mysql_default', mysql_preoperator=None, mysql_postoperator=None, bulk_load=False, *args, **kwargs) 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

将数据从 Hive 移动到 MySQL，请注意，目前数据在被推送到 MySQL 之前已加载到内存中，因此该运算符应该用于少量数据。

参数：

*   `sql( str )` - 对 Hive 服务器执行的 SQL 查询。 （模板）
*   `mysql_table( str )` - 目标 MySQL 表，使用点表示法来定位特定数据库。 （模板）
*   `mysql_conn_id( str )` - 源码 mysql 连接
*   `hiveserver2_conn_id( str )` - 目标配置单元连接
*   `mysql_preoperator( str )` - 在导入之前对 mysql 运行的 sql 语句，通常用于截断删除代替进入的数据，允许任务是幂等的（运行任务两次不会加倍数据）。 （模板）
*   `mysql_postoperator( str )` - 导入后针对 mysql 运行的 sql 语句，通常用于将数据从登台移动到生产并发出清理命令。 （模板）
*   `bulk_load( bool )` - 使用 bulk_load 选项的标志。 这使用 LOAD DATA LOCAL INFILE 命令直接从制表符分隔的文本文件加载 mysql。 此选项需要为目标 MySQL 连接提供额外的连接参数：{'local_infile'：true}。

```py
class airflow.operators.hive_to_samba_operator.Hive2SambaOperator(hql, destination_filepath, samba_conn_id='samba_default', hiveserver2_conn_id='hiveserver2_default', *args, **kwargs) 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在特定的 Hive 数据库中执行 hql 代码，并将查询结果作为 csv 加载到 Samba 位置。

参数：

*   `hql( string )` - 要导出的 hql。 （模板）
*   `hiveserver2_conn_id( string )` - 对 hiveserver2 服务的引用
*   `samba_conn_id( string )` - 对 samba 目标的引用

```py
class airflow.operators.hive_operator.HiveOperator(hql, hive_cli_conn_id=u'hive_cli_default', schema=u'default', hiveconfs=None, hiveconf_jinja_translate=False, script_begin_tag=None, run_as_owner=False, mapred_queue=None, mapred_queue_priority=None, mapred_job_name=None, *args, **kwargs) 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在特定的 Hive 数据库中执行 hql 代码或 hive 脚本。

参数：

*   `hql( string )` - 要执行的 hql。 请注意，您还可以使用（模板）配置单元脚本的 dag 文件中的相对路径。 （模板）
*   `hive_cli_conn_id( string )` - 对 Hive 数据库的引用。 （模板）
*   `hiveconfs( dict )` - 如果已定义，这些键值对将作为`-hiveconf "key"="value"`传递给 hive
*   `hiveconf_jinja_translate( _boolean_ )` - 当为 True 时，hiveconf-type 模板$ {var}被翻译成 jinja-type templating {{var}}，$ {hiveconf：var}被翻译成 jinja-type templating {{var}}。 请注意，您可能希望将此选项与`DAG(user_defined_macros=myargs)`参数一起使用。 查看 DAG 对象文档以获取更多详细信息。
*   `script_begin_tag( str )` - 如果已定义，运算符将在第一次出现`script_begin_tag`之前删除脚本的一部分
*   `mapred_queue( string )` - Hadoop CapacityScheduler 使用的队列。 （模板）
*   `mapred_queue_priority( string )` - CapacityScheduler 队列中的优先级。 可能的设置包括：VERY_HIGH，HIGH，NORMAL，LOW，VERY_LOW
*   `mapred_job_name( string )` - 此名称将出现在 jobtracker 中。 这可以使监控更容易。

```py
class airflow.operators.hive_stats_operator.HiveStatsCollectionOperator(table, partition, extra_exprs=None, col_blacklist=None, assignment_func=None, metastore_conn_id='metastore_default', presto_conn_id='presto_default', mysql_conn_id='airflow_db', *args, **kwargs) 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

使用动态生成的 Presto 查询收集分区统计信息，将统计信息插入到具有此格式的 MySql 表中。 如果重新运行相同的日期/分区，统计信息将覆盖自己。

```py
 CREATE TABLE hive_stats (
    ds VARCHAR ( 16 ),
    table_name VARCHAR ( 500 ),
    metric VARCHAR ( 200 ),
    value BIGINT
);

```

参数：

*   `table( str )` - 源表，格式为`database.table_name` 。 （模板）
*   `partition( dict{col:value} )` - 源分区。 （模板）
*   `extra_exprs( dict )` - 针对表运行的表达式，其中键是度量标准名称，值是 Presto 兼容表达式
*   `col_blacklist( list )` - 列入黑名单的列表，考虑列入黑名单，大型 json 列，......
*   `assignment_func( function )` - 接收列名和类型的函数，返回度量标准名称和 Presto 表达式的 dict。 如果返回 None，则应用全局默认值。 如果返回空字典，则不会为该列计算统计信息。

```py
class airflow.operators.check_operator.IntervalCheckOperator(table, metrics_thresholds, date_filter_column='ds', days_back=-7, conn_id=None, *args, **kwargs) 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

检查作为 SQL 表达式给出的度量值是否在 days_back 之前的某个容差范围内。

请注意，这是一个抽象类，需要定义 get_db_hook。 而 get_db_hook 是钩子，它从外部源获取单个记录。

参数：

*   `table( str )` - 表名
*   `days_back( int )` - ds 与我们要检查的 ds 之间的天数。 默认为 7 天
*   `metrics_threshold( dict )` - 由指标索引的比率字典

```py
class airflow.operators.jdbc_operator.JdbcOperator(sql, jdbc_conn_id='jdbc_default', autocommit=False, parameters=None, *args, **kwargs) 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

使用 jdbc 驱动程序在数据库中执行 sql 代码。

需要 jaydebeapi。

参数：

*   `jdbc_conn_id( string )` - 对预定义数据库的引用
*   `sql(可以接收表示 sql 语句的 str，list[str]（sql 语句）或模板文件的引用。模板引用由以 '.sql' 结尾的 str 识别)` - 要执行的 sql 代码。 （模板）

```py
class airflow.operators.latest_only_operator.LatestOnlyOperator(task_id, owner='Airflow', email=None, email_on_retry=True, email_on_failure=True, retries=0, retry_delay=datetime.timedelta(0, 300), retry_exponential_backoff=False, max_retry_delay=None, start_date=None, end_date=None, schedule_interval=None, depends_on_past=False, wait_for_downstream=False, dag=None, params=None, default_args=None, adhoc=False, priority_weight=1, weight_rule=u'downstream', queue='default', pool=None, sla=None, execution_timeout=None, on_failure_callback=None, on_success_callback=None, on_retry_callback=None, trigger_rule=u'all_success', resources=None, run_as_user=None, task_concurrency=None, executor_config=None, inlets=None, outlets=None, *args, **kwargs) 
```

基类：[`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")，`airflow.models.SkipMixin`

允许工作流跳过在最近的计划间隔期间未运行的任务。

如果任务在最近的计划间隔之外运行，则将跳过所有直接下游任务。

```py
class airflow.operators.mssql_operator.MsSqlOperator（sql，mssql_conn_id ='mssql_default'，parameters = None，autocommit = False，database = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在特定的 Microsoft SQL 数据库中执行 sql 代码

参数：

*   `mssql_conn_id(string)` - 对特定 mssql 数据库的引用
*   `sql(_ 指向扩展名为.sql 的模板文件的 _string 或 _ 字符串。_ _（_ _ 模板化 _ _）_)` - 要执行的 sql 代码
*   `database(string)` - 覆盖连接中定义的数据库的数据库名称

```py
class airflow.operators.mssql_to_hive.MsSqlToHiveTransfer（sql，hive_table，create = True，recreate = False，partition = None，delimiter = u'x01'，mssql_conn_id ='mssql_default'，hive_cli_conn_id ='hive_cli_default'，tblproperties = None，* args ，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

将数据从 Microsoft SQL Server 移动到 Hive。操作员针对 Microsoft SQL Server 运行查询，在将文件加载到 Hive 表之前将其存储在本地。如果将`create`或`recreate`参数设置为`True`，则生成 a 和语句。从游标的元数据推断出 Hive 数据类型。请注意，在 Hive 中生成的表使用的不是最有效的序列化格式。如果加载了大量数据和/或表格被大量查询，您可能只想使用此运算符将数据暂存到临时表中，然后使用 a 将其加载到最终目标中。`CREATE TABLE``DROP TABLE``STORED AS textfile``HiveOperator`

参数：

*   `sql(str)` - 针对 Microsoft SQL Server 数据库执行的 SQL 查询。（模板）
*   `hive_table(str)` - 目标 Hive 表，使用点表示法来定位特定数据库。（模板）
*   `create(bool)` - 是否创建表，如果它不存在
*   `recreate(bool)` - 是否在每次执行时删除并重新创建表
*   `partition(dict)` - 将目标分区作为分区列和值的字典。（模板）
*   `delimiter(str)` - 文件中的字段分隔符
*   `mssql_conn_id(str)` - 源 Microsoft SQL Server 连接
*   `hive_conn_id(str)` - 目标配置单元连接
*   `tblproperties(dict)` - 正在创建的 hive 表的 TBLPROPERTIES

```py
class airflow.operators.mysql_operator.MySqlOperator（sql，mysql_conn_id ='mysql_default'，parameters = None，autocommit = False，database = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在特定的 MySQL 数据库中执行 sql 代码

参数：

*   `mysql_conn_id(string)` - 对特定 mysql 数据库的引用
*   `SQL(_ 可接收表示 SQL 语句中的海峡 _ _，_ _ 海峡列表 _ _（_ _SQL 语句 _ _）_ _，或 _ _ 参照模板文件模板引用在“.SQL”结束海峡认可。_)` -要执行的 SQL 代码。（模板）
*   `database(string)` - 覆盖连接中定义的数据库的数据库名称

```py
class airflow.operators.mysql_to_hive.MySqlToHiveTransfer（sql，hive_table，create = True，recreate = False，partition = None，delimiter = u'x01'，mysql_conn_id ='mysql_default'，hive_cli_conn_id ='hive_cli_default'，tblproperties = None，* args ，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

将数据从 MySql 移动到 Hive。操作员针对 MySQL 运行查询，在将文件加载到 Hive 表之前将文件存储在本地。如果将`create`或`recreate`参数设置为`True`，则生成 a 和语句。从游标的元数据推断出 Hive 数据类型。请注意，在 Hive 中生成的表使用的不是最有效的序列化格式。如果加载了大量数据和/或表格被大量查询，您可能只想使用此运算符将数据暂存到临时表中，然后使用 a 将其加载到最终目标中。`CREATE TABLE``DROP TABLE``STORED AS textfile``HiveOperator`

参数：

*   `sql(str)` - 针对 MySQL 数据库执行的 SQL 查询。（模板）
*   `hive_table(str)` - 目标 Hive 表，使用点表示法来定位特定数据库。（模板）
*   `create(bool)` - 是否创建表，如果它不存在
*   `recreate(bool)` - 是否在每次执行时删除并重新创建表
*   `partition(dict)` - 将目标分区作为分区列和值的字典。（模板）
*   `delimiter(str)` - 文件中的字段分隔符
*   `mysql_conn_id(str)` - 源码 mysql 连接
*   `hive_conn_id(str)` - 目标配置单元连接
*   `tblproperties(dict)` - 正在创建的 hive 表的 TBLPROPERTIES

```py
class airflow.operators.oracle_operator.OracleOperator（sql，oracle_conn_id ='oracle_default'，parameters = None，autocommit = False，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在特定的 Oracle 数据库中执行 sql 代码：param oracle_conn_id：对特定 Oracle 数据库的引用：type oracle_conn_id：string：param sql：要执行的 sql 代码。（模板化）：type sql：可以接收表示 sql 语句的 str，

> str（sql 语句）列表或对模板文件的引用。模板引用由以'.sql'结尾的 str 识别

```py
class airflow.operators.pig_operator.PigOperator（pig，pig_cli_conn_id ='pig_cli_default'，pigparams_jinja_translate = False，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

执行猪脚本。

参数：

*   `pig(string)` - 要执行的猪拉丁文字。（模板）
*   `pig_cli_conn_id(string)` - 对 Hive 数据库的引用
*   `pigparams_jinja_translate(_boolean_)` - 当为 True 时，猪 params 类型的模板$ {var}被转换为 jinja-type templating {{var}}。请注意，您可能希望将此`DAG(user_defined_macros=myargs)`参数与参数一起使用。查看 DAG 对象文档以获取更多详细信息。

```py
class airflow.operators.postgres_operator.PostgresOperator（sql，postgres_conn_id ='postgres_default'，autocommit = False，parameters = None，database = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在特定的 Postgres 数据库中执行 sql 代码

参数：

*   `postgres_conn_id(string)` - 对特定 postgres 数据库的引用
*   `SQL(_ 可接收表示 SQL 语句中的海峡 _ _，_ _ 海峡列表 _ _（_ _SQL 语句 _ _）_ _，或 _ _ 参照模板文件模板引用在“.SQL”结束海峡认可。_)` -要执行的 SQL 代码。（模板）
*   `database(string)` - 覆盖连接中定义的数据库的数据库名称

```py
class airflow.operators.presto_check_operator.PrestoCheckOperator（sql，presto_conn_id ='presto_default'，* args，** kwargs） 
```

基类： [`airflow.operators.check_operator.CheckOperator`](31 "airflow.operators.check_operator.CheckOperator")

对 Presto 执行检查。该`PrestoCheckOperator`预期的 SQL 查询将返回一行。使用 python `bool`强制转换评估第一行的每个值。如果任何值返回，`False`则检查失败并输出错误。

请注意，Python bool 强制转换如下`False`：

*   `False`
*   `0`
*   空字符串（`""`）
*   空列表（`[]`）
*   空字典或集（`{}`）

给定一个查询，它只会在计数时失败。您可以制作更复杂的查询，例如，可以检查表与上游源表的行数相同，或者今天的分区计数大于昨天的分区，或者一组指标是否更少 7 天平均值超过 3 个标准差。`SELECT COUNT(*) FROM foo``== 0`

此运算符可用作管道中的数据质量检查，并且根据您在 DAG 中的位置，您可以选择停止关键路径，防止发布可疑数据，或者在旁边接收电子邮件替代品阻止 DAG 的进展。

参数：

*   `sql(string)` - 要执行的 sql
*   `presto_conn_id(string)` - 对 Presto 数据库的引用

```py
class airflow.operators.presto_check_operator.PrestoIntervalCheckOperator（table，metrics_thresholds，date_filter_column ='ds'，days_back = -7，presto_conn_id ='presto_default'，* args，** kwargs） 
```

基类： [`airflow.operators.check_operator.IntervalCheckOperator`](31 "airflow.operators.check_operator.IntervalCheckOperator")

检查作为 SQL 表达式给出的度量值是否在 days_back 之前的某个容差范围内。

参数：

*   `table(str)` - 表名
*   `days_back(int)` - ds 与我们要检查的 ds 之间的天数。默认为 7 天
*   `metrics_threshold(dict)` - 由指标索引的比率字典
*   `presto_conn_id(string)` - 对 Presto 数据库的引用

```py
class airflow.operators.presto_to_mysql.PrestoToMySqlTransfer（sql，mysql_table，presto_conn_id ='presto_default'，mysql_conn_id ='mysql_default'，mysql_preoperator = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

将数据从 Presto 移动到 MySQL，注意到现在数据在被推送到 MySQL 之前被加载到内存中，因此该运算符应该用于少量数据。

参数：

*   `sql(str)` - 对 Presto 执行的 SQL 查询。（模板）
*   `mysql_table(str)` - 目标 MySQL 表，使用点表示法来定位特定数据库。（模板）
*   `mysql_conn_id(str)` - 源码 mysql 连接
*   `presto_conn_id(str)` - 源 presto 连接
*   `mysql_preoperator(str)` - 在导入之前对 mysql 运行的 sql 语句，通常用于截断删除代替进入的数据，允许任务是幂等的（运行任务两次不会加倍数据）。（模板）

```py
class airflow.operators.presto_check_operator.PrestoValueCheckOperator（sql，pass_value，tolerance = None，presto_conn_id ='presto_default'，* args，** kwargs） 
```

基类： [`airflow.operators.check_operator.ValueCheckOperator`](31 "airflow.operators.check_operator.ValueCheckOperator")

使用 sql 代码执行简单的值检查。

参数：

*   `sql(string)` - 要执行的 sql
*   `presto_conn_id(string)` - 对 Presto 数据库的引用

```py
class airflow.operators.python_operator.PythonOperator（python_callable，op_args = None，op_kwargs = None，provide_context = False，templates_dict = None，templates_exts = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

执行 Python 可调用

参数：

*   `python_callable(_python callable_)` - 对可调用对象的引用
*   `op_kwargs(dict)` - 一个关键字参数的字典，将在你的函数中解压缩
*   `op_args(list)` - 调用 callable 时将解压缩的位置参数列表
*   `provide_context(bool)` - 如果设置为 true，Airflow 将传递一组可在函数中使用的关键字参数。这组 kwargs 完全对应于你在 jinja 模板中可以使用的内容。为此，您需要在函数头中定义`** kwargs`。
*   `templates_dict(_STR 的字典 _)` -一本字典其中的值将由发动机气流之间的某个时候得到模板模板`__init__`和`execute`发生和已应用模板后，您可调用的上下文中提供。（模板）
*   `templates_exts(_list __（_ _str __）_)` - 例如，在处理模板化字段时要解析的文件扩展名列表`['.sql', '.hql']`

```py
class airflow.operators.python_operator.PythonVirtualenvOperator（python_callable，requirements = None，python_version = None，use_dill = False，system_site_packages = True，op_args = None，op_kwargs = None，string_args = None，templates_dict = None，templates_exts = None，* args， ** kwargs） 
```

基类： [`airflow.operators.python_operator.PythonOperator`](31 "airflow.operators.python_operator.PythonOperator")

允许一个人在自动创建和销毁的 virtualenv 中运行一个函数（有一些警告）。

该函数必须使用 def 定义，而不是类的一部分。所有导入必须在函数内部进行，并且不能引用范围之外的变量。名为 virtualenvstringargs 的全局范围变量将可用（由 string_args 填充）。另外，可以通过 op_args 和 op_kwargs 传递内容，并且可以使用返回值。

请注意，如果您的 virtualenv 运行在与 Airflow 不同的 Python 主要版本中，则不能使用返回值，op_args 或 op_kwargs。你可以使用 string_args。

参数：

*   `python_callable(function)` - 一个没有引用外部变量的 python 函数，用 def 定义，将在 virtualenv 中运行
*   `requirements(_list __（_ _str __）_)` - pip install 命令中指定的要求列表
*   `python_version(str)` - 用于运行 virtualenv 的 Python 版本。请注意，2 和 2.7 都是可接受的形式。
*   `use_dill(bool)` - 是否使用 dill 序列化 args 和结果（pickle 是默认值）。这允许更复杂的类型，但要求您在您的要求中包含莳萝。
*   `system_site_packages(bool)` - 是否在 virtualenv 中包含 system_site_packages。有关更多信息，请参阅 virtualenv 文档。
*   `op_args` - 要传递给 python_callable 的位置参数列表。
*   `op_kwargs(dict)` - 传递给 python_callable 的关键字参数的字典。
*   `string_args(_list __（_ _str __）_)` - 全局 var virtualenvstringargs 中存在的字符串，在运行时可用作列表（str）的 python_callable。请注意，args 按换行符分割。
*   `templates_dict(_STR 的字典 _)` -一本字典，其中值是将某个之间的气流引擎获得模板模板`__init__`和`execute`发生，并取得了您的可调用的上下文提供的模板已被应用后，
*   `templates_exts(_list __（_ _str __）_)` - 例如，在处理模板化字段时要解析的文件扩展名列表`['.sql', '.hql']`

```py
class airflow.operators.s3_file_transform_operator.S3FileTransformOperator（source_s3_key，dest_s3_key，transform_script = None，select_expression = None，source_aws_conn_id ='aws_default'，dest_aws_conn_id ='aws_default'，replace = False，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

将数据从源 S3 位置复制到本地文件系统上的临时位置。根据转换脚本的指定对此文件运行转换，并将输出上载到目标 S3 位置。

本地文件系统中的源文件和目标文件的位置作为转换脚本的第一个和第二个参数提供。转换脚本应该从源读取数据，转换它并将输出写入本地目标文件。然后，操作员接管控制并将本地目标文件上载到 S3。

S3 Select 也可用于过滤源内容。如果指定了 S3 Select 表达式，则用户可以省略转换脚本。

参数：

*   `source_s3_key(str)` - 从 S3 检索的密钥。（模板）
*   `source_aws_conn_id(str)` - 源 s3 连接
*   `dest_s3_key(str)` - 从 S3 写入的密钥。（模板）
*   `dest_aws_conn_id(str)` - 目标 s3 连接
*   `replace(bool)` - 替换 dest S3 密钥（如果已存在）
*   `transform_script(str)` - 可执行转换脚本的位置
*   `select_expression(str)` - S3 选择表达式

```py
class airflow.operators.s3_to_hive_operator.S3ToHiveTransfer（s3_key，field_dict，hive_table，delimiter ='，'，create = True，recreate = False，partition = None，headers = False，check_headers = False，wildcard_match = False，aws_conn_id ='aws_default' ，hive_cli_conn_id ='hive_cli_default'，input_compressed = False，tblproperties = None，select_expression = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

将数据从 S3 移动到 Hive。操作员从 S3 下载文件，在将文件加载到 Hive 表之前将其存储在本地。如果将`create`或`recreate`参数设置为`True`，则生成 a 和语句。Hive 数据类型是从游标的元数据中推断出来的。`CREATE TABLE``DROP TABLE`

请注意，在 Hive 中生成的表使用的不是最有效的序列化格式。如果加载了大量数据和/或表格被大量查询，您可能只想使用此运算符将数据暂存到临时表中，然后使用 a 将其加载到最终目标中。`STORED AS textfile``HiveOperator`

参数：

*   `s3_key(str)` - 从 S3 检索的密钥。（模板）
*   `field_dict(dict)` - 字段的字典在文件中命名为键，其 Hive 类型为值
*   `hive_table(str)` - 目标 Hive 表，使用点表示法来定位特定数据库。（模板）
*   `create(bool)` - 是否创建表，如果它不存在
*   `recreate(bool)` - 是否在每次执行时删除并重新创建表
*   `partition(dict)` - 将目标分区作为分区列和值的字典。（模板）
*   `headers(bool)` - 文件是否包含第一行的列名
*   `check_headers(bool)` - 是否应该根据 field_dict 的键检查第一行的列名
*   `wildcard_match(bool)` - 是否应将 s3_key 解释为 Unix 通配符模式
*   `delimiter(str)` - 文件中的字段分隔符
*   `aws_conn_id(str)` - 源 s3 连接
*   `hive_cli_conn_id(str)` - 目标配置单元连接
*   `input_compressed(bool)` - 布尔值，用于确定是否需要文件解压缩来处理标头
*   `tblproperties(dict)` - 正在创建的 hive 表的 TBLPROPERTIES
*   `select_expression(str)` - S3 选择表达式

```py
class airflow.operators.s3_to_redshift_operator.S3ToRedshiftTransfer（schema，table，s3_bucket，s3_key，redshift_conn_id ='redshift_default'，aws_conn_id ='aws_default'，copy_options =（），autocommit = False，parameters = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

执行 COPY 命令将文件从 s3 加载到 Redshift

参数：

*   `schema(string)` - 对 redshift 数据库中特定模式的引用
*   `table(string)` - 对 redshift 数据库中特定表的引用
*   `s3_bucket(string)` - 对特定 S3 存储桶的引用
*   `s3_key(string)` - 对特定 S3 密钥的引用
*   `redshift_conn_id(string)` - 对特定 redshift 数据库的引用
*   `aws_conn_id(string)` - 对特定 S3 连接的引用
*   `copy_options(list)` - 对 COPY 选项列表的引用

```py
class airflow.operators.python_operator.ShortCircuitOperator（python_callable，op_args = None，op_kwargs = None，provide_context = False，templates_dict = None，templates_exts = None，* args，** kwargs） 
```

基类：[`airflow.operators.python_operator.PythonOperator`](31 "airflow.operators.python_operator.PythonOperator")，`airflow.models.SkipMixin`

仅在满足条件时才允许工作流继续。否则，将跳过工作流程“短路”和下游任务。

ShortCircuitOperator 派生自 PythonOperator。如果条件为 False，它会评估条件并使工作流程短路。任何下游任务都标记为“已跳过”状态。如果条件为 True，则下游任务正常进行。

条件由`python_callable`的结果`决定`。

```py
class airflow.operators.http_operator.SimpleHttpOperator（endpoint，method ='POST'，data = None，headers = None，response_check = None，extra_options = None，xcom_push = False，http_conn_id ='http_default'，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在 HTTP 系统上调用端点以执行操作

参数：

*   `http_conn_id(string)` - 运行传感器的连接
*   `endpoint(string)` - 完整 URL 的相对部分。（模板）
*   `method(string)` - 要使用的 HTTP 方法，default =“POST”
*   `data(_ 对于 POST / PUT __，_ _ 取决于 content-type 参数 _ _，_ _ 用于 GET 键/值字符串对的字典 _)` - 要传递的数据。POST / PUT 中的 POST 数据和 GET 请求的 URL 中的 params。（模板）
*   `headers(_ 字符串键/值对的字典 _)` - 要添加到 GET 请求的 HTTP 头
*   `response_check(_lambda _ 或 _ 定义的函数。_)` - 检查'requests'响应对象。对于'pass'返回 True，否则返回 False。
*   `extra_options(_ 选项字典 _ _，_ _ 其中键是字符串，值取决于正在修改的选项。_)` - 'requests'库的额外选项，请参阅'requests'文档（修改超时，ssl 等选项）

```py
class airflow.operators.slack_operator.SlackAPIOperator（slack_conn_id = None，token = None，method = None，api_params = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

Base Slack 运算符 SlackAPIPostOperator 派生自此运算符。在未来，还将从此类派生其他 Slack API 操作符

参数：

*   `slack_conn_id(string)` - Slack 连接 ID，其密码为 Slack API 令牌
*   `token(string)` - Slack API 令牌（[https://api.slack.com/web](https://api.slack.com/web)）
*   `method(string)` - 要调用的 Slack API 方法（[https://api.slack.com/methods](https://api.slack.com/methods)）
*   `api_params(dict)` - API 方法调用参数（[https://api.slack.com/methods](https://api.slack.com/methods)）

```py
construct_api_call_params（） 
```

由 execute 函数使用。允许在构造之前对 api_call_params dict 的源字段进行模板化

覆盖子类。每个 SlackAPIOperator 子类都负责使用 construct_api_call_params 函数，该函数使用 API​​调用参数的字典设置 self.api_call_params（[https://api.slack.com/methods](https://api.slack.com/methods)）

```py
执行（** kwargs） 
```

即使调用不成功，SlackAPIOperator 调用也不会失败。它不应该阻止 DAG 成功完成

```py
class airflow.operators.slack_operator.SlackAPIPostOperator（channel ='＃general'，username ='Airflow'，text ='没有设置任何消息。这是一个猫视频而不是 http://www.youtube.com/watch？v = J --- aiyznGQ'，icon_url ='https：//raw.githubusercontent.com/airbnb/airflow/master/airflow/www/static/pin_100.png'，附件=无，* args，** kwargs） 
```

基类： [`airflow.operators.slack_operator.SlackAPIOperator`](31 "airflow.operators.slack_operator.SlackAPIOperator")

将消息发布到松弛通道

参数：

*   `channel(string)` - 在松弛名称（#general）或 ID（C12318391）上发布消息的通道。（模板）
*   `username(string)` - 气流将发布到 Slack 的用户**名**。（模板）
*   `text(string)` - 要发送到 slack 的消息。（模板）
*   `icon_url(string)` - 用于此消息的图标的 url
*   `附件(哈希数组)` - 额外的格式详细信息。（模板化） - 请参阅[https://api.slack.com/docs/attachments](https://api.slack.com/docs/attachments)。

```py
construct_api_call_params（） 
```

由 execute 函数使用。允许在构造之前对 api_call_params dict 的源字段进行模板化

覆盖子类。每个 SlackAPIOperator 子类都负责使用 construct_api_call_params 函数，该函数使用 API​​调用参数的字典设置 self.api_call_params（[https://api.slack.com/methods](https://api.slack.com/methods)）

```py
class airflow.operators.sqlite_operator.SqliteOperator（sql，sqlite_conn_id ='sqlite_default'，parameters = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在特定的 Sqlite 数据库中执行 sql 代码

参数：

*   `sqlite_conn_id(string)` - 对特定 sqlite 数据库的引用
*   `sql(_ 指向模板文件的 _string 或 _ 字符串。文件必须具有'.sql'扩展名。_)` - 要执行的 sql 代码。（模板）

```py
class airflow.operators.subdag_operator.SubDagOperator（** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

```py
class airflow.operators.dagrun_operator.TriggerDagRunOperator（trigger_dag_id，python_callable = None，execution_date = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

触发指定的 DAG 运行 `dag_id`

参数：

*   `trigger_dag_id(str)` - 要触发的 dag_id
*   `python_callable(_python callable_)` - 对 python 函数的引用，在传递`context`对象时将调用该对象，`obj`并且如果要创建 DagRun，则可调用的占位符对象可以填充并返回。这个`obj`对象包含`run_id`和`payload`属性，你可以在你的函数修改。本`run_id`应为 DAG 运行的唯一标识符，有效载荷必须是在执行该 DAG 运行，这将提供给你的任务 picklable 对象。你的函数头应该是这样的`def foo(context, dag_run_obj):`
*   `execution_date(_datetime.datetime_)` - dag 的执行日期

```py
class airflow.operators.check_operator.ValueCheckOperator（sql，pass_value，tolerance = None，conn_id = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

使用 sql 代码执行简单的值检查。

请注意，这是一个抽象类，需要定义 get_db_hook。而 get_db_hook 是钩子，它从外部源获取单个记录。

参数：`sql(string)` - 要执行的 sql。（模板）

```py
class airflow.operators.redshift_to_s3_operator.RedshiftToS3Transfer（schema，table，s3_bucket，s3_key，redshift_conn_id ='redshift_default'，aws_conn_id ='aws_default'，unload_options =（），autocommit = False，parameters = None，include_header = False，* args，* * kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

执行 UNLOAD 命令，将 s3 作为带标题的 CSV

参数：

*   `schema(string)` - 对 redshift 数据库中特定模式的引用
*   `table(string)` - 对 redshift 数据库中特定表的引用
*   `s3_bucket(string)` - 对特定 S3 存储桶的引用
*   `s3_key(string)` - 对特定 S3 密钥的引用
*   `redshift_conn_id(string)` - 对特定 redshift 数据库的引用
*   `aws_conn_id(string)` - 对特定 S3 连接的引用
*   `unload_options(list)` - 对 UNLOAD 选项列表的引用


#### 传感器

```py
class airflow.sensors.external_task_sensor.ExternalTask​​Sensor（external_dag_id，external_task_id，allowed_states = None，execution_delta = None，execution_date_fn = None，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

等待任务在不同的 DAG 中完成

参数：

*   `external_dag_id(string)` - 包含您要等待的任务的 dag_id
*   `external_task_id(string)` - 包含您要等待的任务的 task_id
*   `allowed_states(list)` - 允许的状态列表，默认为`['success']`
*   `execution_delta(datetime.timedelta)` - 与上一次执行的时间差异来看，默认是与当前任务相同的 execution_date。对于昨天，使用[positive！] datetime.timedelta（days = 1）。execution_delta 或 execution_date_fn 可以传递给 ExternalTask​​Sensor，但不能同时传递给两者。
*   `execution_date_fn(callable)` - 接收当前执行日期并返回所需执行日期以进行查询的函数。execution_delta 或 execution_date_fn 可以传递给 ExternalTask​​Sensor，但不能同时传递给两者。

```py
戳（** kwargs） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.sensors.hdfs_sensor.HdfsSensor（filepath，hdfs_conn_id ='hdfs_default'，ignored_ext = ['_ COPYING_']，ignore_copying = True，file_size = None，hook = <class'airflow.hooks.hdfs_hook.HDFSHook'>，* args ，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

等待文件或文件夹降落到 HDFS 中

```py
static filter_for_filesize（result，size = None） 
```

将测试文件路径结果并测试其大小是否至少为 self.filesize

参数：

*   `结果` - Snakebite ls 返回的 dicts 列表
*   `size` - 文件大小（MB）文件应该至少触发 True

返回值： （bool）取决于匹配标准

```py
static filter_for_ignored_ext（result，ignored_ext，ignore_copying） 
```

如果指示过滤将删除匹配条件的结果

参数：

*   `结果` - Snakebite ls 返回的 dicts（列表）
*   `ignored_ext` - 忽略扩展名的（列表）
*   `ignore_copying` - （bool）我们会忽略吗？

返回值： （清单）未删除的词典

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.sensors.hive_partition_sensor.HivePartitionSensor（table，partition =“ds ='{{ds}}'”，metastore_conn_id ='metastore_default'，schema ='default'，poke_interval = 180，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

等待分区显示在 Hive 中。

注意：因为`partition`支持通用逻辑运算符，所以效率低下。如果您不需要 HivePartitionSensor 的完全灵活性，请考虑使用 NamedHivePartitionSensor。

参数：

*   `table(string)` - 要等待的表的名称，支持点表示法（my_database.my_table）
*   `partition(string)` - 要等待的分区子句。这是原样传递给 Metastore Thrift 客户端`get_partitions_by_filter`方法，显然支持 SQL 作为符号和比较运算符，如`ds='2015-01-01' AND type='value'``"ds&gt;=2015-01-01"`
*   `metastore_conn_id(str)` - 对 Metastore thrift 服务连接 id 的引用

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.sensors.http_sensor.HttpSensor（endpoint，http_conn_id ='http_default'，method ='GET'，request_params = None，headers = None，response_check = None，extra_options = None，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

```py
 执行 HTTP get 语句并在失败时返回 False： 
```

找不到 404 或者 response_check 函数返回 False

参数：

*   `http_conn_id(string)` - 运行传感器的连接
*   `method(string)` - 要使用的 HTTP 请求方法
*   `endpoint(string)` - 完整 URL 的相对部分
*   `request_params(_ 字符串键/值对的字典 _)` - 要添加到 GET URL 的参数
*   `headers(_ 字符串键/值对的字典 _)` - 要添加到 GET 请求的 HTTP 头
*   `response_check(_lambda _ 或 _ 定义的函数。_)` - 检查'requests'响应对象。对于'pass'返回 True，否则返回 False。
*   `extra_options(_ 选项字典 _ _，_ _ 其中键是字符串，值取决于正在修改的选项。_)` - 'requests'库的额外选项，请参阅'requests'文档（修改超时，ssl 等选项）

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.sensors.metastore_partition_sensor.MetastorePartitionSensor（table，partition_name，schema ='default'，mysql_conn_id ='metastore_mysql'，* args，** kwargs） 
```

基类： [`airflow.sensors.sql_sensor.SqlSensor`](31 "airflow.sensors.sql_sensor.SqlSensor")

HivePartitionSensor 的替代方案，直接与 MySQL 数据库对话。这是在观察子分区表时 Metastore thrift 服务生成的子最优查询的结果。Thrift 服务的查询是以不利用索引的方式编写的。

参数：

*   `schema(str)` - 架构
*   `table(str)` - 表格
*   `partition_name(str)` - 分区名称，在 Metastore 的 PARTITIONS 表中定义。字段顺序很重要。示例：`ds=2016-01-01`或`ds=2016-01-01/sub=foo`用于子分区表
*   `mysql_conn_id(str)` - 对 Metastore 的 MySQL conn_id 的引用

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.sensors.named_hive_partition_sensor.NamedHivePartitionSensor（partition_names，metastore_conn_id ='metastore_default'，poke_interval = 180，hook = None，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

等待一组分区显示在 Hive 中。

参数：

*   `partition_names(_ 字符串列表 _)` - 要等待的分区的完全限定名称列表。完全限定名称的格式`schema.table/pk1=pv1/pk2=pv2`为，例如，default.users / ds = 2016-01-01。这将原样传递给 Metastore Thrift 客户端`get_partitions_by_name`方法。请注意，您不能像在 HivePartitionSensor 中那样使用逻辑或比较运算符。
*   `metastore_conn_id(str)` - 对 Metastore thrift 服务连接 id 的引用

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.sensors.s3_key_sensor.S3KeySensor（bucket_key，bucket_name = None，wildcard_match = False，aws_conn_id ='aws_default'，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

等待 S3 存储桶中存在密钥（S3 上的文件类实例）。S3 是键/值，它不支持文件夹。路径只是资源的关键。

参数：

*   `bucket_key(str)` - 正在等待的密钥。支持完整的 s3：//样式 URL 或从根级别的相对路径。
*   `bucket_name(str)` - S3 存储桶的名称
*   `wildcard_match(bool)` - 是否应将 bucket_key 解释为 Unix 通配符模式
*   `aws_conn_id(str)` - 对 s3 连接的引用

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.sensors.s3_prefix_sensor.S3PrefixSensor（bucket_name，prefix，delimiter ='/'，aws_conn_id ='aws_default'，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

等待前缀存在。前缀是键的第一部分，因此可以检查类似于 glob airfl *或 SQL LIKE'airfl％'的构造。有可能精确定界符来指示层次结构或键，这意味着匹配将停止在该定界符处。当前代码接受合理的分隔符，即 Python 正则表达式引擎中不是特殊字符的字符。

参数：

*   `bucket_name(str)` - S3 存储桶的名称
*   `prefix(str)` - 等待的前缀。桶根级别的相对路径。
*   `delimiter(str)` - 用于显示层次结构的分隔符。默认为'/'。

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.sensors.sql_sensor.SqlSensor（conn_id，sql，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

运行 sql 语句，直到满足条件。它将继续尝试，而 sql 不返回任何行，或者如果第一个单元格返回（0，'0'，''）。

参数：

*   `conn_id(string)` - 运行传感器的连接
*   `sql` - 要运行的 sql。要传递，它需要返回至少一个包含非零/空字符串值的单元格。

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.sensors.time_sensor.TimeSensor（target_time，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

等到当天的指定时间。

参数：`target_time(_datetime.time_)` - 作业成功的时间

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.sensors.time_delta_sensor.TimeDeltaSensor（delta，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

在任务的 execution_date + schedule_interval 之后等待 timedelta。在 Airflow 中，标有`execution_date`2016-01-01 的每日任务只能在 2016-01-02 开始运行。timedelta 在此表示执行期间结束后的时间。

参数：`delta(datetime.timedelta)` - 成功之前在 execution_date 之后等待的时间长度

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.sensors.web_hdfs_sensor.WebHdfsSensor（filepath，webhdfs_conn_id ='webhdfs_default'，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

等待文件或文件夹降落到 HDFS 中

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

### 社区贡献的运营商

#### 运营商

```py
class airflow.contrib.operators.awsbatch_operator.AWSBatchOperator（job_name，job_definition，job_queue，overrides，max_retries = 4200，aws_conn_id = None，region_name = None，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在 AWS Batch Service 上执行作业

参数：

*   `job_name(str)` - 将在 AWS Batch 上运行的作业的名称
*   `job_definition(str)` - AWS Batch 上的作业定义名称
*   `job_queue(str)` - AWS Batch 上的队列名称
*   `max_retries(int)` - 服务器未合并时的指数退避重试，4200 = 48 小时
*   `aws_conn_id(str)` - AWS 凭证/区域名称的连接 ID。如果为 None，将使用凭证 boto3 策略（[http://boto3.readthedocs.io/en/latest/guide/configuration.html](http://boto3.readthedocs.io/en/latest/guide/configuration.html)）。
*   `region_name` - 要在 AWS Hook 中使用的区域名称。覆盖连接中的 region_name（如果提供）

| 帕拉姆： | 覆盖：boto3 将在 containerOverrides 上接收的相同参数（模板化）：[http](http://boto3.readthedocs.io/en/latest/reference/services/batch.html)：//boto3.readthedocs.io/en/latest/reference/services/batch.html#submit_job[](http://boto3.readthedocs.io/en/latest/reference/services/batch.html)类型： 覆盖：dict

```py
class airflow.contrib.operators.bigquery_check_operator.BigQueryCheckOperator（sql，bigquery_conn_id ='bigquery_default'，* args，** kwargs） 
```

基类： [`airflow.operators.check_operator.CheckOperator`](31 "airflow.operators.check_operator.CheckOperator")

对 BigQuery 执行检查。该`BigQueryCheckOperator`预期的 SQL 查询将返回一行。使用 python `bool`强制转换评估第一行的每个值。如果任何值返回，`False`则检查失败并输出错误。

请注意，Python bool 强制转换如下`False`：

*   `False`
*   `0`
*   空字符串（`""`）
*   空列表（`[]`）
*   空字典或集（`{}`）

给定一个查询，它只会在计数时失败。您可以制作更复杂的查询，例如，可以检查表与上游源表的行数相同，或者今天的分区计数大于昨天的分区，或者一组指标是否更少 7 天平均值超过 3 个标准差。`SELECT COUNT(*) FROM foo``== 0`

此运算符可用作管道中的数据质量检查，并且根据您在 DAG 中的位置，您可以选择停止关键路径，防止发布可疑数据，或者在旁边接收电子邮件替代品阻止 DAG 的进展。

参数：

*   `sql(string)` - 要执行的 sql
*   `bigquery_conn_id(string)` - 对 BigQuery 数据库的引用

```py
class airflow.contrib.operators.bigquery_check_operator.BigQueryValueCheckOperator（sql，pass_value，tolerance = None，bigquery_conn_id ='bigquery_default'，* args，** kwargs） 
```

基类： [`airflow.operators.check_operator.ValueCheckOperator`](31 "airflow.operators.check_operator.ValueCheckOperator")

使用 sql 代码执行简单的值检查。

参数：`sql(string)` - 要执行的 sql

```py
class airflow.contrib.operators.bigquery_check_operator.BigQueryIntervalCheckOperator（table，metrics_thresholds，date_filter_column ='ds'，days_back = -7，bigquery_conn_id ='bigquery_default'，* args，** kwargs） 
```

基类： [`airflow.operators.check_operator.IntervalCheckOperator`](31 "airflow.operators.check_operator.IntervalCheckOperator")

检查作为 SQL 表达式给出的度量值是否在 days_back 之前的某个容差范围内。

此方法构造一个类似的查询

```py
 SELECT { metrics_thresholddictkey } FROM { table }
    WHERE { date_filter_column } =< date >

```

参数：

*   `table(str)` - 表名
*   `days_back(int)` - ds 与我们要检查的 ds 之间的天数。默认为 7 天
*   `metrics_threshold(dict)` - 由指标索引的比率字典，例如'COUNT（*）'：1.5 将需要当前日和之前的 days_back 之间 50％或更小的差异。

```py
class airflow.contrib.operators.bigquery_get_data.BigQueryGetDataOperator（dataset_id，table_id，max_results ='100'，selected_fields = None，bigquery_conn_id ='bigquery_default'，delegate_to = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

从 BigQuery 表中获取数据（或者为所选列获取数据）并在 python 列表中返回数据。返回列表中的元素数将等于获取的行数。列表中的每个元素将再次是一个列表，其中元素将表示该行的列值。

**结果示例**：`[['Tony', '10'], ['Mike', '20'], ['Steve', '15']]`

注意

如果传递的字段`selected_fields`的顺序与 BQ 表中已有的列的顺序不同，则数据仍将按 BQ 表的顺序排列。例如，如果 BQ 表有 3 列，`[A,B,C]`并且您传递'B，`selected_fields`那么数据中的 A' 仍然是表格`'A,B'`。

**示例** ：

```py
 get_data = BigQueryGetDataOperator (
    task_id = 'get_data_from_bq' ,
    dataset_id = 'test_dataset' ,
    table_id = 'Transaction_partitions' ,
    max_results = '100' ,
    selected_fields = 'DATE' ,
    bigquery_conn_id = 'airflow-service-account'
)

```

参数：

*   `dataset_id` - 请求的表的数据集 ID。（模板）
*   `table_id(string)` - 请求表的表 ID。（模板）
*   `max_results(string)` - 从表中获取的最大记录数（行数）。（模板）
*   `selected_fields(string)` - 要返回的字段列表（逗号分隔）。如果未指定，则返回所有字段。
*   `bigquery_conn_id(string)` - 对特定 BigQuery 钩子的引用。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。

```py
class airflow.contrib.operators.bigquery_operator.BigQueryCreateEmptyTableOperator（dataset_id，table_id，project_id = None，schema_fields = None，gcs_schema_object = None，time_partitioning = {}，bigquery_conn_id ='bigquery_default'，google_cloud_storage_conn_id ='google_cloud_default'，delegate_to = None，* args ，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在指定的 BigQuery 数据集中创建一个新的空表，可选择使用模式。

可以用两种方法之一指定用于 BigQuery 表的模式。您可以直接传递架构字段，也可以将运营商指向 Google 云存储对象名称。Google 云存储中的对象必须是包含架构字段的 JSON 文件。您还可以创建没有架构的表。

参数：

*   `project_id(string)` - 将表创建的项目。（模板）
*   `dataset_id(string)` - 用于创建表的数据集。（模板）
*   `table_id(string)` - 要创建的表的名称。（模板）
*   `schema_fields(list)` -

    如果设置，则此处定义的架构字段列表：[https](https://cloud.google.com/bigquery/docs/reference/rest/v2/jobs)：[//cloud.google.com/bigquery/docs/reference/rest/v2/jobs#configuration.load.schema](https://cloud.google.com/bigquery/docs/reference/rest/v2/jobs)

    **示例** ：

    ```py
     schema_fields = [{ "name" : "emp_name" , "type" : "STRING" , "mode" : "REQUIRED" },
                   { "name" : "salary" , "type" : "INTEGER" , "mode" : "NULLABLE" }]

    ```

*   `gcs_schema_object(string)` - 包含模式（模板化）的 JSON 文件的完整路径。例如：`gs://test-bucket/dir1/dir2/employee_schema.json`
*   `time_partitioning(dict)` -

    配置可选的时间分区字段，即按 API 规范按字段，类型和到期分区。

    也可以看看

    [https://cloud.google.com/bigquery/docs/reference/rest/v2/tables#timePartitioning](https://cloud.google.com/bigquery/docs/reference/rest/v2/tables)

*   `bigquery_conn_id(string)` - 对特定 BigQuery 挂钩的引用。
*   `google_cloud_storage_conn_id(string)` - 对特定 Google 云存储挂钩的引用。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。


**示例（在 GCS 中使用模式 JSON）**：

```py
 CreateTable = BigQueryCreateEmptyTableOperator (
    task_id = 'BigQueryCreateEmptyTableOperator_task' ,
    dataset_id = 'ODS' ,
    table_id = 'Employees' ,
    project_id = 'internal-gcp-project' ,
    gcs_schema_object = 'gs://schema-bucket/employee_schema.json' ,
    bigquery_conn_id = 'airflow-service-account' ,
    google_cloud_storage_conn_id = 'airflow-service-account'
)

```

**对应的 Schema 文件**（`employee_schema.json`）：

```py
 [
  {
    "mode" : "NULLABLE" ,
    "name" : "emp_name" ,
    "type" : "STRING"
  },
  {
    "mode" : "REQUIRED" ,
    "name" : "salary" ,
    "type" : "INTEGER"
  }
]

```

**示例（在 DAG 中使用模式）**：

```py
 CreateTable = BigQueryCreateEmptyTableOperator (
    task_id = 'BigQueryCreateEmptyTableOperator_task' ,
    dataset_id = 'ODS' ,
    table_id = 'Employees' ,
    project_id = 'internal-gcp-project' ,
    schema_fields = [{ "name" : "emp_name" , "type" : "STRING" , "mode" : "REQUIRED" },
                   { "name" : "salary" , "type" : "INTEGER" , "mode" : "NULLABLE" }],
    bigquery_conn_id = 'airflow-service-account' ,
    google_cloud_storage_conn_id = 'airflow-service-account'
)

```

```py
class airflow.contrib.operators.bigquery_operator.BigQueryCreateExternalTableOperator（bucket，source_objects，destination_project_dataset_table，schema_fields = None，schema_object = None，source_format ='CSV'，compression ='NONE'，skip_leading_rows = 0，field_delimiter ='，'，max_bad_records = 0 ，quote_character = None，allow_quoted_newlines = False，allow_jagged_rows = False，bigquery_conn_id ='bigquery_default'，google_cloud_storage_conn_id ='google_cloud_default'，delegate_to = None，src_fmt_configs = {}，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

使用 Google 云端存储中的数据在数据集中创建新的外部表。

可以用两种方法之一指定用于 BigQuery 表的模式。您可以直接传递架构字段，也可以将运营商指向 Google 云存储对象名称。Google 云存储中的对象必须是包含架构字段的 JSON 文件。

参数：

*   `bucket(string)` - 指向外部表的存储桶。（模板）
*   `source_objects` - 指向表格的 Google 云存储 URI 列表。（模板化）如果 source_format 是'DATASTORE_BACKUP'，则列表必须只包含一个 URI。
*   `destination_project_dataset_table(string)` - 用于将数据加载到（模板化）的虚线（&lt;project&gt;。）&lt;dataset&gt;。&lt;table&gt; BigQuery 表。如果未包含&lt;project&gt;，则项目将是连接 json 中定义的项目。
*   `schema_fields(list)` -

    如果设置，则此处定义的架构字段列表：[https](https://cloud.google.com/bigquery/docs/reference/rest/v2/jobs)：[//cloud.google.com/bigquery/docs/reference/rest/v2/jobs#configuration.load.schema](https://cloud.google.com/bigquery/docs/reference/rest/v2/jobs)

    **示例** ：

    ```py
     schema_fields = [{ "name" : "emp_name" , "type" : "STRING" , "mode" : "REQUIRED" },
                   { "name" : "salary" , "type" : "INTEGER" , "mode" : "NULLABLE" }]

    ```

    当 source_format 为'DATASTORE_BACKUP'时，不应设置。

*   `schema_object` - 如果设置，则指向包含表的架构的.json 文件的 GCS 对象路径。（模板）
*   `schema_object` - 字符串
*   `source_format(string)` - 数据的文件格式。
*   `compression(string)` - [可选]数据源的压缩类型。可能的值包括 GZIP 和 NONE。默认值为 NONE。Google Cloud Bigtable，Google Cloud Datastore 备份和 Avro 格式会忽略此设置。
*   `skip_leading_rows(int)` - 从 CSV 加载时要跳过的行数。
*   `field_delimiter(string)` - 用于 CSV 的分隔符。
*   `max_bad_records(int)` - BigQuery 在运行作业时可以忽略的最大错误记录数。
*   `quote_character(string)` - 用于引用 CSV 文件中数据部分的值。
*   `allow_quoted_newlines(_boolean_)` - 是否允许引用的换行符（true）或不允许（false）。
*   `allow_jagged_rows(bool)` - 接受缺少尾随可选列的行。缺失值被视为空值。如果为 false，则缺少尾随列的记录将被视为错误记录，如果错误记录太多，则会在作业结果中返回无效错误。仅适用于 CSV，忽略其他格式。
*   `bigquery_conn_id(string)` - 对特定 BigQuery 挂钩的引用。
*   `google_cloud_storage_conn_id(string)` - 对特定 Google 云存储挂钩的引用。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `src_fmt_configs(dict)` - 配置特定于源格式的可选字段

```py
class airflow.contrib.operators.bigquery_operator.BigQueryOperator（bql = None，sql = None，destination_dataset_table = False，write_disposition ='WRITE_EMPTY'，allow_large_results = False，flatten_results = False，bigquery_conn_id ='bigquery_default'，delegate_to = None，udf_config = False ，use_legacy_sql = True，maximum_billing_tier = None，maximum_bytes_billed = None，create_disposition ='CREATE_IF_NEEDED'，schema_update_options =（），query_params = None，priority ='INTERACTIVE'，time_partitioning = {}，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在特定的 BigQuery 数据库中执行 BigQuery SQL 查询

参数：

*   `BQL(_ 可接收表示 SQL 语句中的海峡 _ _，_ _ 海峡列表 _ _（_ _SQL 语句 _ _）_ _，或 _ _ 参照模板文件模板引用在“.SQL”结束海峡认可。_)` - （不推荐使用。`SQL`参数代替）要执行的 sql 代码（模板化）
*   `SQL(_ 可接收表示 SQL 语句中的海峡 _ _，_ _ 海峡列表 _ _（_ _SQL 语句 _ _）_ _，或 _ _ 参照模板文件模板引用在“.SQL”结束海峡认可。_)` - SQL 代码被执行（模板）
*   `destination_dataset_table(string)` - 一个虚线（&lt;project&gt;。&#124; &lt;project&gt;：）&lt;dataset&gt;。&lt;table&gt;，如果设置，将存储查询结果。（模板）
*   `write_disposition(string)` - 指定目标表已存在时发生的操作。（默认：'WRITE_EMPTY'）
*   `create_disposition(string)` - 指定是否允许作业创建新表。（默认值：'CREATE_IF_NEEDED'）
*   `allow_large_results(_boolean_)` - 是否允许大结果。
*   `flatten_results(_boolean_)` - 如果为 true 且查询使用旧版 SQL 方言，则展平查询结果中的所有嵌套和重复字段。`allow_large_results`必须是`true`如果设置为`false`。对于标准 SQL 查询，将忽略此标志，并且结果永远不会展平。
*   `bigquery_conn_id(string)` - 对特定 BigQuery 钩子的引用。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `udf_config(list)` - 查询的用户定义函数配置。有关详细信息，请参阅[https://cloud.google.com/bigquery/user-defined-functions](https://cloud.google.com/bigquery/user-defined-functions)。
*   `use_legacy_sql(_boolean_)` - 是使用旧 SQL（true）还是标准 SQL（false）。
*   `maximum_billing_tier(_ 整数 _)` - 用作基本价格乘数的正整数。默认为 None，在这种情况下，它使用项目中设置的值。
*   `maximum_bytes_billed(float)` - 限制为此作业计费的字节数。超出此限制的字节数的查询将失败（不会产生费用）。如果未指定，则将其设置为项目默认值。
*   `schema_update_options(_tuple_)` - 允许更新目标表的模式作为加载作业的副作用。
*   `query_params(dict)` - 包含查询参数类型和值的字典，传递给 BigQuery。
*   `priority(string)` - 指定查询的优先级。可能的值包括 INTERACTIVE 和 BATCH。默认值为 INTERACTIVE。
*   `time_partitioning(dict)` - 配置可选的时间分区字段，即按 API 规范按字段，类型和到期分区。请注意，'field'不能与 dataset.table $ partition 一起使用。

```py
class airflow.contrib.operators.bigquery_table_delete_operator.BigQueryTableDeleteOperator（deletion_dataset_table，bigquery_conn_id ='bigquery_default'，delegate_to = None，ignore_if_missing = False，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

删除 BigQuery 表

参数：

*   `deletion_dataset_table(string)` - 一个虚线（&lt;project&gt;。&#124; &lt;project&gt;：）&lt;dataset&gt;。&lt;table&gt;，指示将删除哪个表。（模板）
*   `bigquery_conn_id(string)` - 对特定 BigQuery 钩子的引用。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `ignore_if_missing(_boolean_)` - 如果为 True，则即使请求的表不存在也返回成功。

```py
class airflow.contrib.operators.bigquery_to_bigquery.BigQueryToBigQueryOperator（source_project_dataset_tables，destination_project_dataset_table，write_disposition ='WRITE_EMPTY'，create_disposition ='CREATE_IF_NEEDED'，bigquery_conn_id ='bigquery_default'，delegate_to = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

将数据从一个 BigQuery 表复制到另一个。

也可以看看

有关这些参数的详细信息，请访问：[https](https://cloud.google.com/bigquery/docs/reference/v2/jobs)：[//cloud.google.com/bigquery/docs/reference/v2/jobs#configuration.copy](https://cloud.google.com/bigquery/docs/reference/v2/jobs)

参数：

*   `source_project_dataset_tables(_list &#124; string_)` - 一个或多个点（项目：[&#124;](31)项目。）&lt;dataset&gt;。&lt;table&gt;用作源数据的 BigQuery 表。如果未包含&lt;project&gt;，则项目将是连接 json 中定义的项目。如果有多个源表，请使用列表。（模板）
*   `destination_project_dataset_table(string)` - 目标 BigQuery 表。格式为：（project：[&#124;](31) project。）&lt;dataset&gt;。&lt;table&gt;（模板化）
*   `write_disposition(string)` - 表已存在时的写处置。
*   `create_disposition(string)` - 如果表不存在，则创建处置。
*   `bigquery_conn_id(string)` - 对特定 BigQuery 钩子的引用。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。

```py
class airflow.contrib.operators.bigquery_to_gcs.BigQueryToCloudStorageOperator（source_project_dataset_table，destination_cloud_storage_uris，compression ='NONE'，export_format ='CSV'，field_delimiter ='，'，print_header = True，bigquery_conn_id ='bigquery_default'，delegate_to = None，* args， ** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

将 BigQuery 表传输到 Google Cloud Storage 存储桶。

也可以看看

有关这些参数的详细信息，请访问：[https](https://cloud.google.com/bigquery/docs/reference/v2/jobs)：[//cloud.google.com/bigquery/docs/reference/v2/jobs](https://cloud.google.com/bigquery/docs/reference/v2/jobs)

参数：

*   `source_project_dataset_table(string)` - 用作源数据的虚线（&lt;project&gt;。&#124; &lt;project&gt;：）&lt;dataset&gt;。&lt;table&gt; BigQuery 表。如果未包含&lt;project&gt;，则项目将是连接 json 中定义的项目。（模板）
*   `destination_cloud_storage_uris(list)` - 目标 Google 云端存储 URI（例如 gs：//some-bucket/some-file.txt）。（模板化）遵循此处定义的惯例：https：//cloud.google.com/bigquery/exporting-data-from-bigquery#exportingmultiple
*   `compression(string)` - 要使用的压缩类型。
*   `export_format` - 要导出的文件格式。
*   `field_delimiter(string)` - 提取到 CSV 时使用的分隔符。
*   `print_header(_boolean_)` - 是否打印 CSV 文件提取的标头。
*   `bigquery_conn_id(string)` - 对特定 BigQuery 钩子的引用。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。

```py
class airflow.contrib.operators.cassandra_to_gcs.CassandraToGoogleCloudStorageOperator（cql，bucket，filename，schema_filename = None，approx_max_file_size_bytes = 1900000000，cassandra_conn_id = u'cassandra_default'，google_cloud_storage_conn_id = u'google_cloud_default'，delegate_to = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

将数据从 Cassandra 复制到 JSON 格式的 Google 云端存储

注意：不支持数组数组。

```py
classmethod convert_map_type（name，value） 
```

将映射转换为包含两个字段的重复 RECORD：'key'和'value'，每个字段将转换为 BQ 中的相应数据类型。

```py
classmethod convert_tuple_type（name，value） 
```

将元组转换为包含 n 个字段的 RECORD，每个字段将转换为 bq 中相应的数据类型，并将命名为“field_ &lt;index&gt;”，其中 index 由 cassandra 中定义的元组元素的顺序确定。

```py
classmethod convert_user_type（name，value） 
```

将用户类型转换为包含 n 个字段的 RECORD，其中 n 是属性的数量。用户类型类中的每个元素将在 BQ 中转换为其对应的数据类型。

```py
class airflow.contrib.operators.databricks_operator.DatabricksSubmitRunOperator（json = None，spark_jar_task = None，notebook_task = None，new_cluster = None，existing_cluster_id = None，libraries = None，run_name = None，timeout_seconds = None，databricks_conn_id ='databricks_default'，polling_period_seconds = 30，databricks_retry_limit = 3，do_xcom_push = False，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

使用[api / 2.0 / jobs / runs / submit](https://docs.databricks.com/api/latest/jobs.html) API 端点向 Databricks 提交 Spark 作业运行。

有两种方法可以实例化此运算符。

在第一种方式，你可以把你通常用它来调用的 JSON 有效载荷`api/2.0/jobs/runs/submit`端点并将其直接传递到我们`DatabricksSubmitRunOperator`通过`json`参数。例如

```py
 json = {
  'new_cluster' : {
    'spark_version' : '2.1.0-db3-scala2.11' ,
    'num_workers' : 2
  },
  'notebook_task' : {
    'notebook_path' : '/Users/airflow@example.com/PrepareData' ,
  },
}
notebook_run = DatabricksSubmitRunOperator ( task_id = 'notebook_run' , json = json )

```

另一种完成同样事情的方法是直接使用命名参数`DatabricksSubmitRunOperator`。请注意，`runs/submit`端点中的每个顶级参数都只有一个命名参数。在此方法中，您的代码如下所示：

```py
 new_cluster = {
  'spark_version' : '2.1.0-db3-scala2.11' ,
  'num_workers' : 2
}
notebook_task = {
  'notebook_path' : '/Users/airflow@example.com/PrepareData' ,
}
notebook_run = DatabricksSubmitRunOperator (
    task_id = 'notebook_run' ,
    new_cluster = new_cluster ,
    notebook_task = notebook_task )

```

在提供 json 参数**和**命名参数的情况下，它们将合并在一起。如果在合并期间存在冲突，则命名参数将优先并覆盖顶级`json`键。

```py
 目前 DatabricksSubmitRunOperator 支持的命名参数是 
```

*   `spark_jar_task`
*   `notebook_task`
*   `new_cluster`
*   `existing_cluster_id`
*   `libraries`
*   `run_name`
*   `timeout_seconds`

参数：

*   `json(dict)` -

    包含 API 参数的 JSON 对象，将直接传递给`api/2.0/jobs/runs/submit`端点。其他命名参数（即`spark_jar_task`，`notebook_task`..）到该运营商将与此 JSON 字典合并如果提供他们。如果在合并期间存在冲突，则命名参数将优先并覆盖顶级 json 键。（模板）

    也可以看看

    有关模板的更多信息，请参阅[Jinja 模板](concepts.html)。[https://docs.databricks.com/api/latest/jobs.html#runs-submit](https://docs.databricks.com/api/latest/jobs.html)

*   `spark_jar_task(dict)` -

    JAR 任务的主要类和参数。请注意，实际的 JAR 在`libraries`。中指定。_ 无论是 _ `spark_jar_task` 或 `notebook_task`应符合规定。该字段将被模板化。

    也可以看看

    [https://docs.databricks.com/api/latest/jobs.html#jobssparkjartask](https://docs.databricks.com/api/latest/jobs.html)

*   `notebook_task(dict)` -

    笔记本任务的笔记本路径和参数。_ 无论是 _ `spark_jar_task` 或 `notebook_task`应符合规定。该字段将被模板化。

    也可以看看

    [https://docs.databricks.com/api/latest/jobs.html#jobsnotebooktask](https://docs.databricks.com/api/latest/jobs.html)

*   `new_cluster(dict)` -

    将在其上运行此任务的新群集的规范。_ 无论是 _ `new_cluster` 或 `existing_cluster_id`应符合规定。该字段将被模板化。

    也可以看看

    [https://docs.databricks.com/api/latest/jobs.html#jobsclusterspecnewcluster](https://docs.databricks.com/api/latest/jobs.html)

*   `existing_cluster_id(string)` - 要运行此任务的现有集群的 ID。_ 无论是 _ `new_cluster` 或 `existing_cluster_id`应符合规定。该字段将被模板化。
*   `图书馆(_dicts 列表 _)` -

    这个运行的库将使用。该字段将被模板化。

    也可以看看

    [https://docs.databricks.com/api/latest/libraries.html#managedlibrarieslibrary](https://docs.databricks.com/api/latest/libraries.html)

*   `run_name(string)` - 用于此任务的运行名称。默认情况下，这将设置为 Airflow `task_id`。这`task_id`是超类的必需参数`BaseOperator`。该字段将被模板化。
*   `timeout_seconds(_int32_)` - 此次运行的超时。默认情况下，使用值 0 表示没有超时。该字段将被模板化。
*   `databricks_conn_id(string)` - 要使用的 Airflow 连接的名称。默认情况下，在常见情况下，这将是`databricks_default`。要使用基于令牌的身份验证，请`token`在连接的额外字段中提供密钥。
*   `polling_period_seconds(int)` - 控制我们轮询此运行结果的速率。默认情况下，操作员每 30 秒轮询一次。
*   `databricks_retry_limit(int)` - 如果 Databricks 后端无法访问，则重试的次数。其值必须大于或等于 1。
*   `do_xcom_push(_boolean_)` - 我们是否应该将 run_id 和 run_page_url 推送到 xcom。

```py
class airflow.contrib.operators.dataflow_operator.DataFlowJavaOperator（jar，dataflow_default_options = None，options = None，gcp_conn_id ='google_cloud_default'，delegate_to = None，poll_sleep = 10，job_class = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

启动 Java Cloud DataFlow 批处理作业。操作的参数将传递给作业。

在 dag 的 default_args 中定义 dataflow_ *参数是一个很好的做法，例如项目，区域和分段位置。

```py
 default_args = {
    'dataflow_default_options' : {
        'project' : 'my-gcp-project' ,
        'zone' : 'europe-west1-d' ,
        'stagingLocation' : 'gs://my-staging-bucket/staging/'
    }
}

```

您需要使用`jar`参数将路径作为文件引用传递给数据流，jar 需要是一个自动执行的 jar（请参阅以下文档：[https](https://beam.apache.org/documentation/runners/dataflow/)：[//beam.apache.org/documentation/runners/dataflow/#self-执行 jar](https://beam.apache.org/documentation/runners/dataflow/)。使用`options`转嫁选项你的工作。

```py
 t1 = DataFlowOperation (
    task_id = 'datapflow_example' ,
    jar = '{{var.value.gcp_dataflow_base}}pipeline/build/libs/pipeline-example-1.0.jar' ,
    options = {
        'autoscalingAlgorithm' : 'BASIC' ,
        'maxNumWorkers' : '50' ,
        'start' : '{{ds}}' ,
        'partitionType' : 'DAY' ,
        'labels' : { 'foo' : 'bar' }
    },
    gcp_conn_id = 'gcp-airflow-service-account' ,
    dag = my - dag )

```

这两个`jar`和`options`模板化，所以你可以在其中使用变量。

```py
class airflow.contrib.operators.dataflow_operator.DataflowTemplateOperator（template，dataflow_default_options = None，parameters = None，gcp_conn_id ='google_cloud_default'，delegate_to = None，poll_sleep = 10，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

启动模板化云 DataFlow 批处理作业。操作的参数将传递给作业。在 dag 的 default_args 中定义 dataflow_ *参数是一个很好的做法，例如项目，区域和分段位置。

也可以看看

[https://cloud.google.com/dataflow/docs/reference/rest/v1b3/LaunchTemplateParameters ](https://cloud.google.com/dataflow/docs/reference/rest/v1b3/LaunchTemplateParameters)[https://cloud.google.com/dataflow/docs/reference/rest/v1b3/RuntimeEnvironment](https://cloud.google.com/dataflow/docs/reference/rest/v1b3/RuntimeEnvironment)

```py
 default_args = {
    'dataflow_default_options' : {
        'project' : 'my-gcp-project'
        'zone' : 'europe-west1-d' ,
        'tempLocation' : 'gs://my-staging-bucket/staging/'
        }
    }
}

```

您需要将路径作为带`template`参数的文件引用传递给数据流模板。使用`parameters`来传递参数给你的工作。使用`environment`对运行环境变量传递给你的工作。

```py
 t1 = DataflowTemplateOperator (
    task_id = 'datapflow_example' ,
    template = '{{var.value.gcp_dataflow_base}}' ,
    parameters = {
        'inputFile' : "gs://bucket/input/my_input.txt" ,
        'outputFile' : "gs://bucket/output/my_output.txt"
    },
    gcp_conn_id = 'gcp-airflow-service-account' ,
    dag = my - dag )

```

`template`，`dataflow_default_options`并且`parameters`是模板化的，因此您可以在其中使用变量。

```py
class airflow.contrib.operators.dataflow_operator.DataFlowPythonOperator（py_file，py_options = None，dataflow_default_options = None，options = None，gcp_conn_id ='google_cloud_default'，delegate_to = None，poll_sleep = 10，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

```py
执行（上下文） 
```

执行 python 数据流作业。

```py
class airflow.contrib.operators.dataproc_operator.DataprocClusterCreateOperator（cluster_name，project_id，num_workers，zone，network_uri = None，subnetwork_uri = None，internal_ip_only = None，tags = None，storage_bucket = None，init_actions_uris = None，init_action_timeout ='10m'，metadata =无，image_version =无，属性=无，master_machine_type ='n1-standard-4'，master_disk_size = 500，worker_machine_type ='n1-standard-4'，worker_disk_size = 500，num_preemptible_workers = 0，labels = None，region =' global'，gcp_conn_id ='google_cloud_default'，delegate_to = None，service_account = None，service_account_scopes = None，idle_delete_ttl = None，auto_delete_time = None，auto_delete_ttl = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在 Google Cloud Dataproc 上创建新群集。操作员将等待创建成功或创建过程中发生错误。

参数允许配置群集。请参阅

[https://cloud.google.com/dataproc/docs/reference/rest/v1/projects.regions.clusters](https://cloud.google.com/dataproc/docs/reference/rest/v1/projects.regions.clusters)

有关不同参数的详细说明。链接中详述的大多数配置参数都可作为此运算符的参数。

参数：

*   `cluster_name(string)` - 要创建的 DataProc 集群的名称。（模板）
*   `project_id(string)` - 用于创建集群的 Google 云项目的 ID。（模板）
*   `num_workers(int)` - 旋转的工人数量
*   `storage_bucket(string)` - 要使用的存储桶，设置为 None 允许 dataproc 为您生成自定义存储桶
*   `init_actions_uris(_list __[ __string __]_)` - 包含数据空间初始化脚本的 GCS uri 列表
*   `init_action_timeout(string)` - init_actions_uris 中可执行脚本必须完成的时间
*   `元数据(_ 字典 _)` - 要添加到所有实例的键值 google 计算引擎元数据条目的字典
*   `image_version(string)` - Dataproc 集群内的软件版本
*   `属性(_ 字典 _)` -性能上的配置文件设置的字典（如火花 defaults.conf），见[https://cloud.google.com/dataproc/docs/reference/rest/v1/](https://cloud.google.com/dataproc/docs/reference/rest/v1/) projects.regions.clusters＃SoftwareConfig
*   `master_machine_type(string)` - 计算要用于主节点的引擎机器类型
*   `master_disk_size(int)` - 主节点的磁盘大小
*   `worker_machine_type(string)` - 计算要用于工作节点的引擎计算机类型
*   `worker_disk_size(int)` - 工作节点的磁盘大小
*   `num_preemptible_workers(int)` - 要旋转的可抢占工作节点数
*   `labels(dict)` - 要添加到集群的标签的字典
*   `zone(string)` - 群集所在的区域。（模板）
*   `network_uri(string)` - 用于机器通信的网络 uri，不能用 subnetwork_uri 指定
*   `subnetwork_uri(string)` - 无法使用 network_uri 指定要用于机器通信的子网 uri
*   `internal_ip_only(bool)` - 如果为 true，则群集中的所有实例将只具有内部 IP 地址。这只能为启用子网的网络启用
*   `tags(_list __[ __string __]_)` - 要添加到所有实例的 GCE 标记
*   `地区` - 作为'全球'留下，可能在未来变得相关。（模板）
*   `gcp_conn_id(string)` - 用于连接到 Google Cloud Platform 的连接 ID。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `service_account(string)` - dataproc 实例的服务帐户。
*   `service_account_scopes(_list __[ __string __]_)` - 要包含的服务帐户范围的 URI。
*   `idle_delete_ttl(int)` - 群集在保持空闲状态时保持活动状态的最长持续时间。通过此阈值将导致群集被自动删除。持续时间（秒）。
*   `auto_delete_time(_datetime.datetime_)` - 自动删除群集的时间。
*   `auto_delete_ttl(int)` - 群集的生命周期，群集将在此持续时间结束时自动删除。持续时间（秒）。（如果设置了 auto_delete_time，则将忽略此参数）

```py
class airflow.contrib.operators.dataproc_operator.DataprocClusterScaleOperator（cluster_name，project_id，region ='global'，gcp_conn_id ='google_cloud_default'，delegate_to = None，num_workers = 2，num_preemptible_workers = 0，graceful_decommission_timeout = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在 Google Cloud Dataproc 上进行扩展，向上或向下扩展。操作员将等待，直到重新调整群集。

**示例** ：

```py
 t1 = DataprocClusterScaleOperator（ 
```

task_id ='dataproc_scale'，project_id ='my-project'，cluster_name ='cluster-1'，num_workers = 10，num_preemptible_workers = 10，graceful_decommission_timeout ='1h'dag = dag）

也可以看看

有关扩展群集的更多详细信息，请参阅以下参考：[https](https://cloud.google.com/dataproc/docs/concepts/configuring-clusters/scaling-clusters)：[//cloud.google.com/dataproc/docs/concepts/configuring-clusters/scaling-clusters](https://cloud.google.com/dataproc/docs/concepts/configuring-clusters/scaling-clusters)

参数：

*   `cluster_name(string)` - 要扩展的集群的名称。（模板）
*   `project_id(string)` - 群集运行的 Google 云项目的 ID。（模板）
*   `region(string)` - 数据通路簇的区域。（模板）
*   `gcp_conn_id(string)` - 用于连接到 Google Cloud Platform 的连接 ID。
*   `num_workers(int)` - 新的工人数量
*   `num_preemptible_workers(int)` - 新的可抢占工人数量
*   `graceful_decommission_timeout(string)` - 优雅的 YARN decomissioning 超时。最大值为 1d
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。

```py
class airflow.contrib.operators.dataproc_operator.DataprocClusterDeleteOperator（cluster_name，project_id，region ='global'，gcp_conn_id ='google_cloud_default'，delegate_to = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

删除 Google Cloud Dataproc 上的群集。操作员将等待，直到群集被销毁。

参数：

*   `cluster_name(string)` - 要创建的集群的名称。（模板）
*   `project_id(string)` - 群集运行的 Google 云项目的 ID。（模板）
*   `region(string)` - 保留为“全局”，将来可能会变得相关。（模板）
*   `gcp_conn_id(string)` - 用于连接到 Google Cloud Platform 的连接 ID。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。

```py
class airflow.contrib.operators.dataproc_operator.DataProcPigOperator（query = None，query_uri = None，variables = None，job_name ='{{task.task_id}} _ {{ds_nodash}}'，cluster_name ='cluster-1'，dataproc_pig_properties =无，dataproc_pig_jars =无，gcp_conn_id ='google_cloud_default'，delegate_to =无，region ='全局'，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在 Cloud DataProc 群集上启动 Pig 查询作业。操作的参数将传递给集群。

在 dag 的 default_args 中定义 dataproc_ *参数是一种很好的做法，比如集群名称和 UDF。

```py
 default_args = {
    'cluster_name' : 'cluster-1' ,
    'dataproc_pig_jars' : [
        'gs://example/udf/jar/datafu/1.2.0/datafu.jar' ,
        'gs://example/udf/jar/gpig/1.2/gpig.jar'
    ]
}

```

您可以将 pig 脚本作为字符串或文件引用传递。使用变量传递要在群集上解析的 pig 脚本的变量，或者使用要在脚本中解析的参数作为模板参数。

**示例** ：

```py
 t1 = DataProcPigOperator (
        task_id = 'dataproc_pig' ,
        query = 'a_pig_script.pig' ,
        variables = { 'out' : 'gs://example/output/{{ds}}' },
        dag = dag )

```

也可以看看

有关工作提交的更多详细信息，请参阅以下参考：[https](https://cloud.google.com/dataproc/reference/rest/v1/projects.regions.jobs)：[//cloud.google.com/dataproc/reference/rest/v1/projects.regions.jobs](https://cloud.google.com/dataproc/reference/rest/v1/projects.regions.jobs)

参数：

*   `query(string)` - 对查询文件的查询或引用（pg 或 pig 扩展）。（模板）
*   `query_uri(string)` - 云存储上的猪脚本的 uri。
*   `variables(dict)` - 查询的命名参数的映射。（模板）
*   `job_name(string)` - DataProc 集群中使用的作业名称。默认情况下，此名称是附加执行数据的 task_id，但可以进行模板化。该名称将始终附加一个随机数，以避免名称冲突。（模板）
*   `cluster_name(string)` - DataProc 集群的名称。（模板）
*   `dataproc_pig_properties(dict)` - Pig 属性的映射。非常适合放入默认参数
*   `dataproc_pig_jars(list)` - 在云存储中配置的 jars 的 URI（例如：用于 UDF 和 lib），非常适合放入默认参数。
*   `gcp_conn_id(string)` - 用于连接到 Google Cloud Platform 的连接 ID。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `region(string)` - 创建数据加载集群的指定区域。

```py
class airflow.contrib.operators.dataproc_operator.DataProcHiveOperator（query = None，query_uri = None，variables = None，job_name ='{{task.task_id}} _ {{ds_nodash}}'，cluster_name ='cluster-1'，dataproc_hive_properties =无，dataproc_hive_jars =无，gcp_conn_id ='google_cloud_default'，delegate_to =无，region ='全局'，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在 Cloud DataProc 群集上启动 Hive 查询作业。

参数：

*   `query(string)` - 查询或对查询文件的引用（q 扩展名）。
*   `query_uri(string)` - 云存储上的 hive 脚本的 uri。
*   `variables(dict)` - 查询的命名参数的映射。
*   `job_name(string)` - DataProc 集群中使用的作业名称。默认情况下，此名称是附加执行数据的 task_id，但可以进行模板化。该名称将始终附加一个随机数，以避免名称冲突。
*   `cluster_name(string)` - DataProc 集群的名称。
*   `dataproc_hive_properties(dict)` - Pig 属性的映射。非常适合放入默认参数
*   `dataproc_hive_jars(list)` - 在云存储中配置的 jars 的 URI（例如：用于 UDF 和 lib），非常适合放入默认参数。
*   `gcp_conn_id(string)` - 用于连接到 Google Cloud Platform 的连接 ID。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `region(string)` - 创建数据加载集群的指定区域。

```py
class airflow.contrib.operators.dataproc_operator.DataProcSparkSqlOperator（query = None，query_uri = None，variables = None，job_name ='{{task.task_id}} _ {{ds_nodash}}'，cluster_name ='cluster-1'，dataproc_spark_properties =无，dataproc_spark_jars =无，gcp_conn_id ='google_cloud_default'，delegate_to =无，region ='全局'，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在 Cloud DataProc 集群上启动 Spark SQL 查询作业。

参数：

*   `query(string)` - 查询或对查询文件的引用（q 扩展名）。（模板）
*   `query_uri(string)` - 云存储上的一个 spark sql 脚本的 uri。
*   `variables(dict)` - 查询的命名参数的映射。（模板）
*   `job_name(string)` - DataProc 集群中使用的作业名称。默认情况下，此名称是附加执行数据的 task_id，但可以进行模板化。该名称将始终附加一个随机数，以避免名称冲突。（模板）
*   `cluster_name(string)` - DataProc 集群的名称。（模板）
*   `dataproc_spark_properties(dict)` - Pig 属性的映射。非常适合放入默认参数
*   `dataproc_spark_jars(list)` - 在云存储中配置的 jars 的 URI（例如：用于 UDF 和 lib），非常适合放入默认参数。
*   `gcp_conn_id(string)` - 用于连接到 Google Cloud Platform 的连接 ID。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `region(string)` - 创建数据加载集群的指定区域。

```py
class airflow.contrib.operators.dataproc_operator.DataProcSparkOperator（main_jar = None，main_class = None，arguments = None，archives = None，files = None，job_name ='{{task.task_id}} _ {{ds_nodash}}'，cluster_name ='cluster-1'，dataproc_spark_properties =无，dataproc_spark_jars =无，gcp_conn_id ='google_cloud_default'，delegate_to =无，region ='全局'，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在 Cloud DataProc 群集上启动 Spark 作业。

参数：

*   `main_jar(string)` - 在云存储上配置的作业 jar 的 URI。（使用 this 或 main_class，而不是两者一起）。
*   `main_class(string)` - 作业类的名称。（使用 this 或 main_jar，而不是两者一起）。
*   `arguments(list)` - 作业的参数。（模板）
*   `archives(list)` - 将在工作目录中解压缩的已归档文件列表。应存储在云存储中。
*   `files(list)` - 要复制到工作目录的文件列表
*   `job_name(string)` - DataProc 集群中使用的作业名称。默认情况下，此名称是附加执行数据的 task_id，但可以进行模板化。该名称将始终附加一个随机数，以避免名称冲突。（模板）
*   `cluster_name(string)` - DataProc 集群的名称。（模板）
*   `dataproc_spark_properties(dict)` - Pig 属性的映射。非常适合放入默认参数
*   `dataproc_spark_jars(list)` - 在云存储中配置的 jars 的 URI（例如：用于 UDF 和 lib），非常适合放入默认参数。
*   `gcp_conn_id(string)` - 用于连接到 Google Cloud Platform 的连接 ID。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `region(string)` - 创建数据加载集群的指定区域。

```py
class airflow.contrib.operators.dataproc_operator.DataProcHadoopOperator（main_jar = None，main_class = None，arguments = None，archives = None，files = None，job_name ='{{task.task_id}} _ {{ds_nodash}}'，cluster_name ='cluster-1'，dataproc_hadoop_properties =无，dataproc_hadoop_jars =无，gcp_conn_id ='google_cloud_default'，delegate_to =无，region ='全局'，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在 Cloud DataProc 群集上启动 Hadoop 作业。

参数：

*   `main_jar(string)` - 在云存储上配置的作业 jar 的 URI。（使用 this 或 main_class，而不是两者一起）。
*   `main_class(string)` - 作业类的名称。（使用 this 或 main_jar，而不是两者一起）。
*   `arguments(list)` - 作业的参数。（模板）
*   `archives(list)` - 将在工作目录中解压缩的已归档文件列表。应存储在云存储中。
*   `files(list)` - 要复制到工作目录的文件列表
*   `job_name(string)` - DataProc 集群中使用的作业名称。默认情况下，此名称是附加执行数据的 task_id，但可以进行模板化。该名称将始终附加一个随机数，以避免名称冲突。（模板）
*   `cluster_name(string)` - DataProc 集群的名称。（模板）
*   `dataproc_hadoop_properties(dict)` - Pig 属性的映射。非常适合放入默认参数
*   `dataproc_hadoop_jars(list)` - 在云存储中配置的 jars 的 URI（例如：用于 UDF 和 lib），非常适合放入默认参数。
*   `gcp_conn_id(string)` - 用于连接到 Google Cloud Platform 的连接 ID。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `region(string)` - 创建数据加载集群的指定区域。

```py
class airflow.contrib.operators.dataproc_operator.DataProcPySparkOperator（main，arguments = None，archives = None，pyfiles = None，files = None，job_name ='{{task.task_id}} _ {{ds_nodash}}'，cluster_name =' cluster-1'，dataproc_pyspark_properties = None，dataproc_pyspark_jars = None，gcp_conn_id ='google_cloud_default'，delegate_to = None，region ='global'，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在 Cloud DataProc 群集上启动 PySpark 作业。

参数：

*   `main(string)` - [必需]用作驱动程序的主 Python 文件的 Hadoop 兼容文件系统（HCFS）URI。必须是.py 文件。
*   `arguments(list)` - 作业的参数。（模板）
*   `archives(list)` - 将在工作目录中解压缩的已归档文件列表。应存储在云存储中。
*   `files(list)` - 要复制到工作目录的文件列表
*   `pyfiles(list)` - 要传递给 PySpark 框架的 Python 文件列表。支持的文件类型：.py，.egg 和.zip
*   `job_name(string)` - DataProc 集群中使用的作业名称。默认情况下，此名称是附加执行数据的 task_id，但可以进行模板化。该名称将始终附加一个随机数，以避免名称冲突。（模板）
*   `cluster_name(string)` - DataProc 集群的名称。
*   `dataproc_pyspark_properties(dict)` - Pig 属性的映射。非常适合放入默认参数
*   `dataproc_pyspark_jars(list)` - 在云存储中配置的 jars 的 URI（例如：用于 UDF 和 lib），非常适合放入默认参数。
*   `gcp_conn_id(string)` - 用于连接到 Google Cloud Platform 的连接 ID。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `region(string)` - 创建数据加载集群的指定区域。

```py
class airflow.contrib.operators.dataproc_operator.DataprocWorkflowTemplateBaseOperator（project_id，region ='global'，gcp_conn_id ='google_cloud_default'，delegate_to = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

```py
class airflow.contrib.operators.dataproc_operator.DataprocWorkflowTemplateInstantiateOperator（template_id，* args，** kwargs） 
```

基类： [`airflow.contrib.operators.dataproc_operator.DataprocWorkflowTemplateBaseOperator`](31 "airflow.contrib.operators.dataproc_operator.DataprocWorkflowTemplateBaseOperator")

在 Google Cloud Dataproc 上实例化 WorkflowTemplate。操作员将等待 WorkflowTemplate 完成执行。

也可以看看

请参阅：[https](https://cloud.google.com/dataproc/docs/reference/rest/v1beta2/projects.regions.workflowTemplates/instantiate)：[//cloud.google.com/dataproc/docs/reference/rest/v1beta2/projects.regions.workflowTemplates/instantiate](https://cloud.google.com/dataproc/docs/reference/rest/v1beta2/projects.regions.workflowTemplates/instantiate)

参数：

*   `template_id(string)` - 模板的 id。（模板）
*   `project_id(string)` - 模板运行所在的 Google 云项目的 ID
*   `region(string)` - 保留为“全局”，将来可能会变得相关
*   `gcp_conn_id(string)` - 用于连接到 Google Cloud Platform 的连接 ID。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。

```py
class airflow.contrib.operators.dataproc_operator.DataprocWorkflowTemplateInstantiateInlineOperator（template，* args，** kwargs） 
```

基类： [`airflow.contrib.operators.dataproc_operator.DataprocWorkflowTemplateBaseOperator`](31 "airflow.contrib.operators.dataproc_operator.DataprocWorkflowTemplateBaseOperator")

在 Google Cloud Dataproc 上实例化 WorkflowTemplate 内联。操作员将等待 WorkflowTemplate 完成执行。

也可以看看

请参阅：[https](https://cloud.google.com/dataproc/docs/reference/rest/v1beta2/projects.regions.workflowTemplates/instantiateInline)：[//cloud.google.com/dataproc/docs/reference/rest/v1beta2/projects.regions.workflowTemplates/instantiateInline](https://cloud.google.com/dataproc/docs/reference/rest/v1beta2/projects.regions.workflowTemplates/instantiateInline)

参数：

*   `template(_map_)` - 模板内容。（模板）
*   `project_id(string)` - 模板运行所在的 Google 云项目的 ID
*   `region(string)` - 保留为“全局”，将来可能会变得相关
*   `gcp_conn_id(string)` - 用于连接到 Google Cloud Platform 的连接 ID。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。

```py
class airflow.contrib.operators.datastore_export_operator.DatastoreExportOperator（bucket，namespace = None，datastore_conn_id ='google_cloud_default'，cloud_storage_conn_id ='google_cloud_default'，delegate_to = None，entity_filter = None，labels = None，polling_interval_in_seconds = 10，overwrite_existing = False，xcom_push =假，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

将实体从 Google Cloud Datastore 导出到云存储

参数：

*   `bucket(string)` - 要备份数据的云存储桶的名称
*   `namespace(str)` - 指定云存储桶中用于备份数据的可选命名空间路径。如果 GCS 中不存在此命名空间，则将创建该命名空间。
*   `datastore_conn_id(string)` - 要使用的数据存储区连接 ID 的名称
*   `cloud_storage_conn_id(string)` - 强制写入备份的云存储连接 ID 的名称
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `entity_filter(dict)` - 导出中包含项目中哪些数据的说明，请参阅[https://cloud.google.com/datastore/docs/reference/rest/Shared.Types/EntityFilter](https://cloud.google.com/datastore/docs/reference/rest/Shared.Types/EntityFilter)
*   `labels(dict)` - 客户端分配的云存储标签
*   `polling_interval_in_seconds(int)` - 再次轮询执行状态之前等待的秒数
*   `overwrite_existing(bool)` - 如果存储桶+命名空间不为空，则在导出之前将清空它。这样可以覆盖现有备份。
*   `xcom_push(bool)` - 将操作名称推送到 xcom 以供参考

```py
class airflow.contrib.operators.datastore_import_operator.DatastoreImportOperator（bucket，file，namespace = None，entity_filter = None，labels = None，datastore_conn_id ='google_cloud_default'，delegate_to = None，polling_interval_in_seconds = 10，xcom_push = False，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

将实体从云存储导入 Google Cloud Datastore

参数：

*   `bucket(string)` - 云存储中用于存储数据的容器
*   `file(string)` - 指定云存储桶中备份元数据文件的路径。它应该具有扩展名.overall_export_metadata
*   `namespace(str)` - 指定云存储桶中备份元数据文件的可选命名空间。
*   `entity_filter(dict)` - 导出中包含项目中哪些数据的说明，请参阅[https://cloud.google.com/datastore/docs/reference/rest/Shared.Types/EntityFilter](https://cloud.google.com/datastore/docs/reference/rest/Shared.Types/EntityFilter)
*   `labels(dict)` - 客户端分配的云存储标签
*   `datastore_conn_id(string)` - 要使用的连接 ID 的名称
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `polling_interval_in_seconds(int)` - 再次轮询执行状态之前等待的秒数
*   `xcom_push(bool)` - 将操作名称推送到 xcom 以供参考

```py
class airflow.contrib.operators.discord_webhook_operator.DiscordWebhookOperator（http_conn_id = None，webhook_endpoint = None，message =''，username = None，avatar_url = None，tts = False，proxy = None，* args，** kwargs） 
```

基类： [`airflow.operators.http_operator.SimpleHttpOperator`](31 "airflow.operators.http_operator.SimpleHttpOperator")

此运算符允许您使用传入的 webhook 将消息发布到 Discord。使用默认相对 webhook 端点获取 Discord 连接 ID。可以使用 webhook_endpoint 参数（[https://discordapp.com/developers/docs/resources/webhook](https://discordapp.com/developers/docs/resources/webhook)）覆盖默认端点。

每个 Discord webhook 都可以预先配置为使用特定的用户名和 avatar_url。您可以在此运算符中覆盖这些默认值。

参数：

*   `http_conn_id(str)` - Http 连接 ID，主机为“ [https://discord.com/api/](https://discord.com/api/) ”，默认 webhook 端点在额外字段中，格式为{“webhook_endpoint”：“webhooks / {webhook.id} / { webhook.token}”
*   `webhook_endpoint(str)` - 以“webhooks / {webhook.id} / {webhook.token}”的形式 Discord webhook 端点
*   `message(str)` - 要发送到 Discord 频道的消息（最多 2000 个字符）。（模板）
*   `username(str)` - 覆盖 webhook 的默认用户名。（模板）
*   `avatar_url(str)` - 覆盖 webhook 的默认头像
*   `tts(bool)` - 是一个文本到语音的消息
*   `proxy(str)` - 用于进行 Discord webhook 调用的代理

```py
执行（上下文） 
```

调用 DiscordWebhookHook 发布消息

```py
class airflow.contrib.operators.druid_operator.DruidOperator（json_index_file，druid_ingest_conn_id ='druid_ingest_default'，max_ingestion_time = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

允许直接向德鲁伊提交任务

参数：

*   `json_index_file(str)` - 德鲁伊索引规范的文件路径
*   `druid_ingest_conn_id(str)` - 接受索引作业的德鲁伊霸主的连接 ID

```py
class airflow.contrib.operators.ecs_operator.ECSOperator（task_definition，cluster，overrides，aws_conn_id = None，region_name = None，launch_type ='EC2'，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在 AWS EC2 Container Service 上执行任务

参数：

*   `task_definition(str)` - EC2 容器服务上的任务定义名称
*   `cluster(str)` - EC2 Container Service 上的群集名称
*   `aws_conn_id(str)` - AWS 凭证/区域名称的连接 ID。如果为 None，将使用凭证 boto3 策略（[http://boto3.readthedocs.io/en/latest/guide/configuration.html](http://boto3.readthedocs.io/en/latest/guide/configuration.html)）。
*   `region_name` - 要在 AWS Hook 中使用的区域名称。覆盖连接中的 region_name（如果提供）
*   `launch_type` - 运行任务的启动类型（'EC2'或'FARGATE'）

| 帕拉姆： | 覆盖：boto3 将接收的相同参数（模板化）：[http](http://boto3.readthedocs.org/en/latest/reference/services/ecs.html)：//boto3.readthedocs.org/en/latest/reference/services/ecs.html#ECS.Client.run_task[](http://boto3.readthedocs.org/en/latest/reference/services/ecs.html)类型： 覆盖：dict 类型： launch_type：str

```py
class airflow.contrib.operators.emr_add_steps_operator.EmrAddStepsOperator（job_flow_id，aws_conn_id ='s3_default'，steps = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

向现有 EMR job_flow 添加步骤的运算符。

参数：

*   `job_flow_id` - 要添加步骤的 JobFlow 的 ID。（模板）
*   `aws_conn_id(str)` - 与使用的 aws 连接
*   `步骤(list)` - 要添加到作业流的 boto3 样式步骤。（模板）

```py
class airflow.contrib.operators.emr_create_job_flow_operator.EmrCreateJobFlowOperator（aws_conn_id ='s3_default'，emr_conn_id ='emr_default'，job_flow_overrides = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

创建 EMR JobFlow，从 EMR 连接读取配置。可以传递 JobFlow 覆盖的字典，覆盖连接中的配置。

参数：

*   `aws_conn_id(str)` - 与使用的 aws 连接
*   `emr_conn_id(str)` - 要使用的 emr 连接
*   `job_flow_overrides` - 用于覆盖 emr_connection extra 的 boto3 样式参数。（模板）

```py
class airflow.contrib.operators.emr_terminate_job_flow_operator.EmrTerminateJobFlowOperator（job_flow_id，aws_conn_id ='s3_default'，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

运营商终止 EMR JobFlows。

参数：

*   `job_flow_id` - 要终止的 JobFlow 的 id。（模板）
*   `aws_conn_id(str)` - 与使用的 aws 连接

```py
class airflow.contrib.operators.file_to_gcs.FileToGoogleCloudStorageOperator（src，dst，bucket，google_cloud_storage_conn_id ='google_cloud_default'，mime_type ='application / octet-stream'，delegate_to = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

将文件上传到 Google 云端存储

参数：

*   `src(string)` - 本地文件的路径。（模板）
*   `dst(string)` - 指定存储桶中的目标路径。（模板）
*   `bucket(string)` - 要上传的存储桶。（模板）
*   `google_cloud_storage_conn_id(string)` - 要上传的 Airflow 连接 ID
*   `mime_type(string)` - mime 类型字符串
*   `delegate_to(string)` - 模拟的帐户（如果有）

```py
执行（上下文） 
```

将文件上传到 Google 云端存储

```py
class airflow.contrib.operators.file_to_wasb.FileToWasbOperator（file_path，container_name，blob_name，wasb_conn_id ='wasb_default'，load_options = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

将文件上载到 Azure Blob 存储。

参数：

*   `file_path(str)` - 要加载的文件的路径。（模板）
*   `container_name(str)` - 容器的名称。（模板）
*   `blob_name(str)` - blob 的名称。（模板）
*   `wasb_conn_id(str)` - 对 wasb 连接的引用。
*   `load_options(dict)` - `WasbHook.load_file（）`采用的可选关键字参数。

```py
执行（上下文） 
```

将文件上载到 Azure Blob 存储。

```py
class airflow.contrib.operators.gcp_container_operator.GKEClusterCreateOperator（project_id，location，body = {}，gcp_conn_id ='google_cloud_default'，api_version ='v2'，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

```py
class airflow.contrib.operators.gcp_container_operator.GKEClusterDeleteOperator（project_id，name，location，gcp_conn_id ='google_cloud_default'，api_version ='v2'，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

```py
class airflow.contrib.operators.gcs_download_operator.GoogleCloudStorageDownloadOperator（bucket，object，filename = None，store_to_xcom_key = None，google_cloud_storage_conn_id ='google_cloud_default'，delegate_to = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

从 Google 云端存储下载文件。

参数：

*   `bucket(string)` - 对象所在的 Google 云存储桶。（模板）
*   `object(string)` - 要在 Google 云存储桶中下载的对象的名称。（模板）
*   `filename(string)` - 应将文件下载到的本地文件系统（正在执行操作符的位置）上的文件路径。（模板化）如果未传递文件名，则下载的数据将不会存储在本地文件系统中。
*   `store_to_xcom_key(string)` - 如果设置了此参数，操作员将使用此参数中设置的键将下载文件的内容推送到 XCom。如果未设置，则下载的数据不会被推送到 XCom。（模板）
*   `google_cloud_storage_conn_id(string)` - 连接到 Google 云端存储时使用的连接 ID。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。

```py
class airflow.contrib.operators.gcslistoperator.GoogleCloudStorageListOperator（bucket，prefix = None，delimiter = None，google_cloud_storage_conn_id ='google_cloud_default'，delegate_to = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

使用名称中的给定字符串前缀和分隔符列出存储桶中的所有对象。

```py
 此运算符返回一个 python 列表，其中包含可供其使用的对象的名称 
```

<cite>xcom</cite>在下游任务中。

参数：

*   `bucket(string)` - 用于查找对象的 Google 云存储桶。（模板）
*   `prefix(string)` - 前缀字符串，用于过滤名称以此前缀开头的对象。（模板）
*   `delimiter(string)` - 要过滤对象的分隔符。（模板化）例如，要列出 GCS 目录中的 CSV 文件，您可以使用 delimiter ='。csv'。
*   `google_cloud_storage_conn_id(string)` - 连接到 Google 云端存储时使用的连接 ID。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。

```py
 Example: 
```

以下运算符将列出存储桶中文件`sales/sales-2017`夹中的所有 Avro 文件`data`。

```py
 GCS_Files = GoogleCloudStorageListOperator (
    task_id = 'GCS_Files' ,
    bucket = 'data' ,
    prefix = 'sales/sales-2017/' ,
    delimiter = '.avro' ,
    google_cloud_storage_conn_id = google_cloud_conn_id
)

```

```py
class airflow.contrib.operators.gcs_operator.GoogleCloudStorageCreateBucketOperator（bucket_name，storage_class ='MULTI_REGIONAL'，location ='US'，project_id = None，labels = None，google_cloud_storage_conn_id ='google_cloud_default'，delegate_to = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

创建一个新存储桶。Google 云端存储使用平面命名空间，因此您无法创建名称已在使用中的存储桶。

> 也可以看看
> 
> 有关详细信息，请参阅存储桶命名指南：[https](https://cloud.google.com/storage/docs/bucketnaming.html)：[//cloud.google.com/storage/docs/bucketnaming.html#requirements](https://cloud.google.com/storage/docs/bucketnaming.html)

参数：

*   `bucket_name(string)` - 存储桶的名称。（模板）
*   `storage_class(string)` -

    这定义了存储桶中对象的存储方式，并确定了 SLA 和存储成本（模板化）。价值包括

    *   `MULTI_REGIONAL`
    *   `REGIONAL`
    *   `STANDARD`
    *   `NEARLINE`
    *   `COLDLINE` 。

    如果在创建存储桶时未指定此值，则默认为 STANDARD。

*   `位置(string)` -

    水桶的位置。（模板化）存储桶中对象的对象数据驻留在此区域内的物理存储中。默认为美国。

    也可以看看

    [https://developers.google.com/storage/docs/bucket-locations](https://developers.google.com/storage/docs/bucket-locations)

*   `project_id(string)` - GCP 项目的 ID。（模板）
*   `labels(dict)` - 用户提供的键/值对标签。
*   `google_cloud_storage_conn_id(string)` - 连接到 Google 云端存储时使用的连接 ID。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。

```py
 Example: 
```

以下运算符将在区域中创建`test-bucket`具有`MULTI_REGIONAL`存储类的新存储桶`EU`

```py
 CreateBucket = GoogleCloudStorageCreateBucketOperator (
    task_id = 'CreateNewBucket' ,
    bucket_name = 'test-bucket' ,
    storage_class = 'MULTI_REGIONAL' ,
    location = 'EU' ,
    labels = { 'env' : 'dev' , 'team' : 'airflow' },
    google_cloud_storage_conn_id = 'airflow-service-account'
)

```

```py
class airflow.contrib.operators.gcs_to_bq.GoogleCloudStorageToBigQueryOperator（bucket，source_objects，destination_project_dataset_table，schema_fields = None，schema_object = None，source_format ='CSV'，compression ='NONE'，create_disposition ='CREATE_IF_NEEDED'，skip_leading_rows = 0，write_disposition =' WRITE_EMPTY'，field_delimiter ='，'，max_bad_records = 0，quote_character = None，ignore_unknown_values = False，allow_quoted_newlines = False，allow_jagged_rows = False，max_id_key = None，bigquery_conn_id ='bigquery_default'，google_cloud_storage_conn_id ='google_cloud_default'，delegate_to = None，schema_update_options =（），src_fmt_configs = {}，external_table = False，time_partitioning = {}，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

将文件从 Google 云存储加载到 BigQuery 中。

可以用两种方法之一指定用于 BigQuery 表的模式。您可以直接传递架构字段，也可以将运营商指向 Google 云存储对象名称。Google 云存储中的对象必须是包含架构字段的 JSON 文件。

参数：

*   `bucket(string)` - 要加载的桶。（模板）
*   `source_objects` - 要加载的 Google 云存储 URI 列表。（模板化）如果 source_format 是'DATASTORE_BACKUP'，则列表必须只包含一个 URI。
*   `destination_project_dataset_table(string)` - 用于加载数据的虚线（&lt;project&gt;。）&lt;dataset&gt;。&lt;table&gt; BigQuery 表。如果未包含&lt;project&gt;，则项目将是连接 json 中定义的项目。（模板）
*   `schema_fields(list)` - 如果设置，则此处定义的架构字段列表：[https](https://cloud.google.com/bigquery/docs/reference/v2/jobs)：**//cloud.google.com/bigquery/docs/reference/v2/jobs#configuration.load**当 source_format 为'DATASTORE_BACKUP'时，不应设置。
*   `schema_object` - 如果设置，则指向包含表的架构的.json 文件的 GCS 对象路径。（模板）
*   `schema_object` - 字符串
*   `source_format(string)` - 要导出的文件格式。
*   `compression(string)` - [可选]数据源的压缩类型。可能的值包括 GZIP 和 NONE。默认值为 NONE。Google Cloud Bigtable，Google Cloud Datastore 备份和 Avro 格式会忽略此设置。
*   `create_disposition(string)` - 如果表不存在，则创建处置。
*   `skip_leading_rows(int)` - 从 CSV 加载时要跳过的行数。
*   `write_disposition(string)` - 表已存在时的写处置。
*   `field_delimiter(string)` - 从 CSV 加载时使用的分隔符。
*   `max_bad_records(int)` - BigQuery 在运行作业时可以忽略的最大错误记录数。
*   `quote_character(string)` - 用于引用 CSV 文件中数据部分的值。
*   `ignore_unknown_values(bool)` - [可选]指示 BigQuery 是否应允许表模式中未表示的额外值。如果为 true，则忽略额外值。如果为 false，则将具有额外列的记录视为错误记录，如果错误记录太多，则在作业结果中返回无效错误。
*   `allow_quoted_newlines(_boolean_)` - 是否允许引用的换行符（true）或不允许（false）。
*   `allow_jagged_rows(bool)` - 接受缺少尾随可选列的行。缺失值被视为空值。如果为 false，则缺少尾随列的记录将被视为错误记录，如果错误记录太多，则会在作业结果中返回无效错误。仅适用于 CSV，忽略其他格式。
*   `max_id_key(string)` - 如果设置，则是 BigQuery 表中要加载的列的名称。在加载发生后，Thsi 将用于从 BigQuery 中选择 MAX 值。结果将由 execute（）命令返回，该命令又存储在 XCom 中供将来的操作员使用。这对增量加载很有帮助 - 在将来的执行过程中，您可以从最大 ID 中获取。
*   `bigquery_conn_id(string)` - 对特定 BigQuery 挂钩的引用。
*   `google_cloud_storage_conn_id(string)` - 对特定 Google 云存储挂钩的引用。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `schema_update_options(list)` - 允许更新目标表的模式作为加载作业的副作用。
*   `src_fmt_configs(dict)` - 配置特定于源格式的可选字段
*   `external_table(bool)` - 用于指定目标表是否应为 BigQuery 外部表的标志。默认值为 False。
*   `time_partitioning(dict)` - 配置可选的时间分区字段，即按 API 规范按字段，类型和到期分区。请注意，“field”在 dataset.table $ partition 的并发中不可用。

```py
class airflow.contrib.operators.gcs_to_gcs.GoogleCloudStorageToGoogleCloudStorageOperator（source_bucket，source_object，destination_bucket = None，destination_object = None，move_object = False，google_cloud_storage_conn_id ='google_cloud_default'，delegate_to = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

将对象从存储桶复制到另一个存储桶，并在需要时重命名。

参数：

*   `source_bucket(string)` - 对象所在的源 Google 云存储桶。（模板）
*   `source_object(string)` -

    要在 Google 云存储分区中复制的对象的源名称。（模板化）如果在此参数中使用通配符：

    &gt; 您只能在存储桶中使用一个通配符作为对象（文件名）。通配符可以出现在对象名称内或对象名称的末尾。不支持在存储桶名称中附加通配符。

*   `destination_bucket` - 目标 Google 云端存储分区


对象应该在哪里。（模板化）：type destination_bucket：string：param destination_object：对象的目标名称

> 目标 Google 云存储桶。（模板化）如果在 source_object 参数中提供了通配符，则这是将添加到最终目标对象路径的前缀。请注意，将删除通配符之前的源路径部分; 如果需要保留，则应将其附加到 destination_object。例如，使用 prefix `foo/*`和 destination_object'blah `/``，文件`foo/baz`将被复制到`blah/baz`; 保留前缀写入 destination_object，例如`blah/foo`，在这种情况下，复制的文件将被命名`blah/foo/baz`。

参数：`move_object` - 当移动对象为 True 时，移动对象

```py
 复制到新位置。 
```

这相当于 mv 命令而不是 cp 命令。

参数：

*   `google_cloud_storage_conn_id(string)` - 连接到 Google 云端存储时使用的连接 ID。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。

```py
 Examples: 
```

下面的操作将命名一个文件复制`sales/sales-2017/january.avro`在`data`桶的文件和名为斗`copied_sales/2017/january-backup.avro` in the ``data_backup`

```py
 copy_single_file = GoogleCloudStorageToGoogleCloudStorageOperator (
    task_id = 'copy_single_file' ,
    source_bucket = 'data' ,
    source_object = 'sales/sales-2017/january.avro' ,
    destination_bucket = 'data_backup' ,
    destination_object = 'copied_sales/2017/january-backup.avro' ,
    google_cloud_storage_conn_id = google_cloud_conn_id
)

```

以下运算符会将文件`sales/sales-2017`夹中的所有 Avro 文件（即名称以该前缀开头）复制到存储`data`桶中的`copied_sales/2017`文件夹中`data_backup`。

```py
 copy_files = GoogleCloudStorageToGoogleCloudStorageOperator (
    task_id = 'copy_files' ,
    source_bucket = 'data' ,
    source_object = 'sales/sales-2017/*.avro' ,
    destination_bucket = 'data_backup' ,
    destination_object = 'copied_sales/2017/' ,
    google_cloud_storage_conn_id = google_cloud_conn_id
)

```

以下运算符会将文件`sales/sales-2017`夹中的所有 Avro 文件（即名称以该前缀开头）移动到`data`存储桶中的同一文件夹`data_backup`，删除过程中的原始文件。

```py
 move_files = GoogleCloudStorageToGoogleCloudStorageOperator (
    task_id = 'move_files' ,
    source_bucket = 'data' ,
    source_object = 'sales/sales-2017/*.avro' ,
    destination_bucket = 'data_backup' ,
    move_object = True ,
    google_cloud_storage_conn_id = google_cloud_conn_id
)

```

```py
class airflow.contrib.operators.gcs_to_s3.GoogleCloudStorageToS3Operator（bucket，prefix = None，delimiter = None，google_cloud_storage_conn_id ='google_cloud_storage_default'，delegate_to = None，dest_aws_conn_id = None，dest_s3_key = None，replace = False，* args，** kwargs） 
```

基类： [`airflow.contrib.operators.gcslistoperator.GoogleCloudStorageListOperator`](integration.html "airflow.contrib.operators.gcslistoperator.GoogleCloudStorageListOperator")

将 Google 云端存储分区与 S3 分区同步。

参数：

*   `bucket(string)` - 用于查找对象的 Google Cloud Storage 存储桶。（模板）
*   `prefix(string)` - 前缀字符串，用于过滤名称以此前缀开头的对象。（模板）
*   `delimiter(string)` - 要过滤对象的分隔符。（模板化）例如，要列出 GCS 目录中的 CSV 文件，您可以使用 delimiter ='。csv'。
*   `google_cloud_storage_conn_id(string)` - 连接到 Google 云端存储时使用的连接 ID。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `dest_aws_conn_id(str)` - 目标 S3 连接
*   `dest_s3_key(str)` - 用于存储文件的基本 S3 密钥。（模板）

```py
class airflow.contrib.operators.hipchat_operator.HipChatAPIOperator（token，base_url ='https：//api.hipchat.com/v2'，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

基础 HipChat 运算符。所有派生的 HipChat 运营商都参考了 HipChat 的官方 REST API 文档，[网址](https://www.hipchat.com/docs/apiv2)为[https://www.hipchat.com/docs/apiv2](https://www.hipchat.com/docs/apiv2)。在使用任何 HipChat API 运算符之前，您需要在[https://www.hipchat.com/docs/apiv2/auth](https://www.hipchat.com/docs/apiv2/auth)获取身份验证令牌。在未来，其他 HipChat 运算符也将从此类派生。

参数：

*   `token(str)` - HipChat REST API 身份验证令牌
*   `base_url(str)` - HipChat REST API 基本 URL。

```py
prepare_request（） 
```

由 execute 函数使用。设置 HipChat 的 REST API 调用的请求方法，URL 和正文。在子类中重写。每个 HipChatAPI 子运算符负责具有设置 self.method，self.url 和 self.body 的 prepare_request 方法调用。

```py
class airflow.contrib.operators.hipchat_operator.HipChatAPISendRoomNotificationOperator（room_id，message，* args，** kwargs） 
```

基类： [`airflow.contrib.operators.hipchat_operator.HipChatAPIOperator`](31 "airflow.contrib.operators.hipchat_operator.HipChatAPIOperator")

将通知发送到特定的 HipChat 会议室。更多信息：[https](https://www.hipchat.com/docs/apiv2/method/send_room_notification)：[//www.hipchat.com/docs/apiv2/method/send_room_notification](https://www.hipchat.com/docs/apiv2/method/send_room_notification)

参数：

*   `room_id(str)` - 在 HipChat 上发送通知的房间。（模板）
*   `message(str)` - 邮件正文。（模板）
*   `frm(str)` - 除发件人姓名外还要显示的标签
*   `message_format(str)` - 如何呈现通知：html 或文本
*   `color(str)` - msg 的背景颜色：黄色，绿色，红色，紫色，灰色或随机
*   `attach_to(str)` - 将此通知附加到的消息 ID
*   `notify(bool)` - 此消息是否应触发用户通知
*   `card(dict)` - HipChat 定义的卡片对象

```py
prepare_request（） 
```

由 execute 函数使用。设置 HipChat 的 REST API 调用的请求方法，URL 和正文。在子类中重写。每个 HipChatAPI 子运算符负责具有设置 self.method，self.url 和 self.body 的 prepare_request 方法调用。

```py
class airflow.contrib.operators.hive_to_dynamodb.HiveToDynamoDBTransferOperator（sql，table_name，table_keys，pre_process = None，pre_process_args = None，pre_process_kwargs = None，region_name = None，schema ='default'，hiveserver2_conn_id ='hiveserver2_default'，aws_conn_id ='aws_default' ，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

将数据从 Hive 移动到 DynamoDB，请注意，现在数据在被推送到 DynamoDB 之前已加载到内存中，因此该运算符应该用于少量数据。

参数：

*   `sql(str)` - 针对 hive 数据库执行的 SQL 查询。（模板）
*   `table_name(str)` - 目标 DynamoDB 表
*   `table_keys(list)` - 分区键和排序键
*   `pre_process(function)` - 实现源数据的预处理
*   `pre_process_args(list)` - pre_process 函数参数列表
*   `pre_process_kwargs(dict)` - pre_process 函数参数的字典
*   `region_name(str)` - aws 区域名称（例如：us-east-1）
*   `schema(str)` - hive 数据库模式
*   `hiveserver2_conn_id(str)` - 源配置单元连接
*   `aws_conn_id(str)` - aws 连接

```py
class airflow.contrib.operators.jenkins_job_trigger_operator.JenkinsJobTriggerOperator（jenkins_connection_id，job_name，parameters =''，sleep_time = 10，max_try_before_job_appears = 10，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

触发 Jenkins 作业并监视它的执行情况。此运算符依赖于 python-jenkins 库，版本&gt; = 0.4.15 与 jenkins 服务器通信。您还需要在连接屏幕中配置 Jenkins 连接。：param jenkins_connection_id：用于此作业的 jenkins 连接：type jenkins_connection_id：string：param job_name：要触发的作业的名称：type job_name：string：param parameters：要提供给 jenkins 的参数块。（模板化）：类型参数：string：param sleep_time：操作员在作业的每个状态请求之间休眠多长时间（min 1，默认值为 10）：type sleep_time：int：param max_try_before_job_appears：要发出的最大请求数

> 等待作业出现在 jenkins 服务器上（默认为 10）

```py
build_job（jenkins_server） 
```

此函数对 Jenkins 进行 API 调用以触发'job_name'的构建。它返回了一个带有 2 个键的字典：body 和 headers。headers 还包含一个类似 dict 的对象，可以查询该对象以获取队列中轮询的位置。：param jenkins_server：应该触发作业的 jenkins 服务器：return：包含响应正文（密钥正文）和标题出现的标题（标题）

```py
poll_job_in_queue（location，jenkins_server） 
```

此方法轮询 jenkins 队列，直到执行作业。当我们通过 API 调用触发作业时，首先将作业放入队列而不分配内部版本号。因此，我们必须等待作业退出队列才能知道其内部版本号。为此，我们必须将/ api / json（或/ api / xml）添加到 build_job 调用返回的位置并轮询该文件。当 json 中出现'可执行'块时，表示作业执行已开始，字段'number'则包含内部版本号。 ：param location：轮询的位置，在 build_job 调用的标头中返回：param jenkins_server：要轮询的 jenkins 服务器：return：与触发的作业对应的 build_number

```py
class airflow.contrib.operators.jira_operator.JiraOperator（jira_conn_id ='jira_default'，jira_method = None，jira_method_args = None，result_processor = None，get_jira_resource_method = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

JiraOperator 在 Jira 问题跟踪系统上进行交互并执行操作。此运算符旨在使用 Jira Python SDK：[http](http://jira.readthedocs.io/)：[//jira.readthedocs.io](http://jira.readthedocs.io/)

参数：

*   `jira_conn_id(str)` - 对预定义的 Jira 连接的引用
*   `jira_method(str)` - 要调用的 Jira Python SDK 中的方法名称
*   `jira_method_args(dict)` - **jira_method**所需的方法参数。（模板）
*   `result_processor(function)` - 用于进一步处理 Jira 响应的函数
*   `get_jira_resource_method(function)` - 用于获取将在其上执行提供的 jira_method 的 jira 资源的函数或运算符

```py
class airflow.contrib.operators.kubernetes_pod_operator.KubernetesPodOperator（namespace，image，name，cmds = None，arguments = None，volume_mounts = None，volumes = None，env_vars = None，secrets = None，in_cluster = False，cluster_context = None，labels = None，startup_timeout_seconds = 120，get_logs = True，image_pull_policy ='IfNotPresent'，annotations = None，resources = None，affinity = None，config_file = None，xcom_push = False，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在 Kubernetes Pod 中执行任务

参数：

*   `image(str)` - 您希望启动的 Docker 镜像。默认为 dockerhub.io，但完全限定的 URLS 将指向自定义存储库
*   `cmds(list[str])` - 容器的入口点。（模板化）如果未提供，则使用泊坞窗图像的入口点。
*   `arguments(list[str])` - 入口点的参数。（模板化）如果未提供，则使用泊坞窗图像的 CMD。
*   `volume_mounts(_VolumeMount 列表 _)` - 已启动 pod 的 volumeMounts
*   `卷(**卷**list)` - 已启动 pod 的卷。包括 ConfigMaps 和 PersistentVolumes
*   `labels(dict)` - 要应用于 Pod 的标签
*   `startup_timeout_seconds(int)` - 启动 pod 的超时时间（秒）
*   `name(str)` - 要运行的任务的名称，将用于生成 pod id
*   `env_vars(dict)` - 在容器中初始化的环境变量。（模板）
*   `秘密(**秘密**list)` - Kubernetes 秘密注入容器中，它们可以作为环境变量或卷中的文件公开。
*   `in_cluster(bool)` - 使用 in_cluster 配置运行 kubernetes 客户端
*   `cluster_context(string)` - 指向 kubernetes 集群的上下文。当 in_cluster 为 True 时忽略。如果为 None，则使用 current-context。
*   `get_logs(bool)` - 获取容器的标准输出作为任务的日志
*   `affinity(dict)` - 包含一组关联性调度规则的 dict
*   `config_file(str)` - Kubernetes 配置文件的路径
*   `xcom_push(bool)` - 如果 xcom_push 为 True，容器中的文件/airflow/xcom/return.json 的内容也将在容器完成时被推送到 XCom。

| 帕拉姆： | namespace：在 kubernetes 中运行的命名空间类型： namespace：str

```py
class airflow.contrib.operators.mlengine_operator.MLEngineBatchPredictionOperator（project_id，job_id，region，data_format，input_paths，output_path，model_name = None，version_name = None，uri = None，max_worker_count = None，runtime_version = None，gcp_conn_id ='google_cloud_default'，delegate_to =无，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

启动 Google Cloud ML Engine 预测作业。

注意：对于模型原点，用户应该考虑以下三个选项中的一个：1。仅填充“uri”字段，该字段应该是指向 tensorflow savedModel 目录的 GCS 位置。2.仅填充'model_name'字段，该字段引用现有模型，并将使用模型的默认版本。3.填充“model_name”和“version_name”字段，这些字段指特定模型的特定版本。

在选项 2 和 3 中，模型和版本名称都应包含最小标识符。例如，打电话

```py
 MLEngineBatchPredictionOperator (
    ... ,
    model_name = 'my_model' ,
    version_name = 'my_version' ,
    ... )

```

如果所需的型号版本是“projects / my_project / models / my_model / versions / my_version”。

有关参数的更多文档，请参阅[https://cloud.google.com/ml-engine/reference/rest/v1/projects.jobs](https://cloud.google.com/ml-engine/reference/rest/v1/projects.jobs)。

参数：

*   `project_id(string)` - 提交预测作业的 Google Cloud 项目名称。（模板）
*   `job_id(string)` - Google Cloud ML Engine 上预测作业的唯一 ID。（模板）
*   `data_format(string)` - 输入数据的格式。如果未提供或者不是[“TEXT”，“TF_RECORD”，“TF_RECORD_GZIP”]之一，它将默认为“DATA_FORMAT_UNSPECIFIED”。
*   `input_paths(_ 字符串列表 _)` - 批量预测的输入数据的 GCS 路径列表。接受通配符运算符[*](31)，但仅限于结尾处。（模板）
*   `output_path(string)` - 写入预测结果的 GCS 路径。（模板）
*   `region(string)` - 用于运行预测作业的 Google Compute Engine 区域。（模板化）
*   `model_name(string)` - 用于预测的 Google Cloud ML Engine 模型。如果未提供 version_name，则将使用此模型的默认版本。如果提供了 version_name，则不应为 None。如果提供 uri，则应为 None。（模板）
*   `version_name(string)` - 用于预测的 Google Cloud ML Engine 模型版本。如果提供 uri，则应为 None。（模板）
*   `uri(string)` - 用于预测的已保存模型的 GCS 路径。如果提供了 model_name，则应为 None。它应该是指向张量流 SavedModel 的 GCS 路径。（模板）
*   `max_worker_count(int)` - 用于并行处理的最大 worker 数。如果未指定，则默认为 10。
*   `runtime_version(string)` - 用于批量预测的 Google Cloud ML Engine 运行时版本。
*   `gcp_conn_id(string)` - 用于连接到 Google Cloud Platform 的连接 ID。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用 doamin 范围的委派。

```py
 Raises: 
```

`ValueError` ：如果无法确定唯一的模型/版本来源。

```py
class airflow.contrib.operators.mlengine_operator.MLEngineModelOperator（project_id，model，operation ='create'，gcp_conn_id ='google_cloud_default'，delegate_to = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

管理 Google Cloud ML Engine 模型的运营商。

参数：

*   `project_id(string)` - MLEngine 模型所属的 Google Cloud 项目名称。（模板）
*   `型号(_ 字典 _)` -

    包含有关模型信息的字典。如果`操作`是`create`，则`model`参数应包含有关此模型的所有信息，例如`name`。

    如果`操作`是`get`，则`model`参数应包含`模型`的`名称`。

*   `操作` -

    执行的操作。可用的操作是：

    *   `create`：创建`model`参数提供的新模型。
    *   `get`：获取在模型中指定名称的特定`模型`。
*   `gcp_conn_id(string)` - 获取连接信息时使用的连接 ID。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。

```py
class airflow.contrib.operators.mlengine_operator.MLEngineVersionOperator（project_id，model_name，version_name = None，version = None，operation ='create'，gcp_conn_id ='google_cloud_default'，delegate_to = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

管理 Google Cloud ML Engine 版本的运营商。

参数：

*   `project_id(string)` - MLEngine 模型所属的 Google Cloud 项目名称。
*   `model_name(string)` - 版本所属的 Google Cloud ML Engine 模型的名称。（模板）
*   `version_name(string)` - 用于正在操作的版本的名称。如果没有人及`版本`的说法是没有或不具备的值`名称`键，那么这将是有效载荷中用于填充`名称`键。 （模板）
*   `version(dict)` - 包含版本信息的字典。如果`操作`是`create`，则`version`应包含有关此版本的所有信息，例如 name 和 deploymentUrl。如果`操作`是`get`或`delete`，则`version`参数应包含`版本`的`名称`。如果是 None，则唯一可能的`操作`是`list`。（模板）
*   `操作(string)` -

    执行的操作。可用的操作是：

    *   `create`：在`model_name`指定的`模型中`创建新版本，在这种情况下，`version`参数应包含创建该版本的所有信息（例如`name`，`deploymentUrl`）。
    *   `get`：获取`model_name`指定的`模型中`特定版本的完整信息。应在`version`参数中指定版本的名称。
    *   `list`：列出`model_name`指定的`模型的`所有可用版本。
    *   `delete`：从`model_name`指定的`模型中`删除`version`参数中指定的`版本`。应在`version`参数中指定版本的名称。
*   `gcp_conn_id(string)` - 获取连接信息时使用的连接 ID。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。

```py
class airflow.contrib.operators.mlengine_operator.MLEngineTrainingOperator（project_id，job_id，package_uris，training_python_module，training_args，region，scale_tier = None，runtime_version = None，python_version = None，job_dir = None，gcp_conn_id ='google_cloud_default'，delegate_to = None，mode ='生产'，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

启动 MLEngine 培训工作的操作员。

参数：

*   `project_id(string)` - 应在其中运行 MLEngine 培训作业的 Google Cloud 项目名称（模板化）。
*   `job_id(string)` - 提交的 Google MLEngine 培训作业的唯一模板化 ID。（模板）
*   `package_uris(string)` - MLEngine 培训作业的包位置列表，其中应包括主要培训计划+任何其他依赖项。（模板）
*   `training_python_module(string)` - 安装'package_uris'软件包后，在 MLEngine 培训作业中运行的 Python 模块名称。（模板）
*   `training_args(string)` - 传递给 MLEngine 训练程序的模板化命令行参数列表。（模板）
*   `region(string)` - 用于运行 MLEngine 培训作业的 Google Compute Engine 区域（模板化）。
*   `scale_tier(string)` - MLEngine 培训作业的资源层。（模板）
*   `runtime_version(string)` - 用于培训的 Google Cloud ML 运行时版本。（模板）
*   `python_version(string)` - 训练中使用的 Python 版本。（模板）
*   `job_dir(string)` - 用于存储培训输出和培训所需的其他数据的 Google 云端存储路径。（模板）
*   `gcp_conn_id(string)` - 获取连接信息时使用的连接 ID。
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `mode(string)` - 可以是'DRY_RUN'/'CLOUD'之一。在“DRY_RUN”模式下，不会启动真正的培训作业，但会打印出 MLEngine 培训作业请求。在“CLOUD”模式下，将发出真正的 MLEngine 培训作业创建请求。

```py
class airflow.contrib.operators.mongo_to_s3.MongoToS3Operator（mongo_conn_id，s3_conn_id，mongo_collection，mongo_query，s3_bucket，s3_key，mongo_db = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

```py
 Mongo  - > S3 
```

更具体的 baseOperator 意味着将数据从 mongo 通过 pymongo 移动到 s3 通过 boto

```py
 需要注意的事项 
```

.execute（）编写依赖于.transform（）。transnsform（）旨在由子类扩展，以执行特定于运算符需求的转换

```py
执行（上下文） 
```

由 task_instance 在运行时执行

```py
变换（文档） 
```

```py
 处理 pyMongo 游标并返回每个元素都在的 iterable 
```

一个 JSON 可序列化的字典

Base transform（）假设不需要处理，即。docs 是文档的 pyMongo 游标，只需要传递游标

重写此方法以进行自定义转换

```py
class airflow.contrib.operators.mysql_to_gcs.MySqlToGoogleCloudStorageOperator（sql，bucket，filename，schema_filename = None，approx_max_file_size_bytes = 1900000000，mysql_conn_id ='mysql_default'，google_cloud_storage_conn_id ='google_cloud_default'，schema = None，delegate_to = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

将数据从 MySQL 复制到 JSON 格式的 Google 云存储。

```py
classmethod type_map（mysql_type） 
```

从 MySQL 字段映射到 BigQuery 字段的辅助函数。在设置 schema_filename 时使用。

```py
class airflow.contrib.operators.postgres_to_gcs_operator.PostgresToGoogleCloudStorageOperator（sql，bucket，filename，schema_filename = None，approx_max_file_size_bytes = 1900000000，po​​stgres_conn_id ='postgres_default'，google_cloud_storage_conn_id ='google_cloud_default'，delegate_to = None，parameters = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

将数据从 Postgres 复制到 JSON 格式的 Google 云端存储。

```py
classmethod convert_types（value） 
```

从 Postgres 获取值，并将其转换为对 JSON / Google Cloud Storage / BigQuery 安全的值。日期转换为 UTC 秒。小数转换为浮点数。时间转换为秒。

```py
classmethod type_map（postgres_type） 
```

从 Postgres 字段映射到 BigQuery 字段的 Helper 函数。在设置 schema_filename 时使用。

```py
class airflow.contrib.operators.pubsub_operator.PubSubTopicCreateOperator（project，topic，fail_if_exists = False，gcp_conn_id ='google_cloud_default'，delegate_to = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

创建 PubSub 主题。

默认情况下，如果主题已存在，则此运算符不会导致 DAG 失败。

```py
 with DAG ( 'successful DAG' ) as dag :
    (
        dag
        >> PubSubTopicCreateOperator ( project = 'my-project' ,
                                     topic = 'my_new_topic' )
        >> PubSubTopicCreateOperator ( project = 'my-project' ,
                                     topic = 'my_new_topic' )
    )

```

如果主题已存在，则可以将操作员配置为失败。

```py
 with DAG ( 'failing DAG' ) as dag :
    (
        dag
        >> PubSubTopicCreateOperator ( project = 'my-project' ,
                                     topic = 'my_new_topic' )
        >> PubSubTopicCreateOperator ( project = 'my-project' ,
                                     topic = 'my_new_topic' ,
                                     fail_if_exists = True )
    )

```

这两个`project`和`topic`模板化，所以你可以在其中使用变量。

```py
class airflow.contrib.operators.pubsub_operator.PubSubTopicDeleteOperator（project，topic，fail_if_not_exists = False，gcp_conn_id ='google_cloud_default'，delegate_to = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

删除 PubSub 主题。

默认情况下，如果主题不存在，则此运算符不会导致 DAG 失败。

```py
 with DAG ( 'successful DAG' ) as dag :
    (
        dag
        >> PubSubTopicDeleteOperator ( project = 'my-project' ,
                                     topic = 'non_existing_topic' )
    )

```

如果主题不存在，则可以将运算符配置为失败。

```py
 with DAG ( 'failing DAG' ) as dag :
    (
        dag
        >> PubSubTopicCreateOperator ( project = 'my-project' ,
                                     topic = 'non_existing_topic' ,
                                     fail_if_not_exists = True )
    )

```

这两个`project`和`topic`模板化，所以你可以在其中使用变量。

```py
class airflow.contrib.operators.pubsub_operator.PubSubSubscriptionCreateOperator（topic_project，topic，subscription = None，subscription_project = None，ack_deadline_secs = 10，fail_if_exists = False，gcp_conn_id ='google_cloud_default'，delegate_to = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

创建 PubSub 订阅。

默认情况下，将在中创建订阅`topic_project`。如果`subscription_project`已指定且 GCP 凭据允许，则可以在与其主题不同的项目中创建订阅。

默认情况下，如果订阅已存在，则此运算符不会导致 DAG 失败。但是，该主题必须存在于项目中。

```py
 with DAG ( 'successful DAG' ) as dag :
    (
        dag
        >> PubSubSubscriptionCreateOperator (
            topic_project = 'my-project' , topic = 'my-topic' ,
            subscription = 'my-subscription' )
        >> PubSubSubscriptionCreateOperator (
            topic_project = 'my-project' , topic = 'my-topic' ,
            subscription = 'my-subscription' )
    )

```

如果订阅已存在，则可以将运算符配置为失败。

```py
 with DAG ( 'failing DAG' ) as dag :
    (
        dag
        >> PubSubSubscriptionCreateOperator (
            topic_project = 'my-project' , topic = 'my-topic' ,
            subscription = 'my-subscription' )
        >> PubSubSubscriptionCreateOperator (
            topic_project = 'my-project' , topic = 'my-topic' ,
            subscription = 'my-subscription' , fail_if_exists = True )
    )

```

最后，不需要订阅。如果未通过，则运营商将为订阅的名称生成通用唯一标识符。

```py
 with DAG ( 'DAG' ) as dag :
    (
        dag >> PubSubSubscriptionCreateOperator (
            topic_project = 'my-project' , topic = 'my-topic' )
    )

```

`topic_project`,, `topic`和`subscription`，并且`subscription`是模板化的，因此您可以在其中使用变量。

```py
class airflow.contrib.operators.pubsub_operator.PubSubSubscriptionDeleteOperator（project，subscription，fail_if_not_exists = False，gcp_conn_id ='google_cloud_default'，delegate_to = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

删除 PubSub 订阅。

默认情况下，如果订阅不存在，则此运算符不会导致 DAG 失败。

```py
 with DAG ( 'successful DAG' ) as dag :
    (
        dag
        >> PubSubSubscriptionDeleteOperator ( project = 'my-project' ,
                                            subscription = 'non-existing' )
    )

```

如果订阅已存在，则可以将运算符配置为失败。

```py
 with DAG ( 'failing DAG' ) as dag :
    (
        dag
        >> PubSubSubscriptionDeleteOperator (
             project = 'my-project' , subscription = 'non-existing' ,
             fail_if_not_exists = True )
    )

```

`project`，并且`subscription`是模板化的，因此您可以在其中使用变量。

```py
class airflow.contrib.operators.pubsub_operator.PubSubPublishOperator（project，topic，messages，gcp_conn_id ='google_cloud_default'，delegate_to = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

将消息发布到 PubSub 主题。

每个任务都将所有提供的消息发布到单个 GCP 项目中的同一主题。如果主题不存在，则此任务将失败。

```py
   from base64 import b64encode as b64e

   m1 = {'data': b64e('Hello, World!'),
         'attributes': {'type': 'greeting'}
        }
   m2 = {'data': b64e('Knock, knock')}
   m3 = {'attributes': {'foo': ''}}

   t1 = PubSubPublishOperator(
       project='my-project',topic='my_topic',
       messages=[m1, m2, m3],
       create_topic=True,
       dag=dag)

``project`` , ``topic``, and ``messages`` are templated so you can use

```

其中的变量。

```py
class airflow.contrib.operators.qubole_check_operator.QuboleCheckOperator（qubole_conn_id ='qubole_default'，* args，** kwargs） 
```

基类：[`airflow.operators.check_operator.CheckOperator`](31 "airflow.operators.check_operator.CheckOperator")，[`airflow.contrib.operators.qubole_operator.QuboleOperator`](31 "airflow.contrib.operators.qubole_operator.QuboleOperator")

对 Qubole 命令执行检查。`QuboleCheckOperator`期望一个将在 QDS 上执行的命令。默认情况下，使用 python `bool`强制计算此 Qubole Commmand 结果第一行的每个值。如果返回任何值`False`，则检查失败并输出错误。

请注意，Python bool 强制转换如下`False`：

*   `False`
*   `0`
*   空字符串（`""`）
*   空列表（`[]`）
*   空字典或集（`{}`）

给定一个查询，它只会在计数时失败。您可以制作更复杂的查询，例如，可以检查表与上游源表的行数相同，或者今天的分区计数大于昨天的分区，或者一组指标是否更少 7 天平均值超过 3 个标准差。`SELECT COUNT(*) FROM foo``== 0`

此运算符可用作管道中的数据质量检查，并且根据您在 DAG 中的位置，您可以选择停止关键路径，防止发布可疑数据，或者侧面接收电子邮件警报阻止 DAG 的进展。

参数：`qubole_conn_id(str)` - 由 qds auth_token 组成的连接 ID
kwargs：

> 可以从 QuboleOperator 文档中引用特定于 Qubole 命令的参数。
> 
> &lt;colgroup&gt;&lt;col class="field-name"&gt;&lt;col class="field-body"&gt;&lt;/colgroup&gt;
> | results_parser_callable： |
> | --- |
> |  | 这是一个可选参数，用于扩展将 Qubole 命令的结果解析给用户的灵活性。这是一个 python 可调用的，它可以保存逻辑来解析 Qubole 命令返回的行列表。默认情况下，仅第一行的值用于执行检查。此可调用对象应返回必须在其上执行检查的记录列表。 |

注意

与 QuboleOperator 和 CheckOperator 的模板字段相同的所有字段都是模板支持的。

```py
class airflow.contrib.operators.qubole_check_operator.QuboleValueCheckOperator（pass_value，tolerance = None，qubole_conn_id ='qubole_default'，* args，** kwargs） 
```

基类：[`airflow.operators.check_operator.ValueCheckOperator`](31 "airflow.operators.check_operator.ValueCheckOperator")，[`airflow.contrib.operators.qubole_operator.QuboleOperator`](31 "airflow.contrib.operators.qubole_operator.QuboleOperator")

使用 Qubole 命令执行简单的值检查。默认情况下，此 Qubole 命令的第一行上的每个值都与预定义的值进行比较。如果命令的输出不在预期值的允许限度内，则检查失败并输出错误。

参数：

*   `qubole_conn_id(str)` - 由 qds auth_token 组成的连接 ID
*   `pass_value(_str / int / float_)` - 查询结果的预期值。
*   `tolerance(_int / float_)` - 定义允许的 pass_value 范围，例如，如果 tolerance 为 2，则 Qubole 命令输出可以是-2 * pass_value 和 2 * pass_value 之间的任何值，而不会导致操作员错误输出。


kwargs：

> 可以从 QuboleOperator 文档中引用特定于 Qubole 命令的参数。
> 
> &lt;colgroup&gt;&lt;col class="field-name"&gt;&lt;col class="field-body"&gt;&lt;/colgroup&gt;
> | results_parser_callable： |
> | --- |
> |  | 这是一个可选参数，用于扩展将 Qubole 命令的结果解析给用户的灵活性。这是一个 python 可调用的，它可以保存逻辑来解析 Qubole 命令返回的行列表。默认情况下，仅第一行的值用于执行检查。此可调用对象应返回必须在其上执行检查的记录列表。 |

注意

与 QuboleOperator 和 ValueCheckOperator 的模板字段相同的所有字段都是模板支持的。

```py
class airflow.contrib.operators.qubole_operator.QuboleOperator（qubole_conn_id ='qubole_default'，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在 QDS 上执行任务（命令）（[https://qubole.com](https://qubole.com/)）。

参数：`qubole_conn_id(str)` - 由 qds auth_token 组成的连接 ID

```py
 kwargs： 
```

| COMMAND_TYPE： | 要执行的命令类型，例如 hivecmd，shellcmd，hadoopcmd| 标签： | 要使用该命令分配的标记数组| cluster_label： | 将在其上执行命令的集群标签| 名称： | 要命令的名称| 通知： | 是否在命令完成时发送电子邮件（默认为 False）
**特定于命令类型的参数**

```py
 hivecmd： 
```

| 查询： | 内联查询语句| script_location： |
| --- |
|  | 包含查询语句的 s3 位置 |
| SAMPLE_SIZE： | 要运行查询的样本大小（以字节为单位）| 宏： | 在查询中使用的宏值

```py
 prestocmd： 
```

| 查询： | 内联查询语句| script_location： |
| --- |
|  | 包含查询语句的 s3 位置 |
| 宏： | 在查询中使用的宏值

```py
 hadoopcmd： 
```

| sub_commnad： | 必须是这些[“jar”，“s3distcp”，“流媒体”]后跟一个或多个 args

```py
 shellcmd： 
```

| 脚本： | 带有 args 的内联命令| script_location： |
| --- |
|  | 包含查询语句的 s3 位置 |
| 文件： | s3 存储桶中的文件列表为 file1，file2 格式。这些文件将被复制到正在执行 qubole 命令的工作目录中。| 档案： | s3 存储桶中的存档列表为 archive1，archive2 格式。这些将被解压缩到正在执行 qubole 命令的工作目录中参数：需要传递给脚本的任何额外的 args（仅当提供了 script_location 时）

```py
 pigcmd： 
```

| 脚本： | 内联查询语句（latin_statements）| script_location： |
| --- |
|  | 包含 pig 查询的 s3 位置 |
参数：需要传递给脚本的任何额外的 args（仅当提供了 script_location 时）

```py
 sparkcmd： 
```

| 程序： | Scala，SQL，Command，R 或 Python 中的完整 Spark 程序| CMDLINE： | spark-submit 命令行，必须在 cmdline 本身中指定所有必需的信息。| SQL： | 内联 sql 查询| script_location： |
| --- |
|  | 包含查询语句的 s3 位置 |
| 语言： | 程序的语言，Scala，SQL，Command，R 或 Python| APP_ID： | Spark 作业服务器应用程序的 ID 参数：spark-submit 命令行参数| user_program_arguments： |
| --- |
|  | 用户程序所接受的参数 |
| 宏： | 在查询中使用的宏值

```py
 dbtapquerycmd： 
```

| db_tap_id： | Qubole 中目标数据库的数据存储 ID。| 查询： | 内联查询语句| 宏： | 在查询中使用的宏值

```py
 dbexportcmd： 
```

| 模式： | 1（简单），2（提前）| hive_table： | 配置单元表的名称| partition_spec： | Hive 表的分区规范。| dbtap_id： | Qubole 中目标数据库的数据存储 ID。| db_table： | db 表的名称| db_update_mode： | allowinsert 或 updateonly| db_update_keys： | 用于确定行唯一性的列| export_dir： | HDFS / S3 将从中导出数据的位置。| fields_terminated_by： |
| --- |
|  | 用作数据集中列分隔符的 char 的十六进制 |

```py
 dbimportcmd： 
```

| 模式： | 1（简单），2（提前）| hive_table： | 配置单元表的名称| dbtap_id： | Qubole 中目标数据库的数据存储 ID。| db_table： | db 表的名称| where_clause： | where 子句，如果有的话| 并行： | 用于提取数据的并行数据库连接数| extract_query： | SQL 查询从 db 中提取数据。$ CONDITIONS 必须是 where 子句的一部分。| boundary_query： | 要使用的查询获取要提取的行 ID 范围| split_column： | 用作行 ID 的列将数据拆分为范围（模式 2）
注意

以下字段模板支持：`query`，`script_location`，`sub_command`，`script`，`files`，`archives`，`program`，`cmdline`，`sql`，`where_clause`，`extract_query`，`boundary_query`，`macros`，`tags`，`name`，`parameters`，`dbtap_id`，`hive_table`，`db_table`，`split_column`，`note_id`，`db_update_keys`，`export_dir`，`partition_spec`，`qubole_conn_id`，`arguments`，`user_program_arguments`。

> 您还可以将`.txt`文件用于模板驱动的用例。

注意

在 QuboleOperator 中，有一个用于任务失败和重试的默认处理程序，它通常会杀死在 QDS 上运行的相应任务实例的命令。您可以通过在任务定义中提供自己的失败和重试处理程序来覆盖此行为。

```py
class airflow.contrib.operators.s3listoperator.S3ListOperator（bucket，prefix =''，delimiter =''，aws_conn_id ='aws_default'，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

列出桶中具有名称中给定字符串前缀的所有对象。

此运算符返回一个 python 列表，其中包含可由`xcom`在下游任务中使用的对象名称。

参数：

*   `bucket(string)` - S3 存储桶在哪里找到对象。（模板）
*   `prefix(string)` - 用于过滤名称以此前缀开头的对象的前缀字符串。（模板）
*   `delimiter(string)` - 分隔符标记键层次结构。（模板）
*   `aws_conn_id(string)` - 连接到 S3 存储时使用的连接 ID。

```py
 Example: 
```

以下运算符将列出存储桶中 S3 `customers/2018/04/`键的所有文件（不包括子文件夹）`data`。

```py
 s3_file = S3ListOperator (
    task_id = 'list_3s_files' ,
    bucket = 'data' ,
    prefix = 'customers/2018/04/' ,
    delimiter = '/' ,
    aws_conn_id = 'aws_customers_conn'
)

```

```py
class airflow.contrib.operators.s3_to_gcs_operator.S3ToGoogleCloudStorageOperator（bucket，prefix =''，delimiter =''，aws_conn_id ='aws_default'，dest_gcs_conn_id = None，dest_gcs = None，delegate_to = None，replace = False，* args，** kwargs） 
```

基类： [`airflow.contrib.operators.s3listoperator.S3ListOperator`](integration.html "airflow.contrib.operators.s3listoperator.S3ListOperator")

将 S3 密钥（可能是前缀）与 Google 云端存储目标路径同步。

参数：

*   `bucket(string)` - S3 存储桶在哪里找到对象。（模板）
*   `prefix(string)` - 前缀字符串，用于过滤名称以此前缀开头的对象。（模板）
*   `delimiter(string)` - 分隔符标记键层次结构。（模板）
*   `aws_conn_id(string)` - 源 S3 连接
*   `dest_gcs_conn_id(string)` - 连接到 Google 云端存储时要使用的目标连接 ID。
*   `dest_gcs(string)` - 要存储文件的目标 Google 云端存储**分区**和前缀。（模板）
*   `delegate_to(string)` - 模拟的帐户（如果有）。为此，发出请求的服务帐户必须启用域范围委派。
*   `replace(bool)` - 是否要替换现有目标文件。


**示例**：.. code-block :: python

>

```py
>  s3_to_gcs_op = S3ToGoogleCloudStorageOperator（ 
> ```
> 
> task_id ='s3_to_gcs_example'，bucket ='my-s3-bucket'，prefix ='data / customers-201804'，dest_gcs_conn_id ='google_cloud_default'，dest_gcs ='gs：//my.gcs.bucket/some/customers/' ，replace = False，dag = my-dag）

需要注意的是`bucket`，`prefix`，`delimiter`和`dest_gcs`模板化，所以你可以，如果你想使用变量在其中。

```py
class airflow.contrib.operators.segment_track_event_operator.SegmentTrackEventOperator（user_id，event，properties = None，segment_conn_id ='segment_default'，segment_debug_mode = False，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

将跟踪事件发送到 Segment 以获取指定的 user_id 和事件

参数：

*   `user_id(string)` - 数据库中此用户的 ID。（模板）
*   `event(string)` - 您要跟踪的事件的名称。（模板）
*   `properties(dict)` - 事件属性的字典。（模板）
*   `segment_conn_id(string)` - 连接到 Segment 时使用的连接 ID。
*   `segment_debug_mode(_boolean_)` - 确定 Segment 是否应在调试模式下运行。默认为 False

```py
class airflow.contrib.operators.sftp_operator.SFTPOperator（ssh_hook = None，ssh_conn_id = None，remote_host = None，local_filepath = None，remote_filepath = None，operation ='put'，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

SFTPOperator 用于将文件从远程主机传输到本地或反之亦然。此运算符使用 ssh_hook 打开 sftp trasport 通道，该通道用作文件传输的基础。

参数：

*   `ssh_hook(`SSHHook`)` - 用于远程执行的预定义 ssh_hook
*   `ssh_conn_id(str)` - 来自**airflow** Connections 的连接 ID
*   `remote_host(str)` - 要连接的远程主机
*   `local_filepath(str)` - 要获取或放置的本地文件路径。（模板）
*   `remote_filepath(str)` - 要获取或放置的远程文件路径。（模板）
*   `操作` - 指定操作'get'或'put'，默认为 get

```py
class airflow.contrib.operators.slack_webhook_operator.SlackWebhookOperator（http_conn_id = None，webhook_token = None，message =''，channel = None，username = None，icon_emoji = None，link_names = False，proxy = None，* args，** kwargs ） 
```

基类： [`airflow.operators.http_operator.SimpleHttpOperator`](31 "airflow.operators.http_operator.SimpleHttpOperator")

此运算符允许您使用传入的 webhook 将消息发布到 Slack。直接使用 Slack webhook 令牌和具有 Slack webhook 令牌的连接。如果两者都提供，将使用 Slack webhook 令牌。

每个 Slack webhook 令牌都可以预先配置为使用特定频道，用户名和图标。您可以在此挂钩中覆盖这些默认值。

参数：

*   `conn_id(str)` - 在额外字段中具有 Slack webhook 标记的连接
*   `webhook_token(str)` - Slack webhook 令牌
*   `message(str)` - 要在 Slack 上发送的消息
*   `channel(str)` - 邮件应发布到的频道
*   `username(str)` - 要发布的用户名
*   `icon_emoji(str)` - 用作发布给 Slack 的用户图标的表情符号
*   `link_names(bool)` - 是否在邮件中查找和链接频道和用户名
*   `proxy(str)` - 用于进行 Slack webhook 调用的代理

```py
执行（上下文） 
```

调用 SparkSqlHook 来运行提供的 sql 查询

```py
class airflow.contrib.operators.snowflake_operator.SnowflakeOperator（sql，snowflake_conn_id ='snowflake_default'，parameters = None，autocommit = True，warehouse = None，database = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在 Snowflake 数据库中执行 sql 代码

参数：

*   `snowflake_conn_id(string)` - 对特定雪花连接 ID 的引用
*   `SQL(_ 可接收表示 SQL 语句中的海峡 _ _，_ _ 海峡列表 _ _（_ _SQL 语句 _ _）_ _，或 _ _ 参照模板文件模板引用在“.SQL”结束海峡认可。_)` -要执行的 SQL 代码。（模板）
*   `warehouse(string)` - 覆盖连接中已定义的仓库的仓库名称
*   `database(string)` - 覆盖连接中定义的数据库的数据库名称

```py
class airflow.contrib.operators.spark_jdbc_operator.SparkJDBCOperator（spark_app_name ='airflow-spark-jdbc'，spark_conn_id ='spark-default'，spark_conf = None，spark_py_files = None，spark_files = None，spark_jars = None，num_executors = None，executor_cores = None，executor_memory = None，driver_memory = None，verbose = False，keytab = None，principal = None，cmd_type ='spark_to_jdbc'，jdbc_table = None，jdbc_conn_id ='jdbc-default'，jdbc_driver = None，metastore_table = None，jdbc_truncate = False，save_mode = None，save_format = None，batch_size = None，fetch_size = None，num_partitions = None，partition_column = None，lower_bound = None，upper_bound = None，create_table_column_types = None，* args，** kwargs） 
```

基类： [`airflow.contrib.operators.spark_submit_operator.SparkSubmitOperator`](31 "airflow.contrib.operators.spark_submit_operator.SparkSubmitOperator")

此运算符专门用于扩展 SparkSubmitOperator，以便使用 Apache Spark 执行与基于 JDBC 的数据库之间的数据传输。与 SparkSubmitOperator 一样，它假定 PATH 上有“spark-submit”二进制文件。

参数：

*   `spark_app_name**（str） - 作业名称（默认为**airflow` -spark-jdbc）
*   `spark_conn_id(str)` - 在 Airflow 管理中配置的连接 ID
*   `spark_conf(dict)` - 任何其他 Spark 配置属性
*   `spark_py_files(str)` - 使用的其他 python 文件（.zip，.egg 或.py）
*   `spark_files(str)` - 要上载到运行作业的容器的其他文件
*   `spark_jars(str)` - 要上传并添加到驱动程序和执行程序类路径的其他 jar
*   `num_executors(int)` - 要运行的执行程序的数量。应设置此项以便管理使用 JDBC 数据库建立的连接数
*   `executor_cores(int)` - 每个执行程序的核心数
*   `executor_memory(str)` - 每个执行者的内存（例如 1000M，2G）
*   `driver_memory(str)` - 分配给驱动程序的内存（例如 1000M，2G）
*   `verbose(bool)` - 是否将详细标志传递给 spark-submit 进行调试
*   `keytab(str)` - 包含 keytab 的文件的完整路径
*   `principal(str)` - 用于 keytab 的 kerberos 主体的名称
*   `cmd_type(str)` - 数据应该以哪种方式流动。2 个可能的值：spark_to_jdbc：从 Metastore 到 jdbc jdbc_to_spark 的 spark 写入的数据：来自 jdbc 到 Metastore 的 spark 写入的数据
*   `jdbc_table(str)` - JDBC 表的名称
*   `jdbc_conn_id` - 用于连接 JDBC 数据库的连接 ID
*   `jdbc_driver(str)` - 用于 JDBC 连接的 JDBC 驱动程序的名称。这个驱动程序（通常是一个 jar）应该在'jars'参数中传递
*   `metastore_table(str)` - Metastore 表的名称，
*   `jdbc_truncate(bool)` - （仅限 spark_to_jdbc）Spark 是否应截断或删除并重新创建 JDBC 表。这仅在'save_mode'设置为 Overwrite 时生效。此外，如果架构不同，Spark 无法截断，并将丢弃并重新创建
*   `save_mode(str)` - 要使用的 Spark 保存模式（例如覆盖，追加等）
*   `save_format(str)` - （jdbc_to_spark-only）要使用的 Spark 保存格式（例如镶木地板）
*   `batch_size(int)` - （仅限 spark_to_jdbc）每次往返 JDBC 数据库时要插入的批处理的大小。默认为 1000
*   `fetch_size(int)` - （仅限 jdbc_to_spark）从 JDBC 数据库中每次往返获取的批处理的大小。默认值取决于 JDBC 驱动程序
*   `num_partitions(int)` - Spark 同时可以使用的最大分区数，包括 spark_to_jdbc 和 jdbc_to_spark 操作。这也将限制可以打开的 JDBC 连接数
*   `partition_column(str)` - （jdbc_to_spark-only）用于对 Metastore 表进行分区的数字列。如果已指定，则还必须指定：num_partitions，lower_bound，upper_bound
*   `lower_bound(int)` - （jdbc_to_spark-only）要获取的数字分区列范围的下限。如果已指定，则还必须指定：num_partitions，partition_column，upper_bound
*   `upper_bound(int)` - （jdbc_to_spark-only）要获取的数字分区列范围的上限。如果已指定，则还必须指定：num_partitions，partition_column，lower_bound
*   `create_table_column_types` - （spark_to_jdbc-only）创建表时要使用的数据库列数据类型而不是默认值。应以与 CREATE TABLE 列语法相同的格式指定数据类型信息（例如：“name CHAR（64），comments VARCHAR（1024）”）。指定的类型应该是有效的 spark sql 数据类型。

类型： jdbc_conn_id：str

```py
执行（上下文） 
```

调用 SparkSubmitHook 来运行提供的 spark 作业

```py
class airflow.contrib.operators.spark_sql_operator.SparkSqlOperator（sql，conf = None，conn_id ='spark_sql_default'，total_executor_cores = None，executor_cores = None，executor_memory = None，keytab = None，principal = None，master ='yarn'，name ='default-name'，num_executors = None，yarn_queue ='default'，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

执行 Spark SQL 查询

参数：

*   `sql(str)` - 要执行的 SQL 查询。（模板）
*   `conf(_str __（_ _ 格式：PROP = VALUE __）_)` - 任意 Spark 配置属性
*   `conn_id(str)` - connection_id 字符串
*   `total_executor_cores(int)` - （仅限 Standalone 和 Mesos）所有执行程序的总核心数（默认值：工作程序上的所有可用核心）
*   `executor_cores(int)` - （仅限 Standalone 和 YARN）每个执行程序的核心数（默认值：2）
*   `executor_memory(str)` - 每个执行程序的内存（例如 1000M，2G）（默认值：1G）
*   `keytab(str)` - 包含 keytab 的文件的完整路径
*   `master(str)` - spark：// host：port，mesos：// host：port，yarn 或 local
*   `name(str)` - 作业名称
*   `num_executors(int)` - 要启动的执行程序数
*   `verbose(bool)` - 是否将详细标志传递给 spark-sql
*   `yarn_queue(str)` - 要提交的 YARN 队列（默认值：“default”）

```py
执行（上下文） 
```

调用 SparkSqlHook 来运行提供的 sql 查询

```py
class airflow.contrib.operators.spark_submit_operator.SparkSubmitOperator（application =''，conf = None，conn_id ='spark_default'，files = None，py_files = None，driver_classpath = None，jars = None，java_class = None，packages = None， exclude_packages = None，repositories = None，total_executor_cores = None，executor_cores = None，executor_memory = None，driver_memory = None，keytab = None，principal = None，name ='airflow-spark'，num_executors = None，application_args = None，env_vars =无，详细=假，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

这个钩子是一个围绕 spark-submit 二进制文件的包装器来启动一个 spark-submit 作业。它要求“spark-submit”二进制文件在 PATH 中，或者 spark-home 在连接中的额外设置。

参数：

*   `application(str)` - 作为作业提交的应用程序，jar 或 py 文件。（模板）
*   `conf(dict)` - 任意 Spark 配置属性
*   `conn_id(str)` - Airflow 管理中配置的连接 ID。当提供无效的 connection_id 时，它将默认为 yarn。
*   `files(str)` - 将其他文件上载到运行作业的执行程序，以逗号分隔。文件将放在每个执行程序的工作目录中。例如，序列化对象。
*   `py_files(str)` - 作业使用的其他 python 文件可以是.zip，.egg 或.py。
*   `jars(str)` - 提交其他 jar 以上传并将它们放在执行程序类路径中。
*   `driver_classpath(str)` - 其他特定于驱动程序的类路径设置。
*   `java_class(str)` - Java 应用程序的主要类
*   `packages(str)` - 包含在驱动程序和执行程序类路径上的 jar 的 maven 坐标的逗号分隔列表。（模板）
*   `exclude_packages(str)` - 解析“包”中提供的依赖项时要排除的 jar 的 maven 坐标的逗号分隔列表
*   `repositories(str)` - 以逗号分隔的其他远程存储库列表，用于搜索“packages”给出的 maven 坐标
*   `total_executor_cores(int)` - （仅限 Standalone 和 Mesos）所有执行程序的总核心数（默认值：工作程序上的所有可用核心）
*   `executor_cores(int)` - （仅限 Standalone 和 YARN）每个执行程序的核心数（默认值：2）
*   `executor_memory(str)` - 每个执行程序的内存（例如 1000M，2G）（默认值：1G）
*   `driver_memory(str)` - 分配给驱动程序的内存（例如 1000M，2G）（默认值：1G）
*   `keytab(str)` - 包含 keytab 的文件的完整路径
*   `principal(str)` - 用于 keytab 的 kerberos 主体的名称
*   `name(str)` - 作业名称（默认气流 - 火花）。（模板）
*   `num_executors(int)` - 要启动的执行程序数
*   `application_args(list)` - 正在提交的应用程序的参数
*   `env_vars(dict)` - spark-submit 的环境变量。它也支持纱线和 k8s 模式。
*   `verbose(bool)` - 是否将详细标志传递给 spark-submit 进程进行调试

```py
执行（上下文） 
```

调用 SparkSubmitHook 来运行提供的 spark 作业

```py
class airflow.contrib.operators.sqoop_operator.SqoopOperator（conn_id ='sqoop_default'，cmd_type ='import'，table = None，query = None，target_dir = None，append = None，file_type ='text'，columns = None，num_mappers = None，split_by = None，其中= None，export_dir = None，input_null_string = None，input_null_non_string = None，staging_table = None，clear_staging_table = False，enclosed_by = None，escaped_by = None，input_fields_terminated_by = None，input_lines_terminated_by = None，input_optionally_enclosed_by = None ，batch = False，direct = False，driver = None，verbose = False，relaxed_isolation = False，properties = None，hcatalog_database = None，hcatalog_table = None，create_hcatalog_table = False，extra_import_options = None，extra_export_options = None，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

执行 Sqoop 作业。Apache Sqoop 的文档可以在这里找到：

> [https://sqoop.apache.org/docs/1.4.2/SqoopUserGuide.html](https://sqoop.apache.org/docs/1.4.2/SqoopUserGuide.html)。

```py
执行（上下文） 
```

执行 sqoop 作业

```py
class airflow.contrib.operators.ssh_operator.SSHOperator（ssh_hook = None，ssh_conn_id = None，remote_host = None，command = None，timeout = 10，do_xcom_push = False，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

SSHOperator 使用 ssh_hook 在给定的远程主机上执行命令。

参数：

*   `ssh_hook(`SSHHook`)` - 用于远程执行的预定义 ssh_hook
*   `ssh_conn_id(str)` - 来自**airflow** Connections 的连接 ID
*   `remote_host(str)` - 要连接的远程主机
*   `command(str)` - 在远程主机上执行的命令。（模板）
*   `timeout(int)` - 执行命令的超时（以秒为单位）。
*   `do_xcom_push(bool)` - 返回由气流平台在 xcom 中设置的标准输出

```py
class airflow.contrib.operators.vertica_operator.VerticaOperator（sql，vertica_conn_id ='vertica_default'，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

在特定的 Vertica 数据库中执行 sql 代码

参数：

*   `vertica_conn_id(string)` - 对特定 Vertica 数据库的引用
*   `SQL(_ 可接收表示 SQL 语句中的海峡 _ _，_ _ 海峡列表 _ _（_ _SQL 语句 _ _）_ _，或 _ _ 参照模板文件模板引用在“.SQL”结束海峡认可。_)` -要执行的 SQL 代码。（模板）

```py
class airflow.contrib.operators.vertica_to_hive.VerticaToHiveTransfer（sql，hive_table，create = True，recreate = False，partition = None，delimiter = u'x01'，vertica_conn_id ='vertica_default'，hive_cli_conn_id ='hive_cli_default'，* args，* * kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

将数据从 Vertia 移动到 Hive。操作员针对 Vertia 运行查询，在将文件加载到 Hive 表之前将文件存储在本地。如果将`create`或`recreate`参数设置为`True`，则生成 a 和语句。从游标的元数据推断出 Hive 数据类型。请注意，在 Hive 中生成的表使用的不是最有效的序列化格式。如果加载了大量数据和/或表格被大量查询，您可能只想使用此运算符将数据暂存到临时表中，然后使用 a 将其加载到最终目标中。`CREATE TABLE``DROP TABLE``STORED AS textfile``HiveOperator`

参数：

*   `sql(str)` - 针对 Vertia 数据库执行的 SQL 查询。（模板）
*   `hive_table(str)` - 目标 Hive 表，使用点表示法来定位特定数据库。（模板）
*   `create(bool)` - 是否创建表，如果它不存在
*   `recreate(bool)` - 是否在每次执行时删除并重新创建表
*   `partition(dict)` - 将目标分区作为分区列和值的字典。（模板）
*   `delimiter(str)` - 文件中的字段分隔符
*   `vertica_conn_id(str)` - 源 Vertica 连接
*   `hive_conn_id(str)` - 目标配置单元连接

```py
class airflow.contrib.operators.winrm_operator.WinRMOperator（winrm_hook = None，ssh_conn_id = None，remote_host = None，command = None，timeout = 10，do_xcom_push = False，* args，** kwargs） 
```

基类： [`airflow.models.BaseOperator`](31 "airflow.models.BaseOperator")

WinRMOperator 使用 winrm_hook 在给定的远程主机上执行命令。

参数：

*   `winrm_hook(`WinRMHook`)` - 用于远程执行的预定义 ssh_hook
*   `ssh_conn_id(str)` - 来自**airflow** Connections 的连接 ID
*   `remote_host(str)` - 要连接的远程主机
*   `command(str)` - 在远程主机上执行的命令。（模板）
*   `timeout(int)` - 执行命令的超时。
*   `do_xcom_push(bool)` - 返回由气流平台在 xcom 中设置的标准输出


#### 传感器

```py
class airflow.contrib.sensors.aws_redshift_cluster_sensor.AwsRedshiftClusterSensor（cluster_identifier，target_status ='available'，aws_conn_id ='aws_default'，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

等待 Redshift 群集达到特定状态。

参数：

*   `cluster_identifier(str)` - 要 ping 的集群的标识符。
*   `target_status(str)` - 所需的集群状态。

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.contrib.sensors.bash_sensor.BashSensor（bash_command，env = None，output_encoding ='utf-8'，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

当且仅当返回码为 0 时，执行 bash 命令/脚本并返回 True。

参数：

*   `bash_command(string)` - 要执行的命令，命令集或对 bash 脚本（必须为'.sh'）的引用。
*   `env(dict)` - 如果 env 不是 None，它必须是一个定义新进程的环境变量的映射; 这些用于代替继承当前进程环境，这是默认行为。（模板）
*   `output_encoding(string)` - bash 命令的输出编码。

```py
戳（上下文） 
```

在临时目录中执行 bash 命令，之后将对其进行清理

```py
class airflow.contrib.sensors.bigquery_sensor.BigQueryTableSensor（project_id，dataset_id，table_id，bigquery_conn_id ='bigquery_default_conn'，delegate_to = None，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

检查 Google Bigquery 中是否存在表格。

> &lt;colgroup&gt;&lt;col class="field-name"&gt;&lt;col class="field-body"&gt;&lt;/colgroup&gt;
> | param project_id： |
> | --- |
> |  | 要在其中查找表的 Google 云项目。提供给钩子的连接必须提供对指定项目的访问。 |
> | 类型 project_id： |
> | --- |
> |  | 串 |
> | param dataset_id： |
> | --- |
> |  | 要在其中查找表的数据集的名称。存储桶。 |
> | 类型 dataset_id： |
> | --- |
> |  | 串 |
> | param table_id： | 要检查表的名称是否存在。 |
> | --- | --- |
> | type table_id： | 串 |
> | --- | --- |
> | param bigquery_conn_id： |
> | --- |
> |  | 连接到 Google BigQuery 时使用的连接 ID。 |
> | 输入 bigquery_conn_id： |
> | --- |
> |  | 串 |
> | param delegate_to： |
> | --- |
> |  | 假冒的帐户，如果有的话。为此，发出请求的服务帐户必须启用域范围委派。 |
> | type delegate_to： |
> | --- |
> |  | 串 |

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.contrib.sensors.datadog_sensor.DatadogSensor（datadog_conn_id ='datadog_default'，from_seconds_ago = 3600，up_to_seconds_from_now = 0，priority = None，sources = None，tags = None，response_check = None，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

一个传感器，通过过滤器监听数据对象事件流并确定是否发出了某些事件。

取决于 datadog API，该 API 必须部署在 Airflow 运行的同一服务器上。

参数：

*   `datadog_conn_id` - 与 datadog 的连接，包含 api 密钥的元数据。
*   `datadog_conn_id` - 字符串

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.contrib.sensors.emr_base_sensor.EmrBaseSensor（aws_conn_id ='aws_default'，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

包含 EMR 的一般传感器行为。子类应该实现 get_emr_response（）和 state_from_response（）方法。子类还应实现 NON_TERMINAL_STATES 和 FAILED_STATE 常量。

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.contrib.sensors.emr_job_flow_sensor.EmrJobFlowSensor（job_flow_id，* args，** kwargs） 
```

基类： [`airflow.contrib.sensors.emr_base_sensor.EmrBaseSensor`](31 "airflow.contrib.sensors.emr_base_sensor.EmrBaseSensor")

询问 JobFlow 的状态，直到它达到终端状态。如果传感器错误失败，则任务失败。

参数：`job_flow_id(string)` - job_flow_id 来检查状态

```py
class airflow.contrib.sensors.emr_step_sensor.EmrStepSensor（job_flow_id，step_id，* args，** kwargs） 
```

基类： [`airflow.contrib.sensors.emr_base_sensor.EmrBaseSensor`](31 "airflow.contrib.sensors.emr_base_sensor.EmrBaseSensor")

询问步骤的状态，直到达到终端状态。如果传感器错误失败，则任务失败。

参数：

*   `job_flow_id(string)` - job_flow_id，其中包含检查状态的步骤
*   `step_id(string)` - 检查状态的步骤

```py
class airflow.contrib.sensors.file_sensor.FileSensor（filepath，fs_conn_id ='fs_default2'，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

等待文件或文件夹降落到文件系统中。

如果给定的路径是目录，则该传感器仅在其中存在任何文件时才返回 true（直接或在子目录中）

参数：

*   `fs_conn_id(string)` - 对文件（路径）连接 ID 的引用
*   `filepath` - 文件或文件夹名称（相对于连接中设置的基本路径）

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.contrib.sensors.ftp_sensor.FTPSensor（path，ftp_conn_id ='ftp_default'，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

等待 FTP 上存在文件或目录。

参数：

*   `path(str)` - 远程文件或目录路径
*   `ftp_conn_id(str)` - 运行传感器的连接

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.contrib.sensors.ftp_sensor.FTPSSensor（path，ftp_conn_id ='ftp_default'，* args，** kwargs） 
```

基类： [`airflow.contrib.sensors.ftp_sensor.FTPSensor`](31 "airflow.contrib.sensors.ftp_sensor.FTPSensor")

等待通过 SSL 在 FTP 上存在文件或目录。

```py
class airflow.contrib.sensors.gcs_sensor.GoogleCloudStorageObjectSensor（bucket，object，google_cloud_conn_id ='google_cloud_default'，delegate_to = None，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

检查 Google 云端存储中是否存在文件。创建一个新的 GoogleCloudStorageObjectSensor。

> &lt;colgroup&gt;&lt;col class="field-name"&gt;&lt;col class="field-body"&gt;&lt;/colgroup&gt;
> | param bucket： | 对象所在的 Google 云存储桶。 |
> | --- | --- |
> | 型桶： | 串 |
> | --- | --- |
> | 参数对象： | 要在 Google 云存储分区中检查的对象的名称。 |
> | --- | --- |
> | 类型对象： | 串 |
> | --- | --- |
> | param google_cloud_storage_conn_id： |
> | --- |
> |  | 连接到 Google 云端存储时使用的连接 ID。 |
> | 输入 google_cloud_storage_conn_id： |
> | --- |
> |  | 串 |
> | param delegate_to： |
> | --- |
> |  | 假冒的帐户，如果有的话。为此，发出请求的服务帐户必须启用域范围委派。 |
> | type delegate_to： |
> | --- |
> |  | 串 |

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.contrib.sensors.gcs_sensor.GoogleCloudStorageObjectUpdatedSensor（bucket，object，ts_func = <function ts_function>，google_cloud_conn_id ='google_cloud_default'，delegate_to = None，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

检查 Google Cloud Storage 中是否更新了对象。创建一个新的 GoogleCloudStorageObjectUpdatedSensor。

> &lt;colgroup&gt;&lt;col class="field-name"&gt;&lt;col class="field-body"&gt;&lt;/colgroup&gt;
> | param bucket： | 对象所在的 Google 云存储桶。 |
> | --- | --- |
> | 型桶： | 串 |
> | --- | --- |
> | 参数对象： | 要在 Google 云存储分区中下载的对象的名称。 |
> | --- | --- |
> | 类型对象： | 串 |
> | --- | --- |
> | param ts_func： | 用于定义更新条件的回调。默认回调返回 execution_date + schedule_interval。回调将上下文作为参数。 |
> | --- | --- |
> | 输入 ts_func： | 功能 |
> | --- | --- |
> | param google_cloud_storage_conn_id： |
> | --- |
> |  | 连接到 Google 云端存储时使用的连接 ID。 |
> | 输入 google_cloud_storage_conn_id： |
> | --- |
> |  | 串 |
> | param delegate_to： |
> | --- |
> |  | 假冒的帐户，如果有的话。为此，发出请求的服务帐户必须启用域范围委派。 |
> | type delegate_to： |
> | --- |
> |  | 串 |

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.contrib.sensors.gcs_sensor.GoogleCloudStoragePrefixSensor（bucket，prefix，google_cloud_conn_id ='google_cloud_default'，delegate_to = None，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

检查 Google 云端存储分区中是否存在前缀文件。创建一个新的 GoogleCloudStorageObjectSensor。

> &lt;colgroup&gt;&lt;col class="field-name"&gt;&lt;col class="field-body"&gt;&lt;/colgroup&gt;
> | param bucket： | 对象所在的 Google 云存储桶。 |
> | --- | --- |
> | 型桶： | 串 |
> | --- | --- |
> | 参数前缀： | 要在 Google 云存储分区中检查的前缀的名称。 |
> | --- | --- |
> | 类型前缀： | 串 |
> | --- | --- |
> | param google_cloud_storage_conn_id： |
> | --- |
> |  | 连接到 Google 云端存储时使用的连接 ID。 |
> | 输入 google_cloud_storage_conn_id： |
> | --- |
> |  | 串 |
> | param delegate_to： |
> | --- |
> |  | 假冒的帐户，如果有的话。为此，发出请求的服务帐户必须启用域范围委派。 |
> | type delegate_to： |
> | --- |
> |  | 串 |

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.contrib.sensors.hdfs_sensor.HdfsSensorFolder（be_empty = False，* args，** kwargs） 
```

基类： [`airflow.sensors.hdfs_sensor.HdfsSensor`](31 "airflow.sensors.hdfs_sensor.HdfsSensor")

```py
戳（上下文） 
```

戳一个非空的目录

返回值： Bool 取决于搜索条件

```py
class airflow.contrib.sensors.hdfs_sensor.HdfsSensorRegex（regex，* args，** kwargs） 
```

基类： [`airflow.sensors.hdfs_sensor.HdfsSensor`](31 "airflow.sensors.hdfs_sensor.HdfsSensor")

```py
戳（上下文） 
```

用 self.regex 戳一个目录中的文件

返回值： Bool 取决于搜索条件

```py
class airflow.contrib.sensors.jira_sensor.JiraSensor（jira_conn_id ='jira_default'，method_name = None，method_params = None，result_processor = None，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

如有任何变更，请监控 jira 票。

参数：

*   `jira_conn_id(str)` - 对预定义的 Jira 连接的引用
*   `method_name(str)` - 要执行的 jira-python-sdk 的方法名称
*   `method_params(dict)` - 方法 method_name 的参数
*   `result_processor(function)` - 返回布尔值并充当传感器响应的函数

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.contrib.sensors.pubsub_sensor.PubSubPullSensor（project，subscription，max_messages = 5，return_immediately = False，ack_messages = False，gcp_conn_id ='google_cloud_default'，delegate_to = None，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

从 PubSub 订阅中提取消息并通过 XCom 传递它们。

此传感器操作员将从`max_messages`指定的 PubSub 订阅中提取消息。当订阅返回消息时，将完成 poke 方法的标准，并且将从操作员返回消息并通过 XCom 传递下游任务。

如果`ack_messages`设置为 True，则在返回之前将立即确认消息，否则，下游任务将负责确认消息。

`project`并且`subscription`是模板化的，因此您可以在其中使用变量。

```py
执行（上下文） 
```

重写以允许传递消息

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.contrib.sensors.qubole_sensor.QuboleSensor（data，qubole_conn_id ='qubole_default'，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

所有 Qubole 传感器的基类

参数：

*   `qubole_conn_id(string)` - 用于运行传感器的 qubole 连接
*   `data(_JSON 对象 _)` - 包含有效负载的 JSON 对象，需要检查其存在


注意

两个`data`和`qubole_conn_id`字段都是模板支持的。您可以

还使用`.txt`文件进行模板驱动的用例。

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.contrib.sensors.redis_key_sensor.RedisKeySensor（key，redis_conn_id，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

检查 Redis 数据库中是否存在密钥

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.contrib.sensors.sftp_sensor.SFTPSensor（path，sftp_conn_id ='sftp_default'，* args，** kwargs） 
```

基类： `airflow.operators.sensors.BaseSensorOperator`

等待 SFTP 上存在文件或目录。：param path：远程文件或目录路径：type path：str：param sftp_conn_id：运行传感器的连接：键入 sftp_conn_id：str

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

```py
class airflow.contrib.sensors.wasb_sensor.WasbBlobSensor（container_name，blob_name，wasb_conn_id ='wasb_default'，check_options = None，* args，** kwargs） 
```

基类： [`airflow.sensors.base_sensor_operator.BaseSensorOperator`](31 "airflow.sensors.base_sensor_operator.BaseSensorOperator")

等待 blob 到达 Azure Blob 存储。

参数：

*   `container_name(str)` - 容器的名称。
*   `blob_name(str)` - blob 的名称。
*   `wasb_conn_id(str)` - 对 wasb 连接的引用。
*   `check_options(dict)` - `WasbHook.check_for_blob（）`采用的可选关键字参数。

```py
戳（上下文） 
```

传感器在派生此类时定义的功能应该覆盖。

## 宏

这是可以在模板中使用的变量和宏的列表

### 默认变量

默认情况下，Airflow 引擎会传递一些可在所有模板中访问的变量

<colgroup><col width="30%"><col width="70%"></colgroup>
| 变量 | 描述| `{{ ds }}` | 执行日期为 `YYYY-MM-DD` |
| `{{ ds_nodash }}` | 执行日期为 `YYYYMMDD` |
| `{{ prev_ds }}` | 上一个执行日期为`YYYY-MM-DD`。如果是和是，将是。`{{ ds }}``2016-01-08``schedule_interval``@weekly``{{ prev_ds }}``2016-01-01` |
| `{{ next_ds }}` | 下一个执行日期为`YYYY-MM-DD`。如果是和是，将是。`{{ ds }}``2016-01-01``schedule_interval``@weekly``{{ prev_ds }}``2016-01-08` |
| `{{ yesterday_ds }}` | 昨天的日期是 `YYYY-MM-DD` |
| `{{ yesterday_ds_nodash }}` | 昨天的日期是 `YYYYMMDD` |
| `{{ tomorrow_ds }}` | 明天的日期是 `YYYY-MM-DD` |
| `{{ tomorrow_ds_nodash }}` | 明天的日期是 `YYYYMMDD` |
| `{{ ts }}` | 与...一样 `execution_date.isoformat()` |
| `{{ ts_nodash }}` | 和`ts`没有`-`和`:` |
| `{{ execution_date }}` | execution_date，（datetime.datetime） |
| `{{ prev_execution_date }}` | 上一个执行日期（如果可用）（datetime.datetime） |
| `{{ next_execution_date }}` | 下一个执行日期（datetime.datetime） |
| `{{ dag }}` | DAG 对象 |
| `{{ task }}` | Task 对象 |
| `{{ macros }}` | 对宏包的引用，如下所述 |
| `{{ task_instance }}` | task_instance 对象 |
| `{{ end_date }}` | 与...一样 `{{ ds }}` |
| `{{ latest_date }}` | 与...一样 `{{ ds }}` |
| `{{ ti }}` | 与...一样 `{{ task_instance }}` |
| `{{ params }}` | 对用户定义的 params 字典的引用，如果启用，则可以通过字典覆盖该字典`trigger_dag -c``dag_run_conf_overrides_params` in ``airflow.cfg` |
| `{{ var.value.my_var }}` | 全局定义的变量表示为字典 |
| `{{ var.json.my_var.path }}` | 全局定义的变量表示为具有反序列化 JSON 对象的字典，将路径追加到 JSON 对象中的键 |
| `{{ task_instance_key_str }}` | 格式化任务实例的唯一，人类可读的键 `{dag_id}_{task_id}_{ds}` |
| `{{ conf }}` | 完整的配置对象位于`airflow.configuration.conf`其中，代表您的内容`airflow.cfg` |
| `{{ run_id }}` | 在`run_id`当前 DAG 运行 |
| `{{ dag_run }}` | 对 DagRun 对象的引用 |
| `{{ test_mode }}` | 是否使用 CLI 的 test 子命令调用任务实例 |

请注意，您可以使用简单的点表示法访问对象的属性和方法。这里有什么是可能的一些例子：，，，...参考模型文档上对象的属性和方法的详细信息。`{{ task.owner }}``{{ task.task_id }}``{{ ti.hostname }}`

该`var`模板变量允许您访问气流的 UI 定义的变量。您可以以纯文本或 JSON 格式访问它们。如果使用 JSON，您还可以遍历嵌套结构，例如：`{{ var.json.mydictvar.key1 }}`

### 宏

宏是一种向模板公开对象并在模板中位于`macros`命名空间下的方法。

提供了一些常用的库和方法。

<colgroup><col width="44%"><col width="56%"></colgroup>
| 变量 | 描述| `macros.datetime` | 标准的 lib `datetime.datetime` |
| `macros.timedelta` | 标准的 lib `datetime.timedelta` |
| `macros.dateutil` | 对`dateutil`包的引用 |
| `macros.time` | 标准的 lib `time` |
| `macros.uuid` | 标准的 lib `uuid` |
| `macros.random` | 标准的 lib `random` |

还定义了一些特定于气流的宏：

```py
airflow.macros.ds_add（ds，days） 
```

从 YYYY-MM-DD 添加或减去天数

参数：

*   `ds(str)` - `YYYY-MM-DD`要添加的格式的锚定日期
*   `days(int)` - 要添加到 ds 的天数，可以使用负值

```py
 >>>  ds_add ( '2015-01-01' , 5 )
'2015-01-06'
>>>  ds_add ( '2015-01-06' , - 5 )
'2015-01-01'

```

```py
airflow.macros.ds_format（ds，input_format，output_format） 
```

获取输入字符串并输出输出格式中指定的另一个字符串

参数：

*   `ds(str)` - 包含日期的输入字符串
*   `input_format(str)` - 输入字符串格式。例如％Y-％m-％d
*   `output_format(str)` - 输出字符串格式例如％Y-％m-％d

```py
 >>>  ds_format ( '2015-01-01' , "%Y-%m- %d " , "%m- %d -%y" )
'01-01-15'
>>>  ds_format ( '1/5/2015' , "%m/ %d /%Y" ,  "%Y-%m- %d " )
'2015-01-05'

```

```py
airflow.macros.random（）→x 在区间[0,1）中。 
```

```py
airflow.macros.hive.closest_ds_partition（table，ds，before = True，schema ='default'，metastore_conn_id ='metastore_default'） 
```

此函数在最接近目标日期的列表中查找日期。可以给出可选参数以获得最接近的之前或之后。

参数：

*   `table(str)` - 配置单元表名称
*   `ds(_datetime.date list_)` - _ 日期 _ 戳，`%Y-%m-%d`例如`yyyy-mm-dd`
*   `之前(_bool _ 或 _None_)` - 最接近之前（True），之后（False）或 ds 的任一侧

返回值： 最近的日期| 返回类型： | str 或 None

```py
 >>>  tbl = 'airflow.static_babynames_partitioned'
>>>  closest_ds_partition ( tbl , '2015-01-02' )
'2015-01-01'

```

```py
airflow.macros.hive.max_partition（table，schema ='default'，field = None，filter_map = None，metastore_conn_id ='metastore_default'） 
```

获取表的最大分区。

参数：

*   `schema(string)` - 表所在的 hive 模式
*   `table(string)` - 您感兴趣的 hive 表，支持点符号，如“my_database.my_table”，如果找到一个点，则模式参数被忽略
*   `metastore_conn_id(string)` - 您感兴趣的配置单元连接。如果设置了默认值，则不需要使用此参数。
*   `filter_map(_map_)` - partition_key：用于分区过滤的 partition_value 映射，例如{'key1'：'value1'，'key2'：'value2'}。只有匹配所有 partition_key：partition_value 对的分区才会被视为最大分区的候选者。
*   `field(str)` - 从中​​获取最大值的字段。如果只有一个分区字段，则会推断出这一点

```py
 >>>  max_partition ( 'airflow.static_babynames_partitioned' )
'2015-01-01'

```

## 楷模

模型构建在 SQLAlchemy ORM Base 类之上，实例保留在数据库中。

```py
class airflow.models.BaseOperator（task_id，owner ='Airflow'，email = None，email_on_retry = True，email_on_failure = True，retries = 0，retry_delay = datetime.timedelta（0,300），retry_exponential_backoff = False，max_retry_delay = None， start_date = None，end_date = None，schedule_interval = None，depends_on_past = False，wait_for_downstream = False，dag = None，params = None，default_args = None，adhoc = False，priority_weight = 1，weight_rule = u'downstream'，queue =' default'，pool = None，sla = None，execution_timeout = None，on_failure_callback = None，on_success_callback = None，on_retry_callback = None，trigger_rule = u'all_success'，resources = None，run_as_user = None，task_concurrency = None，executor_config = None， inlets = None，outlets = None，* args，** kwargs） 
```

基类： `airflow.utils.log.logging_mixin.LoggingMixin`

所有运营商的抽象基类。由于运算符创建的对象成为 dag 中的节点，因此 BaseOperator 包含许多用于 dag 爬行行为的递归方法。要派生此类，您需要覆盖构造函数以及“execute”方法。

从此类派生的运算符应同步执行或触发某些任务（等待完成）。运算符的示例可以是运行 Pig 作业（PigOperator）的运算符，等待分区在 Hive（HiveSensorOperator）中着陆的传感器运算符，或者是将数据从 Hive 移动到 MySQL（Hive2MySqlOperator）的运算符。这些运算符（任务）的实例针对特定操作，运行特定脚本，函数或数据传输。

此类是抽象的，不应实例化。实例化从这个派生的类导致创建任务对象，该对象最终成为 DAG 对象中的节点。应使用 set_upstream 和/或 set_downstream 方法设置任务依赖性。

参数：

*   `task_id(string)` - 任务的唯一，有意义的 id
*   `owner(string)` - 使用 unix 用户名的任务的所有者
*   `retries(int)` - 在失败任务之前应执行的重试次数
*   `retry_delay(timedelta)` - 重试之间的延迟
*   `retry_exponential_backoff(bool)` - 允许在重试之间使用指数退避算法在重试之间进行更长时间的等待（延迟将转换为秒）
*   `max_retry_delay(timedelta)` - 重试之间的最大延迟间隔
*   `start_date(datetime)` - `start_date`用于任务，确定`execution_date`第一个任务实例。最佳做法是将 start_date 四舍五入到 DAG 中`schedule_interval`。每天的工作在 00:00:00 有一天的 start_date，每小时的工作在特定时间的 00:00 有 start_date。请注意，Airflow 只是查看最新的`execution_date`并添加`schedule_interval`以确定下一个`execution_date`。同样非常重要的是要注意不同任务的依赖关系需要及时排列。如果任务 A 依赖于任务 B 并且它们的 start_date 以其 execution_date 不排列的方式偏移，则永远不会满足 A 的依赖性。如果您希望延迟任务，例如在凌晨 2 点运行每日任务，请查看`TimeSensor`和`TimeDeltaSensor`。我们建议不要使用动态`start_date`，建议使用固定的。有关更多信息，请阅读有关 start_date 的 FAQ 条目。
*   `end_date(datetime)` - 如果指定，则调度程序不会超出此日期
*   `depends_on_past(bool)` - 当设置为 true 时，任务实例将依次运行，同时依赖上一个任务的计划成功。允许 start_date 的任务实例运行。
*   `wait_for_downstream(bool)` - 当设置为 true 时，任务 X 的实例将等待紧接上一个任务 X 实例下游的任务在运行之前成功完成。如果任务 X 的不同实例更改同一资产，并且任务 X 的下游任务使用此资产，则此操作非常有用。请注意，在使用 wait_for_downstream 的任何地方，depends_on_past 都将强制为 True。
*   `queue(str)` - 运行此作业时要定位到哪个队列。并非所有执行程序都实现队列管理，CeleryExecutor 确实支持定位特定队列。
*   `dag([_DAG_](31 "airflow.models.DAG"))` - 对任务所附的 dag 的引用（如果有的话）
*   `priority_weight(int)` - 此任务相对于其他任务的优先级权重。这允许执行程序在事情得到备份时在其他任务之前触发更高优先级的任务。
*   `weight_rule(str)` - 用于任务的有效总优先级权重的加权方法。选项包括：default 是设置为任务的有效权重时是所有下游后代的总和。因此，上游任务将具有更高的权重，并且在使用正权重值时将更积极地安排。当您有多个 dag 运行实例并希望在每个 dag 可以继续处理下游任务之前完成所有运行的所有上游任务时，这非常有用。设置为时`{ downstream &#124; upstream &#124; absolute }``downstream``downstream``upstream`有效权重是所有上游祖先的总和。这与 downtream 任务具有更高权重并且在使用正权重值时将更积极地安排相反。当你有多个 dag 运行实例并希望在启动其他 dag 的上游任务之前完成每个 dag 时，这非常有用。设置为时`absolute`，有效重量是精确`priority_weight`指定的，无需额外加权。当您确切知道每个任务应具有的优先级权重时，您可能希望这样做。另外，当设置`absolute`为时，对于非常大的 DAGS，存在显着加速任务创建过程的额外效果。可以将选项设置为字符串或使用静态类中定义的常量`airflow.utils.WeightRule`
*   `pool(str)` - 此任务应运行的插槽池，插槽池是限制某些任务的并发性的一种方法
*   `sla(datetime.timedelta)` - 作业预期成功的时间。请注意，这表示`timedelta`期间结束后。例如，如果您将 SLA 设置为 1 小时，则调度程序将在凌晨 1:00 之后发送电子邮件，`2016-01-02`如果`2016-01-01`实例尚未成功的话。调度程序特别关注具有 SLA 的作业，并发送警报电子邮件以防止未命中。SLA 未命中也记录在数据库中以供将来参考。共享相同 SLA 时间的所有任务都捆绑在一封电子邮件中，在此之后很快就会发送。SLA 通知仅为每个任务实例发送一次且仅一次。
*   `execution_timeout(datetime.timedelta)` - 执行此任务实例所允许的最长时间，如果超出它将引发并失败。
*   `on_failure_callback(callable)` - 当此任务的任务实例失败时要调用的函数。上下文字典作为单个参数传递给此函数。Context 包含对任务实例的相关对象的引用，并记录在 API 的宏部分下。
*   `on_retry_callback` - 非常类似于`on_failure_callback`重试发生时执行的。
*   `on_success_callback(callable)` - 很像`on_failure_callback`是在任务成功时执行它。
*   `trigger_rule(str)` - 定义应用依赖项以触发任务的规则。选项包括：默认为。可以将选项设置为字符串或使用静态类中定义的常量`{ all_success &#124; all_failed &#124; all_done &#124; one_success &#124; one_failed &#124; dummy}``all_success``airflow.utils.TriggerRule`
*   `resources(dict)` - 资源参数名称（Resources 构造函数的参数名称）与其值的映射。
*   `run_as_user(str)` - 在运行任务时使用 unix 用户名进行模拟
*   `task_concurrency(int)` - 设置后，任务将能够限制 execution_dates 之间的并发运行
*   `executor_config(dict)` -

    由特定执行程序解释的其他任务级配置参数。参数由 executor 的名称命名。[``](31)示例：通过 KubernetesExecutor MyOperator（...，在特定的 docker 容器中运行此任务）

    &gt; executor_config = {“KubernetesExecutor”：
    &gt; 
    &gt; &gt; {“image”：“myCustomDockerImage”}}

    ）``

```py
清晰（** kwargs） 
```

根据指定的参数清除与任务关联的任务实例的状态。

```py
DAG 
```

如果设置则返回运算符的 DAG，否则引发错误

```py
deps 
```

返回运算符的依赖项列表。这些与执行上下文依赖性的不同之处在于它们特定于任务，并且可以由子类扩展/覆盖。

```py
downstream_list 
```

@property：直接下游的任务列表

```py
执行（上下文） 
```

这是在创建运算符时派生的主要方法。Context 与渲染 jinja 模板时使用的字典相同。

有关更多上下文，请参阅 get_template_context。

```py
get_direct_relative_ids（上游=假） 
```

获取当前任务（上游或下游）的直接相对 ID。

```py
get_direct_relatives（上游=假） 
```

获取当前任务的直接亲属，上游或下游。

```py
get_flat_relative_ids（upstream = False，found_descendants = None） 
```

获取上游或下游的亲属 ID 列表。

```py
get_flat_relatives（上游=假） 
```

获取上游或下游亲属的详细列表。

```py
get_task_instances（session，start_date = None，end_date = None） 
```

获取与特定日期范围的此任务相关的一组任务实例。

```py
has_dag（） 
```

如果已将运算符分配给 DAG，则返回 True。

```py
on_kill（） 
```

当任务实例被杀死时，重写此方法以清除子进程。在操作员中使用线程，子过程或多处理模块的任何使用都需要清理，否则会留下重影过程。

```py
post_execute（context，* args，** kwargs） 
```

调用 self.execute（）后立即触发此挂接。它传递执行上下文和运算符返回的任何结果。

```py
pre_execute（context，* args，** kwargs） 
```

在调用 self.execute（）之前触发此挂钩。

```py
prepare_template（） 
```

模板化字段被其内容替换后触发的挂钩。如果您需要操作员在呈现模板之前更改文件的内容，则应覆盖此方法以执行此操作。

```py
render_template（attr，content，context） 
```

从文件或直接在字段中呈现模板，并返回呈现的结果。

```py
render_template_from_field（attr，content，context，jinja_env） 
```

从字段中呈现模板。如果字段是字符串，它将只是呈现字符串并返回结果。如果它是集合或嵌套的集合集，它将遍历结构并呈现其中的所有字符串。

```py
run（start_date = None，end_date = None，ignore_first_depends_on_past = False，ignore_ti_state = False，mark_success = False） 
```

为日期范围运行一组任务实例。

```py
schedule_interval 
```

DAG 的计划间隔始终胜过单个任务，因此 DAG 中的任务始终排列。该任务仍然需要 schedule_interval，因为它可能未附加到 DAG。

```py
set_downstream（task_or_task_list） 
```

将任务或任务列表设置为直接位于当前任务的下游。

```py
set_upstream（task_or_task_list） 
```

将任务或任务列表设置为直接位于当前任务的上游。

```py
upstream_list 
```

@property：直接上游的任务列表

```py
xcom_pull（context，task_ids = None，dag_id = None，key = u'return_value'，include_prior_dates = None） 
```

请参见 TaskInstance.xcom_pull（）

```py
xcom_push（context，key，value，execution_date = None） 
```

请参见 TaskInstance.xcom_push（）

```py
class airflow.models.Chart（** kwargs） 
```

基类： `sqlalchemy.ext.declarative.api.Base`

```py
class airflow.models.Connection（conn_id = None，conn_type = None，host = None，login = None，password = None，schema = None，port = None，extra = None，uri = None） 
```

基类：`sqlalchemy.ext.declarative.api.Base`，`airflow.utils.log.logging_mixin.LoggingMixin`

占位符用于存储有关不同数据库实例连接信息的信息。这里的想法是脚本使用对数据库实例（conn_id）的引用，而不是在使用运算符或钩子时硬编码主机名，登录名和密码。

```py
extra_dejson 
```

通过反序列化 json 返回额外的属性。

```py
class airflow.models.DAG（dag_id，description = u''，schedule_interval = datetime.timedelta（1），start_date = None，end_date = None，full_filepath = None，template_searchpath = None，user_defined_macros = None，user_defined_filters = None，default_args = None，concurrency = 16，max_active_runs = 16，dagrun_timeout = None，sla_miss_callback = None，default_view = u'tree'，orientation ='LR'，catchup = True，on_success_callback = None，on_failure_callback = None，params = None） 
```

基类：`airflow.dag.base_dag.BaseDag`，`airflow.utils.log.logging_mixin.LoggingMixin`

dag（有向无环图）是具有方向依赖性的任务集合。dag 也有一个时间表，一个开始结束日期（可选）。对于每个计划（例如每天或每小时），DAG 需要在满足其依赖性时运行每个单独的任务。某些任务具有依赖于其自身过去的属性，这意味着它们在完成之前的计划（和上游任务）之前无法运行。

DAG 本质上充当任务的命名空间。task_id 只能添加一次到 DAG。

参数：

*   `dag_id(string)` - DAG 的 ID
*   `description(string)` - 例如，在 Web 服务器上显示 DAG 的描述
*   `schedule_interval(_datetime.timedelta _ 或 _dateutil.relativedelta.relativedelta _ 或 _ 作为 cron 表达式的 str_)` - 定义 DAG 运行的频率，此 timedelta 对象被添加到最新任务实例的 execution_date 以确定下一个计划
*   `start_date(_datetime.datetime_)` - 调度程序将尝试回填的时间戳
*   `end_date(_datetime.datetime_)` - DAG 无法运行的日期，对于开放式调度保留为 None
*   `template_searchpath(string 或 _stings 列表 _)` - 此文件夹列表（非相对）定义 jinja 将在哪里查找模板。订单很重要。请注意，jinja / airflow 默认包含 DAG 文件的路径
*   `user_defined_macros(dict)` - 将在您的 jinja 模板中公开的宏字典。例如，传递`dict(foo='bar')`给此参数允许您在与此 DAG 相关的所有 jinja 模板中。请注意，您可以在此处传递任何类型的对象。`{{ foo }}`
*   `user_defined_filters(dict)` - 将在 jinja 模板中公开的过滤器字典。例如，传递给此参数允许您在与此 DAG 相关的所有 jinja 模板中。`dict(hello=lambda name: 'Hello %s' % name)``{{ 'world' &#124; hello }}`
*   `default_args(dict)` - 初始化运算符时用作构造函数关键字参数的默认参数的字典。请注意，运算符具有相同的钩子，并且在此处定义的钩子之前，这意味着如果您的 dict 包含`'depends_on_past'：`这里为`True`并且`'depends_on_past'：`在运算符的调用`default_args 中`为`False`，实际值将为`False`。
*   `params(dict)` - DAG 级别参数的字典，可在模板中访问，在`params`下命名。这些参数可以在任务级别覆盖。
*   `concurrency(int)` - 允许并发运行的任务实例数
*   `max_active_runs(int)` - 活动 DAG 运行的最大数量，超过运行状态下运行的 DAG 数量，调度程序将不会创建新的活动 DAG 运行
*   `dagrun_timeout(datetime.timedelta)` - 指定在超时/失败之前 DagRun 应该运行多长时间，以便可以创建新的 DagRuns
*   `sla_miss_callback(_types.FunctionType_)` - 指定报告 SLA 超时时要调用的函数。
*   `default_view(string)` - 指定 DAG 默认视图（树，图，持续时间，甘特图，landing_times）
*   `orientation(string)` - 在图表视图中指定 DAG 方向（LR，TB，RL，BT）
*   `catchup(bool)` - 执行调度程序追赶（或只运行最新）？默认为 True
*   `on_failure_callback(callable)` - 当这个 dag 的 DagRun 失败时要调用的函数。上下文字典作为单个参数传递给此函数。
*   `on_success_callback(callable)` - 很像`on_failure_callback`是在 dag 成功时执行的。

```py
add_task（任务） 
```

将任务添加到 DAG

参数：`任务(_task_)` - 要添加的任务

```py
add_tasks（任务） 
```

将任务列表添加到 DAG

参数：`任务(**任务**list)` - 您要添加的任务

```py
清晰（** kwargs） 
```

清除与指定日期范围的当前 dag 关联的一组任务实例。

```py
CLI（） 
```

公开特定于此 DAG 的 CLI

```py
concurrency_reached 
```

返回一个布尔值，指示是否已达到此 DAG 的并发限制

```py
create_dagrun（** kwargs） 
```

从这个 dag 创建一个 dag run，包括与这个 dag 相关的任务。返回 dag run。

参数：

*   `run_id(string)` - 定义此 dag 运行的运行 ID
*   `execution_date(datetime)` - 此 dag 运行的执行日期
*   `州(_ 州 _)` - dag run 的状态
*   `start_date(datetime)` - 应评估此 dag 运行的日期
*   `external_trigger(bool)` - 这个 dag run 是否是外部触发的
*   `session(_Session_)` - 数据库会话

```py
static deactivate_stale_dags（* args，** kwargs） 
```

在到期日期之前，停用调度程序最后触及的所有 DAG。这些 DAG 可能已被删除。

参数：`expiration_date(datetime)` - 设置在此时间之前触摸的非活动 DAG 返回值： 没有

```py
static deactivate_unknown_dags（* args，** kwargs） 
```

给定已知 DAG 列表，停用在 ORM 中标记为活动的任何其他 DAG

参数：`active_dag_ids(_list __[ __unicode __]_)` - 活动的 DAG ID 列表返回值： 没有

```py
文件路径 
```

dag 对象实例化的文件位置

```py
夹 
```

dag 对象实例化的文件夹位置

```py
following_schedule（DTTM） 
```

在当地时间计算此 dag 的以下计划

参数：`dttm` - utc datetime 返回值： utc datetime

```py
get_active_runs（** kwargs） 
```

返回当前运行的 dag 运行执行日期列表

参数：`会议` -返回值： 执行日期清单

```py
get_dagrun（** kwargs） 
```

返回给定执行日期的 dag 运行（如果存在），否则为 none。

参数：

*   `execution_date` - 要查找的 DagRun 的执行日期。
*   `会议` -

返回值： 如果找到 DagRun，否则为 None。

```py
get_last_dagrun（** kwargs） 
```

返回此 dag 的最后一个 dag 运行，如果没有则返回 None。最后的 dag run 可以是任何类型的运行，例如。预定或回填。重写的 DagRuns 被忽略

```py
get_num_active_runs（** kwargs） 
```

返回活动“正在运行”的 dag 运行的数量

参数：

*   `external_trigger(bool)` - 对于外部触发的活动 dag 运行为 True
*   `会议` -

返回值： 活动 dag 运行的数字大于 0

```py
static get_num_task_instances（* args，** kwargs） 
```

返回给定 DAG 中的任务实例数。

参数：

*   `会话` - ORM 会话
*   `dag_id(_unicode_)` - 获取任务并发性的 DAG 的 ID
*   `task_ids(_list __[ __unicode __]_)` - 给定 DAG 的有效任务 ID 列表
*   `states(_list __[ __state __]_)` - 要提供的过滤状态列表

返回值： 正在运行的任务数量| 返回类型： | INT

```py
get_run_dates（start_date，end_date = None） 
```

使用此 dag 的计划间隔返回作为参数接收的间隔之间的日期列表。返回日期可用于执行日期。

参数：

*   `start_date(datetime)` - 间隔的开始日期
*   `end_date(datetime)` - 间隔的结束日期，默认为 timezone.utcnow（）

返回值： dag 计划之后的时间间隔内的日期列表| 返回类型： | 名单

```py
get_template_env（） 
```

返回 jinja2 环境，同时考虑 DAGs template_searchpath，user_defined_macros 和 user_defined_filters

```py
handle_callback（** kwargs） 
```

根据成功的值触发相应的回调，即 on_failure_callback 或 on_success_callback。此方法获取此 DagRun 的单个 TaskInstance 部分的上下文，并将其与“原因”一起传递给 callable，主要用于区分 DagRun 失败。.. 注意：

```py
The logs end up in $AIRFLOW_HOME/logs/scheduler/latest/PROJECT/DAG_FILE.py.log

```

参数：

*   `dagrun` - DagRun 对象
*   `success` - 指定是否应调用失败或成功回调的标志
*   `理由` - 完成原因
*   `session` - 数据库会话

```py
is_paused 
```

返回一个布尔值，指示此 DAG 是否已暂停

```py
latest_execution_date 
```

返回至少存在一个 dag 运行的最新日期

```py
normalize_schedule（DTTM） 
```

返回 dttm + interval，除非 dttm 是第一个区间，然后它返回 dttm

```py
previous_schedule（DTTM） 
```

在当地时间计算此 dag 的先前计划

参数：`dttm` - utc datetime 返回值： utc datetime

```py
run（start_date = None，end_date = None，mark_success = False，local = False，executor = None，donot_pickle = False，ignore_task_deps = False，ignore_first_depends_on_past = False，pool = None，delay_on_limit_secs = 1.0，verbose = False，conf = None， rerun_failed_tasks = FALSE） 
```

运行 DAG。

参数：

*   `start_date(datetime)` - 要运行的范围的开始日期
*   `end_date(datetime)` - 要运行的范围的结束日期
*   `mark_success(bool)` - 如果成功，则将作业标记为成功而不运行它们
*   `local(bool)` - 如果使用 LocalExecutor 运行任务，则为 True
*   `executor(_BaseExecutor_)` - 运行任务的执行程序实例
*   `donot_pickle(bool)` - 正确以避免腌制 DAG 对象并发送给工人
*   `ignore_task_deps(bool)` - 如果为 True 则跳过上游任务
*   `ignore_first_depends_on_past(bool)` - 如果为 True，则忽略第一组任务的 depends_on_past 依赖项
*   `pool(string)` - 要使用的资源池
*   `delay_on_limit_secs(float)` - 达到 max_active_runs 限制时，下次尝试运行 dag run 之前等待的时间（以秒为单位）
*   `verbose(_boolean_)` - 使日志输出更详细
*   `conf(dict)` - 从 CLI 传递的用户定义字典

```py
set_dependency（upstream_task_id，downstream_task_id） 
```

使用 add_task（）在已添加到 DAG 的两个任务之间设置依赖关系的简单实用工具方法

```py
sub_dag（task_regex，include_downstream = False，include_upstream = True） 
```

基于应该与一个或多个任务匹配的正则表达式返回当前 dag 的子集作为当前 dag 的深层副本，并且包括基于传递的标志的上游和下游邻居。

```py
subdags 
```

返回与此 DAG 关联的子标记对象的列表

```py
sync_to_db（** kwargs） 
```

将有关此 DAG 的属性保存到数据库。请注意，可以为 DAG 和 SubDAG 调用此方法。SubDag 实际上是 SubDagOperator。

参数：

*   `dag([_DAG_](31 "airflow.models.DAG"))` - 要保存到 DB 的 DAG 对象
*   `sync_time(datetime)` - 应将 DAG 标记为 sync'ed 的时间

返回值： 没有

```py
test_cycle（） 
```

检查 DAG 中是否有任何循环。如果没有找到循环，则返回 False，否则引发异常。

```py
topological_sort（） 
```

按地形顺序对任务进行排序，以便任务在其任何上游依赖项之后。

深受启发：[http](http://blog.jupo.org/2012/04/06/topological-sorting-acyclic-directed-graphs/)：[//blog.jupo.org/2012/04/06/topological-sorting-acyclic-directed-graphs/](http://blog.jupo.org/2012/04/06/topological-sorting-acyclic-directed-graphs/)

返回值： 拓扑顺序中的任务列表

```py
树视图（） 
```

显示 DAG 的 ascii 树表示

```py
class airflow.models.DagBag（dag_folder = None，executor = None，include_examples = False） 
```

基类：`airflow.dag.base_dag.BaseDagBag`，`airflow.utils.log.logging_mixin.LoggingMixin`

dagbag 是 dags 的集合，从文件夹树中解析出来并具有高级配置设置，例如用作后端的数据库以及用于触发任务的执行器。这样可以更轻松地为生产和开发，测试或不同的团队或安全配置文件运行不同的环境。系统级别设置现在是 dagbag 级别，以便一个系统可以运行多个独立的设置集。

参数：

*   `dag_folder(_unicode_)` - 要扫描以查找 DAG 的文件夹
*   `executor` - 在此 DagBag 中执行任务实例时使用的执行程序
*   `include_examples(bool)` - 是否包含带有气流的示例
*   `has_logged` - 在跳过文件后从 False 翻转为 True 的实例布尔值。这是为了防止用户记录有关跳过文件的消息而使用户过载。因此，每个 DagBag 只有一次被记录的文件被跳过。

```py
bag_dag（dag，parent_dag，root_dag） 
```

将 DAG 添加到包中，递归到子 d .. 如果在此 dag 或其子标记中检测到循环，则抛出 AirflowDagCycleException

```py
collect_dags（dag_folder = None，only_if_updated = True） 
```

给定文件路径或文件夹，此方法查找 python 模块，导入它们并将它们添加到 dagbag 集合中。

请注意，如果在处理目录时发现.airflowignore 文件，它的行为与.gitignore 非常相似，忽略了与文件中指定的任何正则表达式模式匹配的文件。**注意**：.airflowignore 中的模式被视为未锚定的正则表达式，而不是类似 shell 的 glob 模式。

```py
dagbag_report（） 
```

打印有关 DagBag loading stats 的报告

```py
get_dag（dag_id） 
```

从字典中获取 DAG，并在过期时刷新它

```py
kill_zombies（** kwargs） 
```

失败的任务太长时间没有心跳

```py
process_file（filepath，only_if_updated = True，safe_mode = True） 
```

给定 python 模块或 zip 文件的路径，此方法导入模块并在其中查找 dag 对象。

```py
尺寸（） 
```

返回值： 这个 dagbag 中包含的 dags 数量

```py
class airflow.models.DagModel（** kwargs） 
```

基类： `sqlalchemy.ext.declarative.api.Base`

```py
class airflow.models.DagPickle（dag） 
```

基类： `sqlalchemy.ext.declarative.api.Base`

Dags 可以来自不同的地方（用户回购，主回购，......），也可以在不同的地方（不同的执行者）执行。此对象表示 DAG 的一个版本，并成为 BackfillJob 执行的真实来源。pickle 是本机 python 序列化对象，在这种情况下，在作业持续时间内存储在数据库中。

执行程序获取 DagPickle id 并从数据库中读取 dag 定义。

```py
class airflow.models.DagRun（** kwargs） 
```

基类：`sqlalchemy.ext.declarative.api.Base`，`airflow.utils.log.logging_mixin.LoggingMixin`

DagRun 描述了 Dag 的一个实例。它可以由调度程序（用于常规运行）或外部触发器创建

```py
静态查找（* args，** kwargs） 
```

返回给定搜索条件的一组 dag 运行。

参数：

*   `dag_id(_integer __，_ list)` - 用于查找 dag 的 dag_id
*   `run_id(string)` - 定义此 dag 运行的运行 ID
*   `execution_date(datetime)` - 执行日期
*   `州(_ 州 _)` - dag run 的状态
*   `external_trigger(bool)` - 这个 dag run 是否是外部触发的
*   `no_backfills` - 返回无回填（True），全部返回（False）。


默认为 False：键入 no_backfills：bool：param session：数据库会话：类型会话：会话

```py
get_dag（） 
```

返回与此 DagRun 关联的 Dag。

返回值： DAG

```py
classmethod get_latest_runs（** kwargs） 
```

返回每个 DAG 的最新 DagRun。

```py
get_previous_dagrun（** kwargs） 
```

以前的 DagRun，如果有的话

```py
get_previous_scheduled_dagrun（** kwargs） 
```

如果有的话，以前的 SCHEDULED DagRun

```py
static get_run（session，dag_id，execution_date） 
```

参数：

*   `dag_id(_unicode_)` - DAG ID
*   `execution_date(datetime)` - 执行日期

返回值： DagRun 对应于给定的 dag_id 和执行日期
如果存在的话。否则没有。：rtype：DagRun

```py
get_task_instance（** kwargs） 
```

返回此 dag 运行的 task_id 指定的任务实例

参数：`task_id` - 任务 ID

```py
get_task_instances（** kwargs） 
```

返回此 dag 运行的任务实例

```py
refresh_from_db（** kwargs） 
```

从数据库重新加载当前 dagrun：param session：数据库会话

```py
update_state（** kwargs） 
```

根据 TaskInstances 的状态确定 DagRun 的整体状态。

返回值： 州

```py
verify_integrity（** kwargs） 
```

通过检查已删除的任务或尚未在数据库中的任务来验证 DagRun。如果需要，它会将状态设置为已删除或添加任务。

```py
class airflow.models.DagStat（dag_id，state，count = 0，dirty = False） 
```

基类： `sqlalchemy.ext.declarative.api.Base`

```py
静态创建（* args，** kwargs） 
```

为指定的 dag 创建缺少状态 stats 表

参数：

*   `dag_id` - 用于创建统计数据的 dag 的 dag id
*   `会话` - 数据库会话

返回值：

```py
static set_dirty（* args，** kwargs） 
```

参数：

*   `dag_id` - 用于标记脏的 dag_id
*   `会话` - 数据库会话

返回值：

```py
静态更新（* args，** kwargs） 
```

更新脏/不同步 dag 的统计信息

参数：

*   `dag_ids(list)` - 要更新的 dag_ids
*   `dirty_only(bool)` - 仅针对标记的脏更新，默认为 True
*   `session(_Session_)` - 要使用的 db 会话

```py
class airflow.models.ImportError（** kwargs） 
```

基类： `sqlalchemy.ext.declarative.api.Base`

```py
exception airflow.models.InvalidFernetToken 
```

基类： `exceptions.Exception`

```py
class airflow.models.KnownEvent（** kwargs） 
```

基类： `sqlalchemy.ext.declarative.api.Base`

```py
class airflow.models.KnownEventType（** kwargs） 
```

基类： `sqlalchemy.ext.declarative.api.Base`

```py
class airflow.models.KubeResourceVersion（** kwargs） 
```

基类： `sqlalchemy.ext.declarative.api.Base`

```py
class airflow.models.KubeWorkerIdentifier（** kwargs） 
```

基类： `sqlalchemy.ext.declarative.api.Base`

```py
class airflow.models.Log（event，task_instance，owner = None，extra = None，** kwargs） 
```

基类： `sqlalchemy.ext.declarative.api.Base`

用于主动将事件记录到数据库

```py
class airflow.models.NullFernet 
```

基类： `future.types.newobject.newobject`

“Null”加密器类，不加密或解密，但提供与 Fernet 类似的接口。

这样做的目的是使其余代码不必知道差异，并且只显示消息一次，而不是在运行`气流 initdb`时显示 20 次。

```py
class airflow.models.Pool（** kwargs） 
```

基类： `sqlalchemy.ext.declarative.api.Base`

```py
open_slots（** kwargs） 
```

返回此刻打开的插槽数

```py
queued_slots（** kwargs） 
```

返回此刻使用的插槽数

```py
used_slots（** kwargs） 
```

返回此刻使用的插槽数

```py
class airflow.models.SlaMiss（** kwargs） 
```

基类： `sqlalchemy.ext.declarative.api.Base`

存储已错过的 SLA 历史的模型。它用于跟踪 SLA 故障，并避免双重触发警报电子邮件。

```py
class airflow.models.TaskFail（task，execution_date，start_date，end_date） 
```

基类： `sqlalchemy.ext.declarative.api.Base`

TaskFail 跟踪每个任务实例的失败运行持续时间。

```py
class airflow.models.TaskInstance（task，execution_date，state = None） 
```

基类：`sqlalchemy.ext.declarative.api.Base`，`airflow.utils.log.logging_mixin.LoggingMixin`

任务实例存储任务实例的状态。该表是围绕已运行任务及其所处状态的权威和单一事实来源。

SqlAlchemy 模型没有任务或 dag 模型的 SqlAlchemy 外键故意对事务进行更多控制。

此表上的数据库事务应该确保双触发器以及围绕哪些任务实例准备好运行的任何混淆，即使多个调度程序可能正在触发任务实例。

```py
are_dependencies_met（** kwargs） 
```

返回在给定依赖关系的上下文的情况下是否满足所有条件以运行此任务实例（例如，从 UI 强制运行的任务实例将忽略某些依赖关系）。

参数：

*   `dep_context(_DepContext_)` - 确定应评估的依赖项的执行上下文。
*   `session(_Session_)` - 数据库会话
*   `verbose(_boolean_)` - 是否在信息或调试日志级别上的失败依赖项的日志详细信息

```py
are_dependents_done（** kwargs） 
```

检查此任务实例的依赖项是否都已成功。这应该由 wait_for_downstream 使用。

当您不希望在完成依赖项之前开始处理任务的下一个计划时，这非常有用。例如，如果任务 DROPs 并重新创建表。

```py
clear_xcom_data（** kwargs） 
```

从数据库中清除任务实例的所有 XCom 数据

```py
command（mark_success = False，ignore_all_deps = False，ignore_depends_on_past = False，ignore_task_deps = False，ignore_ti_state = False，local = False，pickle_id = None，raw = False，job_id = None，pool = None，cfg_path = None） 
```

返回可在安装气流的任何位置执行的命令。此命令是 orchestrator 发送给执行程序的消息的一部分。

```py
command_as_list（mark_success = False，ignore_all_deps = False，ignore_task_deps = False，ignore_depends_on_past = False，ignore_ti_state = False，local = False，pickle_id = None，raw = False，job_id = None，pool = None，cfg_path = None） 
```

返回可在安装气流的任何位置执行的命令。此命令是 orchestrator 发送给执行程序的消息的一部分。

```py
current_state（** kwargs） 
```

从数据库中获取最新状态，如果会话通过，我们使用并查找状态将成为会话的一部分，否则将使用新会话。

```py
误差（** kwargs） 
```

在数据库中强制将任务实例的状态设置为 FAILED。

```py
static generate_command（dag_id，task_id，execution_date，mark_success = False，ignore_all_deps = False，ignore_depends_on_past = False，ignore_task_deps = False，ignore_ti_state = False，local = False，pickle_id = None，file_path = None，raw = False，job_id = None，pool =无，cfg_path =无） 
```

生成执行此任务实例所需的 shell 命令。

参数：

*   `dag_id(_unicode_)` - DAG ID
*   `task_id(_unicode_)` - 任务 ID
*   `execution_date(datetime)` - 任务的执行日期
*   `mark_success(bool)` - 是否将任务标记为成功
*   `ignore_all_deps(_boolean_)` - 忽略所有可忽略的依赖项。覆盖其他 ignore_ *参数。
*   `ignore_depends_on_past(_boolean_)` - 忽略 DAG 的 depends_on_past 参数（例如，对于回填）
*   `ignore_task_deps(_boolean_)` - 忽略任务特定的依赖项，例如 depends_on_past 和触发器规则
*   `ignore_ti_state(_boolean_)` - 忽略任务实例之前的失败/成功
*   `local(bool)` - 是否在本地运行任务
*   `pickle_id(_unicode_)` - 如果 DAG 序列化到 DB，则与酸洗 DAG 关联的 ID
*   `file_path` - 包含 DAG 定义的文件的路径
*   `原始` - 原始模式（需要更多细节）
*   `job_id` - 工作 ID（需要更多细节）
*   `pool(_unicode_)` - 任务应在其中运行的 Airflow 池
*   `cfg_path(_basestring_)` - 配置文件的路径

返回值： shell 命令，可用于运行任务实例

```py
get_dagrun（** kwargs） 
```

返回此 TaskInstance 的 DagRun

参数：`会议` -返回值： DagRun

```py
init_on_load（） 
```

初始化未存储在 DB 中的属性。

```py
init_run_context（原始=假） 
```

设置日志上下文。

```py
is_eligible_to_retry（） 
```

任务实例是否有资格重试

```py
is_premature 
```

返回任务是否处于 UP_FOR_RETRY 状态且其重试间隔是否已过去。

```py
key 
```

返回唯一标识任务实例的元组

```py
next_retry_datetime（） 
```

如果任务实例失败，则获取下次重试的日期时间。对于指数退避，retry_delay 用作基数并将转换为秒。

```py
pool_full（** kwargs） 
```

返回一个布尔值，指示槽池是否有空间运行此任务

```py
previous_ti 
```

在此任务实例之前运行的任务的任务实例

```py
ready_for_retry（） 
```

检查任务实例是否处于正确状态和重试时间范围。

```py
refresh_from_db（** kwargs） 
```

根据主键刷新数据库中的任务实例

参数：`lock_for_update` - 如果为 True，则表示数据库应锁定 TaskInstance（发出 FOR UPDATE 子句），直到提交会话为止。

```py
try_number 
```

返回该任务编号在实际运行时的尝试编号。

如果 TI 当前正在运行，这将匹配数据库中的列，在所有其他情况下，这将是增量的

```py
xcom_pull（task_ids = None，dag_id = None，key = u'return_value'，include_prior_dates = False） 
```

拉 XComs 可选择满足特定条件。

`key`的默认值将搜索限制为由其他任务返回的 XComs（而不是手动推送的那些）。要删除此过滤器，请传递 key = None（或任何所需的值）。

如果提供了单个 task_id 字符串，则结果是该 task_id 中最新匹配的 XCom 的值。如果提供了多个 task_id，则返回匹配值的元组。无法找到匹配项时返回 None。

参数：

*   `key(string)` - XCom 的密钥。如果提供，将仅返回具有匹配键的 XCom。默认键是'return_value'，也可以作为常量 XCOM_RETURN_KEY 使用。此键自动提供给任务返回的 XComs（而不是手动推送）。要删除过滤器，请传递 key = None。
*   `task_ids(string 或 _ 可迭代的字符串 _ _（_ _ 表示 task_ids __）_)` - 仅提取具有匹配 ID 的任务的 XCom。可以通过 None 删除过滤器。
*   `dag_id(string)` - 如果提供，则仅从此 DAG 中提取 XComs。如果为 None（默认值），则使用调用任务的 DAG。
*   `include_prior_dates(bool)` - 如果为 False，则仅返回当前 execution_date 中的 XComs。如果为 True，则返回之前日期的 XComs。

```py
xcom_push（key，value，execution_date = None） 
```

使 XCom 可用于执行任务。

参数：

*   `key(string)` - XCom 的密钥
*   `value(_ 任何 pickleable 对象 _)` - _XCom_ 的值。该值被 pickle 并存储在数据库中。
*   `execution_date(datetime)` - 如果提供，XCom 将在此日期之前不可见。例如，这可以用于在将来的日期将消息发送到任务，而不会立即显示。

```py
class airflow.models.User（** kwargs） 
```

基类： `sqlalchemy.ext.declarative.api.Base`

```py
class airflow.models.Variable（** kwargs） 
```

基类：`sqlalchemy.ext.declarative.api.Base`，`airflow.utils.log.logging_mixin.LoggingMixin`

```py
classmethod setdefault（key，default，deserialize_json = False） 
```

与 Python 内置的 dict 对象一样，setdefault 返回键的当前值，如果不存在，则存储默认值并返回它。

参数：

*   `key(string)` - 此变量的 Dict 键
*   `default` - 要设置的默认值，如果变量则返回


不在 DB 中：type default：Mixed：param deserialize_json：将其存储为 DB 中的 JSON 编码值

> 并在检索值时取消编码

返回值： 杂

```py
class airflow.models.XCom（** kwargs） 
```

基类：`sqlalchemy.ext.declarative.api.Base`，`airflow.utils.log.logging_mixin.LoggingMixin`

XCom 对象的基类。

```py
classmethod get_many（** kwargs） 
```

检索 XCom 值，可选地满足某些条件 TODO：“pickling”已被弃用，JSON 是首选。

> Airflow 2.0 中将删除“酸洗”。

```py
classmethod get_one（** kwargs） 
```

检索 XCom 值，可选择满足特定条件。TODO：“pickling”已被弃用，JSON 是首选。

> Airflow 2.0 中将删除“酸洗”。

返回值： XCom 值

```py
classmethod set（** kwargs） 
```

存储 XCom 值。TODO：“pickling”已被弃用，JSON 是首选。

> Airflow 2.0 中将删除“酸洗”。

返回值： 没有

```py
airflow.models.clear_task_instances（tis，session，activate_dag_runs = True，dag = None） 
```

清除一组任务实例，但确保正在运行的任务实例被杀死。

参数：

*   `tis` - 任务实例列表
*   `会话` - 当前会话
*   `activate_dag_runs` - 用于检查活动 dag 运行的标志
*   `dag` - DAG 对象

```py
airflow.models.get_fernet（） 
```

Fernet 密钥的延期负载。

此功能可能因为未安装加密或因为 Fernet 密钥无效而失败。

返回值： Fernet 对象| 举： | 如果尝试加载 Fernet 时出现问题，则会出现 AirflowException
## 钩

钩子是外部平台和数据库的接口，在可能的情况下实现通用接口，并充当操作员的构建块。

```py
class airflow.hooks.dbapi_hook.DbApiHook（* args，** kwargs） 
```

基类： `airflow.hooks.base_hook.BaseHook`

sql hooks 的抽象基类。

```py
bulk_dump（table，tmp_file） 
```

将数据库表转储到制表符分隔的文件中

参数：

*   `table(str)` - 源表的名称
*   `tmp_file(str)` - 目标文件的路径

```py
bulk_load（table，tmp_file） 
```

将制表符分隔的文件加载到数据库表中

参数：

*   `table(str)` - 目标表的名称
*   `tmp_file(str)` - 要加载到表中的文件的路径

```py
get_autocommit（康涅狄格州） 
```

获取提供的连接的自动提交设置。如果 conn.autocommit 设置为 True，则返回 True。如果未设置 conn.autocommit 或将其设置为 False 或 conn 不支持自动提交，则返回 False。：param conn：从中获取自动提交设置的连接。：type conn：连接对象。：return：连接自动提交设置。：rtype 布尔。

```py
get_conn（） 
```

返回一个连接对象

```py
get_cursor（） 
```

返回一个游标

```py
get_first（sql，parameters = None） 
```

执行 sql 并返回第一个结果行。

参数：

*   `sql(_str _ 或 list)` - 要执行的 sql 语句（str）或要执行的 sql 语句列表
*   `参数(_mapping _ 或 _iterable_)` - 用于呈现 SQL 查询的参数。

```py
get_pandas_df（sql，parameters = None） 
```

执行 sql 并返回一个 pandas 数据帧

参数：

*   `sql(_str _ 或 list)` - 要执行的 sql 语句（str）或要执行的 sql 语句列表
*   `参数(_mapping _ 或 _iterable_)` - 用于呈现 SQL 查询的参数。

```py
get_records（sql，parameters = None） 
```

执行 sql 并返回一组记录。

参数：

*   `sql(_str _ 或 list)` - 要执行的 sql 语句（str）或要执行的 sql 语句列表
*   `参数(_mapping _ 或 _iterable_)` - 用于呈现 SQL 查询的参数。

```py
insert_rows（table，rows，target_fields = None，commit_every = 1000，replace = False） 
```

将一组元组插入表中的通用方法是，每个 commit_every 行都会创建一个新事务

参数：

*   `table(str)` - 目标表的名称
*   `rows(_ 可迭代的元组 _)` - 要插入表中的行
*   `target_fields(_ 可迭代的字符串 _)` - 要填充表的列的名称
*   `commit_every(int)` - 要在一个事务中插入的最大行数。设置为 0 以在一个事务中插入所有行。
*   `replace(bool)` - 是否替换而不是插入

```py
run（sql，autocommit = False，parameters = None） 
```

运行命令或命令列表。将 sql 语句列表传递给 sql 参数，以使它们按顺序执行

参数：

*   `sql(_str _ 或 list)` - 要执行的 sql 语句（str）或要执行的 sql 语句列表
*   `autocommit(bool)` - 在执行查询之前将连接的自动提交设置设置为什么。
*   `参数(_mapping _ 或 _iterable_)` - 用于呈现 SQL 查询的参数。

```py
set_autocommit（conn，autocommit） 
```

设置连接上的自动提交标志

```py
class airflow.hooks.docker_hook.DockerHook（docker_conn_id ='docker_default'，base_url = None，version = None，tls = None） 
```

基类：`airflow.hooks.base_hook.BaseHook`，`airflow.utils.log.logging_mixin.LoggingMixin`

与私有 Docker 注册表交互。

参数：`docker_conn_id(str)` - 存储凭证和额外配置的 Airflow 连接的 ID

```py
class airflow.hooks.hive_hooks.HiveCliHook（hive_cli_conn_id = u'hive_cli_default'，run_as = None，mapred_queue = None，mapred_queue_priority = None，mapred_job_name = None） 
```

基类： `airflow.hooks.base_hook.BaseHook`

围绕 hive CLI 的简单包装。

它还支持`beeline`运行 JDBC 的轻量级 CLI，并取代较重的传统 CLI。要启用`beeline`，请在连接的额外字段中设置 use_beeline 参数，如下所示`{ "use_beeline": true }`

请注意，您还`hive_cli_params`可以使用要在连接中使用的默认配置单元 CLI 参数，因为在此处传递的参数可以被 run_cli 的 hive_conf 参数覆盖。`{"hive_cli_params": "-hiveconf mapred.job.tracker=some.jobtracker:444"}`

额外的连接参数`auth`将按原样在`jdbc`连接字符串中传递。

参数：

*   `mapred_queue(string)` - Hadoop 调度程序使用的队列（容量或公平）
*   `mapred_queue_priority(string)` - 作业队列中的优先级。可能的设置包括：VERY_HIGH，HIGH，NORMAL，LOW，VERY_LOW
*   `mapred_job_name(string)` - 此名称将出现在 jobtracker 中。这可以使监控更容易。

```py
load_df（df，table，field_dict = None，delimiter = u'，'，encoding = u'utf8'，pandas_kwargs = None，** kwargs） 
```

将 pandas DataFrame 加载到配置单元中。

如果未传递 Hive 数据类型，则会推断 Hive 数据类型，但不会清理列名称。

参数：

*   `df(_DataFrame_)` - 要加载到 Hive 表中的 DataFrame
*   `table(str)` - 目标 Hive 表，使用点表示法来定位特定数据库
*   `field_dict(_OrderedDict_)` - 从列名映射到 hive 数据类型。请注意，它必须是 OrderedDict 才能保持列的顺序。
*   `delimiter(str)` - 文件中的字段分隔符
*   `encoding(str)` - 将 DataFrame 写入文件时使用的字符串编码
*   `pandas_kwargs(dict)` - 传递给 DataFrame.to_csv
*   `kwargs` - 传递给 self.load_file

```py
load_file（filepath，table，delimiter = u'，'，field_dict = None，create = True，overwrite = True，partition = None，recreate = False，tblproperties = None） 
```

将本地文件加载到 Hive 中

请注意，在 Hive 中生成的表使用的不是最有效的序列化格式。如果加载了大量数据和/或表格被大量查询，您可能只想使用此运算符将数据暂存到临时表中，然后使用 a 将其加载到最终目标中。`STORED AS textfile``HiveOperator`

参数：

*   `filepath(str)` - 要加载的文件的本地文件路径
*   `table(str)` - 目标 Hive 表，使用点表示法来定位特定数据库
*   `delimiter(str)` - 文件中的字段分隔符
*   `field_dict(_OrderedDict_)` - 文件中字段的字典名称作为键，其 Hive 类型作为值。请注意，它必须是 OrderedDict 才能保持列的顺序。
*   `create(bool)` - 是否创建表，如果它不存在
*   `overwrite(bool)` - 是否覆盖表或分区中的数据
*   `partition(dict)` - 将目标分区作为分区列和值的字典
*   `recreate(bool)` - 是否在每次执行时删除并重新创建表
*   `tblproperties(dict)` - 正在创建的 hive 表的 TBLPROPERTIES

```py
run_cli（hql，schema = None，verbose = True，hive_conf = None） 
```

使用 hive cli 运行 hql 语句。如果指定了 hive_conf，它应该是一个 dict，并且条目将在 HiveConf 中设置为键/值对

参数：`hive_conf(dict)` - 如果指定，这些键值对将作为传递给 hive 。请注意，它们将在之后传递，因此将覆盖数据库中指定的任何值。`-hiveconf "key"="value"``hive_cli_params`

```py
 >>>  hh = HiveCliHook ()
>>>  result = hh . run_cli ( "USE airflow;" )
>>>  ( "OK" in result )
True

```

```py
test_hql（HQL） 
```

使用 hive cli 和 EXPLAIN 测试 hql 语句

```py
class airflow.hooks.hive_hooks.HiveMetastoreHook（metastore_conn_id = u'metastore_default'） 
```

基类： `airflow.hooks.base_hook.BaseHook`

包装器与 Hive Metastore 交互

```py
check_for_named_pa​​rtition（schema，table，partition_name） 
```

检查是否存在具有给定名称的分区

参数：

*   `schema(string)` - @table 所属的 hive 模式（数据库）的名称
*   `table` - @partition 所属的配置单元表的名称

| 划分： | 要检查的分区的名称（例如`a = b / c = d`）| 返回类型： | 布尔

```py
 >>>  hh = HiveMetastoreHook ()
>>>  t = 'static_babynames_partitioned'
>>>  hh . check_for_named_partition ( 'airflow' , t , "ds=2015-01-01" )
True
>>>  hh . check_for_named_partition ( 'airflow' , t , "ds=xxx" )
False

```

```py
check_for_partition（架构，表，分区） 
```

检查分区是否存在

参数：

*   `schema(string)` - @table 所属的 hive 模式（数据库）的名称
*   `table` - @partition 所属的配置单元表的名称

| 划分： | 与要检查的分区匹配的表达式（例如`a ='b'和 c ='d'`）| 返回类型： | 布尔

```py
 >>>  hh = HiveMetastoreHook ()
>>>  t = 'static_babynames_partitioned'
>>>  hh . check_for_partition ( 'airflow' , t , "ds='2015-01-01'" )
True

```

```py
get_databases（图案= U '*'） 
```

获取 Metastore 表对象

```py
get_metastore_client（） 
```

返回一个 Hive thrift 客户端。

```py
get_partitions（schema，table_name，filter = None） 
```

返回表中所有分区的列表。仅适用于小于 32767（java short max val）的表。对于子分区表，该数字可能很容易超过此值。

```py
 >>>  hh = HiveMetastoreHook ()
>>>  t = 'static_babynames_partitioned'
>>>  parts = hh . get_partitions ( schema = 'airflow' , table_name = t )
>>>  len ( parts )
1
>>>  parts
[{'ds': '2015-01-01'}]

```

```py
get_table（table_name，db = u'default'） 
```

获取 Metastore 表对象

```py
 >>>  hh = HiveMetastoreHook ()
>>>  t = hh . get_table ( db = 'airflow' , table_name = 'static_babynames' )
>>>  t . tableName
'static_babynames'
>>>  [ col . name for col in t . sd . cols ]
['state', 'year', 'name', 'gender', 'num']

```

```py
get_tables（db，pattern = u'*'） 
```

获取 Metastore 表对象

```py
max_partition（schema，table_name，field = None，filter_map = None） 
```

返回表中给定字段的所有分区的最大值。如果表中只存在一个分区键，则该键将用作字段。filter_map 应该是 partition_key：partition_value map，将用于过滤掉分区。

参数：

*   `schema(string)` - 模式名称。
*   `table_name(string)` - 表名。
*   `field(string)` - 从中​​获取最大分区的分区键。
*   `filter_map(_map_)` - partition_key：用于分区过滤的 partition_value 映射。

```py
 >>>  hh = HiveMetastoreHook ()
>>>  filter_map = { 'ds' : '2015-01-01' , 'ds' : '2014-01-01' }
>>>  t = 'static_babynames_partitioned'
>>>  hh . max_partition ( schema = 'airflow' ,        ... table_name = t , field = 'ds' , filter_map = filter_map )
'2015-01-01'

```

```py
table_exists（table_name，db = u'default'） 
```

检查表是否存在

```py
 >>>  hh = HiveMetastoreHook ()
>>>  hh . table_exists ( db = 'airflow' , table_name = 'static_babynames' )
True
>>>  hh . table_exists ( db = 'airflow' , table_name = 'does_not_exist' )
False

```

```py
class airflow.hooks.hive_hooks.HiveServer2Hook（hiveserver2_conn_id = u'hiveserver2_default'） 
```

基类： `airflow.hooks.base_hook.BaseHook`

pyhive 库周围的包装

请注意，默认的 authMechanism 是 PLAIN，要覆盖它，您可以`extra`在 UI 中的连接中指定它，如在

```py
get_pandas_df（hql，schema = u'default'） 
```

从 Hive 查询中获取 pandas 数据帧

```py
 >>>  hh = HiveServer2Hook ()
>>>  sql = "SELECT * FROM airflow.static_babynames LIMIT 100"
>>>  df = hh . get_pandas_df ( sql )
>>>  len ( df . index )
100

```

```py
get_records（hql，schema = u'default'） 
```

从 Hive 查询中获取一组记录。

```py
 >>>  hh = HiveServer2Hook ()
>>>  sql = "SELECT * FROM airflow.static_babynames LIMIT 100"
>>>  len ( hh . get_records ( sql ))
100

```

```py
class airflow.hooks.http_hook.HttpHook（method ='POST'，http_conn_id ='http_default'） 
```

基类： `airflow.hooks.base_hook.BaseHook`

与 HTTP 服务器交互。：param http_conn_id：具有基本 API 网址的连接，即[https://www.google.com/](https://www.google.com/)

> 和可选的身份验证凭据 也可以在 Json 格式的 Extra 字段中指定默认标头。

参数：`method(str)` - 要调用的 API 方法

```py
check_response（响应） 
```

检查状态代码并在非 2XX 或 3XX 状态代码上引发 AirflowException 异常：param response：请求响应对象：type response：requests.response

```py
get_conn（头=无） 
```

返回用于请求的 http 会话：param headers：要作为字典传递的其他标头：type headers：dict

```py
run（endpoint，data = None，headers = None，extra_options = None） 
```

执行请求：param endpoint：要调用的端点，即 resource / v1 / query？：type endpoint：str：param data：要上载的有效负载或请求参数：type data：dict：param headers：要作为字典传递的其他头：type headers：dict：param extra_options：执行时要使用的其他选项请求

> ie {'check_response'：False}以避免在非 2XX 或 3XX 状态代码上检查引发异常

```py
run_and_check（session，prepped_request，extra_options） 
```

获取超时等额外选项并实际运行请求，检查结果：param session：用于执行请求的会话：type session：requests.Session：param prepped_request：在 run（）中生成的准备好的请求：type prepped_request ：session.prepare_request：param extra_options：执行请求时要使用的其他选项

> ie {'check_response'：False}以避免在非 2XX 或 3XX 状态代码上检查引发异常

```py
run_with_advanced_retry（_retry_args，* args，** kwargs） 
```

运行 Hook.run（）并附加一个 Tenacity 装饰器。这对于可能受间歇性问题干扰并且不应立即失效的连接器非常有用。：param _retry_args：定义重试行为的参数。

> 请参阅[https://github.com/jd/tenacity 上的](https://github.com/jd/tenacity) Tenacity 文档[](https://github.com/jd/tenacity)

```py
 示例：:: 
```

hook = HttpHook（http_conn_id ='my_conn'，method ='GET'）retry_args = dict（

> &gt; wait = tenacity.wait_exponential（），stop = tenacity.stop_after_attempt（10），retry = requests.exceptions.ConnectionError
> 
> ）hook.run_with_advanced_retry（
> 
> &gt; &gt; endpoint ='v1 / test'，_ retry_args = retry_args
> &gt; 
> &gt; ）

```py
class airflow.hooks.druid_hook.DruidDbApiHook（* args，** kwargs） 
```

基类： [`airflow.hooks.dbapi_hook.DbApiHook`](31 "airflow.hooks.dbapi_hook.DbApiHook")

与德鲁伊经纪人互动

这个钩子纯粹是供用户查询德鲁伊经纪人。摄取，请使用 druidHook。

```py
get_conn（） 
```

建立与德鲁伊经纪人的联系。

```py
get_pandas_df（sql，parameters = None） 
```

执行 sql 并返回一个 pandas 数据帧

参数：

*   `sql(_str _ 或 list)` - 要执行的 sql 语句（str）或要执行的 sql 语句列表
*   `参数(_mapping _ 或 _iterable_)` - 用于呈现 SQL 查询的参数。

```py
get_uri（） 
```

获取德鲁伊经纪人的连接 uri。

例如：druid：// localhost：8082 / druid / v2 / sql /

```py
insert_rows（table，rows，target_fields = None，commit_every = 1000） 
```

将一组元组插入表中的通用方法是，每个 commit_every 行都会创建一个新事务

参数：

*   `table(str)` - 目标表的名称
*   `rows(_ 可迭代的元组 _)` - 要插入表中的行
*   `target_fields(_ 可迭代的字符串 _)` - 要填充表的列的名称
*   `commit_every(int)` - 要在一个事务中插入的最大行数。设置为 0 以在一个事务中插入所有行。
*   `replace(bool)` - 是否替换而不是插入

```py
set_autocommit（conn，autocommit） 
```

设置连接上的自动提交标志

```py
class airflow.hooks.druid_hook.DruidHook（druid_ingest_conn_id ='druid_ingest_default'，timeout = 1，max_ingestion_time = None） 
```

基类： `airflow.hooks.base_hook.BaseHook`

与德鲁伊霸主的联系以供摄取

参数：

*   `druid_ingest_conn_id(string)` - 接受索引作业的德鲁伊霸王机器的连接 ID
*   `timeout(int)` - 轮询 Druid 作业以获取摄取作业状态之间的间隔
*   `max_ingestion_time(int)` - 假定作业失败前的最长摄取时间

```py
class airflow.hooks.hdfs_hook.HDFSHook（hdfs_conn_id ='hdfs_default'，proxy_user = None，autoconfig = False） 
```

基类： `airflow.hooks.base_hook.BaseHook`

与 HDFS 互动。这个类是 snakebite 库的包装器。

参数：

*   `hdfs_conn_id` - 用于获取连接信息的连接 ID
*   `proxy_user(string)` - HDFS 操作的有效用户
*   `autoconfig(bool)` - 使用 snakebite 自动配置的客户端

```py
get_conn（） 
```

返回一个 snakebite HDFSClient 对象。

```py
class airflow.hooks.jdbc_hook.JdbcHook（* args，** kwargs） 
```

基类： [`airflow.hooks.dbapi_hook.DbApiHook`](31 "airflow.hooks.dbapi_hook.DbApiHook")

jdbc db 访问的常规挂钩。

JDBC URL，用户名和密码将取自预定义的连接。请注意，必须在 DB 中的“host”字段中指定整个 JDBC URL。如果给定的连接 ID 不存在，则引发气流错误。

```py
get_conn（） 
```

返回一个连接对象

```py
set_autocommit（conn，autocommit） 
```

启用或禁用给定连接的自动提交。

参数：`conn` - 连接返回值：

```py
class airflow.hooks.mssql_hook.MsSqlHook（* args，** kwargs） 
```

基类： [`airflow.hooks.dbapi_hook.DbApiHook`](31 "airflow.hooks.dbapi_hook.DbApiHook")

与 Microsoft SQL Server 交互。

```py
get_conn（） 
```

返回一个 mssql 连接对象

```py
set_autocommit（conn，autocommit） 
```

设置连接上的自动提交标志

```py
class airflow.hooks.mysql_hook.MySqlHook（* args，** kwargs） 
```

基类： [`airflow.hooks.dbapi_hook.DbApiHook`](31 "airflow.hooks.dbapi_hook.DbApiHook")

与 MySQL 交互。

您可以在连接的额外字段中指定 charset 。您也可以选择光标。有关更多详细信息，请参阅 MySQLdb.cursors。`{"charset": "utf8"}``{"cursor": "SSCursor"}`

```py
bulk_dump（table，tmp_file） 
```

将数据库表转储到制表符分隔的文件中

```py
bulk_load（table，tmp_file） 
```

将制表符分隔的文件加载到数据库表中

```py
get_autocommit（康涅狄格州） 
```

MySql 连接以不同的方式进行自动提交。：param conn：连接以获取自动提交设置。：type conn：连接对象。：return：connection autocommit setting：rtype bool

```py
get_conn（） 
```

返回一个 mysql 连接对象

```py
set_autocommit（conn，autocommit） 
```

MySql 连接以不同的方式设置自动提交。

```py
class airflow.hooks.oracle_hook.OracleHook（* args，** kwargs） 
```

基类： [`airflow.hooks.dbapi_hook.DbApiHook`](31 "airflow.hooks.dbapi_hook.DbApiHook")

与 Oracle SQL 交互。

```py
bulk_insert_rows（table，rows，target_fields = None，commit_every = 5000） 
```

cx_Oracle 的高性能批量插入，它通过`executemany（）`使用预处理语句。为获得最佳性能，请将`行`作为迭代器传入。

```py
get_conn（） 
```

返回 oracle 连接对象使用自定义 DSN 连接的可选参数（而不是使用来自 tnsnames.ora 的服务器别名）dsn（数据源名称）是 TNS 条目（来自 Oracle 名称服务器或 tnsnames.ora 文件）或者是一个像 makedsn（）返回的字符串。

参数：

*   `dsn` - Oracle 服务器的主机地址
*   `service_name` - 要连接的数据库的 db_unique_name（TNS 的 CONNECT_DATA 部分）


您可以在连接的额外字段中设置这些参数，如 `{ "dsn":"some.host.address" , "service_name":"some.service.name" }`

```py
insert_rows（table，rows，target_fields = None，commit_every = 1000） 
```

将一组元组插入表中的通用方法，整个插入集被视为一个事务从标准 DbApiHook 实现更改： - cx_Oracle 中的 Oracle SQL 查询不能以分号（';'）终止 - 替换 NaN 使用 numpy.nan_to_num 的 NULL 值（不使用 is_nan（）

> 因为字符串的输入类型错误）

*   在插入期间将日期时间单元格强制转换为 Oracle DATETIME 格式

```py
class airflow.hooks.pig_hook.PigCliHook（pig_cli_conn_id ='pig_cli_default'） 
```

基类： `airflow.hooks.base_hook.BaseHook`

猪 CLI 的简单包装。

请注意，您还`pig_properties`可以使用要在连接中使用的默认 pig CLI 属性来设置`{"pig_properties": "-Dpig.tmpfilecompression=true"}`

```py
run_cli（pig，verbose = True） 
```

使用猪 cli 运行猪脚本

```py
 >>>  ph = PigCliHook ()
>>>  result = ph . run_cli ( "ls /;" )
>>>  ( "hdfs://" in result )
True

```

```py
class airflow.hooks.postgres_hook.PostgresHook（* args，** kwargs） 
```

基类： [`airflow.hooks.dbapi_hook.DbApiHook`](31 "airflow.hooks.dbapi_hook.DbApiHook")

与 Postgres 互动。您可以在连接的额外字段中指定 ssl 参数。`{"sslmode": "require", "sslcert": "/path/to/cert.pem", etc}`

注意：对于 Redshift，请在额外的连接参数中使用 keepalives_idle 并将其设置为小于 300 秒。

```py
bulk_dump（table，tmp_file） 
```

将数据库表转储到制表符分隔的文件中

```py
bulk_load（table，tmp_file） 
```

将制表符分隔的文件加载到数据库表中

```py
copy_expert（sql，filename，open = <内置函数打开>） 
```

使用 psycopg2 copy_expert 方法执行 SQL。必须在不访问超级用户的情况下执行 COPY 命令。

注意：如果使用“COPY FROM”语句调用此方法并且指定的输入文件不存在，则会创建一个空文件并且不会加载任何数据，但操作会成功。因此，如果用户想要知道输入文件何时不存在，他们必须自己检查它的存在。

```py
get_conn（） 
```

返回一个连接对象

```py
class airflow.hooks.presto_hook.PrestoHook（* args，** kwargs） 
```

基类： [`airflow.hooks.dbapi_hook.DbApiHook`](31 "airflow.hooks.dbapi_hook.DbApiHook")

通过 PyHive 与 Presto 互动！

```py
 >>>  ph = PrestoHook ()
>>>  sql = "SELECT count(1) AS num FROM airflow.static_babynames"
>>>  ph . get_records ( sql )
[[340698]]

```

```py
get_conn（） 
```

返回一个连接对象

```py
get_first（hql，parameters = None） 
```

无论查询返回多少行，都只返回第一行。

```py
get_pandas_df（hql，parameters = None） 
```

从 sql 查询中获取 pandas 数据帧。

```py
get_records（hql，parameters = None） 
```

从 Presto 获取一组记录

```py
insert_rows（table，rows，target_fields = None） 
```

将一组元组插入表中的通用方法。

参数：

*   `table(str)` - 目标表的名称
*   `rows(_ 可迭代的元组 _)` - 要插入表中的行
*   `target_fields(_ 可迭代的字符串 _)` - 要填充表的列的名称

```py
run（hql，parameters = None） 
```

执行针对 Presto 的声明。可用于创建视图。

```py
class airflow.hooks.S3_hook.S3Hook（aws_conn_id ='aws_default'） 
```

基类： [`airflow.contrib.hooks.aws_hook.AwsHook`](31 "airflow.contrib.hooks.aws_hook.AwsHook")

使用 boto3 库与 AWS S3 交互。

```py
check_for_bucket（BUCKET_NAME） 
```

检查 bucket_name 是否存在。

参数：`bucket_name(str)` - 存储桶的名称

```py
check_for_key（key，bucket_name = None） 
```

检查存储桶中是否存在密钥

参数：

*   `key(str)` - 指向文件的 S3 键
*   `bucket_name(str)` - 存储文件的存储桶的名称

```py
check_for_prefix（bucket_name，prefix，delimiter） 
```

检查存储桶中是否存在前缀

```py
check_for_wildcard_key（wildcard_key，bucket_name = None，delimiter =''） 
```

检查桶中是否存在与通配符表达式匹配的密钥

```py
get_bucket（BUCKET_NAME） 
```

返回 boto3.S3.Bucket 对象

参数：`bucket_name(str)` - 存储桶的名称

```py
get_key（key，bucket_name = None） 
```

返回 boto3.s3.Object

参数：

*   `key(str)` - 密钥的路径
*   `bucket_name(str)` - 存储桶的名称

```py
get_wildcard_key（wildcard_key，bucket_name = None，delimiter =''） 
```

返回与通配符表达式匹配的 boto3.s3.Object 对象

参数：

*   `wildcard_key(str)` - 密钥的路径
*   `bucket_name(str)` - 存储桶的名称

```py
list_keys（bucket_name，prefix =''，delimiter =''，page_size = None，max_items = None） 
```

列出前缀下的存储桶中的密钥，但不包含分隔符

参数：

*   `bucket_name(str)` - 存储桶的名称
*   `prefix(str)` - 一个密钥前缀
*   `delimiter(str)` - 分隔符标记键层次结构。
*   `page_size(int)` - 分页大小
*   `max_items(int)` - 要返回的最大项目数

```py
list_prefixes（bucket_name，prefix =''，delimiter =''，page_size = None，max_items = None） 
```

列出前缀下的存储桶中的前缀

参数：

*   `bucket_name(str)` - 存储桶的名称
*   `prefix(str)` - 一个密钥前缀
*   `delimiter(str)` - 分隔符标记键层次结构。
*   `page_size(int)` - 分页大小
*   `max_items(int)` - 要返回的最大项目数

```py
load_bytes（bytes_data，key，bucket_name = None，replace = False，encrypt = False） 
```

将字节加载到 S3

这是为了方便在 S3 中删除字符串。它使用 boto 基础结构将文件发送到 s3。

参数：

*   `bytes_data(_bytes_)` - 设置为密钥内容的字节。
*   `key(str)` - 指向文件的 S3 键
*   `bucket_name(str)` - 存储文件的存储桶的名称
*   `replace(bool)` - 一个标志，用于决定是否覆盖密钥（如果已存在）
*   `encrypt(bool)` - 如果为 True，则文件将在服务器端由 S3 加密，并在 S3 中静止时以加密形式存储。

```py
load_file（filename，key，bucket_name = None，replace = False，encrypt = False） 
```

将本地文件加载到 S3

参数：

*   `filename(str)` - 要加载的文件的名称。
*   `key(str)` - 指向文件的 S3 键
*   `bucket_name(str)` - 存储文件的存储桶的名称
*   `replace(bool)` - 一个标志，用于决定是否覆盖密钥（如果已存在）。如果 replace 为 False 且密钥存在，则会引发错误。
*   `encrypt(bool)` - 如果为 True，则文件将在服务器端由 S3 加密，并在 S3 中静止时以加密形式存储。

```py
load_string（string_data，key，bucket_name = None，replace = False，encrypt = False，encoding ='utf-8'） 
```

将字符串加载到 S3

这是为了方便在 S3 中删除字符串。它使用 boto 基础结构将文件发送到 s3。

参数：

*   `string_data(str)` - 要设置为键的内容的字符串。
*   `key(str)` - 指向文件的 S3 键
*   `bucket_name(str)` - 存储文件的存储桶的名称
*   `replace(bool)` - 一个标志，用于决定是否覆盖密钥（如果已存在）
*   `encrypt(bool)` - 如果为 True，则文件将在服务器端由 S3 加密，并在 S3 中静止时以加密形式存储。

```py
read_key（key，bucket_name = None） 
```

从 S3 读取密钥

参数：

*   `key(str)` - 指向文件的 S3 键
*   `bucket_name(str)` - 存储文件的存储桶的名称

```py
select_key（key，bucket_name = None，expression ='SELECT * FROM S3Object'，expression_type ='SQL'，input_serialization = {'CSV'：{}}，output_serialization = {'CSV'：{}}） 
```

使用 S3 Select 读取密钥。

参数：

*   `key(str)` - 指向文件的 S3 键
*   `bucket_name(str)` - 存储文件的存储桶的名称
*   `expression(str)` - S3 选择表达式
*   `expression_type(str)` - S3 选择表达式类型
*   `input_serialization(dict)` - S3 选择输入数据序列化格式
*   `output_serialization(dict)` - S3 选择输出数据序列化格式

返回值： 通过 S3 Select 检索原始数据的子集| 返回类型： | 海峡
也可以看看

有关 S3 Select 参数的更多详细信息：[http](http://boto3.readthedocs.io/en/latest/reference/services/s3.html)：[//boto3.readthedocs.io/en/latest/reference/services/s3.html#S3.Client.selectobjectcontent](http://boto3.readthedocs.io/en/latest/reference/services/s3.html)

```py
class airflow.hooks.samba_hook.SambaHook（samba_conn_id） 
```

基类： `airflow.hooks.base_hook.BaseHook`

允许与 samba 服务器交互。

```py
class airflow.hooks.slack_hook.SlackHook（token = None，slack_conn_id = None） 
```

基类： `airflow.hooks.base_hook.BaseHook`

使用 slackclient 库与 Slack 交互。

```py
class airflow.hooks.sqlite_hook.SqliteHook（* args，** kwargs） 
```

基类： [`airflow.hooks.dbapi_hook.DbApiHook`](31 "airflow.hooks.dbapi_hook.DbApiHook")

与 SQLite 交互。

```py
get_conn（） 
```

返回一个 sqlite 连接对象

```py
class airflow.hooks.webhdfs_hook.WebHDFSHook（webhdfs_conn_id ='webhdfs_default'，proxy_user = None） 
```

基类： `airflow.hooks.base_hook.BaseHook`

与 HDFS 互动。这个类是 hdfscli 库的包装器。

```py
check_for_path（hdfs_path） 
```

通过查询 FileStatus 检查 HDFS 中是否存在路径。

```py
get_conn（） 
```

返回 hdfscli InsecureClient 对象。

```py
load_file（source，destination，overwrite = True，parallelism = 1，** kwargs） 
```

将文件上传到 HDFS

参数：

*   `source(str)` - 文件或文件夹的本地路径。如果是文件夹，则会上传其中的所有文件（请注意，这意味着不会远程创建文件夹空文件夹）。
*   `destination(str)` - PTarget HDFS 路径。如果它已经存在并且是目录，则文件将被上传到内部。
*   `overwrite(bool)` - 覆盖任何现有文件或目录。
*   `parallelism(int)` - 用于并行化的线程数。值`0`（或负数）使用与文件一样多的线程。
*   `** kwargs` - 转发给的关键字参数`upload()`。

```py
class airflow.hooks.zendesk_hook.ZendeskHook（zendesk_conn_id） 
```

基类： `airflow.hooks.base_hook.BaseHook`

与 Zendesk 交谈的钩子

```py
call（path，query = None，get_all_pages = True，side_loading = False） 
```

调用 Zendesk API 并返回结果

参数：

*   `path` - 要调用的 Zendesk API
*   `query` - 查询参数
*   `get_all_pages` - 在返回之前累积所有页面的结果。由于严格的速率限制，这通常会超时。等待超时后尝试之间的建议时间段。
*   `side_loading` - 作为单个请求的一部分检索相关记录。为了启用侧载，请添加一个'include'查询参数，其中包含要加载的以逗号分隔的资源列表。有关侧载的更多信息，请参阅[https://developer.zendesk.com/rest_api/docs/core/side_loading](https://developer.zendesk.com/rest_api/docs/core/side_loading)


### 社区贡献了钩子

```py
class airflow.contrib.hooks.aws_dynamodb_hook.AwsDynamoDBHook（table_keys = None，table_name = None，region_name = None，* args，** kwargs） 
```

基类： [`airflow.contrib.hooks.aws_hook.AwsHook`](31 "airflow.contrib.hooks.aws_hook.AwsHook")

与 AWS DynamoDB 交互。

参数：

*   `table_keys(list)` - 分区键和排序键
*   `table_name(str)` - 目标 DynamoDB 表
*   `region_name(str)` - aws 区域名称（例如：us-east-1）

```py
write_batch_data（项目） 
```

将批处理项目写入 dynamodb 表，并提供整个容量。

```py
class airflow.contrib.hooks.aws_hook.AwsHook（aws_conn_id ='aws_default'） 
```

基类： `airflow.hooks.base_hook.BaseHook`

与 AWS 互动。这个类是 boto3 python 库的一个瘦包装器。

```py
get_credentials（REGION_NAME =无） 
```

获取底层的`botocore.Credentials`对象。

它包含以下属性：access_key，secret_key 和 token。

```py
get_session（REGION_NAME =无） 
```

获取底层的 boto3.session。

```py
class airflow.contrib.hooks.aws_lambda_hook.AwsLambdaHook（function_name，region_name = None，log_type ='None'，qualifier ='$ LATEST'，invocation_type ='RequestResponse'，* args，** kwargs） 
```

基类： [`airflow.contrib.hooks.aws_hook.AwsHook`](31 "airflow.contrib.hooks.aws_hook.AwsHook")

与 AWS Lambda 互动

参数：

*   `function_name(str)` - AWS Lambda 函数名称
*   `region_name(str)` - AWS 区域名称（例如：us-west-2）
*   `log_type(str)` - 尾调用请求
*   `qualifier(str)` - AWS Lambda 函数版本或别名
*   `invocation_type(str)` - AWS Lambda 调用类型（RequestResponse，Event 等）

```py
invoke_lambda（有效载荷） 
```

调用 Lambda 函数

```py
class airflow.contrib.hooks.azure_data_lake_hook.AzureDataLakeHook（azure_data_lake_conn_id ='azure_data_lake_default'） 
```

基类： `airflow.hooks.base_hook.BaseHook`

与 Azure Data Lake 进行交互。

客户端 ID 和客户端密钥应该在用户和密码参数中。租户和帐户名称应为{“租户”：“&lt;TENANT&gt;”，“account_name”：“ACCOUNT_NAME”}的额外字段。

参数：`azure_data_lake_conn_id(str)` - 对 Azure Data Lake 连接的引用。

```py
check_for_file（FILE_PATH） 
```

检查 Azure Data Lake 上是否存在文件。

参数：`file_path(str)` - 文件的路径和名称。返回值： 如果文件存在则为 True，否则为 False。
：rtype 布尔

```py
download_file（local_path，remote_path，nthreads = 64，overwrite = True，buffersize = 4194304，blocksize = 4194304） 
```

从 Azure Blob 存储下载文件。

参数：

*   `local_path(str)` - 本地路径。如果下载单个文件，将写入此特定文件，除非它是现有目录，在这种情况下，将在其中创建文件。如果下载多个文件，这是要写入的根目录。将根据需要创建目录。
*   `remote_path(str)` - 用于查找远程文件的远程路径/ globstring。不支持使用`**的`递归 glob 模式。
*   `nthreads(int)` - 要使用的线程数。如果为 None，则使用核心数。
*   `overwrite(bool)` - 是否强制覆盖现有文件/目录。如果 False 和远程路径是目录，则无论是否覆盖任何文件都将退出。如果为 True，则实际仅覆盖匹配的文件名。
*   `buffersize(int)` - int [2 ** 22]内部缓冲区的字节数。此块不能大于块，并且不能小于块。
*   `blocksize(int)` - int [2 ** 22]块的字节数。在每个块中，我们为每个 API 调用编写一个较小的块。这个块不能大于块。

```py
get_conn（） 
```

返回 AzureDLFileSystem 对象。

```py
upload_file（local_path，remote_path，nthreads = 64，overwrite = True，buffersize = 4194304，blocksize = 4194304） 
```

将文件上载到 Azure Data Lake。

参数：

*   `local_path(str)` - 本地路径。可以是单个文件，目录（在这种情况下，递归上传）或 glob 模式。不支持使用`**的`递归 glob 模式。
*   `remote_path(str)` - 要上传的远程路径; 如果有多个文件，这就是要写入的 dircetory 根目录。
*   `nthreads(int)` - 要使用的线程数。如果为 None，则使用核心数。
*   `overwrite(bool)` - 是否强制覆盖现有文件/目录。如果 False 和远程路径是目录，则无论是否覆盖任何文件都将退出。如果为 True，则实际仅覆盖匹配的文件名。
*   `buffersize(int)` - int [2 ** 22]内部缓冲区的字节数。此块不能大于块，并且不能小于块。
*   `blocksize(int)` - int [2 ** 22]块的字节数。在每个块中，我们为每个 API 调用编写一个较小的块。这个块不能大于块。

```py
class airflow.contrib.hooks.azure_fileshare_hook.AzureFileShareHook（wasb_conn_id ='wasb_default'） 
```

基类： `airflow.hooks.base_hook.BaseHook`

与 Azure FileShare 存储交互。

在连接的“额外”字段中传递的其他选项将传递给`FileService（）`构造函数。

参数：`wasb_conn_id(str)` - 对 wasb 连接的引用。

```py
check_for_directory（share_name，directory_name，** kwargs） 
```

检查 Azure 文件共享上是否存在目录。

参数：

*   `share_name(str)` - 共享的名称。
*   `directory_name(str)` - 目录的名称。
*   `kwargs(object)` - `FileService.exists（）`采用的可选关键字参数。

返回值： 如果文件存在则为 True，否则为 False。
：rtype 布尔

```py
check_for_file（share_name，directory_name，file_name，** kwargs） 
```

检查 Azure 文件共享上是否存在文件。

参数：

*   `share_name(str)` - 共享的名称。
*   `directory_name(str)` - 目录的名称。
*   `file_name(str)` - 文件名。
*   `kwargs(object)` - `FileService.exists（）`采用的可选关键字参数。

返回值： 如果文件存在则为 True，否则为 False。
：rtype 布尔

```py
create_directory（share_name，directory_name，** kwargs） 
```

在 Azure 文件共享上创建新的目标。

参数：

*   `share_name(str)` - 共享的名称。
*   `directory_name(str)` - 目录的名称。
*   `kwargs(object)` - `FileService.create_directory（）`采用的可选关键字参数。

返回值： 文件和目录列表
：rtype 列表

```py
get_conn（） 
```

返回 FileService 对象。

```py
get_file（file_path，share_name，directory_name，file_name，** kwargs） 
```

从 Azure 文件共享下载文件。

参数：

*   `file_path(str)` - 存储文件的位置。
*   `share_name(str)` - 共享的名称。
*   `directory_name(str)` - 目录的名称。
*   `file_name(str)` - 文件名。
*   `kwargs(object)` - `FileService.get_file_to_path（）`采用的可选关键字参数。

```py
get_file_to_stream（stream，share_name，directory_name，file_name，** kwargs） 
```

从 Azure 文件共享下载文件。

参数：

*   `stream(类 _ 文件对象 _)` - 用于存储文件的文件句柄。
*   `share_name(str)` - 共享的名称。
*   `directory_name(str)` - 目录的名称。
*   `file_name(str)` - 文件名。
*   `kwargs(object)` - `FileService.get_file_to_stream（）`采用的可选关键字参数。

```py
list_directories_and_files（share_name，directory_name = None，** kwargs） 
```

返回存储在 Azure 文件共享中的目录和文件列表。

参数：

*   `share_name(str)` - 共享的名称。
*   `directory_name(str)` - 目录的名称。
*   `kwargs(object)` - `FileService.list_directories_and_files（）`采用的可选关键字参数。

返回值： 文件和目录列表
：rtype 列表

```py
load_file（file_path，share_name，directory_name，file_name，** kwargs） 
```

将文件上载到 Azure 文件共享。

参数：

*   `file_path(str)` - 要加载的文件的路径。
*   `share_name(str)` - 共享的名称。
*   `directory_name(str)` - 目录的名称。
*   `file_name(str)` - 文件名。
*   `kwargs(object)` - `FileService.create_file_from_path（）`采用的可选关键字参数。

```py
load_stream（stream，share_name，directory_name，file_name，count，** kwargs） 
```

将流上载到 Azure 文件共享。

参数：

*   `stream(类 _ 文件 _)` - 打开的文件/流作为文件内容上传。
*   `share_name(str)` - 共享的名称。
*   `directory_name(str)` - 目录的名称。
*   `file_name(str)` - 文件名。
*   `count(int)` - 流的大小（以字节为单位）
*   `kwargs(object)` - `FileService.create_file_from_stream（）`采用的可选关键字参数。

```py
load_string（string_data，share_name，directory_name，file_name，** kwargs） 
```

将字符串上载到 Azure 文件共享。

参数：

*   `string_data(str)` - 要加载的字符串。
*   `share_name(str)` - 共享的名称。
*   `directory_name(str)` - 目录的名称。
*   `file_name(str)` - 文件名。
*   `kwargs(object)` - `FileService.create_file_from_text（）`采用的可选关键字参数。

```py
class airflow.contrib.hooks.bigquery_hook.BigQueryHook（bigquery_conn_id ='bigquery_default'，delegate_to = None，use_legacy_sql = True） 
```

基类：[`airflow.contrib.hooks.gcp_api_base_hook.GoogleCloudBaseHook`](31 "airflow.contrib.hooks.gcp_api_base_hook.GoogleCloudBaseHook")，[`airflow.hooks.dbapi_hook.DbApiHook`](31 "airflow.hooks.dbapi_hook.DbApiHook")，`airflow.utils.log.logging_mixin.LoggingMixin`

与 BigQuery 交互。此挂钩使用 Google Cloud Platform 连接。

```py
get_conn（） 
```

返回 BigQuery PEP 249 连接对象。

```py
get_pandas_df（sql，parameters = None，dialect = None） 
```

返回 BigQuery 查询生成的结果的 Pandas DataFrame。必须重写 DbApiHook 方法，因为 Pandas 不支持 PEP 249 连接，但 SQLite 除外。看到：

[https://github.com/pydata/pandas/blob/master/pandas/io/sql.py#L447 ](https://github.com/pydata/pandas/blob/master/pandas/io/sql.py)[https://github.com/pydata/pandas/issues/6900](https://github.com/pydata/pandas/issues/6900)

参数：

*   `sql(string)` - 要执行的 BigQuery SQL。
*   `参数(_ 映射 _ 或 _ 可迭代 _)` - 用于呈现 SQL 查询的参数（未使用，请保留覆盖超类方法）
*   `dialect(_{'legacy' __，_ _'standard'}中的 _string)` - BigQuery SQL 的方言 - 遗留 SQL 或标准 SQL 默认使用`self.use_legacy_sql（`如果未指定）

```py
get_service（） 
```

返回一个 BigQuery 服务对象。

```py
insert_rows（table，rows，target_fields = None，commit_every = 1000） 
```

目前不支持插入。从理论上讲，您可以使用 BigQuery 的流 API 将行插入表中，但这尚未实现。

```py
table_exists（project_id，dataset_id，table_id） 
```

检查 Google BigQuery 中是否存在表格。

参数：

*   `project_id(string)` - 要在其中查找表的 Google 云项目。提供给钩子的连接必须提供对指定项目的访问。
*   `dataset_id(string)` - 要在其中查找表的数据集的名称。
*   `table_id(string)` - 要检查的表的名称。

```py
class airflow.contrib.hooks.cassandra_hook.CassandraHook（cassandra_conn_id ='cassandra_default'） 
```

基类：`airflow.hooks.base_hook.BaseHook`，`airflow.utils.log.logging_mixin.LoggingMixin`

胡克曾经与卡桑德拉互动

可以在连接的“hosts”字段中将联系点指定为逗号分隔的字符串。

可以在连接的端口字段中指定端口。

如果在 Cassandra 中启用了 SSL，则将额外字段中的 dict 作为 kwargs 传入`ssl.wrap_socket()`。例如：

>

```py
>  { 
> ```
> 
>

```py
>  'ssl_options'：{ 
> ```
> 
> 'ca_certs'：PATH_TO_CA_CERTS
> 
> }
> 
> }

```py
默认负载平衡策略是 RoundRobinPolicy。要指定不同的 LB 策略：
```

*

```py
     DCAwareRoundRobinPolicy 
    ```

```py
     { 
    ```

    &gt; 'load_balancing_policy'：'DCAwareRoundRobinPolicy'，'load_balancing_policy_args'：{
    &gt; 
    &gt; &gt; 'local_dc'：LOCAL_DC_NAME，//可选'used_hosts_per_remote_dc'：SOMEintVALUE，//可选
    &gt; 
    &gt; }

    }

*

```py
     WhiteListRoundRobinPolicy 
    ```

```py
     { 
    ```

    'load_balancing_policy'：'WhiteListRoundRobinPolicy'，'load_balancing_policy_args'：{

    &gt; '主持人'：['HOST1'，'HOST2'，'HOST3']

    }

    }

*

```py
     TokenAwarePolicy 
    ```

```py
     { 
    ```

    'load_balancing_policy'：'TokenAwarePolicy'，'load_balancing_policy_args'：{

    &gt; 'child_load_balancing_policy'：CHILD_POLICY_NAME，//可选'child_load_balancing_policy_args'：{...} //可选

    }

    }

有关群集配置的详细信息，请参阅 cassandra.cluster。

```py
get_conn（） 
```

返回一个 cassandra Session 对象

```py
record_exists（表，键） 
```

检查 Cassandra 中是否存在记录

参数：

*   `table(string)` - 目标 Cassandra 表。使用点表示法来定位特定键空间。
*   `keys(dict)` - 用于检查存在的键及其值。

```py
shutdown_cluster（） 
```

关闭与此群集关联的所有会话和连接。

```py
class airflow.contrib.hooks.cloudant_hook.CloudantHook（cloudant_conn_id ='cloudant_default'） 
```

基类： `airflow.hooks.base_hook.BaseHook`

与 Cloudant 互动。

这个类是 cloudant python 库的一个薄包装器。请参阅[此处](https://github.com/cloudant-labs/cloudant-python)的文档。

```py
D b（） 
```

返回此挂接的 Database 对象。

请参阅 cloudant-python 的文档[https://github.com/cloudant-labs/cloudant-python](https://github.com/cloudant-labs/cloudant-python)。

```py
class airflow.contrib.hooks.databricks_hook.DatabricksHook（databricks_conn_id ='databricks_default'，timeout_seconds = 180，retry_limit = 3） 
```

基类：`airflow.hooks.base_hook.BaseHook`，`airflow.utils.log.logging_mixin.LoggingMixin`

与 Databricks 互动。

```py
submit_run（JSON） 
```

用于调用`api/2.0/jobs/runs/submit`端点的实用程序功能。

参数：`json(dict)` - 在`submit`端点请求体中使用的数据。返回值： run_id 为字符串| 返回类型： | 串

```py
class airflow.contrib.hooks.datadog_hook.DatadogHook（datadog_conn_id ='datadog_default'） 
```

基类：`airflow.hooks.base_hook.BaseHook`，`airflow.utils.log.logging_mixin.LoggingMixin`

使用 datadog API 发送几乎任何可测量的度量标准，因此可以跟踪插入/删除的 db 记录数，从文件读取的记录以及许多其他有用的度量标准。

取决于 datadog API，该 API 必须部署在 Airflow 运行的同一服务器上。

参数：

*   `datadog_conn_id` - 与 datadog 的连接，包含 api 密钥的元数据。
*   `datadog_conn_id` - 字符串

```py
post_event（title，text，tags = None，alert_type = None，aggregation_key = None） 
```

将事件发布到 datadog（处理完成，潜在警报，其他问题）将此视为维持警报持久性的一种方法，而不是提醒自己。

参数：

*   `title(string)` - 事件的标题
*   `text(string)` - 事件正文（更多信息）
*   `tags(list)` - 要应用于事件的字符串标记列表
*   `alert_type(string)` - 事件的警报类型，[“错误”，“警告”，“信息”，“成功”之一]
*   `aggregation_key(string)` - 可用于在流中聚合此事件的键

```py
query_metric（query，from_seconds_ago，to_seconds_ago） 
```

查询特定度量标准的数据路径，可能会应用某些功能并返回结果。

参数：

*   `query(string)` - 要执行的 datadog 查询（请参阅 datadog docs）
*   `from_seconds_ago(int)` - 开始查询的秒数。
*   `to_seconds_ago(int)` - 最多查询前几秒。

```py
send_metric（metric_name，datapoint，tags = None） 
```

将单个数据点度量标准发送到 DataDog

参数：

*   `metric_name(string)` - 度量标准的名称
*   `datapoint(_ 整数 _ 或 _ 浮点数 _)` - 与度量标准相关的单个整数或浮点数
*   `tags(list)` - 与度量标准关联的标记列表

```py
class airflow.contrib.hooks.datastore_hook.DatastoreHook（datastore_conn_id ='google_cloud_datastore_default'，delegate_to = None） 
```

基类： [`airflow.contrib.hooks.gcp_api_base_hook.GoogleCloudBaseHook`](31 "airflow.contrib.hooks.gcp_api_base_hook.GoogleCloudBaseHook")

与 Google Cloud Datastore 互动。此挂钩使用 Google Cloud Platform 连接。

此对象不是线程安全的。如果要同时发出多个请求，则需要为每个线程创建一个钩子。

```py
allocate_ids（partialKeys） 
```

为不完整的密钥分配 ID。请参阅[https://cloud.google.com/datastore/docs/reference/rest/v1/projects/allocateIds](https://cloud.google.com/datastore/docs/reference/rest/v1/projects/allocateIds)

参数：`partialKeys` - 部分键列表返回值： 完整密钥列表。

```py
begin_transaction（） 
```

获取新的事务处理

> 也可以看看
> 
> [https://cloud.google.com/datastore/docs/reference/rest/v1/projects/beginTransaction](https://cloud.google.com/datastore/docs/reference/rest/v1/projects/beginTransaction)

返回值： 交易句柄

```py
提交（体） 
```

提交事务，可选地创建，删除或修改某些实体。

也可以看看

[https://cloud.google.com/datastore/docs/reference/rest/v1/projects/commit](https://cloud.google.com/datastore/docs/reference/rest/v1/projects/commit)

参数：`body` - 提交请求的主体返回值： 提交请求的响应主体

```py
delete_operation（名称） 
```

删除长时间运行的操作

参数：`name` - 操作资源的名称

```py
export_to_storage_bucket（bucket，namespace = None，entity_filter = None，labels = None） 
```

将实体从 Cloud Datastore 导出到 Cloud Storage 进行备份

```py
get_conn（版本= 'V1'） 
```

返回 Google 云端存储服务对象。

```py
GET_OPERATION（名称） 
```

获取长时间运行的最新状态

参数：`name` - 操作资源的名称

```py
import_from_storage_bucket（bucket，file，namespace = None，entity_filter = None，labels = None） 
```

将备份从云存储导入云数据存储

```py
lookup（keys，read_consistency = None，transaction = None） 
```

按键查找一些实体

也可以看看

[https://cloud.google.com/datastore/docs/reference/rest/v1/projects/lookup](https://cloud.google.com/datastore/docs/reference/rest/v1/projects/lookup)

参数：

*   `keys` - 要查找的键
*   `read_consistency` - 要使用的读取一致性。默认，强或最终。不能与事务一起使用。
*   `transaction` - 要使用的事务，如果有的话。

返回值： 查找请求的响应主体。

```py
poll_operation_until_done（name，polling_interval_in_seconds） 
```

轮询备份操作状态直到完成

```py
回滚（事务） 
```

回滚交易

也可以看看

[https://cloud.google.com/datastore/docs/reference/rest/v1/projects/rollback](https://cloud.google.com/datastore/docs/reference/rest/v1/projects/rollback)

参数：`transaction` - 要回滚的事务

```py
run_query（体） 
```

运行实体查询。

也可以看看

[https://cloud.google.com/datastore/docs/reference/rest/v1/projects/runQuery](https://cloud.google.com/datastore/docs/reference/rest/v1/projects/runQuery)

参数：`body` - 查询请求的主体返回值： 批量查询结果。

```py
class airflow.contrib.hooks.discord_webhook_hook.DiscordWebhookHook（http_conn_id = None，webhook_endpoint = None，message =''，username = None，avatar_url = None，tts = False，proxy = None，* args，** kwargs） 
```

基类： [`airflow.hooks.http_hook.HttpHook`](31 "airflow.hooks.http_hook.HttpHook")

此挂钩允许您使用传入的 webhooks 将消息发布到 Discord。使用默认相对 webhook 端点获取 Discord 连接 ID。可以使用 webhook_endpoint 参数（[https://discordapp.com/developers/docs/resources/webhook](https://discordapp.com/developers/docs/resources/webhook)）覆盖默认端点。

每个 Discord webhook 都可以预先配置为使用特定的用户名和 avatar_url。您可以在此挂钩中覆盖这些默认值。

参数：

*   `http_conn_id(str)` - Http 连接 ID，主机为“ [https://discord.com/api/](https://discord.com/api/) ”，默认 webhook 端点在额外字段中，格式为{“webhook_endpoint”：“webhooks / {webhook.id} / { webhook.token}”
*   `webhook_endpoint(str)` - 以“webhooks / {webhook.id} / {webhook.token}”的形式 Discord webhook 端点
*   `message(str)` - 要发送到 Discord 频道的消息（最多 2000 个字符）
*   `username(str)` - 覆盖 webhook 的默认用户名
*   `avatar_url(str)` - 覆盖 webhook 的默认头像
*   `tts(bool)` - 是一个文本到语音的消息
*   `proxy(str)` - 用于进行 Discord webhook 调用的代理

```py
执行（） 
```

执行 Discord webhook 调用

```py
class airflow.contrib.hooks.emr_hook.EmrHook（emr_conn_id = None，* args，** kwargs） 
```

基类： [`airflow.contrib.hooks.aws_hook.AwsHook`](31 "airflow.contrib.hooks.aws_hook.AwsHook")

与 AWS EMR 交互。emr_conn_id 只是使用 create_job_flow 方法所必需的。

```py
create_job_flow（job_flow_overrides） 
```

使用 EMR 连接中的配置创建作业流。json 额外哈希的键可以具有 boto3 run_job_flow 方法的参数。此配置的覆盖可以作为 job_flow_overrides 传递。

```py
class airflow.contrib.hooks.fs_hook.FSHook（conn_id ='fs_default'） 
```

基类： `airflow.hooks.base_hook.BaseHook`

允许与文件服务器交互。

连接应具有名称和额外指定的路径：

示例：Conn Id：fs_test Conn 类型：文件（路径）主机，Shchema，登录，密码，端口：空额外：{“path”：“/ tmp”}

```py
class airflow.contrib.hooks.ftp_hook.FTPHook（ftp_conn_id ='ftp_default'） 
```

基类：`airflow.hooks.base_hook.BaseHook`，`airflow.utils.log.logging_mixin.LoggingMixin`

与 FTP 交互。

可能在整个过程中发生的错误应该在下游处理。

```py
close_conn（） 
```

关闭连接。如果未打开连接，则会发生错误。

```py
create_directory（路径） 
```

在远程系统上创建目录。

参数：`path(str)` - 要创建的远程目录的完整路径

```py
delete_directory（路径） 
```

删除远程系统上的目录。

参数：`path(str)` - 要删除的远程目录的完整路径

```py
DELETE_FILE（路径） 
```

删除 FTP 服务器上的文件。

参数：`path(str)` - 远程文件的完整路径

```py
describe_directory（路径） 
```

返回远程系统上所有文件的{filename：{attributes}}字典（支持 MLSD 命令）。

参数：`path(str)` - 远程目录的完整路径

```py
get_conn（） 
```

返回 FTP 连接对象

```py
list_directory（path，nlst = False） 
```

返回远程系统上的文件列表。

参数：`path(str)` - 要列出的远程目录的完整路径

```py
重命名（from_name，to_name） 
```

重命名文件。

参数：

*   `from_name` - 从名称重命名文件
*   `to_name` - 将文件重命名为 name

```py
retrieve_file（remote_full_path，local_full_path_or_buffer） 
```

将远程文件传输到本地位置。

如果 local_full_path_or_buffer 是字符串路径，则该文件将放在该位置; 如果它是类似文件的缓冲区，则该文件将被写入缓冲区但不会被关闭。

参数：

*   `remote_full_path(str)` - 远程文件的完整路径
*   `local_full_path_or_buffer(_str __ 或类 _ _ 文件缓冲区 _)` - 本地文件的完整路径或类文件缓冲区

```py
store_file（remote_full_path，local_full_path_or_buffer） 
```

将本地文件传输到远程位置。

如果 local_full_path_or_buffer 是字符串路径，则将从该位置读取该文件; 如果它是类似文件的缓冲区，则从缓冲区读取文件但不关闭。

参数：

*   `remote_full_path(str)` - 远程文件的完整路径
*   `local_full_path_or_buffer(_str __ 或类 _ _ 文件缓冲区 _)` - 本地文件的完整路径或类文件缓冲区

```py
class airflow.contrib.hooks.ftp_hook.FTPSHook（ftp_conn_id ='ftp_default'） 
```

基类： [`airflow.contrib.hooks.ftp_hook.FTPHook`](31 "airflow.contrib.hooks.ftp_hook.FTPHook")

```py
get_conn（） 
```

返回 FTPS 连接对象。

```py
class airflow.contrib.hooks.gcp_api_base_hook.GoogleCloudBaseHook（gcp_conn_id ='google_cloud_default'，delegate_to = None） 
```

基类：`airflow.hooks.base_hook.BaseHook`，`airflow.utils.log.logging_mixin.LoggingMixin`

谷歌云相关钩子的基础钩子。Google 云有一个共享的 REST API 客户端，无论您使用哪种服务，都以相同的方式构建。此类有助于构建和授权所需的凭据，然后调用 apiclient.discovery.build（）来实际发现和构建 Google 云服务的客户端。

该类还包含一些其他辅助函数。

从此基本挂钩派生的所有挂钩都使用“Google Cloud Platform”连接类型。支持两种身份验证方式：

默认凭据：只需要“项目 ID”。您需要设置默认凭据，例如`GOOGLE_APPLICATION_DEFAULT`环境变量或 Google Compute Engine 上的元数据服务器。

JSON 密钥文件：指定“项目 ID”，“密钥路径”和“范围”。

不支持旧版 P12 密钥文件。

```py
class airflow.contrib.hooks.gcp_container_hook.GKEClusterHook（project_id，location） 
```

基类： `airflow.hooks.base_hook.BaseHook`

```py
create_cluster（cluster，retry = <object object>，timeout = <object object>） 
```

创建一个群集，由指定数量和类型的 Google Compute Engine 实例组成。

参数：

*   `cluster(_dict _ 或 _google.cloud.container_v1.types.Cluster_)` - 群集 protobuf 或 dict。如果提供了 dict，它必须与 protobuf 消息的格式相同 google.cloud.container_v1.types.Cluster
*   `重试(_google.api_core.retry.Retry_)` - 用于重试请求的重试对象（google.api_core.retry.Retry）。如果指定 None，则不会重试请求。
*   `timeout(float)` - 等待请求完成的时间（以秒为单位）。请注意，如果指定了重试，则超时适用于每次单独尝试。

返回值： 新集群或现有集群的完整 URL

```py
 ：加薪 
```

ParseError：在尝试转换 dict 时出现 JSON 解析问题 AirflowException：cluster 不是 dict 类型也不是 Cluster proto 类型

```py
delete_cluster（name，retry = <object object>，timeout = <object object>） 
```

删除集群，包括 Kubernetes 端点和所有工作节点。在群集创建期间配置的防火墙和路由也将被删除。群集可能正在使用的其他 Google Compute Engine 资源（例如，负载均衡器资源）如果在初始创建时不存在，则不会被删除。

参数：

*   `name(str)` - 要删除的集群的名称
*   `重试(_google.api_core.retry.Retry_)` - 重 _ 试用 _ 于确定何时/是否重试请求的对象。如果指定 None，则不会重试请求。
*   `timeout(float)` - 等待请求完成的时间（以秒为单位）。请注意，如果指定了重试，则超时适用于每次单独尝试。

返回值： 如果成功则删除操作的完整 URL，否则为 None

```py
get_cluster（name，retry = <object object>，timeout = <object object>） 
```

获取指定集群的详细信息：param name：要检索的集群的名称：type name：str：param retry：用于重试请求的重试对象。如果指定了 None，

> 请求不会被重试。

参数：`timeout(float)` - 等待请求完成的时间（以秒为单位）。请注意，如果指定了重试，则超时适用于每次单独尝试。返回值： 一个 google.cloud.container_v1.types.Cluster 实例

```py
GET_OPERATION（OPERATION_NAME） 
```

从 Google Cloud 获取操作：param operation_name：要获取的操作的名称：type operation_name：str：return：来自 Google Cloud 的新的更新操作

```py
wait_for_operation（操作） 
```

给定操作，持续从 Google Cloud 获取状态，直到完成或发生错误：param 操作：等待的操作：键入操作：google.cloud.container_V1.gapic.enums.Operator：return：a new，updated 从 Google Cloud 获取的操作

```py
class airflow.contrib.hooks.gcp_dataflow_hook.DataFlowHook（gcp_conn_id ='google_cloud_default'，delegate_to = None，poll_sleep = 10） 
```

基类： [`airflow.contrib.hooks.gcp_api_base_hook.GoogleCloudBaseHook`](31 "airflow.contrib.hooks.gcp_api_base_hook.GoogleCloudBaseHook")

```py
get_conn（） 
```

返回 Google 云端存储服务对象。

```py
class airflow.contrib.hooks.gcp_dataproc_hook.DataProcHook（gcp_conn_id ='google_cloud_default'，delegate_to = None，api_version ='v1beta2'） 
```

基类： [`airflow.contrib.hooks.gcp_api_base_hook.GoogleCloudBaseHook`](31 "airflow.contrib.hooks.gcp_api_base_hook.GoogleCloudBaseHook")

Hook for Google Cloud Dataproc API。

```py
等待（操作） 
```

等待 Google Cloud Dataproc Operation 完成。

```py
get_conn（） 
```

返回 Google Cloud Dataproc 服务对象。

```py
等待（操作） 
```

等待 Google Cloud Dataproc Operation 完成。

```py
class airflow.contrib.hooks.gcp_mlengine_hook.MLEngineHook（gcp_conn_id ='google_cloud_default'，delegate_to = None） 
```

基类： [`airflow.contrib.hooks.gcp_api_base_hook.GoogleCloudBaseHook`](31 "airflow.contrib.hooks.gcp_api_base_hook.GoogleCloudBaseHook")

```py
create_job（project_id，job，use_existing_job_fn = None） 
```

启动 MLEngine 作业并等待它达到终端状态。

参数：

*   `project_id(string)` - 将在其中启动 MLEngine 作业的 Google Cloud 项目 ID。
*   `工作(_ 字典 _)` -

    应该提供给 MLEngine API 的 MLEngine Job 对象，例如：

```py
     {
      'jobId' : 'my_job_id' ,
      'trainingInput' : {
        'scaleTier' : 'STANDARD_1' ,
        ...
      }
    }

    ```

*   `use_existing_job_fn(function)` - 如果已存在具有相同 job_id 的 MLEngine 作业，则此方法（如果提供）将决定是否应使用此现有作业，继续等待它完成并返回作业对象。它应该接受 MLEngine 作业对象，并返回一个布尔值，指示是否可以重用现有作业。如果未提供“use_existing_job_fn”，我们默认重用现有的 MLEngine 作业。

返回值： 如果作业成功到达终端状态（可能是 FAILED 或 CANCELED 状态），则为 MLEngine 作业对象。| 返回类型： | 字典

```py
create_model（project_id，model） 
```

创建一个模型。阻止直到完成。

```py
create_version（project_id，model_name，version_spec） 
```

在 Google Cloud ML Engine 上创建版本。

如果版本创建成功则返回操作，否则引发错误。

```py
delete_version（project_id，model_name，version_name） 
```

删除给定版本的模型。阻止直到完成。

```py
get_conn（） 
```

返回 Google MLEngine 服务对象。

```py
get_model（project_id，model_name） 
```

获取一个模型。阻止直到完成。

```py
list_versions（project_id，model_name） 
```

列出模型的所有可用版本。阻止直到完成。

```py
set_default_version（project_id，model_name，version_name） 
```

将版本设置为默认值。阻止直到完成。

```py
class airflow.contrib.hooks.gcp_pubsub_hook.PubSubHook（gcp_conn_id ='google_cloud_default'，delegate_to = None） 
```

基类： [`airflow.contrib.hooks.gcp_api_base_hook.GoogleCloudBaseHook`](31 "airflow.contrib.hooks.gcp_api_base_hook.GoogleCloudBaseHook")

用于访问 Google Pub / Sub 的 Hook。

应用操作的 GCP 项目由嵌入在 gcp_conn_id 引用的 Connection 中的项目确定。

```py
确认（项目，订阅，ack_ids） 
```

`max_messages`从 Pub / Sub 订阅中提取消息。

参数：

*   `project(string)` - 用于创建主题的 GCP 项目名称或 ID
*   `subscription(string)` - 要删除的 Pub / Sub 订阅名称; 不包括'projects / {project} / topics /'前缀。
*   `ack_ids(list)` - 来自先前拉取响应的 ReceivedMessage 确认列表

```py
create_subscription（topic_project，topic，subscription = None，subscription_project = None，ack_deadline_secs = 10，fail_if_exists = False） 
```

如果 Pub / Sub 订阅尚不存在，则创建它。

参数：

*   `topic_project(string)` - 订阅将绑定到的主题的 GCP 项目 ID。
*   `topic(string)` - 订阅将绑定创建的发布/订阅主题名称; 不包括`projects/{project}/subscriptions/`前缀。
*   `subscription(string)` - 发布/订阅订阅名称。如果为空，将使用 uuid 模块生成随机名称
*   `subscription_project(string)` - 将在其中创建订阅的 GCP 项目 ID。如果未指定，`topic_project`将使用。
*   `ack_deadline_secs(int)` - 订户必须确认从订阅中提取的每条消息的秒数
*   `fail_if_exists(bool)` - 如果设置，则在主题已存在时引发异常

返回值： 订阅名称，如果`subscription`未提供参数，它将是系统生成的值| 返回类型： | 串

```py
create_topic（project，topic，fail_if_exists = False） 
```

如果 Pub / Sub 主题尚不存在，则创建它。

参数：

*   `project(string)` - 用于创建主题的 GCP 项目 ID
*   `topic(string)` - 要创建的发布/订阅主题名称; 不包括`projects/{project}/topics/`前缀。
*   `fail_if_exists(bool)` - 如果设置，则在主题已存在时引发异常

```py
delete_subscription（project，subscription，fail_if_not_exists = False） 
```

删除发布/订阅订阅（如果存在）。

参数：

*   `project(string)` - 订阅所在的 GCP 项目 ID
*   `subscription(string)` - 要删除的 Pub / Sub 订阅名称; 不包括`projects/{project}/subscriptions/`前缀。
*   `fail_if_not_exists(bool)` - 如果设置，则在主题不存在时引发异常

```py
delete_topic（项目，主题，fail_if_not_exists = False） 
```

删除 Pub / Sub 主题（如果存在）。

参数：

*   `project(string)` - 要删除主题的 GCP 项目 ID
*   `topic(string)` - 要删除的发布/订阅主题名称; 不包括`projects/{project}/topics/`前缀。
*   `fail_if_not_exists(bool)` - 如果设置，则在主题不存在时引发异常

```py
get_conn（） 
```

返回 Pub / Sub 服务对象。

| 返回类型： | apiclient.discovery.Resource

```py
发布（项目，主题，消息） 
```

将消息发布到 Pub / Sub 主题。

参数：

*   `project(string)` - 要发布的 GCP 项目 ID
*   `topic(string)` - 要发布的发布/订阅主题; 不包括`projects/{project}/topics/`前缀。
*   `消息(PubSub 消息列表;请参阅[http://cloud.google.com/pubsub/docs/reference/rest/v1/PubsubMessage](http://cloud.google.com/pubsub/docs/reference/rest/v1/PubsubMessage))` - 要发布的消息; 如果设置了消息中的数据字段，则它应该已经是 base64 编码的。

```py
pull（项目，订阅，max_messages，return_immediately = False） 
```

`max_messages`从 Pub / Sub 订阅中提取消息。

参数：

*   `project(string)` - 订阅所在的 GCP 项目 ID
*   `subscription(string)` - 要从中提取的 Pub / Sub 订阅名称; 不包括'projects / {project} / topics /'前缀。
*   `max_messages(int)` - 从 Pub / Sub API 返回的最大消息数。
*   `return_immediately(bool)` - 如果设置，如果没有可用的消息，Pub / Sub API 将立即返回。否则，请求将阻止未公开但有限的时间段

```py
 ：return 每个包含的 Pub / Sub ReceivedMessage 对象列表 
```

的`ackId`属性和`message`属性，其中包括 base64 编码消息内容。请参阅[https://cloud.google.com/pubsub/docs/reference/rest/v1/](https://cloud.google.com/pubsub/docs/reference/rest/v1/) projects.subscriptions / pull＃ReceivedMessage

```py
class airflow.contrib.hooks.gcs_hook.GoogleCloudStorageHook（google_cloud_storage_conn_id ='google_cloud_default'，delegate_to = None） 
```

基类： [`airflow.contrib.hooks.gcp_api_base_hook.GoogleCloudBaseHook`](31 "airflow.contrib.hooks.gcp_api_base_hook.GoogleCloudBaseHook")

与 Google 云端存储互动。此挂钩使用 Google Cloud Platform 连接。

```py
copy（source_bucket，source_object，destination_bucket = None，destination_object = None） 
```

将对象从存储桶复制到另一个存储桶，并在需要时重命名。

destination_bucket 或 destination_object 可以省略，在这种情况下使用源桶/对象，但不能同时使用两者。

参数：

*   `source_bucket(string)` - 要从中复制的对象的存储桶。
*   `source_object(string)` - 要复制的对象。
*   `destination_bucket(string)` - 要复制到的对象的目标。可以省略; 然后使用相同的桶。
*   `destination_object` - 给定对象的（重命名）路径。可以省略; 然后使用相同的名称。

```py
create_bucket（bucket_name，storage_class ='MULTI_REGIONAL'，location ='US'，project_id = None，labels = None） 
```

创建一个新存储桶。Google 云端存储使用平面命名空间，因此您无法创建名称已在使用中的存储桶。

也可以看看

有关详细信息，请参阅存储桶命名指南：[https](https://cloud.google.com/storage/docs/bucketnaming.html)：[//cloud.google.com/storage/docs/bucketnaming.html#requirements](https://cloud.google.com/storage/docs/bucketnaming.html)

参数：

*   `bucket_name(string)` - 存储桶的名称。
*   `storage_class(string)` -

    这定义了存储桶中对象的存储方式，并确定了 SLA 和存储成本。价值包括

    *   `MULTI_REGIONAL`
    *   `REGIONAL`
    *   `STANDARD`
    *   `NEARLINE`
    *   `COLDLINE` 。

    如果在创建存储桶时未指定此值，则默认为 STANDARD。

*   `位置(string)` -

    水桶的位置。存储桶中对象的对象数据驻留在此区域内的物理存储中。默认为美国。

    也可以看看

    [https://developers.google.com/storage/docs/bucket-locations](https://developers.google.com/storage/docs/bucket-locations)

*   `project_id(string)` - GCP 项目的 ID。
*   `labels(dict)` - 用户提供的键/值对标签。

返回值： 如果成功，则返回`id`桶的内容。

```py
删除（桶，对象，生成=无） 
```

如果未对存储桶启用版本控制，或者使用了生成参数，则删除对象。

参数：

*   `bucket(string)` - 对象所在的存储桶的名称
*   `object(string)` - 要删除的对象的名称
*   `generation(string)` - 如果存在，则永久删除该代的对象

返回值： 如果成功则为真

```py
下载（bucket，object，filename = None） 
```

从 Google 云端存储中获取文件。

参数：

*   `bucket(string)` - 要获取的存储桶。
*   `object(string)` - 要获取的对象。
*   `filename(string)` - 如果设置，则应写入文件的本地文件路径。

```py
存在（桶，对象） 
```

检查 Google 云端存储中是否存在文件。

参数：

*   `bucket(string)` - 对象所在的 Google 云存储桶。
*   `object(string)` - 要在 Google 云存储分区中检查的对象的名称。

```py
get_conn（） 
```

返回 Google 云端存储服务对象。

```py
get_crc32c（bucket，object） 
```

获取 Google Cloud Storage 中对象的 CRC32c 校验和。

参数：

*   `bucket(string)` - 对象所在的 Google 云存储桶。
*   `object(string)` - 要在 Google 云存储分区中检查的对象的名称。

```py
get_md5hash（bucket，object） 
```

获取 Google 云端存储中对象的 MD5 哈希值。

参数：

*   `bucket(string)` - 对象所在的 Google 云存储桶。
*   `object(string)` - 要在 Google 云存储分区中检查的对象的名称。

```py
get_size（bucket，object） 
```

获取 Google 云端存储中文件的大小。

参数：

*   `bucket(string)` - 对象所在的 Google 云存储桶。
*   `object(string)` - 要在 Google 云存储分区中检查的对象的名称。

```py
is_updated_after（bucket，object，ts） 
```

检查 Google Cloud Storage 中是否更新了对象。

参数：

*   `bucket(string)` - 对象所在的 Google 云存储桶。
*   `object(string)` - 要在 Google 云存储分区中检查的对象的名称。
*   `ts(datetime)` - 要检查的时间戳。

```py
list（bucket，versions = None，maxResults = None，prefix = None，delimiter = None） 
```

使用名称中的给定字符串前缀列出存储桶中的所有对象

参数：

*   `bucket(string)` - 存储桶名称
*   `versions(_boolean_)` - 如果为 true，则列出对象的所有版本
*   `maxResults(_ 整数 _)` - 在单个响应页面中返回的最大项目数
*   `prefix(string)` - 前缀字符串，用于过滤名称以此前缀开头的对象
*   `delimiter(string)` - 根据分隔符过滤对象（例如'.csv'）

返回值： 与过滤条件匹配的对象名称流

```py
重写（source_bucket，source_object，destination_bucket，destination_object = None） 
```

具有与复制相同的功能，除了可以处理超过 5 TB 的文件，以及在位置和/或存储类之间复制时。

destination_object 可以省略，在这种情况下使用 source_object。

参数：

*   `source_bucket(string)` - 要从中复制的对象的存储桶。
*   `source_object(string)` - 要复制的对象。
*   `destination_bucket(string)` - 要复制到的对象的目标。
*   `destination_object` - 给定对象的（重命名）路径。可以省略; 然后使用相同的名称。

```py
upload（bucket，object，filename，mime_type ='application / octet-stream'） 
```

将本地文件上传到 Google 云端存储。

参数：

*   `bucket(string)` - 要上传的存储桶。
*   `object(string)` - 上载本地文件时要设置的对象名称。
*   `filename(string)` - 要上载的文件的本地文件路径。
*   `mime_type(string)` - 上载文件时要设置的 MIME 类型。

```py
class airflow.contrib.hooks.jenkins_hook.JenkinsHook（conn_id ='jenkins_default'） 
```

基类： `airflow.hooks.base_hook.BaseHook`

挂钩管理与 jenkins 服务器的连接

```py
class airflow.contrib.hooks.jira_hook.JiraHook（jira_conn_id ='jira_default'，proxies = None） 
```

基类：`airflow.hooks.base_hook.BaseHook`，`airflow.utils.log.logging_mixin.LoggingMixin`

Jira 交互钩，JIRA Python SDK 的 Wrapper。

参数：`jira_conn_id(string)` - 对预定义的 Jira 连接的引用

```py
class airflow.contrib.hooks.mongo_hook.MongoHook（conn_id ='mongo_default'，* args，** kwargs） 
```

基类： `airflow.hooks.base_hook.BaseHook`

PyMongo Wrapper 与 Mongo 数据库进行交互 Mongo 连接文档[https://docs.mongodb.com/manual/reference/connection-string/index.html](https://docs.mongodb.com/manual/reference/connection-string/index.html)您可以在连接的额外字段中指定连接字符串选项[https：//docs.mongodb .com / manual / reference / connection-string / index.html＃connection-string-options](https://docs.mongodb.com/manual/reference/connection-string/index.html) ex。

> {replicaSet：test，ssl：True，connectTimeoutMS：30000}

```py
aggregate（mongo_collection，aggregate_query，mongo_db = None，** kwargs） 
```

运行聚合管道并返回结果[https://api.mongodb.com/python/current/api/pymongo/collection.html#pymongo.collection.Collection.aggregate ](https://api.mongodb.com/python/current/api/pymongo/collection.html)[https://api.mongodb.com/python/current /examples/aggregation.html](https://api.mongodb.com/python/current/examples/aggregation.html)

```py
find（mongo_collection，query，find_one = False，mongo_db = None，** kwargs） 
```

运行 mongo 查找查询并返回结果[https://api.mongodb.com/python/current/api/pymongo/collection.html#pymongo.collection.Collection.find](https://api.mongodb.com/python/current/api/pymongo/collection.html)

```py
get_collection（mongo_collection，mongo_db = None） 
```

获取用于查询的 mongo 集合对象。

除非指定，否则将连接模式用作 DB。

```py
get_conn（） 
```

获取 PyMongo 客户端

```py
insert_many（mongo_collection，docs，mongo_db = None，** kwargs） 
```

将许多文档插入到 mongo 集合中。[https://api.mongodb.com/python/current/api/pymongo/collection.html#pymongo.collection.Collection.insert_many](https://api.mongodb.com/python/current/api/pymongo/collection.html)

```py
insert_one（mongo_collection，doc，mongo_db = None，** kwargs） 
```

将单个文档插入到 mongo 集合中[https://api.mongodb.com/python/current/api/pymongo/collection.html#pymongo.collection.Collection.insert_one](https://api.mongodb.com/python/current/api/pymongo/collection.html)

```py
class airflow.contrib.hooks.pinot_hook.PinotDbApiHook（* args，** kwargs） 
```

基类： [`airflow.hooks.dbapi_hook.DbApiHook`](31 "airflow.hooks.dbapi_hook.DbApiHook")

连接到 pinot db（[https://github.com/linkedin/pinot](https://github.com/linkedin/pinot)）以发出 pql

```py
get_conn（） 
```

通过 pinot dbqpi 建立与 pinot 代理的连接。

```py
get_first（SQL） 
```

执行 sql 并返回第一个结果行。

参数：`sql(_str _ 或 list)` - 要执行的 sql 语句（str）或要执行的 sql 语句列表

```py
get_pandas_df（sql，parameters = None） 
```

执行 sql 并返回一个 pandas 数据帧

参数：

*   `sql(_str _ 或 list)` - 要执行的 sql 语句（str）或要执行的 sql 语句列表
*   `参数(_mapping _ 或 _iterable_)` - 用于呈现 SQL 查询的参数。

```py
get_records（SQL） 
```

执行 sql 并返回一组记录。

参数：`sql(str)` - 要执行的 sql 语句（str）或要执行的 sql 语句列表

```py
get_uri（） 
```

获取 pinot 经纪人的连接 uri。

例如：[http：// localhost：9000 / pql](http://localhost:9000/pql)

```py
insert_rows（table，rows，target_fields = None，commit_every = 1000） 
```

将一组元组插入表中的通用方法是，每个 commit_every 行都会创建一个新事务

参数：

*   `table(str)` - 目标表的名称
*   `rows(_ 可迭代的元组 _)` - 要插入表中的行
*   `target_fields(_ 可迭代的字符串 _)` - 要填充表的列的名称
*   `commit_every(int)` - 要在一个事务中插入的最大行数。设置为 0 以在一个事务中插入所有行。
*   `replace(bool)` - 是否替换而不是插入

```py
set_autocommit（conn，autocommit） 
```

设置连接上的自动提交标志

```py
class airflow.contrib.hooks.qubole_hook.QuboleHook（* args，** kwargs） 
```

基类：`airflow.hooks.base_hook.BaseHook`，`airflow.utils.log.logging_mixin.LoggingMixin`

```py
get_jobs_id（TI） 
```

获取与 Qubole 命令相关的作业：param ti：任务 dag 的实例，用于确定 Quboles 命令 id：return：与命令关联的作业信息

```py
get_log（TI） 
```

从 Qubole 获取命令的日志：param ti：dag 的任务实例，用于确定 Quboles 命令 id：return：命令日志作为文本

```py
get_results（ti = None，fp = None，inline = True，delim = None，fetch = True） 
```

从 Qubole 获取命令的结果（或仅 s3 位置）并保存到文件中：param ti：任务 dag 的实例，用于确定 Quboles 命令 id：param fp：可选文件指针，将创建一个并返回如果 None 传递：param inline：True 表示只下载实际结果，False 表示仅获取 s3 位置：param delim：用给定的 delim 替换 CTL-A 字符，默认为'，'：param fetch：当 inline 为 True 时，直接从中获取结果 s3（如果大）：return：包含实际结果的文件位置或结果的 s3 位置

```py
杀（TI） 
```

杀死（取消）一个 Qubole 委托：param ti：任务 dag 的实例，用于确定 Quboles 命令 id：return：来自 Qubole 的响应

```py
class airflow.contrib.hooks.redis_hook.RedisHook（redis_conn_id ='redis_default'） 
```

基类：`airflow.hooks.base_hook.BaseHook`，`airflow.utils.log.logging_mixin.LoggingMixin`

挂钩与 Redis 数据库交互

```py
get_conn（） 
```

返回 Redis 连接。

```py
key_exists（钥匙​​） 
```

检查 Redis 数据库中是否存在密钥

参数：`key(string)` - 检查存在的关键。

```py
class airflow.contrib.hooks.redshift_hook.RedshiftHook（aws_conn_id ='aws_default'） 
```

基类： [`airflow.contrib.hooks.aws_hook.AwsHook`](31 "airflow.contrib.hooks.aws_hook.AwsHook")

使用 boto3 库与 AWS Redshift 交互

```py
cluster_status（cluster_identifier） 
```

返回群集的状态

参数：`cluster_identifier(str)` - 集群的唯一标识符

```py
create_cluster_snapshot（snapshot_identifier，cluster_identifier） 
```

创建群集的快照

参数：

*   `snapshot_identifier(str)` - 群集快照的唯一标识符
*   `cluster_identifier(str)` - 集群的唯一标识符

```py
delete_cluster（cluster_identifier，skip_final_cluster_snapshot = True，final_cluster_snapshot_identifier =''） 
```

删除群集并可选择创建快照

参数：

*   `cluster_identifier(str)` - 集群的唯一标识符
*   `skip_final_cluster_snapshot(bool)` - 确定群集快照创建
*   `final_cluster_snapshot_identifier(str)` - 最终集群快照的名称

```py
describe_cluster_snapshots（cluster_identifier） 
```

获取群集的快照列表

参数：`cluster_identifier(str)` - 集群的唯一标识符

```py
restore_from_cluster_snapshot（cluster_identifier，snapshot_identifier） 
```

从其快照还原群集

参数：

*   `cluster_identifier(str)` - 集群的唯一标识符
*   `snapshot_identifier(str)` - 群集快照的唯一标识符

```py
class airflow.contrib.hooks.segment_hook.SegmentHook（segment_conn_id ='segment_default'，segment_debug_mode = False，* args，** kwargs） 
```

基类：`airflow.hooks.base_hook.BaseHook`，`airflow.utils.log.logging_mixin.LoggingMixin`

```py
on_error（错误，项目） 
```

在将 segment_debug_mode 设置为 True 的情况下使用 Segment 时处理错误回调

```py
class airflow.contrib.hooks.sftp_hook.SFTPHook（ftp_conn_id ='sftp_default'） 
```

基类： `airflow.hooks.base_hook.BaseHook`

与 SFTP 互动。旨在与 FTPHook 互换。

```py
 陷阱： - 与 FTPHook 相比，describe_directory 只返回大小，类型和 
```

> 修改。它不返回 unix.owner，unix.mode，perm，unix.group 和 unique。

*   retrieve_file 和 store_file 只接受本地完整路径而不是缓冲区。
*   如果没有模式传递给 create_directory，它将以 777 权限创建。

可能在整个过程中发生的错误应该在下游处理。

```py
close_conn（） 
```

关闭连接。如果连接未被打开，则会发生错误。

```py
create_directory（path，mode = 777） 
```

在远程系统上创建目录。：param path：要创建的远程目录的完整路径：type path：str：param mode：int 表示目录的八进制模式

```py
delete_directory（路径） 
```

删除远程系统上的目录。：param path：要删除的远程目录的完整路径：type path：str

```py
DELETE_FILE（路径） 
```

删除 FTP 服务器上的文件：param path：远程文件的完整路径：type path：str

```py
describe_directory（路径） 
```

返回远程系统上所有文件的{filename：{attributes}}字典（支持 MLSD 命令）。：param path：远程目录的完整路径：type path：str

```py
get_conn（） 
```

返回 SFTP 连接对象

```py
list_directory（路径） 
```

返回远程系统上的文件列表。：param path：要列出的远程目录的完整路径：type path：str

```py
retrieve_file（remote_full_path，local_full_path） 
```

将远程文件传输到本地位置。如果 local_full_path 是字符串路径，则文件将放在该位置：param remote_full_path：远程文件的完整路径：type remote_full_path：str：param local_full_path：本地文件的完整路径：type local_full_path：str

```py
store_file（remote_full_path，local_full_path） 
```

将本地文件传输到远程位置。如果 local_full_path_or_buffer 是字符串路径，则将从该位置读取文件：param remote_full_path：远程文件的完整路径：type remote_full_path：str：param local_full_path：本地文件的完整路径：type local_full_path：str

```py
class airflow.contrib.hooks.slack_webhook_hook.SlackWebhookHook（http_conn_id = None，webhook_token = None，message =''，channel = None，username = None，icon_emoji = None，link_names = False，proxy = None，* args，** kwargs ） 
```

基类： [`airflow.hooks.http_hook.HttpHook`](31 "airflow.hooks.http_hook.HttpHook")

此挂钩允许您使用传入的 webhook 将消息发布到 Slack。直接使用 Slack webhook 令牌和具有 Slack webhook 令牌的连接。如果两者都提供，将使用 Slack webhook 令牌。

每个 Slack webhook 令牌都可以预先配置为使用特定频道，用户名和图标。您可以在此挂钩中覆盖这些默认值。

参数：

*   `http_conn_id(str)` - 在额外字段中具有 Slack webhook 标记的连接
*   `webhook_token(str)` - Slack webhook 令牌
*   `message(str)` - 要在 Slack 上发送的消息
*   `channel(str)` - 邮件应发布到的频道
*   `username(str)` - 要发布的用户名
*   `icon_emoji(str)` - 用作发布给 Slack 的用户图标的表情符号
*   `link_names(bool)` - 是否在邮件中查找和链接频道和用户名
*   `proxy(str)` - 用于进行 Slack webhook 调用的代理

```py
执行（） 
```

远程 Popen（实际上执行松弛的 webhook 调用）

参数：

*   `cmd` - 远程执行的命令
*   `kwargs` - **Popen 的**额外参数（参见 subprocess.Popen）

```py
class airflow.contrib.hooks.snowflake_hook.SnowflakeHook（* args，** kwargs） 
```

基类： [`airflow.hooks.dbapi_hook.DbApiHook`](31 "airflow.hooks.dbapi_hook.DbApiHook")

与 Snowflake 互动。

get_sqlalchemy_engine（）依赖于 snowflake-sqlalchemy

```py
get_conn（） 
```

返回一个 snowflake.connection 对象

```py
get_uri（） 
```

覆盖 get_sqlalchemy_engine（）的 DbApiHook get_uri 方法

```py
set_autocommit（conn，autocommit） 
```

设置连接上的自动提交标志

```py
class airflow.contrib.hooks.spark_jdbc_hook.SparkJDBCHook（spark_app_name ='airflow-spark-jdbc'，spark_conn_id ='spark-default'，spark_conf = None，spark_py_files = None，spark_files = None，spark_jars = None，num_executors = None，executor_cores = None，executor_memory = None，driver_memory = None，verbose = False，principal = None，keytab = None，cmd_type ='spark_to_jdbc'，jdbc_table = None，jdbc_conn_id ='jdbc-default'，jdbc_driver = None，metastore_table = None，jdbc_truncate = False，save_mode = None，save_format = None，batch_size = None，fetch_size = None，num_partitions = None，partition_column = None，lower_bound = None，upper_bound = None，create_table_column_types = None，* args，** kwargs） 
```

基类： [`airflow.contrib.hooks.spark_submit_hook.SparkSubmitHook`](31 "airflow.contrib.hooks.spark_submit_hook.SparkSubmitHook")

此钩子扩展了 SparkSubmitHook，专门用于使用 Apache Spark 执行与基于 JDBC 的数据库之间的数据传输。

参数：

*   `spark_app_name**（str） - 作业名称（默认为**airflow` -spark-jdbc）
*   `spark_conn_id(str)` - 在 Airflow 管理中配置的连接 ID
*   `spark_conf(dict)` - 任何其他 Spark 配置属性
*   `spark_py_files(str)` - 使用的其他 python 文件（.zip，.egg 或.py）
*   `spark_files(str)` - 要上载到运行作业的容器的其他文件
*   `spark_jars(str)` - 要上传并添加到驱动程序和执行程序类路径的其他 jar
*   `num_executors(int)` - 要运行的执行程序的数量。应设置此项以便管理使用 JDBC 数据库建立的连接数
*   `executor_cores(int)` - 每个执行程序的核心数
*   `executor_memory(str)` - 每个执行者的内存（例如 1000M，2G）
*   `driver_memory(str)` - 分配给驱动程序的内存（例如 1000M，2G）
*   `verbose(bool)` - 是否将详细标志传递给 spark-submit 进行调试
*   `keytab(str)` - 包含 keytab 的文件的完整路径
*   `principal(str)` - 用于 keytab 的 kerberos 主体的名称
*   `cmd_type(str)` - 数据应该以哪种方式流动。2 个可能的值：spark_to_jdbc：从 Metastore 到 jdbc jdbc_to_spark 的 spark 写入的数据：来自 jdbc 到 Metastore 的 spark 写入的数据
*   `jdbc_table(str)` - JDBC 表的名称
*   `jdbc_conn_id` - 用于连接 JDBC 数据库的连接 ID
*   `jdbc_driver(str)` - 用于 JDBC 连接的 JDBC 驱动程序的名称。这个驱动程序（通常是一个 jar）应该在'jars'参数中传递
*   `metastore_table(str)` - Metastore 表的名称，
*   `jdbc_truncate(bool)` - （仅限 spark_to_jdbc）Spark 是否应截断或删除并重新创建 JDBC 表。这仅在'save_mode'设置为 Overwrite 时生效。此外，如果架构不同，Spark 无法截断，并将丢弃并重新创建
*   `save_mode(str)` - 要使用的 Spark 保存模式（例如覆盖，追加等）
*   `save_format(str)` - （jdbc_to_spark-only）要使用的 Spark 保存格式（例如镶木地板）
*   `batch_size(int)` - （仅限 spark_to_jdbc）每次往返 JDBC 数据库时要插入的批处理的大小。默认为 1000
*   `fetch_size(int)` - （仅限 jdbc_to_spark）从 JDBC 数据库中每次往返获取的批处理的大小。默认值取决于 JDBC 驱动程序
*   `num_partitions(int)` - Spark 同时可以使用的最大分区数，包括 spark_to_jdbc 和 jdbc_to_spark 操作。这也将限制可以打开的 JDBC 连接数
*   `partition_column(str)` - （jdbc_to_spark-only）用于对 Metastore 表进行分区的数字列。如果已指定，则还必须指定：num_partitions，lower_bound，upper_bound
*   `lower_bound(int)` - （jdbc_to_spark-only）要获取的数字分区列范围的下限。如果已指定，则还必须指定：num_partitions，partition_column，upper_bound
*   `upper_bound(int)` - （jdbc_to_spark-only）要获取的数字分区列范围的上限。如果已指定，则还必须指定：num_partitions，partition_column，lower_bound
*   `create_table_column_types` - （spark_to_jdbc-only）创建表时要使用的数据库列数据类型而不是默认值。应以与 CREATE TABLE 列语法相同的格式指定数据类型信息（例如：“name CHAR（64），comments VARCHAR（1024）”）。指定的类型应该是有效的 spark sql 数据类型。

类型： jdbc_conn_id：str

```py
class airflow.contrib.hooks.spark_sql_hook.SparkSqlHook（sql，conf = None，conn_id ='spark_sql_default'，total_executor_cores = None，executor_cores = None，executor_memory = None，keytab = None，principal = None，master ='yarn'，name ='default-name'，num_executors =无，verbose = True，yarn_queue ='default'） 
```

基类： `airflow.hooks.base_hook.BaseHook`

这个钩子是 spark-sql 二进制文件的包装器。它要求“spark-sql”二进制文件在 PATH 中。：param sql：要执行的 SQL 查询：type sql：str：param conf：arbitrary Spark 配置属性：type conf：str（format：PROP = VALUE）：param conn_id：connection_id string：type conn_id：str：param total_executor_cores :(仅限 Standalone 和 Mesos）所有执行程序的总核心数

> （默认值：工作者的所有可用核心）

参数：

*   `executor_cores(int)` - （仅限 Standalone 和 YARN）每个执行程序的核心数（默认值：2）
*   `executor_memory(str)` - 每个执行程序的内存（例如 1000M，2G）（默认值：1G）
*   `keytab(str)` - 包含 keytab 的文件的完整路径
*   `master(str)` - spark：// host：port，mesos：// host：port，yarn 或 local
*   `name(str)` - 作业名称。
*   `num_executors(int)` - 要启动的执行程序数
*   `verbose(bool)` - 是否将详细标志传递给 spark-sql
*   `yarn_queue(str)` - 要提交的 YARN 队列（默认值：“default”）

```py
run_query（cmd =''，** kwargs） 
```

Remote Popen（实际执行 Spark-sql 查询）

参数：

*   `cmd` - 远程执行的命令
*   `kwargs` - **Popen 的**额外参数（参见 subprocess.Popen）

```py
class airflow.contrib.hooks.spark_submit_hook.SparkSubmitHook（conf = None，conn_id ='spark_default'，files = None，py_files = None，driver_classpath = None，jars = None，java_class = None，packages = None，exclude_packages = None，repositories = None，total_executor_cores = None，executor_cores = None，executor_memory = None，driver_memory = None，keytab = None，principal = None，name ='default-name'，num_executors = None，application_args = None，env_vars = None，verbose = False ） 
```

基类：`airflow.hooks.base_hook.BaseHook`，`airflow.utils.log.logging_mixin.LoggingMixin`

这个钩子是一个围绕 spark-submit 二进制文件的包装器来启动一个 spark-submit 作业。它要求“spark-submit”二进制文件在 PATH 或 spark_home 中提供。：param conf：任意 Spark 配置属性：类型 conf：dict：param conn_id：Airflow 管理中配置的连接 ID。当一个

> 提供的 connection_id 无效，默认为 yarn。

参数：

*   `files(str)` - 将其他文件上载到运行作业的执行程序，以逗号分隔。文件将放在每个执行程序的工作目录中。例如，序列化对象。
*   `py_files(str)` - 作业使用的其他 python 文件可以是.zip，.egg 或.py。
*   `driver_classpath(str)` - 其他特定于驱动程序的类路径设置。
*   `jars(str)` - 提交其他 jar 以上传并将它们放在执行程序类路径中。
*   `java_class(str)` - Java 应用程序的主要类
*   `packages` - 包含在的**包的**逗号分隔的 maven 坐标列表


驱动程序和执行程序类路径：类型包：str：param exclude_packages：解析“packages”中提供的依赖项时要排除的 jar 的 maven 坐标的逗号分隔列表：type exclude_packages：str：param repositories：以逗号分隔的其他远程存储库列表搜索“packages”给出的 maven 坐标：type repositories：str：param total_executor_cores :(仅限 Standalone 和 Mesos）所有执行程序的总核心数（默认值：worker 上的所有可用核心）：type total_executor_cores：int：param executor_cores :(仅限 Standalone，YARN 和 Kubernetes）每个执行程序的核心数（默认值：2）：类型 executor_cores：int：param executor_memory：每个执行程序的内存（例如 1000M，2G）（默认值：1G）：类型 executor_memory：str：param driver_memory ：分配给驱动程序的内存（例如 1000M，2G）（默认值：1G）：类型 driver_memory：str：param keytab：包含 keytab 的文件的完整路径：type keytab：str：param principal：用于 keytab 的 curb eros principal 的名称：type principal：str： param name：作业名称（默认为 airflow-spark）：类型名称：str：param num_executors：要启动的执行程序数：类型 num_executors：int：param application_args：正在提交的应用程序的参数：type application_args：list：param env_vars ：spark-submit 的环境变量。type num_executors：int：param application_args：正在提交的应用程序的参数：type application_args：list：param env_vars：spark-submit 的环境变量。type num_executors：int：param application_args：正在提交的应用程序的参数：type application_args：list：param env_vars：spark-submit 的环境变量。它

> 也支持纱线和 k8s 模式。

参数：`verbose(bool)` - 是否将详细标志传递给 spark-submit 进程进行调试

```py
提交（application =''，** kwargs） 
```

Remote Popen 执行 spark-submit 作业

参数：

*   `application(str)` - 提交的应用程序，jar 或 py 文件
*   `kwargs` - **Popen 的**额外参数（参见 subprocess.Popen）

```py
class airflow.contrib.hooks.sqoop_hook.SqoopHook（conn_id ='sqoop_default'，verbose = False，num_mappers = None，hcatalog_database = None，hcatalog_table = None，properties = None） 
```

基类：`airflow.hooks.base_hook.BaseHook`，`airflow.utils.log.logging_mixin.LoggingMixin`

这个钩子是 sqoop 1 二进制文件的包装器。为了能够使用钩子，需要“sqoop”在 PATH 中。

可以通过 sqoop 连接的“额外”JSON 字段传递的其他参数：

* job_tracker：Job tracker local | jobtracker：port。
* namenode：Namenode。
* lib_jars：逗号分隔的 jar 文件，包含在类路径中。
* files：要复制到地图缩减群集的逗号分隔文件。
* archives：要在计算机上取消归档的逗号分隔的归档

> 机器。

*   password_file：包含密码的文件路径。

参数：

*   `conn_id(str)` - 对 sqoop 连接的引用。
*   `verbose(bool)` - 将 sqoop 设置为 verbose。
*   `num_mappers(int)` - 要并行导入的地图任务数。
*   `properties(dict)` - 通过-D 参数设置的属性

```py
Popen（cmd，** kwargs） 
```

远程 Popen

参数：

*   `cmd` - 远程执行的命令
*   `kwargs` - **Popen 的**额外参数（参见 subprocess.Popen）

返回值： 处理子进程

```py
export_table（table，export_dir，input_null_string，input_null_non_string，staging_table，clear_staging_table，enclosed_by，escaped_by，input_fields_terminated_by，input_lines_terminated_by，input_optionally_enclosed_by，batch，relaxed_isolation，extra_export_options = None） 
```

将 Hive 表导出到远程位置。参数是直接 sqoop 命令行的副本参数：param table：表远程目标：param export_dir：要导出的 Hive 表：param input_null_string：要解释为 null 的字符串

> 字符串列

参数：

*   `input_null_non_string` - 非字符串列的解释为 null 的字符串
*   `staging_table` - 在插入目标表之前将暂存数据的表
*   `clear_staging_table` - 指示可以删除登台表中存在的任何数据
*   `enclosed_by` - 设置包含字符的必填字段
*   `escaped_by` - 设置转义字符
*   `input_fields_terminated_by` - 设置字段分隔符
*   `input_lines_terminated_by` - 设置行尾字符
*   `input_optionally_enclosed_by` - 设置包含字符的字段
*   `batch` - 使用批处理模式执行基础语句
*   `relaxed_isolation` - 为映射器读取未提交的事务隔离
*   `extra_export_options` - 作为 dict 传递的额外导出选项。如果某个键没有值，只需将空字符串传递给它即可。对于 sqoop 选项，不要包含 - 的前缀。

```py
import_query（query，target_dir，append = False，file_type ='text'，split_by = None，direct = None，driver = None，extra_import_options = None） 
```

从 rdbms 导入特定查询到 hdfs：param 查询：要运行的自由格式查询：param target_dir：HDFS 目标 dir：param append：将数据附加到 HDFS 中的现有数据集：param file_type：“avro”，“sequence”，“文字“或”镶木地板“

> 将数据导入 hdfs 为指定格式。默认为文字。

参数：

*   `split_by` - 用于拆分工作单元的表的列
*   `直接` - 使用直接导入快速路径
*   `driver` - 手动指定要使用的 JDBC 驱动程序类
*   `extra_import_options` - 作为 dict 传递的额外导入选项。如果某个键没有值，只需将空字符串传递给它即可。对于 sqoop 选项，不要包含 - 的前缀。

```py
import_table（table，target_dir = None，append = False，file_type ='text'，columns = None，split_by = None，where = None，direct = False，driver = None，extra_import_options = None） 
```

将表从远程位置导入到目标目录。参数是直接 sqoop 命令行参数的副本：param table：要读取的表：param target_dir：HDFS 目标 dir：param append：将数据附加到 HDFS 中的现有数据集：param file_type：“avro”，“sequence”，“text”或“镶木地板”。

> 将数据导入指定的格式。默认为文字。

参数：

*   `columns` - &lt;col，col，col ...&gt;要从表导入的列
*   `split_by` - 用于拆分工作单元的表的列
*   `where` - 导入期间要使用的 WHERE 子句
*   `direct` - 如果存在数据库，则使用直接连接器
*   `driver` - 手动指定要使用的 JDBC 驱动程序类
*   `extra_import_options` - 作为 dict 传递的额外导入选项。如果某个键没有值，只需将空字符串传递给它即可。对于 sqoop 选项，不要包含 - 的前缀。

```py
class airflow.contrib.hooks.ssh_hook.SSHHook（ssh_conn_id = None，remote_host = None，username = None，password = None，key_file = None，port = 22，timeout = 10，keepalive_interval = 30） 
```

基类：`airflow.hooks.base_hook.BaseHook`，`airflow.utils.log.logging_mixin.LoggingMixin`

使用 Paramiko 进行 ssh 远程执行的钩子。ref：[https](https://github.com/paramiko/paramiko)：[//github.com/paramiko/paramiko](https://github.com/paramiko/paramiko)这个钩子还允许你创建 ssh 隧道并作为 SFTP 文件传输的基础

参数：

*   `ssh_conn_id(str)` - 来自气流的连接 ID 从中可以获取所有必需参数的连接，如用户名，密码或 key_file。认为优先权是在 init 期间传递的参数
*   `remote_host(str)` - 要连接的远程主机
*   `username(str)` - 用于连接 remote_host 的用户名
*   `password(str)` - 连接到 remote_host 的用户名的密码
*   `key_file(str)` - 用于连接 remote_host 的密钥文件。
*   `port(int)` - 要连接的远程主机的端口（默认为 paramiko SSH_PORT）
*   `timeout(int)` - 尝试连接到 remote_host 的超时。
*   `keepalive_interval(int)` - 每隔 keepalive_interval 秒向远程主机发送一个 keepalive 数据包

```py
create_tunnel（** kwds） 
```

在两台主机之间创建隧道。与 ssh -L &lt;LOCAL_PORT&gt;类似：host：&lt;REMOTE_PORT&gt;。记得关闭（）返回的“隧道”对象，以便在完成隧道后自行清理。

参数：

*   `local_port(int)` -
*   `remote_port(int)` -
*   `remote_host(str)` -

返回值：

```py
class airflow.contrib.hooks.vertica_hook.VerticaHook（* args，** kwargs） 
```

基类： [`airflow.hooks.dbapi_hook.DbApiHook`](31 "airflow.hooks.dbapi_hook.DbApiHook")

与 Vertica 互动。

```py
get_conn（） 
```

返回 verticaql 连接对象

```py
class airflow.contrib.hooks.wasb_hook.WasbHook（wasb_conn_id ='wasb_default'） 
```

基类： `airflow.hooks.base_hook.BaseHook`

通过 wasb：//协议与 Azure Blob 存储进行交互。

在连接的“额外”字段中传递的其他选项将传递给`BlockBlockService（）`构造函数。例如，通过添加{“sas_token”：“YOUR_TOKEN”}使用 SAS 令牌进行身份验证。

参数：`wasb_conn_id(str)` - 对 wasb 连接的引用。

```py
check_for_blob（container_name，blob_name，** kwargs） 
```

检查 Azure Blob 存储上是否存在 Blob。

参数：

*   `container_name(str)` - 容器的名称。
*   `blob_name(str)` - blob 的名称。
*   `kwargs(object)` - `BlockBlobService.exists（）`采用的可选关键字参数。

返回值： 如果 blob 存在则为 True，否则为 False。
：rtype 布尔

```py
check_for_prefix（container_name，prefix，** kwargs） 
```

检查 Azure Blob 存储上是否存在前缀。

参数：

*   `container_name(str)` - 容器的名称。
*   `prefix(str)` - blob 的前缀。
*   `kwargs(object)` - `BlockBlobService.list_blobs（）`采用的可选关键字参数。

返回值： 如果存在与前缀匹配的 blob，则为 True，否则为 False。
：rtype 布尔

```py
get_conn（） 
```

返回 BlockBlobService 对象。

```py
get_file（file_path，container_name，blob_name，** kwargs） 
```

从 Azure Blob 存储下载文件。

参数：

*   `file_path(str)` - 要下载的文件的路径。
*   `container_name(str)` - 容器的名称。
*   `blob_name(str)` - blob 的名称。
*   `kwargs(object)` - `BlockBlobService.create_blob_from_path（）`采用的可选关键字参数。

```py
load_file（file_path，container_name，blob_name，** kwargs） 
```

将文件上载到 Azure Blob 存储。

参数：

*   `file_path(str)` - 要加载的文件的路径。
*   `container_name(str)` - 容器的名称。
*   `blob_name(str)` - blob 的名称。
*   `kwargs(object)` - `BlockBlobService.create_blob_from_path（）`采用的可选关键字参数。

```py
load_string（string_data，container_name，blob_name，** kwargs） 
```

将字符串上载到 Azure Blob 存储。

参数：

*   `string_data(str)` - 要加载的字符串。
*   `container_name(str)` - 容器的名称。
*   `blob_name(str)` - blob 的名称。
*   `kwargs(object)` - `BlockBlobService.create_blob_from_text（）`采用的可选关键字参数。

```py
read_file（container_name，blob_name，** kwargs） 
```

从 Azure Blob Storage 读取文件并以字符串形式返回。

参数：

*   `container_name(str)` - 容器的名称。
*   `blob_name(str)` - blob 的名称。
*   `kwargs(object)` - `BlockBlobService.create_blob_from_path（）`采用的可选关键字参数。

```py
class airflow.contrib.hooks.winrm_hook.WinRMHook（ssh_conn_id = None，remote_host = None，username = None，password = None，key_file = None，timeout = 10，keepalive_interval = 30） 
```

基类：`airflow.hooks.base_hook.BaseHook`，`airflow.utils.log.logging_mixin.LoggingMixin`

使用 pywinrm 进行 winrm 远程执行的钩子。

参数：

*   `ssh_conn_id(str)` - 来自气流的连接 ID 从中可以获取所有必需参数的连接，如用户名，密码或 key_file。认为优先权是在 init 期间传递的参数
*   `remote_host(str)` - 要连接的远程主机
*   `username(str)` - 用于连接 remote_host 的用户名
*   `password(str)` - 连接到 remote_host 的用户名的密码
*   `key_file(str)` - 用于连接 remote_host 的密钥文件。
*   `timeout(int)` - 尝试连接到 remote_host 的超时。
*   `keepalive_interval(int)` - 每隔 keepalive_interval 秒向远程主机发送一个 keepalive 数据包


## 执行人

执行程序是运行任务实例的机制。

```py
class airflow.executors.local_executor.LocalExecutor（parallelism = 32） 
```

基类： `airflow.executors.base_executor.BaseExecutor`

LocalExecutor 并行本地执行任务。它使用多处理 Python 库和队列来并行执行任务。

```py
结束（） 
```

当调用者完成提交作业并且想要同步等待先前提交的作业全部完成时，调用此方法。

```py
execute_async（key，command，queue = None，executor_config = None） 
```

此方法将异步执行该命令。

```py
开始（） 
```

执行人员可能需要开始工作。例如，LocalExecutor 启动 N 个工作者。

```py
同步（） 
```

将通过心跳方法定期调用同步。执行者应该重写此操作以执行收集状态。

```py
class airflow.executors.celery_executor.CeleryExecutor（parallelism = 32） 
```

基类： `airflow.executors.base_executor.BaseExecutor`

CeleryExecutor 推荐用于 Airflow 的生产使用。它允许将任务实例的执行分配给多个工作节点。

Celery 是一个简单，灵活和可靠的分布式系统，用于处理大量消息，同时为操作提供维护此类系统所需的工具。

```py
端（同步=假） 
```

当调用者完成提交作业并且想要同步等待先前提交的作业全部完成时，调用此方法。

```py
execute_async（key，command，queue ='default'，executor_config = None） 
```

此方法将异步执行该命令。

```py
开始（） 
```

执行人员可能需要开始工作。例如，LocalExecutor 启动 N 个工作者。

```py
同步（） 
```

将通过心跳方法定期调用同步。执行者应该重写此操作以执行收集状态。

```py
class airflow.executors.sequential_executor.SequentialExecutor 
```

基类： `airflow.executors.base_executor.BaseExecutor`

此执行程序一次只运行一个任务实例，可用于调试。它也是唯一可以与 sqlite 一起使用的执行器，因为 sqlite 不支持多个连接。

由于我们希望气流能够开箱即用，因此在您首次安装时，它会默认使用此 SequentialExecutor 和 sqlite。

```py
结束（） 
```

当调用者完成提交作业并且想要同步等待先前提交的作业全部完成时，调用此方法。

```py
execute_async（key，command，queue = None，executor_config = None） 
```

此方法将异步执行该命令。

```py
同步（） 
```

将通过心跳方法定期调用同步。执行者应该重写此操作以执行收集状态。

### 社区贡献的遗嘱执行人

```py
class airflow.contrib.executors.mesos_executor.MesosExecutor（parallelism = 32） 
```

基类：`airflow.executors.base_executor.BaseExecutor`，`airflow.www.utils.LoginMixin`

MesosExecutor 允许将任务实例的执行分配给多个 mesos worker。

Apache Mesos 是一个分布式系统内核，可以将 CPU，内存，存储和其他计算资源从机器（物理或虚拟）中抽象出来，从而可以轻松构建和运行容错和弹性分布式系统。见[http://mesos.apache.org/](http://mesos.apache.org/)

```py
结束（） 
```

当调用者完成提交作业并且想要同步等待先前提交的作业全部完成时，调用此方法。

```py
execute_async（key，command，queue = None，executor_config = None） 
```

此方法将异步执行该命令。

```py
开始（） 
```

执行人员可能需要开始工作。例如，LocalExecutor 启动 N 个工作者。

```py
同步（） 
```

将通过心跳方法定期调用同步。执行者应该重写此操作以执行收集状态。

