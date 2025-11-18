# WeChat Login Flows: Mini Program vs. Donut (iOS/Android)

Purpose
- Explain the login experience from two angles:
  1) User perspective (what users see/do)
  2) Process-flow perspective (what the app/SDK/backend do)
- Provide ASCII diagrams and an index of relevant files/functions to speed onboarding for future engineers.

Scope
- Mini Program client (MP)
- Donut mobile shells (iOS/Android) using the WeChat “多端应用” login service


Important Note (WeChat Doc 5.1: “页面不存在”)
- When using Donut (multi-end) login, WeChat requires the mini program to include a special authorization page. You must rebuild and publish the mini program for it to take effect.
- If the mini program isn’t re-published after the config, users who jump from the App to the mini program for authorization can see “页面不存在”.
- Reference: 快速接入小程序登录服务 → Section 5.1 “页面不存在”
  - https://developers.weixin.qq.com/miniprogram/dev/platform-capabilities/miniapp/quickstart/auth.html#_5-1-%E9%A1%B5%E9%9D%A2%E4%B8%8D%E5%AD%98%E5%9C%A8

---

1) User Perspective

Mini Program (MP)
- From the index page, the user taps Login.
- If WeChat privacy consent is required, the user sees and accepts it.
- A short “登录成功” toast appears; the page stays in the mini program.
- If profile is incomplete, the app navigates to a profile onboarding screen; otherwise it loads the form and normal UX continues.

iOS/Android (Donut shells)
- From the index page, the user taps Login.
- The app opens the custom Donut login page. The user checks consent and taps the login button.
- The login page coordinates with WeChat to obtain a code and calls our backend to authenticate.
- On success, the login page closes; the user sees a “登录成功” toast on return and stays in the app. Profile onboarding flows as needed.

---

2) Process-Flow Perspective

Mini Program (MP)
1. Index page triggers auth: `startAuthentication()` → check `wx.getPrivacySetting()`.
2. If needed, call `wx.requirePrivacyAuthorize()`.
3. Proceed with `performLogin()` → `authManager.authenticate()`.
4. MP branch: `authManager.wechatLogin()` → `wx.login()` → code → backend `/auth/wechat-login`.
5. Backend returns `{ access_token, user, openid }`.
6. Client persists token and user in storage and `getApp().globalData`.
7. UI feedback; optional profile onboarding; load form schema.

ASCII (MP)

```
User         Index Page          AuthManager           WeChat SDK            Backend
 |    tap Login    |                   |                    |                   |
 |---------------->| startAuthentication()                   |                   |
 |                 |-- wx.getPrivacySetting/require -->      |                   |
 |                 | performLogin()                          |                   |
 |                 |-------------------> authenticate()      |                   |
 |                 |                    wechatLogin() (MP)   |                   |
 |                 |---------------------------------------->| wx.login()        |
 |                 |                           code <--------|                   |
 |                 |---------------------------------------------- POST /auth/wechat-login
 |                 |                                     {access_token,user,openid} <---|
 |                 | store token/user; update globalData                           |
 |       toast     |<--------------------------------------------------------------------|
```

Donut iOS/Android
1. Index page calls unified `performLogin()` → `authManager.authenticate()`.
2. Donut branch: `authManager.wechatLogin()` → `donutLogin()` navigates to `/pages/index/login`.
3. Donut login page:
   - Calls `wx.weixinMiniProgramLogin()` to bridge to the mini program authorization flow.
   - Then calls `wx.login()` to obtain a code.
   - Calls backend `/auth/wechat-login`.
4. On success, emits `donutLoginComplete` back to `authManager.donutLogin()`.
5. Auth manager persists token/user and resolves back to the index.
6. UI feedback; optional profile onboarding; load form schema.

ASCII (Donut)

```
User       Index Page        AuthManager         Donut Login Page     WeChat SDK/Miniapp       Backend
 |  tap Login  |                   |                     |                     |                   |
 |------------>| performLogin()    |                     |                     |                   |
 |             |---- authenticate() ------------------> donutLogin()           |                   |
 |             |                   |       navigateTo('/pages/index/login')    |                   |
 |             |                   |                     |-- weixinMiniProgramLogin() --> [Authorization Page]
 |             |                   |                     |-- wx.login() ----------------------------->|
 |             |                   |                     |            code                           |
 |             |                   |                     |----------------- POST /auth/wechat-login ->|
 |             |                   |                     |    {access_token,user,openid}             |
 |             |                   |<-- donutLoginComplete (persist token/user)                       |
 |     toast   |<--------------------------------------------------------------------------------------|
```

---

R

