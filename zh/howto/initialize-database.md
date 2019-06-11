# 初始化数据库

> 贡献者：[@ImPerat0R\_](https://github.com/tssujt)、[@ThinkingChen](https://github.com/cdmikechen)

如果您想对 Airflow 进行真正的试使用，您应该考虑设置一个真正的数据库后端并切换到 LocalExecutor。

由于 Airflow 是使用优秀的 SqlAlchemy 库与其元数据进行交互而构建的，因此您可以使用任何 SqlAlchemy 所支持的数据库作为后端数据库。我们推荐使用**MySQL**或**Postgres**。

> 注意
> 我们依赖更严格的 MySQL SQL 设置来获得合理的默认值。确保在[mysqld]下的 my.cnf 中指定了 explicit_defaults_for_timestamp = 1;

> 注意
> 如果您决定使用**Postgres**，我们建议您使用`psycopg2`驱动程序并在 SqlAlchemy 连接字符串中指定它。另请注意，由于 SqlAlchemy 没有公开在 Postgres 连接 URI 中定位特定模式的方法，因此您可能需要使用类似于`ALTER ROLE username SET search_path = airflow, foobar;`的命令为您的角色设置默认模式

一旦您设置好管理 Airflow 的数据库以后，您需要更改配置文件`$AIRFLOW_HOME/airflow.cfg`中的 SqlAlchemy 连接字符串。然后，您还应该将“executor”设置更改为使用“LocalExecutor”，这是一个可以在本地并行化任务实例的执行程序。

```py
# 初始化数据库
airflow initdb
```
