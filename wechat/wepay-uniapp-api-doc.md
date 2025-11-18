API /多端框架新增API /微信支付
wx.miniapp.requestPayment
App 调起微信支付

接入前注意事项
在接入支付前需详细阅读下方说明：

该能力依赖「微信 Open SDK」，因此需按照文档前往微信开放平台申请移动应用账号、前往微信商户平台创建商户号，并且将两者进行绑定等操作，详情可查看
完成上述账号操作后，需将移动应用账号与多端应用账号进行绑定，详情可查看
最后，由于该能力依赖「微信 Open SDK」，因此在 project.miniapp.json 中需勾选 Open SDK；注意， iOS 版的 Open SDK 需要勾选含支付的版本；
补充：该能力需要使用微信支付商户号，需配置 Api key 配置商户号证书等内容，详情可查看APP支付接入前准备
关于微信支付签名的注意事项
微信支付的 Api 和签名有 V2 和 V3 两个版本，开发者需使用正确版本的 Api 和对应的签名才可以正常调用，否则会出现「签名不一致」的报错
wx.miniapp.requestPayment 中 sign 仅支持 V3 的签名，即开发者需使用 V3 的下单接口配合使用
API_V2
使用 V2 的下单接口，需使用 V2 的签名
API_V3
使用 V3 的下单接口，需使用 V3 的签名
参数
参数对齐微信支付 - APP调起支付参数，方便理解
参数的大小写以下文为准，即在参数的大小写上没有与微信支付 - APP调起支付对齐
此外，在微信支付 - APP调起支付中还有个 appid 参数，该参数在本接口中不需要开发者传递，基础库会自动将该多端应用所绑定的移动应用 appid 传递给支付侧
属性	类型	默认值	必填	说明
mchId	string		是	商户号
prepayId	string		是	预支付交易会话ID
nonceStr	string		是	随机字符串
package	string		是	随机字符串，暂填写固定值Sign=WXPay
timeStamp	string		是	时间戳
sign	string		是	签名；注意：签名方式一定要与统一下单接口使用的一致，仅支持 V3 的签名
代码例子
    wx.miniapp.requestPayment({
      timeStamp: '1667792176',
      mchId: '1800009365',
      prepayId: 'wx07113616363804b19dde94884922030000',
      package: 'Sign=WXPay',
      nonceStr: '8ne443gjxxg',
      sign: '4FF5900870B5C5BCB089789BC004156426C46512CE566DB17C131747E09ADEBA',
      success: (res) => {
        console.warn('wx.miniapp.requestPayment success:', res)
      },
      fail: (res) => {
        console.error('wx.miniapp.requestPayment res:', res)
      },
      complete: (res) => {
        console.error('wx.miniapp.requestPayment res:', res)
      }
    })
一些常见的问题
v2接口签名报错排查指引
APP调起支付返回：-1
签名报错的情况下，还可以看下这个文档里的关于 Android 的部分
如果依据上述的自助排查指引依旧无法解决问题，可在App 支付开发指引文档中心的右侧栏寻找「技术支持」
商户传入的appid参数不正确
如果出现下方的报错，请按照下方思路自行排查
首先，截图中的 appid 指的是「移动应用的appid」，不是指「多端应用的id」；而在多端应用中，移动应用 appid 在本接口中不需要开发者传递，基础库会自动将该多端应用所绑定的移动应用 appid 传递给支付侧
所以，开发者需先自行确认「多端应用 - 移动应用 - 商户号」这三者的绑定关系是否正确
可以在「微信开发者工具-多端应用模式-详情」那里看多端应用 id 和移动应用 appid 信息
可以在「微信开放平台 - 移动应用 - 详情」那里看下绑定的商户号信息（或者在微信支付商户平台那里看下绑定的移动应用账号信息）
最后注意事项则是：必须构建安装包装到手机上测试，不可以使用移动应用助手测试。

tips
如果向微信支付寻求「技术支持」，可将支付侧对应的下单接口和签名文档发给他们，不要将本接口文档地址发给支付侧
通常，向微信支付寻求「技术支持」，他们需获取你的移动应用 appid ，以及该移动应用账号上配置的包名、签名、bundleid 等信息，以及你的商户号信息，便于帮助定位问题
此外，他们还会询问相关的接口文档地址，你可以将微信支付 - APP调起支付文档发给他们，以及对应的 v2 和 V3 的下单接口和签名文档链接发给他们即可