# 实验性 Rest API

> 贡献者：[@ImPerat0R\_](https://github.com/tssujt)

Airflow 公开了一个实验性的 Rest API。它可以通过 Web 服务器获得来请求。 API 端点以/api/experimental/开头，同时请注意，我们也希望端点定义发生变化。

## 端点

这是占位符，直到 swagger 定义处于活动状态

* /api/experimental/dags/<DAG_ID>/tasks/<TASK_ID> 返回任务信息（GET）。
* /api/experimental/dags/<DAG_ID>/dag_runs 为给定的 dag id 创建一个 dag_run（POST）。

## CLI

对于某些功能，cli 可以使用 API​​。 要配置 CLI 选项使得在可用时能够使用 API​​，请按如下方式配置：

```py
[cli]
api_client = airflow.api.client.json_client
endpoint_url = http://<WEBSERVER>:<PORT>
```

## 认证

API 的身份验证与 Web 身份验证分开处理。 默认情况下，不需要对 API 进行任何身份验证 - 即默认情况下全开。 如果您的 Airflow 网络服务器可公开访问，那么不建议这样做，您应该使用拒绝所有后端请求：
```py
[api]
auth_backend = airflow.api.auth.backend.deny_all
```

API 目前支持两种“真实”的身份验证方法。

要启用密码身份验证，请在配置中进行以下设置：
```py
[api]
auth_backend = airflow.contrib.auth.backends.password_auth
```

它的用法类似于用于 Web 界面的密码验证。

要启用 Kerberos 身份验证，请在配置中设置以下内容：
```py
[api]
auth_backend = airflow.api.auth.backend.kerberos_auth

[kerberos]
keytab = <KEYTAB>
```

Kerberos 服务配置为`airflow/fully.qualified.domainname@REALM`。确保密钥表文件中存在此配置。
