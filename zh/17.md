# 使用 upstart 运行 Airflow

> 贡献者：[@ImPerat0R\_](https://github.com/tssujt)、[@ThinkingChen](https://github.com/cdmikechen)

Airflow 可以与基于 upstart 的系统集成。Upstart 会在系统启动时自动启动在`/etc/init`具有相应`*.conf`文件的所有 Airflow 服务。失败时，upstart 会自动重启进程（直到达到`*.conf`文件中设置的重启限制）。

您可以在`scripts/upstart`目录中找到示例 upstart 作业文件。这些文件已在 Ubuntu 14.04 LTS 上测试过。您可能需要调整`start on`和`stop on`设置，以使其适用于其他 upstart 系统。`scripts/upstart/README`中列出了一些可能的选项。

您可以根据需要修改`*.conf`文件并将它们复制到`/etc/init`目录。这是基于假设 Airflow 将在`airflow:airflow`用户/组下运行的情况。如果您使用其他用户/组，请在`*.conf`文件中更改`setuid`和`setgid`。

您可以使用`initctl`手动启动，停止，查看已与 upstart 集成的 Airflow 过程的状态

```py
initctl airflow-webserver status
```
