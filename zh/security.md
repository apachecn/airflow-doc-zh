# 安全

> 贡献者：[@ImPerat0R\_](https://github.com/tssujt) [@zhongjiajie](https://github.com/zhongjiajie)

默认情况下，所有门都是打开的。限制对 Web 应用程序的访问的一种简单方法是在网络级别执行此操作，比如使用 SSH 隧道。

但也可以通过使用其中一个已提供的认证后端或创建自己的认证后端来打开身份验证。

请务必查看[Experimental Rest API](zh/api.md)以保护 API。

## Web 身份验证

### 密码

最简单的身份验证机制之一是要求用户在登录前输入密码。密码身份验证需要在 requirements 文件中使用`password`子扩展包。它在存储密码之前使用了 bcrypt 扩展包来对密码进行哈希。

```
[webserver]
authenticate = True
auth_backend = airflow.contrib.auth.backends.password_auth
```

启用密码身份验证后，需要先创建初始用户凭据，然后才能登录其他人。未在此身份验证后端的迁移中创建初始用户，以防止默认 Airflow 安装受到攻击。必须通过安装 Airflow 的同一台机器上的 Python REPL 来创建新用户。

```sh
# navigate to the airflow installation directory
$ cd ~/airflow
$ python
Python 2.7.9 (default, Feb 10 2015, 03:28:08)
Type "help", "copyright", "credits" or "license" for more information.
>>> import airflow
>>> from airflow import models, settings
>>> from airflow.contrib.auth.backends.password_auth import PasswordUser
>>> user = PasswordUser(models.User())
>>> user.username = 'new_user_name'
>>> user.email = 'new_user_email@example.com'
>>> user.password = 'set_the_password'
>>> session = settings.Session()
>>> session.add(user)
>>> session.commit()
>>> session.close()
>>> exit()
```

### LDAP

要打开 LDAP 身份验证，请按如下方式配置`airflow.cfg`。请注意，该示例使用与 ldap 服务器的加密连接，因为您可能不希望密码在网络级别上可读。 但是，如果您真的想要，可以在不加密的情况下进行配置。

此外，如果您使用的是 Active Directory，并且没有明确指定用户所在的 OU，则需要将`search_scope`更改为“SUBTREE”。

有效的 search_scope 选项可以在[ldap3 文档中](https://ldap3.readthedocs.org/searches.html%3Fhighlight%3Dsearch_scope)找到

```
[webserver]
authenticate = True
auth_backend = airflow.contrib.auth.backends.ldap_auth

[ldap]
# set a connection without encryption: uri = ldap://<your.ldap.server>:<port>
uri = ldaps://<your.ldap.server>:<port>
user_filter = objectClass=*
# in case of Active Directory you would use: user_name_attr = sAMAccountName
user_name_attr = uid
# group_member_attr should be set accordingly with *_filter
# eg :
#     group_member_attr = groupMembership
#     superuser_filter = groupMembership=CN=airflow-super-users...
group_member_attr = memberOf
superuser_filter = memberOf=CN=airflow-super-users,OU=Groups,OU=RWC,OU=US,OU=NORAM,DC=example,DC=com
data_profiler_filter = memberOf=CN=airflow-data-profilers,OU=Groups,OU=RWC,OU=US,OU=NORAM,DC=example,DC=com
bind_user = cn=Manager,dc=example,dc=com
bind_password = insecure
basedn = dc=example,dc=com
cacert = /etc/ca/ldap_ca.crt
# Set search_scope to one of them: BASE, LEVEL, SUBTREE
# Set search_scope to SUBTREE if using Active Directory, and not specifying an Organizational Unit
search_scope = LEVEL
```

superuser_filter 和 data_profiler_filter 是可选的。如果已定义，则这些配置允许您指定用户必须属于的 LDAP 组，以便拥有超级用户（admin）和数据分析权限。如果未定义，则所有用户都将成为超级用户和拥有数据分析的权限。

### 创建自定义

Airflow 使用了`flask_login`扩展包并在`airflow.default_login`模块中公开了一组钩子。您可以更改内容并使其成为`PYTHONPATH`一部分，并将其配置为`airflow.cfg`的认证后端。

```
[webserver]
authenticate = True
auth_backend = mypackage.auth
```

## 多租户

通过在配置中设置`webserver:filter_by_owner`，可以在启用身份验证时按所有者名称筛选`webserver:filter_by_owner`。有了这个，用户将只看到他所有者的 dags，除非他是超级用户。

```
[webserver]
filter_by_owner = True
```

## Kerberos

Airflow 天然支持 Kerberos。这意味着 Airflow 可以为自己更新 kerberos 票证并将其存储在票证缓存中。钩子和 dags 可以使用票证来验证 kerberized 服务。

### 限制

请注意，此时并未调整所有钩子以使用此功能。此外，它没有将 kerberos 集成到 Web 界面中，您现在必须依赖网络级安全性来确保您的服务保持安全。

Celery 集成尚未经过试用和测试。 但是，如果您为每个主机生成一个 key tab，并在每个 Worker 旁边启动一个 ticket renewer，那么它很可能会起作用。

### 启用 kerberos

#### Airflow

要启用 kerberos，您需要生成（服务）key tab。

```py
# in the kadmin.local or kadmin shell, create the airflow principal
kadmin:  addprinc -randkey airflow/fully.qualified.domain.name@YOUR-REALM.COM

# Create the airflow keytab file that will contain the airflow principal
kadmin:  xst -norandkey -k airflow.keytab airflow/fully.qualified.domain.name
```

现在将此文件存储在 airflow 用户可以读取的位置（chmod 600）。然后将以下内容添加到`airflow.cfg`

```
[core]
security = kerberos

[kerberos]
keytab = /etc/airflow/airflow.keytab
reinit_frequency = 3600
principal = airflow

```

启动 ticket renewer

```sh
# run ticket renewer
airflow kerberos
```

#### Hadoop

如果要使用模拟，则需要在 hadoop 配置的`core-site.xml`中启用。

```
<property>
  <name>hadoop.proxyuser.airflow.groups</name>
  <value>*</value>
</property>

<property>
  <name>hadoop.proxyuser.airflow.users</name>
  <value>*</value>
</property>

<property>
  <name>hadoop.proxyuser.airflow.hosts</name>
  <value>*</value>
</property>

```

当然，如果您需要加强安全性，请用更合适的东西替换星号。

### 使用 kerberos 身份验证

已更新 Hive 钩子以利用 kerberos 身份验证。要允许 DAG 使用它，只需更新连接详细信息，例如：

```
{"use_beeline": true, "principal": "hive/_HOST@EXAMPLE.COM"}
```

请根据您的设置进行调整。_HOST 部分将替换为服务器的完全限定域名。

您可以指定是否要将 dag 所有者用作连接的用户或连接的登录部分中指定的用户。对于登录用户，请将以下内容指定为额外：
```
{"use_beeline": true, "principal": "hive/_HOST@EXAMPLE.COM", "proxy_user": "login"}
```

对于 DAG 所有者使用：
```
{"use_beeline": true, "principal": "hive/_HOST@EXAMPLE.COM", "proxy_user": "owner"}
```

在 DAG 中，初始化 HiveOperator 时，请指定：
```
run_as_owner = True
```

为了使用 kerberos 认证，请务必在安装 Airflow 时添加 kerberos 子扩展包：
```
pip install airflow[kerberos]
```

## OAuth 认证

### GitHub Enterprise（GHE）认证

GitHub Enterprise 认证后端可用于对使用 OAuth2 安装 GitHub Enterprise 的用户进行身份认证。您可以选择指定团队白名单（由 slug cased 团队名称组成）以限制仅允许这些团队的成员登录。

```
[webserver]
authenticate = True
auth_backend = airflow.contrib.auth.backends.github_enterprise_auth

[github_enterprise]
host = github.example.com
client_id = oauth_key_from_github_enterprise
client_secret = oauth_secret_from_github_enterprise
oauth_callback_route = /example/ghe_oauth/callback
allowed_teams = 1, 345, 23
```

注意

如果您未指定团队白名单，那么在 GHE 安装中拥有有效帐户的任何人都可以登录 Airflow。

#### 设置 GHE 身份验证

必须先在 GHE 中设置应用程序，然后才能使用 GHE 身份验证后端。 要设置应用程序：

1. 导航到您的 GHE 配置文件
2. 从左侧导航栏中选择“应用程序”
3. 选择“开发者应用程序”选项卡
4. 点击“注册新申请”
5. 填写所需信息（“授权回调 URL”必须完全合格，例如[http://airflow.example.com/example/ghe_oauth/callback](http://airflow.example.com/example/ghe_oauth/callback) ）
6. 点击“注册申请”
7. 根据上面的示例，将“客户端 ID”，“客户端密钥”和回调路由复制到 airflow.cfg

#### 在 github.com 上使用 GHE 身份验证

可以在 github.com 上使用 GHE 身份验证：

1. [创建一个 Oauth 应用程序](https://developer.github.com/apps/building-oauth-apps/creating-an-oauth-app/)
2. 根据上面的示例，将“客户端 ID”，“客户端密钥”复制到 airflow.cfg
3. 在 airflow.cfg 设置`host = github.com`和`oauth_callback_route = /oauth/callback`

### Google 身份验证

Google 身份验证后端可用于使用 OAuth2 对 Google 用户进行身份验证。您必须指定电子邮件域以限制登录（以逗号分隔），仅限于这些域的成员。

```
[webserver]
authenticate = True
auth_backend = airflow.contrib.auth.backends.google_auth

[google]
client_id = google_client_id
client_secret = google_client_secret
oauth_callback_route = /oauth2callback
domain = "example1.com,example2.com"
```

#### 设置 Google 身份验证

必须先在 Google API 控制台中设置应用程序，然后才能使用 Google 身份验证后端。 要设置应用程序：

1. 导航到[https://console.developers.google.com/apis/](https://console.developers.google.com/apis/)
2. 从左侧导航栏中选择“凭据”
3. 点击“创建凭据”，然后选择“OAuth 客户端 ID”
4. 选择“Web 应用程序”
5. 填写所需信息（'授权重定向 URI'必须完全合格，例如[http://airflow.example.com/oauth2callback](http://airflow.example.com/oauth2callback) ）
6. 点击“创建”
7. 根据上面的示例，将“客户端 ID”，“客户端密钥”和重定向 URI 复制到 airflow.cfg

## SSL

可以通过提供证书和密钥来启用 SSL。启用后，请务必在浏览器中使用“[https//](https:)”。

```
[webserver]
web_server_ssl_cert = <path to cert>
web_server_ssl_key = <path to key>
```

启用 S​​SL 不会自动更改 Web 服务器端口。如果要使用标准端口 443，则还需要配置它。请注意，侦听端口 443 需要超级用户权限（或 Linux 上的 cap_net_bind_service）。

```py
# Optionally, set the server to listen on the standard SSL port.
web_server_port = 443
base_url = http://<hostname or IP>:443
```

使用 SSL 启用 CeleryExecutor。 确保正确生成客户端和服务器证书和密钥。

```
[celery]
CELERY_SSL_ACTIVE = True
CELERY_SSL_KEY = <path to key>
CELERY_SSL_CERT = <path to cert>
CELERY_SSL_CACERT = <path to cacert>
```

## 模拟

Airflow 能够在运行任务实例时模拟 unix 用户，该任务实例基于任务的`run_as_user`参数，该参数采用用户的名称。

**注意：**
要模拟工作，必须使用`sudo`运行 Airflow，因为使用`sudo -u`运行子任务并更改文件的权限。此外，unix 用户需要存在于 worker 中。假设 airflow 是`airflow`用户运行，一个简单的 sudoers 文件可能看起来像这样。请注意，这意味着必须以与 root 用户相同的方式信任和处理 airflow 用户。

```
airflow ALL=(ALL) NOPASSWD: ALL
```

带模拟的子任务仍将记录到同一文件夹，但他们登录的文件将更改权限，只有 unix 用户才能写入。

### 默认模拟

要防止不使用模拟的任务以`sudo`权限运行，如果未设置`run_as_user`，可以在`core:default_impersonation`配置中来设置默认的模拟用户。

```
[core]
default_impersonation = airflow
```

## Flower 认证

Celery Flower 的基础认证是支持的。

可以在 Flower 进程启动命令中指定可选参数，或者在`airflow.cfg`指定配置项。对于这两种情况，请提供用逗号分隔的 user:password 对。

```sh
airflow flower --basic_auth=user1:password1,user2:password2
```

```
[celery]
flower_basic_auth = user1:password1,user2:password2
```
