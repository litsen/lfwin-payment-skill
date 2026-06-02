# 支付创建流程

## 统一收银台

适用场景：希望最快完成通用支付对接，不想分别处理微信、支付宝、云闪付等通道差异。

- 请求路径：`POST /index/Payment/pre_order`
- 必填：`apikey`、`money`、`mch_orderid`、`nonce_str`、`sign`
- 选填：`succ_url`、`err_url`、`notify_url`、`goods_tag`、`goods_info`、`split_info`、`split`、`time_expire`
- 响应字段 `data` 为支付跳转 URL 或支付参数。
- 原始文档没有要求传 `service` 参数，按普通签名规则处理即可。

## 条码支付：商户扫顾客付款码

推荐使用聚合条码支付，由 LFWin 自动识别微信、支付宝、云闪付。

- 请求路径：`POST /payapi/pay/barcode`
- service：`pay.comm.barcode`
- 必填：`service`、`apikey`、`money`、`dynamic_id`、`nonce_str`、`sign`
- 选填：`mch_orderid`、`mid`、`remarks`、`guid`、`time_expire`、`goods_info`、`split_info`、`scene`、`terminal_params`、`good_name`、`sign_type`

指定通道时，请求结构相同，只替换 service：

| 通道 | service |
|---|---|
| 支付宝 | `pay.alipay.barcode` |
| 微信 | `pay.wxpay.barcode` |
| 云闪付 | `pay.unpay.barcode` |

付款码格式参考：

- 微信：`10`、`11`、`12` 开头，18 位纯数字。
- 支付宝：通常 `25` 开头，16-24 位。
- 云闪付：通常 `62` 开头，19 位。

条码支付如果没有得到明确终态，应先查单。仍无法确认且符合条件时，再调用撤销订单。

## 条码支付前解码

适用场景：扣款前需要先把付款码解析为用户身份标识。

- 请求路径：`POST /payapi/pay/decode_bar`
- service：`pay.alipay.decode_bar`
- 必填：`service`、`apikey`、`dynamic_id`、`nonce_str`、`sign`
- 选填：`sence_no`、`sub_appid`
- 返回可能包含：`user_id`、`openid`、`sub_openid`、`sub_appid`、`appid`
- 如果解码时传了 `sence_no`，后续条码支付也要传相同值。

## 扫码支付：顾客扫商户二维码

聚合扫码：

- 请求路径：`POST /payapi/trans/kxpay`
- service：`pay.comm.jspay`
- 必填：`service`、`apikey`、`money`、`nonce_str`、`sign`
- 选填：`mch_orderid`、`notify_url`、`succ_url`、`err_url`、`remarks`、`store_dis`、`goods_info`、`split_info`、`split`、`attach`、`sign_type`
- 响应 `url` 用来生成二维码展示给顾客。

指定通道扫码：

- 请求路径：`POST /payapi/pay/qrcode`
- service：`pay.alipay.qrcode`、`pay.wxpay.qrcode`、`pay.unpay.qrcode`
- 必填：`service`、`apikey`、`money`、`nonce_str`、`sign`
- 响应包含 `qr_code`，可能包含 `code_url`。

处理规则：

- `url` 或 `qr_code` 是二维码内容；`code_url` 可作为图片地址展示。
- 创建订单后用 `pay.comm.query_order` 轮询支付状态。
- 二维码默认约 15 分钟超时。
- 测试环境不会推送 `notify_url`。

## H5/链接跳转支付

适用场景：公众号、WebView 或 H5 页面中需要生成跳转支付链接，尤其是需要避免支付主体不一致问题时。

- 请求路径：`POST /payapi/pay/jspay3`
- service：`alipay.comm.jspay`、`wxpay.comm.jspay`、`unpay.comm.jspay`
- 必填：`service`、`apikey`、`money`、`nonce_str`、`sign`
- 选填：`mch_orderid`、`notify_url`、`remarks`、`store_dis`、`time_expire`、`split_info`、`sign_type`
- 响应 `url` 为 H5 支付页面地址。

需要提前在 LFWin 后台配置平台 APPID 和支付目录。原文提到公众号支付安全目录配置为 `http://www.lfwin.com/wxpay/`。

## 微信公众号和小程序支付

- 请求路径：`POST /payapi/mini/wxpay`
- service：公众号支付用 `comm.js.pay`，小程序支付用 `comm.mini.pay`
- 必填：`service`、`apikey`、`money`、`nonce_str`、`sub_appid`、`sub_openid`、`sign`
- 选填：`mch_orderid`、`notify_url`、`time_expire`、`goods_info`、`goods_tag`、`sub_wx_mchid`、`split_info`、`split`、`attach`、`good_name`、`sign_type`
- 响应包含前端调起支付字段：`appId`、`timeStamp`、`nonceStr`、`signType`、`package`、`paySign`

注意：`sign_type` 是 LFWin 接口自身签名方式，`signType` 是微信前端调起支付的签名方式，不能混淆。前端字段 `timeStamp` 大小写敏感。

## 支付宝生活号和小程序支付

- 请求路径：`POST /payapi/trade/alipay`
- service：生活号支付用 `comm.js.pay`，小程序支付用 `comm.mini.pay`
- 必填：`service`、`apikey`、`money`、`nonce_str`、`sign`，以及 `buyer_id` 或 `buyer_open_id` 二选一
- 选填：`sub_appid`、`mch_orderid`、`remarks`、`notify_url`、`order_url`、`goods_info`、`split_info`、`split`、`attach`
- 响应包含 `trade_no`、`orderid`、`order_time`、`mch_orderid`

`buyer_id` 是支付宝用户 ID，通常以 `2088` 开头；`buyer_open_id` 暂时仅支持官方通道小程序。

## goods_info 商品详情

单品优惠活动时该字段必传，格式为 JSON 字符串。

```json
[
  {"goods_id":"apple-01","goods_name":"iPhone6s 32G","quantity":"1","price":"528800"}
]
```

字段说明：

- `goods_id`：商品编码。
- `goods_name`：商品名称。
- `quantity`：购买数量，字符串。
- `price`：商品单价，单位为分；有优惠时传优惠后的单价。
