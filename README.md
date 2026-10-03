# Titanloom

Titanloom 是一个面向企业场景设计的综合数据与业务平台，覆盖填报与流程、考勤签到、工具自动化、数据处理、知识与文档、数据可视化，以及统一门户、后台治理、插件扩展、集成与消息通道和平台级 Agent。

**状态：设计阶段，尚无实现代码。** 当前内容是设计基线 V1.0-rc1：设计文档、契约 JSON Schema 草案及其测试。文档版本不代表软件发行版，也不表示性能已验收或安全已评审。

## 目录

| 路径 | 内容 |
| --- | --- |
| `docs/` | 总框架、公共规范、公共平台能力、各业务域子方案、架构决策记录、实施路线、变更记录 |
| `titanloom-contracts/` | 契约 JSON Schema 草案、状态机、示例与测试 |

建议从 `docs/Titanloom-总框架设计方案-V1.0.md` 与 `docs/Titanloom-实施路线与首期工程基线-V0.2.md` 开始阅读。

## 运行契约测试

需要 Python 3 与 `jsonschema`：

```
pip install jsonschema
cd titanloom-contracts
for t in test_contracts*.py; do python3 "$t" || exit 1; done
```

## 许可证

本项目以 [Apache License 2.0](LICENSE) 发布。

Copyright 2026 Titanloom contributors

第三方组件及其许可证见 THIRD_PARTY_NOTICES.md。贡献方式见 CONTRIBUTING.md，安全问题的报告方式见 SECURITY.md。
