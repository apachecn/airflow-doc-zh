# 保护连接

> 贡献者：[@ImPerat0R\_](https://github.com/tssujt)、[@ThinkingChen](https://github.com/cdmikechen)

默认情况下，Airflow 在元数据数据库中是以纯文本格式保存连接的密码。在安装过程中强烈建议使用`crypto`包。`crypto`包要求您的操作系统安装了 libffi-dev。

如果最初未安装`crypto`软件包，您仍可以通过以下步骤为连接启用加密：

1. 安装 crypto 包`pip install apache-airflow[crypto]`
2. 使用下面的代码片段生成 fernet_key。 fernet_key 必须是 base64 编码的 32 字节密钥。

```py
from cryptography.fernet import Fernet
fernet_key= Fernet.generate_key()
print(fernet_key.decode()) # 你的 fernet_key，把它放在安全的地方
```

3.将`airflow.cfg`中的 fernet_key 值替换为步骤 2 中的值。或者，可以将 fernet_key 存储在 OS 环境变量中。在这种情况下，您不需要更改`airflow.cfg`，因为 Airflow 将使用环境变量而不是`airflow.cfg`中的值：

```py
# 注意双下划线
export AIRFLOW__CORE__FERNET_KEY=your_fernet_key
```

1. 重启 Airflow webserver。
2. 对于现有连接（在安装`airflow[crypto]`和创建 Fernet 密钥之前已定义的连接），您需要在连接管理 UI 中打开每个连接，重新键入密码并保存。
