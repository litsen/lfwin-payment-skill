# 分账功能

本文件用于实现 LFWin 分账能力，包括分账接收方绑定、订单分账、分账查询和分账订单退款。

## 分账模式

固定门店分账：如果门店总是按固定规则分给固定接收方，优先在后台配置门店自动分账。

动态接口分账有两种方式：

- 支付后自动分账：创建支付订单时传 `split_info`，订单支付成功后系统自动执行分账。
- 延迟分账：创建支付订单时传 `split=1` 标记为分账订单，支付成功后再调用分账请求接口，传入实际 `split_info`。

注意：创建支付订单时，`split` 和 `split_info` 不能同时传。

## split_info 格式

通用格式：

```json
[
  {"account":"E1800305780","amount":"10.00","is_original":1},
  {"account":"E1321546","amount":"0.02"}
]
```

字段说明：

- `account`：分账接收方商户号，不能传发起方自身。
- `amount`：分账金额，单位元。
- `is_original`：可选；支持的通道中，传 `1` 表示接收方收到扣除手续费后的金额。
- 同一笔订单可以传多个接收方，但接收方不能重复。

支付宝生活号/小程序文档还出现过支付宝专用格式：

```json
[
  {"mer_email":"youlian313@163.com","money":"0.01"}
]
```

如果涉及具体通道差异，必须回到原始文档或通道配置确认。

## 分账接收方绑定

添加接收方：

- 请求路径：`POST /payapi/pay/sharing_receiver_bind`
- service 示例：`pay.lspay.sharing_receiver_bind`
- service 前缀按通道变化，可为 `alipay`、`lspay`、`lspay2`、`heli`、`yop`
- 必填：`service`、`apikey`、`nonce_str`、`sign`、`sharing_merchant_id`、`receiver_merchant_id`、`receiver_merchant_name`
- 选填：`receiver_merchant_type`、`protocol_pic`、`remark`

查询接收方关联：

- 请求路径：`POST /payapi/pay/sharing_receiver_query`
- service 示例：`pay.lspay.sharing_receiver_query`
- 必填：`service`、`apikey`、`nonce_str`、`sign`、`sharing_merchant_id`

解除接收方关联：

- 请求路径：`POST /payapi/pay/sharing_receiver_unbind`
- service 示例：`pay.lspay.sharing_receiver_unbind`
- 必填：`service`、`apikey`、`nonce_str`、`sign`、`sharing_merchant_id`、`receiver_merchant_id`

## 延迟分账请求

- 请求路径：`POST /payapi/pay/order_sharing`
- service 示例：`pay.lspay.order_sharing`
- 必填：`service`、`apikey`、`orderid`、`nonce_str`、`sign`、`money`、`split_info`
- 选填：`mch_orderid`、`order_time`、`version`、`sign_type`
- `status == 10000` 表示分账请求提交成功，不代表最终分账完成；后续要查询分账结果。
- 不同通道资金入账可能有延迟。延迟分账建议支付成功约 30 秒后再发起。

`split_info` 示例：

```json
[
  {"account":"5215519353","amount":"1.25","remark":"测试1"},
  {"account":"5215519351","amount":"1.95","remark":"测试2"}
]
```

## 分账查询

- 请求路径：`POST /payapi/pay/query_sharing`
- service 示例：`pay.lspay.query_sharing`
- 必填：`service`、`apikey`、`orderid`、`nonce_str`、`sign`
- 选填：`mch_orderid`、`order_time`
- 响应可能包含 `sharing_status`、`sharing_orderid`、`out_order_no`、`lists`
- 返回 `sharing_status == FINISHED`，或明细中 `result == FINISHED` 时，可视为该分账结果完成。

## 分账订单退款

分账订单退款仍调用普通退款请求接口，但需要传 `split_info` 描述分账退回的出资方和金额。

实现要求：

- 第三方系统自行计算部分退款时各接收方应退回的金额。
- 确保出资方余额充足。
- 提交退款后仍然使用退款查询接口确认最终结果。
- 如果错误地对分账订单调用普通退款或撤销流程，可能返回 `4970`。

## 协议图片上传

部分乐刷接收方绑定流程需要上传协议图片。

- 请求路径：`POST /merchant/register/pic_nosql`
- service：文档出现 `lspay.apply.pic` / `heli.apply.pic`
- 该接口不是普通支付接口，不传 `apikey`，而是传 `agent_num`
- 签名密钥需联系技术获取。
- 上传 `pic` 文件或传 `pic_http_url`；响应 `mer_pic` 用作接收方绑定接口的 `protocol_pic`。

