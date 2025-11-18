WeChat Pay (WePay) Integration Guide — Yijiu AI
================================================

This document explains how WeChat Pay (V3) is wired into Yijiu AI for both Mini Program (JSAPI) and iOS/Android (Open SDK APP pay) builds, what SDK capabilities we use, the issues we hit and how we resolved them, and a reference index of important files and functions.

Overview
--------

We support two payment channels, selected automatically at build-time via Uniapp conditional compilation:
- Mini Program (MP-Weixin): JSAPI pay
- Native iOS/Android (Donut Open SDK): APP pay

Both channels share the same backend and database. We rely on the WeChat Pay V3 SDK (wechatpayv3[async]) for signing, requests, callback verification, and decryption.

High-Level Flows
----------------

Mini Program (JSAPI):

```
+-------------+     POST /api/payments/wechat/jsapi        +----------------------+   verify+decrypt   +---------+
| MiniProgram | -----------------------------------------> | FastAPI (create pay) | <------------------ | WeChat  |
|  (MP)       | <--- wx.requestPayment params 200 -------- |  AsyncWeChatPay.pay  |                    | Server  |
|             | -- wx.requestPayment() --> WeChat Client   |                      | -- callback 200 -->|  (V3)   |
|             |                                            | + DB: pending -> ok  |                    +---------+
+-------------+                                            +----------------------+
```

iOS/Android (Open SDK APP pay):

```
+-----------+      POST /api/payments/wechat/app           +----------------------+   verify+decrypt   +---------+
| iOS/Android| ------------------------------------------> | FastAPI (create pay) | <------------------ | WeChat  |
| (Donut)   | <--- wx.miniapp.requestPayment params 200 -- |  AsyncWeChatPay.pay  |                    | Server  |
|           | -- wx.miniapp.requestPayment() --> WeChat    |                      | -- callback 200 -->|  (V3)   |
|           |                                              | + DB: pending -> ok  |                    +---------+
+-----------+                                              +----------------------+
```

Callback verification & decryption:

```
WeChat -> POST /api/payments/wechat/notify
         headers: Wechatpay-* (signature, timestamp, serial)
         body: JSON with encrypted resource

Backend:
  - sdk.callback(headers, body)  # verify signature + decrypt
  - resource.event_type == TRANSACTION.SUCCESS ?
      DB: mark pay succeeded, set transaction_id, success_time, grant credits
      return 200/204
```

Backend Wiring
--------------

We built a thin payment layer on top of wechatpayv3 async client.

Environment Variables
---------------------

- WECHATPAY_MCHID — merchant ID
- WECHATPAY_PRIVATE_KEY_PATH — merchant API private key (apiclient_key.pem)
- WECHATPAY_CERT_SERIAL_NO — merchant cert serial number
- WECHATPAY_APPID — Mini Program AppID (JSAPI)
- WECHATPAY_APIV3_KEY — V3 key (AES-256-GCM decrypt)
- WECHATPAY_NOTIFY_URL — public HTTPS callback URL
- WECHATPAY_CERT_DIR — platform cert cache dir (certificate mode)
- WECHATPAY_PUBLIC_KEY / WECHATPAY_PUBLIC_KEY_ID — set these ONLY for “平台公钥”模式
- WECHATPAY_PARTNER_MODE — false for our use case
- WECHAT_APPID_OAUTH — mobile app AppID (微信开放平台，APP 支付用)

Notes
- 我们当前使用“平台证书模式（商户证书模式）”，不是“平台公钥模式”。
- notify_url 必须是公网 HTTPS（Cloudflared/ngrok 都可），且需要反向到同一后端实例。

SDK Capabilities Used (wechatpayv3)
-----------------------------------

- AsyncWeChatPay(**config) — initialize V3 client
- pay(...) — we call with WeChatPayType.JSAPI for MP and WeChatPayType.APP for iOS/Android
- sign([...]) — build invoke signatures (JSAPI uses [appId, timeStamp, nonceStr, package]; APP uses [appId, timeStamp, nonceStr, prepay_id])
- callback(headers, body) — verify Wechatpay-* signature, decrypt resource
- query(out_trade_no=...) — optional reconcile

