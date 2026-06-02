# 订单、退款与异步通知

## 查询订单

- 请求路径：`POST /payapi/pay/query_order`
- service：`pay.comm.query_order`
- 必填：`service`、`apikey`、`nonce_str`、`sign`，以及 `orderid`、`mch_orderid`、`dis_name` 三选一
- 如果只传 `mch_orderid`，必须同时传 `order_time`，用于定位订单所在月份的数据表。

支付终态判断：

| 条件 | 本地状态 |
|---|---|
| `status == 10000 && paystatus == 1` | 支付成功 |
| `status == 10000 && paystatus == 2` | 支付失败 |
| 其他响应、HTTP 超时、504、网络失败 | 待支付/未知，继续查询或人工确认 |

推荐轮询频率：约每 2 秒一次。收银台场景一般查询 1 分钟左右即可停止前端等待，本地订单继续保持待支付，后续由通知或人工对账处理。

## 支付成功异步通知

生产环境中，如果下单时传了 `notify_url`，支付成功后 LFWin 会尝试通知。

- 请求方式：`POST`
- 请求体：QueryString/form 表单参数
- 支付通知字段包括：`orderid`、`trade_no`、`dis_name`、`paystatus`、`paymoney`、`pri_paymoney`、`order_time`、`paytime`、`mch_orderid`、`notify_url`、`buyer_account`、可选 `attach`、`sign`
- 状态更新前必须验签。
- 处理成功后必须输出纯文本 `success`，不能带引号、JSON 或其他字符。
- 未输出 `success` 会被视为失败，重试时间大约为第 1、5、15、30 分钟。

通知处理要求：

1. 解析表单字段，不要丢弃未知字段。
2. `notify_url` 参与签名前先 URL Decode。
3. 根据 `sign_type` 使用 MD5 或 RSA 验签。
4. 只有 `paystatus == 1` 才能标记支付成功。
5. 通过 `orderid` 或 `mch_orderid` 幂等更新本地订单，重复成功通知直接返回成功。
6. 数据持久化成功后再输出 `success`。

## 发起退款

- 请求路径：`POST /payapi/pay/refund_order`
- service：`pay.comm.refund_order`
- 必填：`service`、`apikey`、`orderid`/`mch_orderid`/`dis_name` 三选一、`nonce_str`、`sign`、`refundmoney`、`version=4.0`
- 选填：`mch_refund_no`、`reason`、`mid`、`auth_mid`、`guid`、`split_info`、`notify_url`、`sign_type`
- 如果只传 `mch_orderid`，必须传 `order_time`。

重要规则：退款请求返回 `status == 4001` 或 `status == 10000`，只表示退款请求已受理或正在处理，不能表示退款成功。必须调用退款查询确认结果。

建议使用 `mch_refund_no` 做退款幂等。同一订单多次部分退款时，必须在上一笔退款状态明确后，再使用新的 `mch_refund_no` 发起下一笔。

## 查询退款

- 请求路径：`POST /payapi/pay/query_refund`
- service：`pay.comm.query_refund`
- 必填：`service`、`apikey`、`nonce_str`、`sign`，以及 `orderid` 或 `mch_orderid`
- 选填：`refund_no`、`mch_refund_no`、`order_time`、`page_no`、`page_size`、`version=4.0`、`sign_type`

推荐时机：提交退款约 1 分钟后开始查询；后续如需轮询，约每 2 秒一次。

退款终态判断：

| 条件 | 本地状态 |
|---|---|
| `status == 4001` | 退款处理中，继续查询 |
| `status == 10000 && refund_status == 1` | 退款成功 |
| `status == 10000 && refund_status == 2` | 退款失败 |
| `status` 为 `1002`、`4000`、`4002` | 退款失败或需要人工处理 |
| `status` 为 `3000`、`3900`、`3010`、`3030`、`3040` | 停止重试，修复配置或请求参数 |

## 退款完成异步通知

如果退款时传了 `notify_url`，LFWin 可能推送退款完成通知。

通知字段包括：`refund_orderid`、`orderid`、`addtime`、`notify_url`、`rt_trade_no`、`mch_refund_no`、`refundmoney`、`refund_handmoney`、`refund_status`、`refundtime`、可选 `attach`、`sign`。

处理方式与支付通知相同：先验签，再幂等更新。`refund_status == 1` 表示成功，`2` 表示失败，`0` 表示处理中。

## 关闭订单

适用场景：订单创建后消费者长时间未付款，需要关闭未支付订单。

- 请求路径：`POST /payapi/pay/close_order`
- service：`pay.comm.close_order`
- 必填：`service`、`apikey`、`orderid` 或 `mch_orderid`、`nonce_str`、`sign`
- 如果只传 `mch_orderid`，必须传 `order_time`。
- 仅未支付订单可关闭；已成功或已失败订单不能关闭。

## 撤销条码支付订单

适用场景：条码支付订单长时间没有终态或一直超时。撤销前必须先查单。

- 请求路径：`POST /payapi/pay/cancel_order`
- service：`pay.comm.cancel_order`
- 必填：`service`、`apikey`、`orderid` 或 `mch_orderid`、`nonce_str`、`sign`
- 最早可撤销时间：订单创建 15 秒后。
- 撤销时间限制：1:00-23:00。
- 如果银行或通道已判定支付成功或支付失败，撤销可能失败。

## 订单列表/订单详情

- 请求路径：`POST /payapi/pay/order_list`
- service：`order.list`
- 适用于对账、订单详情和批量查询。
- 订单列表响应验签时，`lists` 是数组，需要先序列化成 JSON 字符串再参与签名。
- `version=4.0` 可在有退款时返回最近一笔退款状态。

## 状态值

`paystatus`：

- `0`：待支付
- `1`：支付成功
- `2`：支付失败

`refund_status`：

- `0`：待处理
- `1`：处理成功
- `2`：退款失败

`is_refund`：

- `0`：普通订单
- `1`：撤销订单
- `2`：退款订单
- `3`：关闭订单
