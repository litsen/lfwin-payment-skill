# LFWin Payment Skill

LFWIN 酷收银聚合支付接口对接操作指南。适用于 Codex/Agent 实现或修改 LFWin 支付能力，包括签名、下单、查单、退款、异步通知、关单、撤销、分账和支付故障排查。

## 内容

- `SKILL.md`：skill 入口说明，包含触发描述、阅读顺序、接口选择和实现检查清单。
- `agents/openai.yaml`：Codex UI 展示元数据。
- `references/`：按支付创建、订单退款通知、分账、核心协议拆分的参考资料。
- `references/source-docs/`：原始接口文档 Markdown 版本。

## 安装

将本仓库克隆到 Codex skills 目录：

```powershell
git clone git@github.com:litsen/lfwin-payment-skill.git $env:USERPROFILE\.codex\skills\lfwin-payment
```

或复制当前目录到：

```text
%USERPROFILE%\.codex\skills\lfwin-payment
```

## 使用

在 Codex 中请求 LFWin 支付相关开发任务时，引用或触发 `lfwin-payment` skill，例如：

```text
使用 LFWin Payment skill 帮我实现下单、查单、退款和异步通知验签。
```

## 发布

不需要额外维护 `RELEASE.md`。建议使用 Git tag 和 GitHub Releases 管理版本，例如 `v0.1.0`、`v0.2.0`。

## License

MIT