FastAPI Routes
--------------

- POST /api/payments/wechat/jsapi
  - Input: { sku } (or { amount_cents, description })
  - Action: create pending payment row → pay(type=JSAPI) → return wx.requestPayment params
- POST /api/payments/wechat/app
  - Input: { sku } (or { amount_cents, description })
  - Action: create pending payment row → pay(type=APP, appid=WECHAT_APPID_OAUTH) → return wx.miniapp.requestPayment params
- POST /api/payments/wechat/notify
  - Action: sdk.callback() verify/decrypt → mark succeeded + grant credits (idempotent)
- GET /api/payments/wechat/orders — list user orders
- GET /api/payments/wechat/orders/{out_trade_no} — order detail; ?reconcile=true optionally queries WeChat to refresh status

Database Layer
--------------

- On create: insert payments row (provider='wechat_pay', status='pending',
  provider_order_id = out_trade_no, product_id, credits_granted, amount_cents, provider_app_id = appid)
- On callback: update to succeeded, store transaction_id, success_time, grant credits once (idempotent)
- Helper functions in postgresql_payment_service.py:
  - insert_wechat_pending, update_wechat_prepay, mark_wechat_failed
  - mark_wechat_paid_and_grant
  - list_user_payments, get_user_payment_by_order

ASCII: Backend Module Relationships
-----------------------------------

```
+------------------------+     uses     +-----------------------+
| routes/wechat_payment  | -----------> | services/wechatpay_  |
|  - jsapi/app/notify    |              |   service (Async)     |
+------------------------+              +-----------------------+
          |                                      |
          |            calls                     | callback/sign
          v                                      v
+------------------------+              +-----------------------+
| db/postgresql_payment |<-------------| wechatpayv3.async_    |
|  - pending/success    |              |   (SDK)               |
+------------------------+              +-----------------------+
```

Frontend Wiring (Uniapp/Donut)
------------------------------

We use conditional compilation to select the proper channel at build-time.

- Mini Program (MP)
  - Calls /api/payments/wechat/jsapi
  - Use wx.requestPayment({ timeStamp, nonceStr, package, signType: 'RSA', paySign })
- iOS/Android (NATIVE = IOS || ANDROID)
  - Calls /api/payments/wechat/app
  - Use wx.miniapp.requestPayment({ timeStamp, mchId, prepayId, package: 'Sign=WXPay', nonceStr, sign })

ASCII: Conditional Call Sites
-----------------------------

```
// #if MP
  call /wechat/jsapi  -> wx.requestPayment(...JSAPI params...)
// #elif NATIVE
  call /wechat/app    -> wx.miniapp.requestPayment(...APP params...)
// #endif
```

Important: iOS/Android 的 AppID 和小程序 AppID 不同。
- APP 支付使用 WECHAT_APPID_OAUTH（微信开放平台“移动应用”）
- JSAPI 支付使用 WECHAT_APPID（微信公众平台“小程序”）

Issues We Hit And Solutions
---------------------------

1) Service returned 503 (WeChat Pay disabled)
- Cause: private key path invalid or missing env; or notify_url missing protocol
- Fix: ensure WECHATPAY_PRIVATE_KEY_PATH points to a real file; set WECHATPAY_NOTIFY_URL to full http/https; restart

2) Orders stuck at “pending” (credits not granted)
- Cause: notify_url not publicly reachable (no HTTPS or tunnel), so callback never hit backend
- Fix: expose backend via Cloudflared/ngrok; set WECHATPAY_NOTIFY_URL to public HTTPS; verify 200 in logs

3) Postgres error: could not determine data type of parameter $3
- Cause: SQL CASE with NULL param caused unknown type inference
- Fix: build JSON in Python and pass single $2::jsonb payload to UPDATE; cast COALESCE($3::text,'CNY') where needed

4) 404 spam on root or `/` POST
- Cause: scanners/monitors probing public tunnel
- Fix: ignore or add root 200 handler; critical path (/api/payments/wechat/notify) was fine

