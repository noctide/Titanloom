# 贡献指南

> 许可证以仓库根目录的 LICENSE 为准。

## 贡献者权利：DCO

本项目采用 Developer Certificate of Origin（DCO）。每个提交须带签署行：

```
git commit -s -m "说明"
```

这表示你声明：该贡献由你创作或你有权按本项目许可证提交，且你同意按 Apache-2.0 授权。不要提交：

- 任何企业的真实数据、模板原件、内部接口细节、凭据或内部文档；
- 你无权授权的第三方代码、素材、字体、模型权重。

测试数据一律使用合成数据。

## 提交前

- 新增依赖须写明许可证，并更新 THIRD_PARTY_NOTICES（发布时生成）。
- 契约与状态机变更须同步 titanloom-contracts 的 Schema 与测试，必要时新增 ADR。
