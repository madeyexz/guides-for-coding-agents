APP下单
更新时间：2025.03.31
用户在商户APP端选择微信支付后，商户需调用该接口在微信支付下单，生成用于调起支付的预支付交易会话标识(prepay_id)。

接口说明
支持商户：【普通商户】

请求方式：【POST】/v3/pay/transactions/app

请求域名：【主域名】https://api.mch.weixin.qq.com 使用该域名将访问就近的接入点

　　　　　【备域名】https://api2.mch.weixin.qq.com 使用该域名将访问异地的接入点 ，指引点击查看

请求参数
Header  HTTP头参数

 Authorization 　必填　string

请参考签名认证生成认证信息

 Accept 　必填　string

请设置为application/json

 Content-Type 　必填　string

请设置为application/json

body  包体参数

 appid 　必填   string(32)

【商户应用ID】APPID是商户移动应用唯一标识，在开放平台(移动应用)申请。此处需填写与mchid完成绑定的appid，详见：商户模式开发必要参数说明。

 mchid 　必填   string(32)

【商户号】是由微信支付系统生成并分配给每个商户的唯一标识符，商户号获取方式请参考商户模式开发必要参数说明。

 description 　必填   string(127)

【商品描述】商品信息描述，用户微信账单的商品字段中可见(可参考APP支付示例说明-账单示意图)，商户需传递能真实代表商品信息的描述，不能超过127个字符。

 out_trade_no 　必填   string(32)

【商户订单号】商户系统内部订单号，要求6-32个字符内，只能是数字、大小写字母_-|* 且在同一个商户号下唯一。

 time_expire 　选填   string(64)

【支付结束时间】

1、定义：支付结束时间是指用户能够完成该笔订单支付的最后时限，并非订单关闭的时间。超过此时间后，用户将无法对该笔订单进行支付。如需关闭订单，请调用关闭订单API接口。

2、格式要求：支付结束时间需遵循rfc3339标准格式：yyyy-MM-DDTHH:mm:ss+TIMEZONE。yyyy-MM-DD 表示年月日；T 字符用于分隔日期和时间部分；HH:mm:ss 表示具体的时分秒；TIMEZONE 表示时区（例如，+08:00 对应东八区时间，即北京时间）。

示例：2015-05-20T13:29:35+08:00 表示北京时间2015年5月20日13点29分35秒。

3、注意事项：

time_expire 参数仅在用户首次下单时可设置，且不允许后续修改，尝试修改将导致错误。

若用户实际进行支付的时间超过了订单设置的支付结束时间，商户需使用新的商户订单号下单，生成新的订单供用户进行支付。若未超过支付结束时间，则可使用原参数重新请求下单接口，以获取当前订单最新的prepay_id 进行支付。

支付结束时间不能早于下单时间后1分钟，若设置的支付结束时间早于该时间，系统将自动调整为下单时间后1分钟作为支付结束时间。

 attach 　选填   string(128)

【商户数据包】商户在创建订单时可传入自定义数据包，该数据对用户不可见，用于存储订单相关的商户自定义信息，其总长度限制在128字符以内。支付成功后查询订单API和支付成功回调通知均会将此字段返回给商户，并且该字段还会体现在交易账单。

 notify_url 　必填   string(255)

【商户回调地址】商户接收支付成功回调通知的地址，需按照notify_url填写注意事项规范填写。

 goods_tag 　选填   string(32)

【订单优惠标记】代金券在创建时可以配置多个订单优惠标记，标记的内容由创券商户自定义设置。详细参考：创建代金券批次API。如果代金券有配置订单优惠标记，则必须在该参数传任意一个配置的订单优惠标记才能使用券。如果代金券没有配置订单优惠标记，则可以不传该参数。

示例：
如有两个活动，活动A设置了两个优惠标记：WXG1、WXG2；活动B设置了两个优惠标记：WXG1、WXG3；
下单时优惠标记传WXG2，则订单参与活动A的优惠；
下单时优惠标记传WXG3，则订单参与活动B的优惠；
下单时优惠标记传共同的WXG1，则订单参与活动A、B两个活动的优惠；

 support_fapiao 　选填   boolean

