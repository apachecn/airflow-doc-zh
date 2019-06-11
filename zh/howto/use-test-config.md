# 使用测试模式配置

> 贡献者：[@ImPerat0R\_](https://github.com/tssujt)、[@ThinkingChen](https://github.com/cdmikechen) [@zhongjiajie](https://github.com/zhongjiajie)

Airflow 具有一组固定的“test mode”配置选项。您可以随时通过调用`airflow.configuration.load_test_config()`来加载它们（注意此操作不可逆！）。但是，在您有机会调用 load_test_config()之前，会加载一些选项（如 DAG_FOLDER）。为了更快加载测试配置，请在 airflow.cfg 中设置 test_mode：

```py
[tests]
unit_test_mode = True
```

由于 Airflow 的自动环境变量扩展（请参阅[设置配置选项](zh/howto/set-config.md) ），您还可以设置环境变量`AIRFLOW__CORE__UNIT_TEST_MODE`以临时覆盖 airflow.cfg。
