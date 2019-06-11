# 安装

> 贡献者：[@ThinkingChen](https://github.com/cdmikechen)、[@zhongjiajie](https://github.com/zhongjiajie)

## 获取 Airflow

安装最新稳定版 Airflow 的最简单方法是使用`pip` ：

```bash
pip install apache-airflow
```

您还可以安装 Airflow 的一些别的支持功能组件，例如 `gcp_api` 或者 `postgres`：

```bash
pip install apache-airflow[postgres,gcp_api]
```

## 额外的扩展包

通过 PyPI 的 `apache-airflow` 命令下载的基本包只含有启动的基础部分内容。您可以根据您环境的需要下载您的扩展包。例如，如果您不需要连接 Postgres，那么您就不需要使用 yum 命令安装 `postgres-devel`，或者在您使用的系统上面安装 postgre 应用，并在安装中的经历一些痛苦过程。

除此之外，Airflow 可以按照需求导入这些扩展包来使用。

如下是列举出来的子包列表和他的功能:

| 包名 | 安装命令 | 说明 |
| :------| :------ | :------ |
| all | `pip install apache-airflow[all]` | 所有 Airflow 功能 |
| all_dbs | `pip install apache-airflow[all_dbs]` | 所有集成的数据库 |
| async | `pip install apache-airflow[async]` | Gunicorn 的异步 worker classes |
| celery | `pip install apache-airflow[celery]` | CeleryExecutor |
| cloudant | `pip install apache-airflow[cloudant]` | Cloudant hook  |
| crypto | `pip install apache-airflow[crypto]` | 加密元数据 db 中的连接密码 |
| devel | `pip install apache-airflow[devel]` | 最小开发工具要求 |
| devel_hadoop | `pip install apache-airflow[devel_hadoop]` | Airflow + Hadoop stack 的依赖 |
| druid | `pip install apache-airflow[druid]` | Druid.io 相关的 operators 和 hooks |
| gcp_api | `pip install apache-airflow[gcp_api]` | Google 云平台 hooks 和 operators（使用`google-api-python-client` ） |
| github_enterprise | `pip install apache-airflow[github_enterprise]` | Github 企业版身份认证 |
| google_auth | `pip install apache-airflow[google_auth]` | Google 身份认证 |
| hdfs | `pip install apache-airflow[hdfs]` | HDFS hooks 和 operators |
| hive | `pip install apache-airflow[hive]` | 所有 Hive 相关的 operators |
| jdbc | `pip install apache-airflow[jdbc]` | JDBC hooks 和 operators |
| kerberos | `pip install apache-airflow[kerberos]` | Kerberos 集成 Kerberized Hadoop |
| kubernetes | `pip install apache-airflow[kubernetes]` | Kubernetes Executor 以及 operator |
| ldap | `pip install apache-airflow[ldap]` | 用户的 LDAP 身份验证 |
| mssql | `pip install apache-airflow[mssql]` | Microsoft SQL Server operators 和 hook，作为 Airflow 后端支持 |
| mysql | `pip install apache-airflow[mysql]` | MySQL operators 和 hook，支持作为 Airflow 后端。 MySQL 服务器的版本必须是 5.6.4+。 确切的版本上限取决于`mysqlclient`包的版本。 例如， `mysqlclient` 1.3.12 只能与 MySQL 服务器 5.6.4 到 5.7 一起使用。 |
| password | `pip install apache-airflow[password]` | 用户密码验证 |
| postgres | `pip install apache-airflow[postgres]` | Postgres operators 和 hook，作为 Airflow 后端支持 |
| qds | `pip install apache-airflow[qds]` | 启用 QDS（Qubole 数据服务）支持 |
| rabbitmq | `pip install apache-airflow[rabbitmq]` | rabbitmq 作为 Celery 后端支持 |
| redis | `pip install apache-airflow[redis]` | Redis hooks 和 sensors |
| s3 | `pip install apache-airflow[s3]` | `S3KeySensor`，`S3PrefixSensor` |
| samba | `pip install apache-airflow[samba]` | `Hive2SambaOperator` |
| slack | `pip install apache-airflow[slack]` | `SlackAPIPostOperator` |
| ssh | `pip install apache-airflow[ssh]` | SSH hooks 及 Operator |
| vertica | `pip install apache-airflow[vertica]` | 做为 Airflow 后端的 Vertica hook 支持 |

## 初始化 Airflow 数据库

在您运行任务之前，Airflow 需要初始化数据库。 如果您只是在试验和学习 Airflow，您可以坚持使用默认的 SQLite 选项。 如果您不想使用 SQLite，请查看[初始化数据库后端](zh/howto/initialize-database.md)以设置其他数据库。

配置完成后，若您想要运行任务，需要先初始化数据库：

```bash
airflow initdb
```
