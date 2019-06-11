# 管理连接

> 贡献者：[@ImPerat0R\_](https://github.com/tssujt)、[@ThinkingChen](https://github.com/cdmikechen) [@zhongjiajie](https://github.com/zhongjiajie)

Airflow 需要知道如何连接到您的环境。其他系统和服务的主机名，端口，登录名和密码等信息在 UI 的`Admin->Connection`部分中处理。您编写的 pipeline（管道）代码将引用 Connection 对象的“conn_id”。

![https://airflow.apache.org/_images/connections.png](../img/b1caba93dd8fce8b3c81bfb0d58cbf95.jpg)

可以使用 UI 或环境变量创建和管理连接。

有关更多信息，请参阅[Connenctions Concepts](zh/concepts.md)文档。

## 使用 UI 创建连接

打开 UI 的`Admin->Connection`部分。 单击`Create`按钮以创建新连接。

![https://airflow.apache.org/_images/connection_create.png](../img/635aacab53c55192ad3e31c28e65eb43.jpg)

1. 使用所需的连接 ID 填写`Conn Id`字段。建议您使用小写字符和单独的带下划线的单词。
2. 使用`Conn Type`字段选择连接类型。
3. 填写其余字段。有关属于不同连接类型的字段的说明，请参阅连接类型。
4. 单击`Save`按钮以创建连接。

## 使用 UI 编辑连接

打开 UI 的`Admin->Connection`部分。 单击连接列表中要编辑的连接旁边的铅笔图标。

![https://airflow.apache.org/_images/connection_edit.png](../img/08e0f3fedf871b535c850d202dda1422.jpg)

修改连接属性，然后单击`Save`按钮以保存更改。

## 使用环境变量创建连接

可以使用环境变量创建 Airflow pipeline（管道）中的连接。环境变量需要具有`AIRFLOW_CONN_`的前缀的 URI 格式才能正确使用连接。

在引用 Airflow pipeline（管道）中的连接时，`conn_id`应该是没有前缀的变量的名称。例如，如果`conn_id`名为`postgres_master`，则环境变量应命名为`AIRFLOW_CONN_POSTGRES_MASTER`（请注意，环境变量必须全部为大写）。Airflow 假定环境变量返回的值为 URI 格式（例如`postgres://user:password@localhost:5432/master`或`s3://accesskey:secretkey@S3` ）。

## 连接类型

### Google Cloud Platform

Google Cloud Platform 连接类型支持[GCP 集成](zh/integration.md) 。

#### 对 GCP 进行身份验证

有两种方法可以使用 Airflow 连接到 GCP。

1. 使用[应用程序默认凭据](https://google-auth.readthedocs.io/en/latest/reference/google.auth.html) ，例如在 Google Compute Engine 上运行时通过元数据服务器。
2. 在磁盘上使用[服务帐户](https://cloud.google.com/docs/authentication/)密钥文件（JSON 格式）。

#### 默认连接 ID

默认情况下使用以下连接 ID。

```py
bigquery_default
```

由[`BigQueryHook`](zh/integration.md)钩子使用。

```py
google_cloud_datastore_default
```

由[`DatastoreHook`](zh/integration.md)钩子使用。

```py
google_cloud_default
```

由[`GoogleCloudBaseHook`](zh/integration.md)，[`DataFlowHook`](zh/integration.md)，[`DataProcHook`](zh/integration.md)，[`MLEngineHook`](zh/integration.md)和[`GoogleCloudStorageHook`](zh/integration.md)挂钩使用。

#### 配置连接

##### Project Id (必填)

要连接的 Google Cloud 项目 ID。

##### Keyfile Path

磁盘上[服务帐户](https://cloud.google.com/docs/authentication/)密钥文件（JSON 格式）的路径。

如果使用应用程序默认凭据则不需要

##### Keyfile JSON

磁盘上的[服务帐户](https://cloud.google.com/docs/authentication/)密钥文件（JSON 格式）的内容。 如果使用此方法进行身份验证，建议[保护您的连接](zh/howto/secure-connections.md) 。

如果使用应用程序默认凭据则不需要

##### Scopes (逗号分隔)

要通过身份验证的逗号分隔[Google 云端范围](https://developers.google.com/identity/protocols/googlescopes)列表。

> 注意
> 使用应用程序默认凭据时，将忽略范围。 请参阅[AIRFLOW-2522](https://issues.apache.org/jira/browse/AIRFLOW-2522) 。

### MySQL

MySQL 连接类型允许连接 MySQL 数据库。

#### 配置连接

##### 主机（必填）

要连接的主机。

##### 模式（选填）

指定要在数据库中使用的模式名称。

##### 登陆（必填）

指定要连接的用户名

##### 密码（必填）

指定要连接的密码

##### 额外参数（选填）

指定可以在 mysql 连接中使用的额外参数（作为 json 字典）。支持以下参数：

 - charset：指定连接的 charset
 - cursor：指定使用的 cursor，为“sscursor”、“dictcurssor”、“ssdictcursor”其中之一
 - local_infile：控制 MySQL 的 LOCAL 功能（允许客户端加载本地数据）。有关详细信息，请参阅[MySQLdb 文档](https://mysqlclient.readthedocs.io/user_guide.html)。
 - unix_socket：使用 UNIX 套接字而不是默认套接字
 - ssl：控制使用 SSL 连接的 SSL 参数字典（这些参数是特定于服务器的，应包含“ca”，“cert”，“key”，“capath”，“cipher”参数。有关详细信息，请参阅[MySQLdb 文档](https://mysqlclient.readthedocs.io/user_guide.html)。请注意，以便在 URL 表示法中很有用，此参数也可以是 SSL 字典是字符串编码的 JSON 字典的字符串。

示例“extras”字段：

```py
{
   "charset": "utf8",
   "cursorclass": "sscursor",
   "local_infile": true,
   "unix_socket": "/var/socket",
   "ssl": {
     "cert": "/tmp/client-cert.pem",
     "ca": "/tmp/server-ca.pem'",
     "key": "/tmp/client-key.pem"
   }
}
```

或

```py
{
   "charset": "utf8",
   "cursorclass": "sscursor",
   "local_infile": true,
   "unix_socket": "/var/socket",
   "ssl": "{\"cert\": \"/tmp/client-cert.pem\", \"ca\": \"/tmp/server-ca.pem\", \"key\": \"/tmp/client-key.pem\"}"
}
```

将连接指定为 URI（在 AIRFLOW_CONN_ *变量中）时，应按照 DB 连接的标准语法指定它，其中附加项作为 URI 的参数传递（请注意，URI 的所有组件都应进行 URL 编码）。

举个例子：

```py
mysql://mysql_user:XXXXXXXXXXXX@1.1.1.1:3306/mysqldb?ssl=%7B%22cert%22%3A+%22%2Ftmp%2Fclient-cert.pem%22%2C+%22ca%22%3A+%22%2Ftmp%2Fserver-ca.pem%22%2C+%22key%22%3A+%22%2Ftmp%2Fclient-key.pem%22%7D
```

> 注意
> 如果在使用 MySQL 连接时遇到 UnicodeDecodeError，请检查定义的字符集是否与数据库字符集匹配。