5) iOS APP pay 404
- Cause: backend without /api/payments/wechat/app (stale process) or iOS hitting a different base URL
- Fix: restart backend with new route; ensure iOS/Android apiBase points to same host; confirm via /openapi.json

6) APP pay used wrong AppID (签名/校验报错)
- Cause: APP 支付必须用“移动应用 AppID（开放平台）”，不是“小程序 AppID”
- Fix: create_app_prepay(appid=WECHAT_APPID_OAUTH) and sign with that AppID; ensure mchId↔AppID binding exists

7) Uniapp条件编译 JS 语法冲突（变量重名）
- Cause: 定义了重复的常量名（MP/NATIVE 解构出的 timeStamp 等）
- Fix: 使用平台前缀变量名（mpTimeStamp/appTimeStamp），或提取外层临时变量（outTradeNo）

Testing Checklist (Abbreviated)
-------------------------------

- MP: jsapi 下单→wx.requestPayment→notify 200→订单 succeeded→credits +N
- iOS/Android: app 下单→wx.miniapp.requestPayment→notify 200→订单 succeeded→credits +N
- Cancel flow: 用户取消→订单 pending，无加分
- Idempotency: 重复回调不重复加分
- Low amount: ¥0.01（amount_cents=1）正确显示与扣款

File & Function Index
---------------------

Backend
- backend/config.py
  - Adds WECHATPAY_* envs; WECHAT_APPID_OAUTH for APP pay
- backend/requirements.txt
  - wechatpayv3[async]
- backend/services/wechatpay_service.py
  - init_wechatpay(), shutdown_wechatpay() — client lifecycle
  - sku_catalog() — SKU→credits & cents
  - generate_out_trade_no() — unique merchant order id
  - create_jsapi_prepay(), build_jsapi_invoke_params() — JSAPI
  - create_app_prepay(), build_app_invoke_params() — APP（uses WECHAT_APPID_OAUTH）
  - verify_callback(), query_order()
- backend/routes/wechat_payment.py
  - POST /api/payments/wechat/jsapi → JSAPI invoke params
  - POST /api/payments/wechat/app   → APP invoke params
  - POST /api/payments/wechat/notify→ callback verify/decrypt → mark paid + grant
  - GET /api/payments/wechat/orders and /orders/{out_trade_no}
- backend/database/postgresql_payment_service.py
  - insert_wechat_pending, update_wechat_prepay
  - mark_wechat_failed, mark_wechat_paid_and_grant
  - list_user_payments, get_user_payment_by_order
  - Export: postgresql_payment_service
- backend/main.py
  - app.include_router(wechat_payment_router, prefix="/api")
  - Startup/shutdown: init/close WeChat Pay client

Frontend (Uniapp/Donut)
- frontend/yijiu_ai_wechat_program/pages/pay/index.js
  - // #if MP → call /wechat/jsapi + wx.requestPayment
  - // #elif NATIVE → call /wechat/app + wx.miniapp.requestPayment
  - Uses outTradeNo to navigate to Orders page
- frontend/yijiu_ai_wechat_program/pages/pay/index.wxml/wxss/json — purchase UI
- frontend/yijiu_ai_wechat_program/pages/orders/index.* — orders list
- frontend/yijiu_ai_wechat_program/app.json — registers pages
- frontend/yijiu_ai_wechat_program/pages/settings/settings.* — entry points to Purchase/Orders

Docs (supporting)
- docs/weixin/wechatpay-integration-plan.md — plan
- docs/weixin/wechatpayv3-notes.md — V3 SDK usage notes & pitfalls
- docs/weixin/wepay/doc-wx.miniapp.requestPayment.md — APP 调起支付规范（上游文档）
- docs/weixin/uniapp-conditional-compling-intro.md — 条件编译指南

Operations Tips
---------------

- Always restart backend when changing .env.local to re-init the SDK.
- Watch logs for: WeChat Pay client initialized successfully, then POST /api/payments/wechat/notify 200.
- Ignore periodic GET / or POST / 404s from scanners on public tunnels.
- Prefer low amount test SKU (¥0.01) for live tests.

If you need auxiliary tooling (e.g., a /health route for callback host or a /debug/notify echo), consider adding separately to avoid mixing with payment code.