【电子发票入口开放标识】 传入true时，支付成功消息和支付详情页将出现开票入口。需要在微信支付商户平台或微信公众平台开通电子发票功能，传此字段才可生效。 详细参考：电子发票介绍
true：是
false：否

 amount 　必填   object

【订单金额】订单金额信息

	属性
 detail 　选填   object

【优惠功能】 优惠功能

	属性
 scene_info 　选填   object

【场景信息】 场景信息

	属性
settle_info 　选填   object

【结算信息】 结算信息

	属性
请求示例

curl
Java
Go
POST

curl -X POST \
  https://api.mch.weixin.qq.com/v3/pay/transactions/app \
  -H "Authorization: WECHATPAY2-SHA256-RSA2048 mchid=\"1900000001\",..." \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "appid" : "wxd678efh567hg6787",
    "mchid" : "1230000109",
    "description" : "Image形象店-深圳腾大-QQ公仔",
    "out_trade_no" : "1217752501201407033233368018",
    "time_expire" : "2018-06-08T10:34:56+08:00",
    "attach" : "自定义数据说明",
    "notify_url" : " https://www.weixin.qq.com/wxpay/pay.php",
    "goods_tag" : "WXG",
    "support_fapiao" : false,
    "amount" : {
      "total" : 100,
      "currency" : "CNY"
    },
    "detail" : {
      "cost_price" : 608800,
      "invoice_id" : "微信123",
      "goods_detail" : [
        {
          "merchant_goods_id" : "1246464644",
          "wechatpay_goods_id" : "1001",
          "goods_name" : "iPhoneX 256G",
          "quantity" : 1,
          "unit_price" : 528800
        }
      ]
    },
    "scene_info" : {
      "payer_client_ip" : "14.23.150.211",
      "device_id" : "013467007045764",
      "store_info" : {
        "id" : "0001",
        "name" : "腾讯大厦分店",
        "area_code" : "440305",
        "address" : "广东省深圳市南山区科技中一道10000号"
      }
    },
    "settle_info" : {
      "profit_sharing" : false
    }
  }'
应答参数
200 OK

 prepay_id 　必填   string(64)

【预支付交易会话标识】预支付交易会话标识，APP调起支付时需要使用的参数，有效期为2小时，失效后需要重新请求该接口以获取新的prepay_id。

应答示例

200 OK

{
  "prepay_id" : "wx201410272009395522657a690389285100"
}
 

错误码
公共错误码
状态码

错误码

描述

解决方案

400

PARAM_ERROR

参数错误

请根据错误提示正确传入参数

400

INVALID_REQUEST

HTTP 请求不符合微信支付 APIv3 接口规则

请参阅 接口规则检查传入的参数

401

SIGN_ERROR

验证不通过

请参阅 签名常见问题排查

500

SYSTEM_ERROR

系统异常，请稍后重试

请稍后重试

业务错误码
状态码

错误码

描述

解决方案

400

APPID_MCHID_NOT_MATCH

AppID和mch_id不匹配

请确认AppID和mch_id是否匹配，查询指引参考：查询商户号绑定的APPID

400

INVALID_REQUEST

无效请求

请根据接口返回的详细信息检查

400

MCH_NOT_EXISTS

商户号不存在

请检查商户号是否正确，商户号获取方式请参考：普通商户模式开发必要参数说明

400

PARAM_ERROR

参数错误

请根据接口返回的错误描述检查参数，参数需按API文档字段填写说明填写

401

SIGN_ERROR

签名错误

请检查签名参数和方法是否都符合签名算法要求，参考：如何生成签名

403

NO_AUTH

商户无权限

请商户前往商户平台申请此接口相关权限，参考：权限申请

403

OUT_TRADE_NO_USED

商户订单号重复

请核实商户订单号是否重复提交

429

FREQUENCY_LIMITED

频率超限

请求频率超限，请降低请求接口频率

500

SYSTEM_ERROR

系统错误

系统异常，请用相同参数重新调用

