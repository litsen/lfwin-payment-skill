---
name: lfwin-payment
description: LFWIN酷收银聚合支付接口对接操作指南。使用本 skill 实现或修改 LFWin 聚合支付能力，包括 MD5/RSA 签名、表单 POST、条码支付、扫码支付、H5/JSAPI/小程序支付、微信小程序跳转支付、统一收银台、订单轮询、异步通知、退款、关单、撤销、分账和支付故障排查。
---

# LFWin Payment

使用本 skill 对接 LFWin 聚合支付能力。它不是普通接口说明书，而是给 Agent 和研发人员使用的操作指南：先判断业务场景，再选择接口，最后按签名、验签、状态机、回调和退款流程落地实现。

## 阅读顺序

先读本文件，再按任务加载对应 reference：

- 核心协议、签名、验签、状态码：[references/core.md](references/core.md)
- 支付创建类接口：统一收银台、条码、扫码、H5、微信/支付宝小程序：[references/payment-flows.md](references/payment-flows.md)
- 查单、退款、异步通知、关单、撤销：[references/order-refund-notify.md](references/order-refund-notify.md)
<!-- - 分账能力：[references/profit-sharing.md](references/profit-sharing.md)
- 原始完整接口文档：[references/source-docs](references/source-docs) -->

## 推荐对接流程

除非用户或现有代码明确要求其他方式，默认按这个流程实现：

1. 在本地订单表创建商户订单，生成唯一 `mch_orderid`，金额使用“元”的字符串格式，状态置为待支付。
2. 根据支付场景选择 LFWin 接口，使用 `application/x-www-form-urlencoded` 发起 POST。
3. 除 `sign` 外，所有实际提交的参数都参与签名；提交了空值也要参与；`signkey` 只用于签名，不能提交。
4. 创建支付响应 `status == 10000` 时，如果响应包含 `sign`，先验签，再保存 `orderid`、`order_time`、渠道字段、支付链接/二维码/调起参数。
5. 扫码、H5、JSAPI、小程序支付创建后，前端展示支付能力，同时后端用 `pay.comm.query_order` 约每 2 秒轮询一次，生产环境也接收异步通知。
6. 只有 `status == 10000 && paystatus == 1` 才能标记支付成功；只有 `status == 10000 && paystatus == 2` 才能标记支付失败。HTTP 超时、网络失败、其他状态都先按待支付/未知处理。
7. 异步通知必须幂等：验签、锁定或比较本地订单状态、只允许合法状态流转，处理成功后输出纯文本 `success`。
8. 退款请求返回 `status == 10000` 或 `status == 4001` 只表示退款请求已受理或处理中，不能当成退款成功；必须再调用退款查询确认 `refund_status`。

## 环境

- 测试环境：`http://api2uat.lfwin.com`
- 生产环境：`http://api2.lfwin.com`
- 文档测试账号：`apikey=00014005`，`signkey=punr8ucu`
- 测试环境只支持测试 `apikey`，只支持当月订单，并且不发送异步通知。
- 生产环境的 `apikey` 和 `signkey` 从商户后台设备列表获取。

## 接口选择


| 业务需求         | 请求路径                       | service                                                   | 详见                       |
| ------------ | -------------------------- | --------------------------------------------------------- | ------------------------ |
| 快速对接统一用收银台    | `/index/Payment/pre_order` | 文档未要求 service                                             | `payment-flows.md`       |
| 条码支付（B扫C）     | `/payapi/pay/barcode`      | 推荐 `pay.comm.barcode`                                     | `payment-flows.md`       |
| 聚合扫码支付     | `/payapi/trans/kxpay`      | `pay.comm.jspay`                                          | `payment-flows.md`       |
| 扫码支付（C扫B、支付宝链接跳转）      | `/payapi/pay/qrcode`       | `pay.alipay.qrcode`、`pay.wxpay.qrcode`、`pay.unpay.qrcode` | `payment-flows.md`       |
| H5/链接跳转支付    | `/payapi/pay/jspay3`       | `alipay.comm.jspay`、`wxpay.comm.jspay`、`unpay.comm.jspay` | `payment-flows.md`       |
| 微信公众号/小程序支付  | `/payapi/mini/wxpay`       | `comm.js.pay`、`comm.mini.pay`                             | `payment-flows.md`       |
| 微信小程序跳转支付    | `/payapi/mini/mini_url`    | `comm.mini.url`                                            | `payment-flows.md`       |
| 支付宝生活号/小程序支付 | `/payapi/trade/alipay`     | `comm.js.pay`、`comm.mini.pay`                             | `payment-flows.md`       |
| 查询支付状态       | `/payapi/pay/query_order`  | `pay.comm.query_order`                                    | `order-refund-notify.md` |
| 发起退款         | `/payapi/pay/refund_order` | `pay.comm.refund_order`                                   | `order-refund-notify.md` |
| 查询退款         | `/payapi/pay/query_refund` | `pay.comm.query_refund`                                   | `order-refund-notify.md` |
| 关闭未支付订单      | `/payapi/pay/close_order`  | `pay.comm.close_order`                                    | `order-refund-notify.md` |
| 撤销条码支付订单     | `/payapi/pay/cancel_order` | `pay.comm.cancel_order`                                   | `order-refund-notify.md` |



## 签名规则红线

- 默认使用 MD5，除非明确配置 `sign_type=RSA`。
- 待签名串取所有提交或返回参数，排除 `sign`，按字段名 ASCII 升序排序。
- 拼接格式为 `key=value&key=value`，使用原始值，不对字段名和值做 URL Encode。
- MD5 签名在末尾追加 `&signkey=<商户密钥>`，再计算 32 位小写 MD5。
- RSA 签名不追加 `signkey`；请求用商户 RSA 私钥签名，响应/通知用平台 RSA 公钥验签。
- `notify_url` 参与签名前要先做 URL Decode。
- 验签不要写死字段列表；平台可能新增返回字段。
- `lists`、`fund_bill_list` 等数组/对象字段，在验签前按 JSON 字符串参与签名。
- `status != 10000` 的失败响应通常不签名；没有 `sign` 时不要强行验签。

## 实现检查清单

- 支付密钥不能写死在代码里，必须从配置或密钥系统读取。
- 金额在 API 边界使用字符串，单位为元，例如 `0.01`。
- `nonce_str` 长度不超过 32 位。
- 每次下单生成唯一 `mch_orderid`；如果用 `mch_orderid` 查单或退款，要同时传大致 `order_time`。
- HTTP 请求使用表单 POST；只有 `goods_info`、`split_info` 这类字段值本身是 JSON 字符串。
- 同时保存本地订单号和 LFWin `orderid`；拿到 LFWin `orderid` 后优先用它查单、退款。
- 回调处理和退款请求都要做幂等，退款用 `mch_refund_no` 标识一次退款。
- 至少补充签名串生成、通知验签、支付状态流转、退款状态流转的单元测试。

## 常见坑

- 不要把 LFWin 响应的 `sign_type` 和微信前端调起支付的 `signType` 混淆。
- 不要把 `/payapi/mini/mini_url` 微信小程序跳转支付和 `/payapi/mini/wxpay` 小程序内直接调起支付混淆；前者返回 Scheme/URL Link，后者返回前端调起支付参数。
- 不要只依赖异步通知；状态不明时必须主动查单。
- 不要把退款请求的 `status == 10000` 当成退款成功。
- 不要高频查单或查退款；推荐约 2 秒一次。
- 下单时不要同时传 `split` 和 `split_info`。
- 条码支付撤销必须在订单创建 15 秒后，且只能在 1:00-23:00 操作。
