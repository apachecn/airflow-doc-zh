# 命令行接口

> 贡献者：[@ImPerat0R\_](https://github.com/tssujt)

Airflow 具有非常丰富的命令行接口，允许在 DAG 上执行多种类型的操作，启动服务以及支持开发和测试。

```py
usage: airflow [-h]
               {resetdb,render,variables,connections,create_user,pause,task_failed_deps,version,trigger_dag,initdb,test,unpause,dag_state,run,list_tasks,backfill,list_dags,kerberos,worker,webserver,flower,scheduler,task_state,pool,serve_logs,clear,upgradedb,delete_dag}
               ...
```

## 必填参数

| 子命令 | 可能的选择：resetdb，render，variables，connections，create_user，pause，task_failed_deps，version，trigger_dag，initdb，test，unpause，dag_state，run，list_tasks，backfill，list_dags，kerberos，worker，webserver，flower，scheduler，task_state，pool ，serve_logs，clear，upgrab，delete_dag 子命令帮助 |

## 子命令：

### resetdb

删除并重建元数据数据库

```py
airflow resetdb [-h] [-y]
```

#### 可选参数

| -y, --yes | 不要提示确认重置。请小心使用！默认值：False |

### render

渲染任务实例的模板

```py
airflow render [-h] [-sd SUBDIR] dag_id task_id execution_date
```

#### 必填参数

| dag_id | dag 的 id |
| task_id | 任务的 id |
| execution_date | DAG 的执行日期 |

#### 可选参数

| -sd, --subdir | 从中查找 dag 的文件位置或目录 默认值：“[AIRFLOW_HOME]/dags” |

### 变量

对变量的 CRUD 操作

```py
airflow variables [-h] [-s KEY VAL] [-g KEY] [-j] [-d VAL] [-i FILEPATH]
                  [-e FILEPATH] [-x KEY]
```

#### 可选参数

| -s, --set | 设置变量 |
| -g, --get | 获取变量的值 |
| -j, --json | 反序列化 JSON 变量默认值：False |
| -d, --default | 如果变量不存在，则返回默认值 |
| -i, --import | 从 JSON 文件导入变量 |
| -e, --export | 将变量导出到 JSON 文件 |
| -x, --delete | 删除变量 |

### connections

列表/添加/删除连接

```py
airflow connections [-h] [-l] [-a] [-d] [--conn_id CONN_ID]
                    [--conn_uri CONN_URI] [--conn_extra CONN_EXTRA]
                    [--conn_type CONN_TYPE] [--conn_host CONN_HOST]
                    [--conn_login CONN_LOGIN] [--conn_password CONN_PASSWORD]
                    [--conn_schema CONN_SCHEMA] [--conn_port CONN_PORT]
```

#### 可选参数

| -l，--list | 列出所有连接，默认值：False |
| -a，--add | 添加连接，默认值：False |
| -d，--delete | 删除连接，默认值：False |
| --conn_id | 连接 ID，添加/删除连接时必填 |
| --conn_uri | 连接 URI，添加没有 conn_type 的连接时必填 |
| --conn_extra | 连接的 Extra 字段，添加连接时可选 |
| --conn_type | 连接类型，添加没有 conn_uri 的连接时时必填 |
| --conn_host | 连接主机，添加连接时可选 |
| --conn_login | 连接登录，添加连接时可选 |
| --conn_password | 连接密码，添加连接时可选 |
| --conn_schema | 连接架构，添加连接时可选 |
| --conn_port | 连接端口，添加连接时可选 |

### create_user

创建管理员帐户

```py
airflow create_user [-h] [-r ROLE] [-u USERNAME] [-e EMAIL] [-f FIRSTNAME]
                    [-l LASTNAME] [-p PASSWORD] [--use_random_password]
```

#### 可选参数

| -r，--role | 用户的角色。现有角色包括 Admin，User，Op，Viewer 和 Public |
| -u，--username | 用户的用户名 |
| -e，--电子邮件 | 用户的电子邮件 |
| -f，--firstname | 用户的名字 |
| -l，--lastname | 用户的姓氏 |
| -p，--password | 用户密码 |
| --use_random_password | 不提示输入密码。改为使用随机字符串默认值：False |

### pause

暂停 DAG

```py
airflow pause [-h] [-sd SUBDIR] dag_id
```

#### 必填参数

| dag_id | dag 的 id |

#### 可选参数


| -sd，--subdir | 从中查找 dag 的文件位置或目录，默认值：“[AIRFLOW_HOME]/dags” |

### task_failed_deps

从调度程序的角度返回任务实例的未满足的依赖项。 换句话说，为什么任务实例不会被调度程序调度然后排队，然后由执行程序运行。

```py
airflow task_failed_deps [-h] [-sd SUBDIR] dag_id task_id execution_date
```

#### 必填参数

| dag_id | dag 的 id |
| task_id | 任务的 id |
| execution_date | DAG 的执行日期 |

#### 可选参数

| -sd，--subdir | 从中查找 dag 的文件位置或目录，默认值：“[AIRFLOW_HOME]/dags” |

### version

显示版本

```py
airflow version [-h]
```
### trigger_dag

触发 DAG 运行

```py
airflow trigger_dag [-h] [-sd SUBDIR] [-r RUN_ID] [-c CONF] [-e EXEC_DATE]
                    dag_id
```

#### 必填参数

| dag_id | dag 的 id |

#### 可选参数

| -sd，--subdir | 从中查找 dag 的文件位置或目录，默认值：“[AIRFLOW_HOME]/dags” |
| -r，--run_id | 帮助识别此次运行 |
| -c，--conf | JSON 字符串被腌制到 DagRun 的 conf 属性中 |
| -e，--exec_date | DAG 的执行日期 |

### initdb

初始化元数据数据库

```py
airflow initdb [-h]
```

### 测试

测试任务实例。这将在不检查依赖关系或在数据库中记录其状态的情况下运行任务。

```py
airflow test [-h] [-sd SUBDIR] [-dr] [-tp TASK_PARAMS]
             dag_id task_id execution_date
```

#### 必填参数

| dag_id | dag 的 id |
| task_id | 任务的 id |
| execution_date | DAG 的执行日期 |

#### 可选参数


| -sd，--subdir | 从中查找 dag 的文件位置或目录，默认值：“[AIRFLOW_HOME]/dags” |
| -dr，--dr_run | 进行干运行默认值：False |
| -tp，--task_params | 向任务发送 JSON params dict |

### unpause

恢复暂停的 DAG

```py
airflow unpause [-h] [-sd SUBDIR] dag_id
```

#### 必填参数


| dag_id | dag 的 id |

#### 可选参数


| -sd，--subdir | 从中查找 dag 的文件位置或目录，默认值：“[AIRFLOW_HOME]/dags” |

### dag_state

获取 dag run 的状态

```py
airflow dag_state [-h] [-sd SUBDIR] dag_id execution_date
```

#### 必填参数

| dag_id | dag 的 id |
| execution_date | DAG 的执行日期 |

#### 可选参数

| -sd，--subdir | 从中查找 dag 的文件位置或目录，默认值：“[AIRFLOW_HOME]/dags” |

### run

运行单个任务实例

```py
airflow run [-h] [-sd SUBDIR] [-m] [-f] [--pool POOL] [--cfg_path CFG_PATH]
            [-l] [-A] [-i] [-I] [--ship_dag] [-p PICKLE] [-int]
            dag_id task_id execution_date
```

#### 必填参数

| dag_id | dag 的 id |
| task_id | 任务的 id |
| execution_date | DAG 的执行日期 |

#### 可选参数

| -sd，--subdir | 从中查找 dag 的文件位置或目录，默认值：“[AIRFLOW_HOME]/dags” |
| -m，--mark_success | 将作业标记为成功而不运行它们默认值：False |
| -f，--force | 忽略先前的任务实例状态，无论任务是否已成功/失败，都重新运行，默认值：False |
| --pool | 要使用的资源池 |
| --cfg_path | 要使用的配置文件的路径而不是 airflow.cfg |
| -l，--local | 使用 LocalExecutor 运行任务，默认值：False |
| -A，--ignore_all_dependencies | 忽略所有非关键依赖项，包括 ignore_ti_state 和 ignore_task_deps，默认值：False |
| -i，--ignore_dependencies | 忽略特定于任务的依赖项，例如 upstream，depends_on_past 和重试延迟依赖项，默认值：False |
| -I，--signore_depends_on_past | 忽略 depends_on_past 依赖项（但尊重上游依赖项），默认值：False |
| --ship_dag | 泡菜（序列化）DAG 并将其运送给工人，默认值：False |
| -p，--pickle | 整个 dag 的序列化 pickle 对象（内部使用） |
| -int，--interactive | 不捕获标准输出和错误流（对交互式调试很有用），默认值：False |

### list_tasks

列出 DAG 中的任务

```py
airflow list_tasks [-h] [-t] [-sd SUBDIR] dag_id
```

#### 必填参数

| dag_id | dag 的 id |

#### 可选参数

| -t，--tree | 树视图，默认值：False |
| -sd，--subdir | 从中查找 dag 的文件位置或目录，默认值：“[AIRFLOW_HOME]/dags” |

### backfill

在指定的日期范围内运行 DAG 的子部分 如果使用 reset_dag_run 选项，则回填将首先提示用户 Airflow 是否应清除回填日期范围内的所有先前 dag_run 和 task_instances。如果使用 rerun_failed_tasks，则回填将自动重新运行回填日期范围内的先前失败的任务实例。

```py
airflow backfill [-h] [-t TASK_REGEX] [-s START_DATE] [-e END_DATE] [-m] [-l]
                 [-x] [-i] [-I] [-sd SUBDIR] [--pool POOL]
                 [--delay_on_limit DELAY_ON_LIMIT] [-dr] [-v] [-c CONF]
                 [--reset_dagruns] [--rerun_failed_tasks]
                 dag_id
```

#### 必填参数

| dag_id | dag 的 id |

#### 可选参数

| -t，--task_regex |
|  | 用于过滤特定 task_ids 以回填的正则表达式（可选） |
| -s，--start_date |
|  | 覆盖 start_date YYYY-MM-DD |
| -e，--end_date | 覆盖 end_date YYYY-MM-DD |
| -m，--mark_success |
|  | 将作业标记为成功而不运行它们，默认值：False |
| -l，--local | 使用 LocalExecutor 运行任务，默认值：False |
| -x，--donot_pickle |
|  | 不要试图挑选 DAG 对象发送给工人，只要告诉工人运行他们的代码版本。默认值：False |
| -i，--ignore_dependencies |
|  | 跳过上游任务，仅运行与正则表达式匹配的任务。仅适用于 task_regex，默认值：False |
| -I，--signore_first_depends_on_past |
|  | 仅忽略第一组任务的 depends_on_past 依赖关系（回填 DO 中的后续执行依赖 depends_on_past）。默认值：False |
| -sd，--subdir | 从中查找 dag 的文件位置或目录，默认值：“[AIRFLOW_HOME]/dags” |
| --pool | 要使用的资源池 |
| --delay_on_limit |
|  | 在尝试再次执行 dag 运行之前达到最大活动 dag 运行限制（max_active_runs）时等待的时间（以秒为单位）。默认值：1.0 |
| -dr，--dr_run | 进行干运行，默认值：False |
| -v，--verbose | 使日志输出更详细，默认值：False |
| -c，--conf | JSON 字符串被腌制到 DagRun 的 conf 属性中 |
| --reset_dagruns |
|  | 如果设置，则回填将删除现有的与回填相关的 DAG 运行，并重新开始运行新的 DAG 运行，默认值：False |
| --rerun_failed_tasks |
|  | 如果设置，则回填将自动重新运行回填日期范围的所有失败任务，而不是抛出异常，默认值：False |

### list_dags

列出所有 DAG

```py
airflow list_dags [-h] [-sd SUBDIR] [-r]
```

#### 可选参数

| -sd，--subdir | 从中查找 dag 的文件位置或目录，默认值：“[AIRFLOW_HOME]/dags” |
| -r，--report | 显示 DagBag 加载报告，默认值：False |

### kerberos

启动 kerberos 票证续订

```py
airflow kerberos [-h] [-kt [KEYTAB]] [--pid [PID]] [-D] [--stdout STDOUT]
                 [--stderr STDERR] [-l LOG_FILE]
                 [principal]

```

#### 必填参数

|  principal | kerberos principal 默认值：airflow |

#### 可选参数

| -kt，--keytab | 密钥表默认值：airflow.keytab |
| --pid | PID 文件位置 |
| -D，--daemon | 守护进程而不是在前台运行默认值：False |
| --stdout | 将 stdout 重定向到此文件 |
| --stderr | 将 stderr 重定向到此文件 |
| -l，--log-file | 日志文件的位置 |

### worker

启动 Celery 工作节点

```py
airflow worker [-h] [-p] [-q QUEUES] [-c CONCURRENCY] [-cn CELERY_HOSTNAME]
               [--pid [PID]] [-D] [--stdout STDOUT] [--stderr STDERR]
               [-l LOG_FILE]
```

#### 可选参数

| -p，--do_pickle |
|  | 尝试将 DAG 对象发送给工作人员，而不是让工作人员运行他们的代码版本。默认值：False |
| -q，--queue | 以逗号分隔的队列列表，默认值：default |
| -c, --concurrency |
|  | 工作进程的数量，默认值：16 |
| -cn，--slowry_hostname |
|  | 如果一台计算机上有多个 worker，请设置 celery worker 的主机名。 |
| --pid | PID 文件位置 |
| -D，--daemon | 守护进程而不是在前台运行，默认值：False |
| --stdout | 将 stdout 重定向到此文件 |
| --stderr | 将 stderr 重定向到此文件 |
| -l，--log-file | 日志文件的位置 |

### webserver

启动 Airflow 网络服务器实例

```py
airflow webserver [-h] [-p PORT] [-w WORKERS]
                  [-k {sync,eventlet,gevent,tornado}] [-t WORKER_TIMEOUT]
                  [-hn HOSTNAME] [--pid [PID]] [-D] [--stdout STDOUT]
                  [--stderr STDERR] [-A ACCESS_LOGFILE] [-E ERROR_LOGFILE]
                  [-l LOG_FILE] [--ssl_cert SSL_CERT] [--ssl_key SSL_KEY] [-d]
```

#### 可选参数

| -p，--port | 运行服务器的端口，默认值：8080 |
| -w，--workers | 运行 Web 服务器的工作者数量，默认值：4 |
| -k，--workerclass |
|  | 可能的选择：sync，eventlet，gevent，tornado 用于 Gunicorn 的 worker class，默认值：sync |
| -t，--worker_timeout |
|  | 等待 Web 服务器工作者的超时时间，默认值：120 |
| -hn，--hostname |
|  | 设置运行 Web 服务器的主机名，默认值：0.0.0.0 |
| --pid | PID 文件位置 |
| -D，--daemon | 守护进程而不是在前台运行，默认值：False |
| --stdout | 将 stdout 重定向到此文件 |
| --stderr | 将 stderr 重定向到此文件 |
| -A，--access_logfile |
|  | 用于存储 Web 服务器访问日志的日志文件。 使用'-'打印到 stderr。默认值：- |
| -E，--error_logfile |
|  | 用于存储 Web 服务器错误日志的日志文件。 使用'-'打印到 stderr。默认值：- |
| -l，--log-file | 日志文件的位置 |
| --ssl_cert | Web 服务器的 SSL 证书的路径 |
| --ssl_key | 用于 SSL 证书的密钥的路径 |
| -d，--debug | 在调试模式下使用 Flask 附带的服务器，默认值：False |

### flower

运行 Celery Flower

```py
airflow flower [-h] [-hn HOSTNAME] [-p PORT] [-fc FLOWER_CONF] [-u URL_PREFIX]
               [-a BROKER_API] [--pid [PID]] [-D] [--stdout STDOUT]
               [--stderr STDERR] [-l LOG_FILE]
```

#### 可选参数

| -hn，--hostname |
|  | 设置运行服务器的主机名，默认值：0.0.0.0 |
| -p，--port | 运行服务器的端口，默认值：5555 |
| -fc，--flowers_conf |
|  | celery 的配置文件 |
| -u，--url_prefix |
|  | Flower 的 URL 前缀 |
| -a，--broker_api |
|  | Broker api |
| --pid | PID 文件位置 |
| -D，--daemon | 守护进程而不是在前台运行，默认值：False |
| --stdout | 将 stdout 重定向到此文件 |
| --stderr | 将 stderr 重定向到此文件 |
| -l，--log-file | 日志文件的位置 |

### scheduler

启动调度程序实例

```py
airflow scheduler [-h] [-d DAG_ID] [-sd SUBDIR] [-r RUN_DURATION]
                  [-n NUM_RUNS] [-p] [--pid [PID]] [-D] [--stdout STDOUT]
                  [--stderr STDERR] [-l LOG_FILE]
```

#### 可选参数

| -d，--dag_id | 要运行的 dag 的 id |
| -sd，--subdir | 从中查找 dag 的文件位置或目录，默认值：“[AIRFLOW_HOME]/dags” |
| -r，--run-duration |
|  | 设置退出前执行的秒数 |
| -n，--num_runs | 设置退出前要执行的运行次数，默认值：-1 |
| -p，--do_pickle |
|  | 尝试将 DAG 对象发送给工作人员，而不是让工作人员运行他们的代码版本。默认值：False |
| --pid | PID 文件位置 |
| -D，--daemon | 守护进程而不是在前台运行默认值：False |
| --stdout | 将 stdout 重定向到此文件 |
| --stderr | 将 stderr 重定向到此文件 |
| -l，--log-file | 日志文件的位置 |

### task_state

获取任务实例的状态

```py
airflow task_state [-h] [-sd SUBDIR] dag_id task_id execution_date
```

#### 必填参数

| dag_id | dag 的 id |
| task_id | 任务的 id |
| execution_date | DAG 的执行日期 |

#### 可选参数

| -sd，--subdir | 从中查找 dag 的文件位置或目录，默认值：“[AIRFLOW_HOME]/dags” |

### pool

pool 的 CRUD 操作

```py
airflow pool [-h] [-s NAME SLOT_COUNT POOL_DESCRIPTION] [-g NAME] [-x NAME]
```

#### 可选参数

| -s，--set | 分别设置池槽数和描述 |
| -g，--get | 获取池信息 |
| -x，--delete | 删除池 |

### serve_logs

由 worker 生成的服务日志

```py
airflow serve_logs [-h]
```

### clear

清除一组任务实例，就好像它们从未运行过一样

```py
airflow clear [-h] [-t TASK_REGEX] [-s START_DATE] [-e END_DATE] [-sd SUBDIR]
              [-u] [-d] [-c] [-f] [-r] [-x] [-xp] [-dx]
              dag_id

```

#### 必填参数

| dag_id | dag 的 id |

#### 可选参数

| -t，--task_regex |
|  | 用于过滤特定 task_ids 以回填的正则表达式（可选） |
| -s，--start_date |
|  | 覆盖 start_date YYYY-MM-DD |
| -e，--end_date | 覆盖 end_date YYYY-MM-DD |
| -sd，--subdir | 从中查找 dag 的文件位置或目录，默认值：“[AIRFLOW_HOME]/dags” |
| -u，--upstream | 包括上游任务，默认值：False |
| -d，--downstream |
|  | 包括下游任务，默认值：False |
| -c，--no_confirm |
|  | 请勿要求确认，默认值：False |
| -f，--only_failed |
|  | 只有失败的工作，默认值：False |
| -r，--only_running |
|  | 只运行工作，默认值：False |
| -x，--exclude_subdags |
|  | 排除子标记，默认值：False |
| -dx，--dag_regex |
|  | 将 dag_id 搜索为正则表达式而不是精确字符串，默认值：False |

### upgradedb

将元数据数据库升级到最新版本

```py
airflow upgradedb [-h]
```

### delete_dag

删除与指定 DAG 相关的所有 DB 记录

```py
airflow delete_dag [-h] [-y] dag_id
```

#### 必填参数

| dag_id | dag 的 id |

#### 可选参数

| -y，--是的 | 不要提示确认重置。 小心使用！默认值：False |
