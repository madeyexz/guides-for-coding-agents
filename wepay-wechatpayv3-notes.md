# WeChat Pay v3 SDK (wechatpayv3)

https://github.com/minibear2021/wechatpayv3/

This document captures practical knowledge for integrating the `wechatpayv3` Python SDK into our stack (FastAPI backend + Uniapp/WeChat Mini Program frontend). 

## Key Concepts
- API v3 only; signatures use RSA-SHA256 with merchant API private key.
- Two init modes (SDK supports both):
  - Platform certificate mode: SDK downloads and caches WeChat Pay platform certs (needs `APIV3_KEY`, `CERT_DIR`).
  - Platform public-key mode (new merchants from 2024-09): supply `WECHATPAY_PUBLIC_KEY` and `WECHATPAY_PUBLIC_KEY_ID`; no cert downloads.
- Mini Program payments use JSAPI: backend creates prepay → returns invoke params → mini program calls `wx.requestPayment`.
- Final payment success comes from WeChat callback (server-to-server). Client success/failure UI is not authoritative.

## Required Config (env)
- `WECHATPAY_MCHID`: Merchant ID (mchid)
- `WECHATPAY_PRIVATE_KEY_PATH`: Path to merchant API private key (`apiclient_key.pem`)
- `WECHATPAY_CERT_SERIAL_NO`: Merchant cert serial number
- `WECHATPAY_APPID`: Mini Program AppID (must be bound to mchid)
- `WECHATPAY_APIV3_KEY`: API v3 symmetric key (AES-256-GCM for decrypt)
- `WECHATPAY_NOTIFY_URL`: Public HTTPS callback URL (WeChat server posts here)
- One of:
  - `WECHATPAY_CERT_DIR` (empty initially) – certificate mode; or
  - `WECHATPAY_PUBLIC_KEY` and `WECHATPAY_PUBLIC_KEY_ID` – public-key mode
- Optional: `WECHATPAY_PARTNER_MODE=false` (we use direct merchant mode)

Tip: Server clock must be accurate (signature/cert time validation relies on it).

## SDK Usage Mapping
- Init (async): `AsyncWeChatPay(**config)`; use once at app startup/lifespan.
- Pay (JSAPI): `await wxpay.pay(description, out_trade_no, amount={total,currency}, payer={'openid'}, pay_type=JSAPI)` → returns `prepay_id`.
- Sign invoke params: RSA over `[appId, timeStamp, nonceStr, package]` using `wxpay.sign([...])`.
- Verify/decrypt callback: `wxpay.callback(headers, body)` returns dict with `resource` on success. Idempotent processing required.
- Query order: `await wxpay.query(out_trade_no=...)` (optional reconcile)

## Our Backend Endpoints
- `POST /api/payments/wechat/jsapi` → returns invoke params for `wx.requestPayment` and `out_trade_no`.
- `POST /api/payments/wechat/notify` → verifies/decrypts, updates DB, grants credits (idempotent).
- `GET /api/payments/wechat/orders` → list current user’s WeChat payments.
- `GET /api/payments/wechat/orders/{out_trade_no}` → detail, optional reconcile.

## Our Data Model (payments)
- `provider='wechat_pay'`, `status in ('pending','succeeded','failed','refunded')`
- `provider_order_id` = `out_trade_no` (merchant order id)
- `provider_txn_id` = `transaction_id` (WeChat platform id)
- `raw_payload` JSONB stores prepay/callback snippets for audit.
- Idempotency: unique `(provider, provider_order_id)`; callback processing must be safe on retries.

## Mini Program Flow
1) Ensure login → JWT available.
2) Call `POST /api/payments/wechat/jsapi` with `{ sku }` (maps to credits and CNY amount). Receive invoke params.
3) Call `wx.requestPayment({ timeStamp, nonceStr, package, signType: 'RSA', paySign })`.
4) Show status UI; navigate to “支付记录” page; server callback finalizes state.

## Pitfalls & Troubleshooting
- Callback signature fails: ensure you pass FastAPI `request.headers` and raw `await request.body()` to `wxpay.callback`. Don’t coerce/alter body.
- CERT vs PUBLIC-KEY mode: new merchants often require `WECHATPAY_PUBLIC_KEY` + `WECHATPAY_PUBLIC_KEY_ID`. If `/v3/certificates` returns 500 repeatedly, switch to public-key mode.
- `APPID_MCHID_NOT_MATCH`: ensure Mini Program appid is bound to mchid in WeChat Pay console.
- `OUT_TRADE_NO_USED`: ensure unique `out_trade_no`. We use `wj_yyyyMMddHHmmss_<8hex>`.
- Duplicate callbacks: normal; process idempotently.
- Dev testing: Native pay (QR) is simplest for param validation, but we’ve integrated JSAPI directly; use a dev appid/mchid binding.

## Next Steps When Credentials Ready
1) Populate `backend/.env.local` with WECHATPAY_*.
2) Ensure callback URL is reachable and set in WeChat Pay console.
3) Run backend, confirm “client initialized successfully” in logs.
4) Use a low-amount SKU and complete a real transaction in dev.
5) Verify DB row shows `succeeded` and credits increased.

## References
- SDK examples (FastAPI async): `examples/server/async_examples/fastapi_example.py` in `wechatpayv3` repo
- Q&A and crypto details: `docs/Q&A.md` and `docs/security.md` in `wechatpayv3`
- Integration plan: `docs/weixin/wechatpay-integration-plan.md`

