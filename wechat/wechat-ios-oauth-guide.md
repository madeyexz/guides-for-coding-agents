# Implementing WeChat Login in an iOS (Swift) App

## Overview and context

WeChat login enables users to sign into your iOS app using their WeChat identity. It is built on OAuth 2.0 and requires installing the WeChat app. Because it is a *third‑party social login* on iOS, Apple’s App Store Review Guidelines require that you **also provide Sign in with Apple** or another first‑party login method so users can choose a privacy‑preserving option.

WeChat’s iOS login SDK is written in Objective‑C, so your Swift project must include a bridging header. After integrating the SDK and configuring Info.plist, your app will call WeChat to request authorization. WeChat will return an authorization code, which you exchange for an access token and user information via WeChat’s API.

This document walks through the complete integration process for a Swift app built with Xcode 16 (iOS 18), based on WeChat’s open‑platform documentation and developer experiences[[1]](https://cclc.github.io/mess-knowledge/wechat-ios-login.html#:~:text=,URLSchemes%EF%BC%8C%E6%B7%BB%E5%8A%A0%20ATS)[[2]](https://stackoverflow.com/questions/35718897/how-to-add-the-wechat-api-to-a-swift-project#:~:text=pod%20%27WechatOpenSDK%27).

## Prerequisites

1. **WeChat Open Platform registration:** Register a WeChat Open Platform developer account and create a **mobile application**. Provide basic information, upload the app logo, and configure **Bundle ID** and **universal link** (a domain under your control that resolves to your app). After approval, you receive an **AppID** and **AppSecret**; these are needed to authenticate API requests.
2. **Xcode project:** Your project should target iOS 13 or later (universal links require iOS 9+). Ensure you have a **valid Apple Developer account** to configure universal links and handle Sign in with Apple.
3. **WeChat app installed:** WeChat login only works when the WeChat client is installed. The SDK provides a method (WXApi.isWXAppInstalled()) to check this[[3]](https://cclc.github.io/mess-knowledge/wechat-ios-login.html#:~:text=%2F).

## Step 1 – Download and integrate WeChat iOS SDK

1. **Add the SDK:** Either integrate manually or via CocoaPods:

# CocoaPods
pod 'WechatOpenSDK'

Alternatively, download the latest **OpenSDK\_iOS** from the WeChat Open Platform resource page and drag the framework and header files into Xcode[[4]](https://stackoverflow.com/questions/35718897/how-to-add-the-wechat-api-to-a-swift-project#:~:text=Download%20the%20SDK%20from%20the,if%20you%20install%20using%20CocoaPods).

1. **Add required system frameworks:** Import the following frameworks (if you integrate manually) in Xcode → **Build Phases → Link Binary With Libraries**:
2. SystemConfiguration.framework, Security.framework, CoreTelephony.framework, CFNetwork.framework
3. libsqlite3.0.tbd, libz.tbd, libc++.tbd
4. **Create a bridging header:** Because the SDK is written in Objective‑C, add a bridging header (e.g., YourProjectName-Bridging-Header.h) and import the WeChat header inside it:

#import "WXApi.h"

The StackOverflow answer for WeChat integration notes that a bridging header is needed to call Objective‑C code from Swift[[5]](https://stackoverflow.com/questions/35718897/how-to-add-the-wechat-api-to-a-swift-project#:~:text=As%20,to%20add%20a%20bridging%20header).

1. **Add Swift package import:** In your Swift files, you do not import WXApi directly; all Objective‑C declarations become available automatically once the bridging header is configured.

## Step 2 – Configure Info.plist

WeChat login requires several Info.plist entries:

1. **CFBundleURLTypes:** Add a URL scheme so that WeChat can return to your app. Set the scheme to your AppID (prefixed with wx), for example:

<key>CFBundleURLTypes</key>
<array>
 <dict>
 <key>CFBundleURLSchemes</key>
 <array>
 <string>wxYOUR\_APP\_ID</string>
 </array>
 </dict>
</array>

1. **LSApplicationQueriesSchemes:** iOS prevents apps from querying arbitrary URL schemes. Add the weixin scheme so that your app can check if the WeChat client is installed[[6]](https://stackoverflow.com/questions/35718897/how-to-add-the-wechat-api-to-a-swift-project#:~:text=):

<key>LSApplicationQueriesSchemes</key>
<array>
 <string>weixin</string>
</array>

1. **NSAppTransportSecurity:** WeChat’s OAuth API still uses HTTP. iOS 9+ enforces App Transport Security (ATS), which blocks plain HTTP. To allow WeChat network calls, add an ATS exception:

<key>NSAppTransportSecurity</key>
<dict>
 <key>NSAllowsArbitraryLoads</key>
 <true/>
</dict>

The cclc blog notes that ATS forces HTTP requests to TLS and recommends adding NSAllowsArbitraryLoads to the Info.plist[[1]](https://cclc.github.io/mess-knowledge/wechat-ios-login.html#:~:text=,URLSchemes%EF%BC%8C%E6%B7%BB%E5%8A%A0%20ATS).

1. **Associated Domains:** To support universal links, enable the **Associated Domains** capability in the Xcode Signing & Capabilities tab and add an entry like:

applinks:yourdomain.example.com

On your domain, host an apple-app-site-association file containing your app’s Bundle ID. The universal link will be used when registering the app with WeChat.

1. **Other privacy keys (optional):** If you request the user’s gender or nickname via the WeChat user info API, you must provide a privacy statement in your app under Settings → Privacy as required by Apple.

## Step 3 – Register the app at launch

In your AppDelegate implement UIApplicationDelegate and WXApiDelegate. During app launch, call WXApi.registerApp with your AppID and universal link:

import UIKit

@main
class AppDelegate: UIResponder, UIApplicationDelegate, WXApiDelegate {
 func application(\_ application: UIApplication,
 didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
 // Replace with your WeChat AppID and universal link
 WXApi.registerApp("wxYOUR\_APP\_ID", universalLink: "https://yourdomain.example.com/app/")
 return true
 }

 // MARK: - URL handling (iOS 9+)
 func application(\_ app: UIApplication, open url: URL,
 options: [UIApplication.OpenURLOptionsKey : Any] = [:]) -> Bool {
 // Forward the URL to WeChat SDK and return the result
 return WXApi.handleOpen(url, delegate: self)
 }

 // MARK: - Universal link handling (iOS 13+)
 func application(\_ application: UIApplication, continue userActivity: NSUserActivity,
 restorationHandler: @escaping ([UIUserActivityRestoring]?) -> Void) -> Bool {
 return WXApi.handleOpenUniversalLink(userActivity, delegate: self)
 }

 // MARK: - WXApiDelegate
 func onReq(\_ req: BaseReq) {
 // Not used for login
 }

 func onResp(\_ resp: BaseResp) {
 // See Step 6 for handling SendAuthResp
 }
}

The StackOverflow answer shows that prior to iOS 9 two handleOpenURL methods are used, while on iOS 9+ you should use application(\_:open:options:)[[7]](https://stackoverflow.com/questions/35718897/how-to-add-the-wechat-api-to-a-swift-project#:~:text=func%20application,handleOpen%28url%2C%20delegate%3A%20self%29). Your delegate should return the result of WXApi.handleOpen(url, delegate: self)[[8]](https://stackoverflow.com/questions/35718897/how-to-add-the-wechat-api-to-a-swift-project#:~:text=class%20AppDelegate%3A%20UIResponder%2C%20UIApplicationDelegate%2C%20WXApiDelegate,).

## Step 4 – Add a WeChat login button and send authorization request

1. **Hide the button if WeChat is not installed:** WeChat login only works when the WeChat client is installed. Use WXApi.isWXAppInstalled() to determine this. The cclc blog emphasises that if the client is not installed, you should hide the WeChat login button and offer other login options[[9]](https://cclc.github.io/mess-knowledge/wechat-ios-login.html#:~:text=%2F)[[10]](https://stackoverflow.com/questions/35718897/how-to-add-the-wechat-api-to-a-swift-project#:~:text=To%20check%20if%20WeChat%20app,phone%2C%20use%20the%20following%20code).
2. **Send an OAuth request:** When the user taps the WeChat login button, create a SendAuthReq object. Set the scope parameter to either snsapi\_userinfo (obtain user info) or snsapi\_base (silent login – returns only openid). Optionally set a custom state string. Then call WXApi.send to launch the WeChat app:

@IBAction func weChatLoginButtonTapped(\_ sender: UIButton) {
 guard WXApi.isWXAppInstalled() else {
 // Show alert that WeChat is not installed
 return
 }
 let req = SendAuthReq()
 req.scope = "snsapi\_userinfo" // Request user info
 req.state = "AppWeChatLogin"
 WXApi.send(req)
}

The Objective‑C example in the cclc blog calls isWXAppInstalled before creating a SendAuthReq and sending it via WXApi[[3]](https://cclc.github.io/mess-knowledge/wechat-ios-login.html#:~:text=%2F).

## Step 5 – Handle the authorization response (onResp)

When the user approves or denies the login in the WeChat app, WeChat returns to your app via the URL scheme or universal link. The WXApiDelegate method onResp(\_:) is triggered. Cast the response to SendAuthResp and extract the code property:

func onResp(\_ resp: BaseResp) {
 if let authResp = resp as? SendAuthResp {
 if authResp.errCode == WXSuccess.rawValue {
 let authCode = authResp.code
 // Exchange auth code for access token and openid
 requestAccessToken(with: authCode)
 } else {
 // The user cancelled or an error occurred
 print("WeChat login failed: code\(authResp.errCode)")
 }
 }
}

The cclc blog illustrates that onResp receives a SendAuthResp and constructs a URL like https://api.weixin.qq.com/sns/oauth2/access\_token?appid=APPID&secret=SECRET&code=CODE&grant\_type=authorization\_code to request the access token[[11]](https://cclc.github.io/mess-knowledge/wechat-ios-login.html#:~:text=,%E5%90%91%E5%BE%AE%E4%BF%A1%E8%AF%B7%E6%B1%82%E6%8E%88%E6%9D%83%E5%90%8E%2C%E5%BE%97%E5%88%B0%E5%93%8D%E5%BA%94%E7%BB%93%E6%9E%9C). The response contains access\_token, openid and refresh\_token[[12]](https://cclc.github.io/mess-knowledge/wechat-ios-login.html#:~:text=NSString%20%2AaccessUrlStr%20%3D%20%5BNSStringstringWithFormat%3A%40%22,code).

## Step 6 – Exchange the code for an access token and user info

### 6.1 Obtain the access token

Use an HTTP client (e.g., URLSession) to send a GET request to WeChat’s OAuth endpoint:

func requestAccessToken(with authCode: String) {
 let appId = "wxYOUR\_APP\_ID"
 let appSecret = "YOUR\_APP\_SECRET"
 let urlString = "https://api.weixin.qq.com/sns/oauth2/access\_token?appid=\(appId)&secret=\(appSecret)&code=\(authCode)&grant\_type=authorization\_code"
 guard let url = URL(string: urlString) else { return }

 let task = URLSession.shared.dataTask(with: url) { data, response, error in
 guard let data = data, error == nil else { return }
 if let dict = try? JSONSerialization.jsonObject(with: data) as? [String: Any] {
 if let accessToken = dict["access\_token"] as? String,
 let openId = dict["openid"] as? String,
 let refreshToken = dict["refresh\_token"] as? String {
 // Save tokens securely (e.g., Keychain)
 self.saveWeChatTokens(accessToken: accessToken, openId: openId, refreshToken: refreshToken)
 // Fetch user info
 self.requestUserInfo(accessToken: accessToken, openId: openId)
 }
 }
 }
 task.resume()
}

The blog’s Objective‑C code uses AFNetworking to request access\_token and parse access\_token, open\_id and refresh\_token, storing them in NSUserDefaults[[12]](https://cclc.github.io/mess-knowledge/wechat-ios-login.html#:~:text=NSString%20%2AaccessUrlStr%20%3D%20%5BNSStringstringWithFormat%3A%40%22,code).

### 6.2 Refresh the token when expired

The access\_token is valid for about 2 hours. You can refresh it using refresh\_token via https://api.weixin.qq.com/sns/oauth2/refresh\_token?appid=APPID&grant\_type=refresh\_token&refresh\_token=REFRESH\_TOKEN. The cclc example shows how to call the refresh endpoint and update stored tokens if the response contains access\_token[[13]](https://cclc.github.io/mess-knowledge/wechat-ios-login.html#:~:text=%2F%2F%20%E6%9B%B4%E6%96%B0access_token%E3%80%81refresh_token%E3%80%81open_id).

If both access\_token and refresh\_token are expired (WeChat uses a 30‑day validity for refresh tokens), you must ask the user to log in again.

### 6.3 Retrieve user profile

Once you have a valid access\_token and openid, request the user’s profile via https://api.weixin.qq.com/sns/userinfo?access\_token=ACCESS\_TOKEN&openid=OPENID. A successful response contains nickname, headimgurl, sex, and unionid. The cclc blog demonstrates requesting user data and parsing the JSON into a dictionary[[14]](https://cclc.github.io/mess-knowledge/wechat-ios-login.html#:~:text=,AFHTTPRequestOperationManagermanager). In Swift:

func requestUserInfo(accessToken: String, openId: String) {
 let urlString = "https://api.weixin.qq.com/sns/userinfo?access\_token=\(accessToken)&openid=\(openId)"
 guard let url = URL(string: urlString) else { return }
 let task = URLSession.shared.dataTask(with: url) { data, response, error in
 guard let data = data, error == nil else { return }
 if let userDict = try? JSONSerialization.jsonObject(with: data) as? [String: Any] {
 // Use userDict["unionid"], userDict["nickname"], userDict["headimgurl"], etc.
 print("WeChat user: \(userDict)")
 // Send user info to your backend for authentication/registration
 }
 }
 task.resume()
}

### 6.4 Persist tokens securely

Persist the access\_token, refresh\_token and openid in the Keychain rather than UserDefaults so that they survive app restarts and remain secure. When launching the app, check whether valid tokens exist; if they do, refresh or reuse them to avoid repeatedly prompting the user[[15]](https://cclc.github.io/mess-knowledge/wechat-ios-login.html#:~:text=%2F%2F%20%E6%9B%B4%E6%96%B0access_token%E3%80%81refresh_token%E3%80%81open_id).

## Step 7 – Server‑side considerations

1. **Verify tokens on your server:** Once you obtain user information, send the access\_token and openid to your backend. The server should call https://api.weixin.qq.com/sns/auth?access\_token=ACCESS\_TOKEN&openid=OPENID to verify that the token is still valid.
2. **Create or link user accounts:** Use the returned unionid as the unique identifier across multiple apps (if unionid is enabled for your account). If unionid is unavailable, openid is unique per application. Link these identifiers with existing user accounts or create new accounts.
3. **Handle token expiration:** When the access\_token expires, your server can use the refresh\_token to obtain a new access\_token without user interaction. If refreshing fails, ask the client to initiate login again.

## Step 8 – Comply with Apple’s guidelines

Apple’s App Store Review Guidelines require any app that uses a third‑party sign‑in service (such as WeChat login) as the primary method to also offer **Sign in with Apple**. Implement Sign in with Apple (or another equivalent first‑party login) so users have a choice. Additionally, provide a privacy policy explaining how you collect and use WeChat user data.

## Conclusion

Integrating WeChat login in an iOS Swift app involves several steps: registering the app with the WeChat Open Platform, integrating the Objective‑C SDK through a bridging header, configuring Info.plist, registering your app during launch, sending a SendAuthReq to obtain an authorization code, exchanging it for an access token and user information, and refreshing the token when it expires. Using WXApi.isWXAppInstalled() to ensure the WeChat client is available and adding ATS exceptions are important details[[3]](https://cclc.github.io/mess-knowledge/wechat-ios-login.html#:~:text=%2F)[[1]](https://cclc.github.io/mess-knowledge/wechat-ios-login.html#:~:text=,URLSchemes%EF%BC%8C%E6%B7%BB%E5%8A%A0%20ATS). Because it’s a social login, also include Sign in with Apple to comply with iOS policies. With careful implementation and server‑side verification, WeChat login can provide a seamless sign‑on experience for users in the Chinese ecosystem.

[[1]](https://cclc.github.io/mess-knowledge/wechat-ios-login.html#:~:text=,URLSchemes%EF%BC%8C%E6%B7%BB%E5%8A%A0%20ATS) [[3]](https://cclc.github.io/mess-knowledge/wechat-ios-login.html#:~:text=%2F) [[9]](https://cclc.github.io/mess-knowledge/wechat-ios-login.html#:~:text=%2F) [[11]](https://cclc.github.io/mess-knowledge/wechat-ios-login.html#:~:text=,%E5%90%91%E5%BE%AE%E4%BF%A1%E8%AF%B7%E6%B1%82%E6%8E%88%E6%9D%83%E5%90%8E%2C%E5%BE%97%E5%88%B0%E5%93%8D%E5%BA%94%E7%BB%93%E6%9E%9C) [[12]](https://cclc.github.io/mess-knowledge/wechat-ios-login.html#:~:text=NSString%20%2AaccessUrlStr%20%3D%20%5BNSStringstringWithFormat%3A%40%22,code) [[13]](https://cclc.github.io/mess-knowledge/wechat-ios-login.html#:~:text=%2F%2F%20%E6%9B%B4%E6%96%B0access_token%E3%80%81refresh_token%E3%80%81open_id) [[14]](https://cclc.github.io/mess-knowledge/wechat-ios-login.html#:~:text=,AFHTTPRequestOperationManagermanager) [[15]](https://cclc.github.io/mess-knowledge/wechat-ios-login.html#:~:text=%2F%2F%20%E6%9B%B4%E6%96%B0access_token%E3%80%81refresh_token%E3%80%81open_id) 〖iOS〗实现微信第三方登录 | cclc's backup

<https://cclc.github.io/mess-knowledge/wechat-ios-login.html>

[[2]](https://stackoverflow.com/questions/35718897/how-to-add-the-wechat-api-to-a-swift-project#:~:text=pod%20%27WechatOpenSDK%27) [[4]](https://stackoverflow.com/questions/35718897/how-to-add-the-wechat-api-to-a-swift-project#:~:text=Download%20the%20SDK%20from%20the,if%20you%20install%20using%20CocoaPods) [[5]](https://stackoverflow.com/questions/35718897/how-to-add-the-wechat-api-to-a-swift-project#:~:text=As%20,to%20add%20a%20bridging%20header) [[6]](https://stackoverflow.com/questions/35718897/how-to-add-the-wechat-api-to-a-swift-project#:~:text=) [[7]](https://stackoverflow.com/questions/35718897/how-to-add-the-wechat-api-to-a-swift-project#:~:text=func%20application,handleOpen%28url%2C%20delegate%3A%20self%29) [[8]](https://stackoverflow.com/questions/35718897/how-to-add-the-wechat-api-to-a-swift-project#:~:text=class%20AppDelegate%3A%20UIResponder%2C%20UIApplicationDelegate%2C%20WXApiDelegate,) [[10]](https://stackoverflow.com/questions/35718897/how-to-add-the-wechat-api-to-a-swift-project#:~:text=To%20check%20if%20WeChat%20app,phone%2C%20use%20the%20following%20code) ios - How to add the WeChat API to a Swift project? - Stack Overflow

<https://stackoverflow.com/questions/35718897/how-to-add-the-wechat-api-to-a-swift-project>
