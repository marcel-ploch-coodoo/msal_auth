# MSAL Auth

Microsoft Authentication 🔐 Library for Flutter.

`msal_auth` plugin provides Microsoft authentication in Android and iOS devices using native MSAL library. This is very straightforward and easy to use.

## Features 🚀

- Option to set one of the following Middleware:
  - MS Authenticator App
  - Browser
  - In-App WebView
- Supports authentication against Entra ID (formerly Azure Active Directory)/Microsoft Account tenants as well as Azure Active Directory (AD) B2C tenants
- Get auth token silently
- Get auth token interactive
- Logout
- Auth Token information
- Microsoft User information

---

To implement `MSAL` in `Flutter`, You need to setup an app in `Azure Portal` and required some of the platform specific configurations.

➡ Follow the step-by-step guide below ⬇️

## Create an App in Azure Portal

- First of all, You need to Sign up and create an app on [Azure Portal].
- To create the app, Search for `App registrations`, click on it and go to `New registration`.
- Fill the `Name` and select `Supported account types` and register it.
- Your application is created now and you should see the `Application (client) ID` and `Directory (tenant) ID`. Those values are required in Dart code.

  ![Azure Dashboard](/Screenshots/Azure-Dashboard.png)

- Now You need to add `Android` and `iOS` platform specific configuration in Azure portal. to do that, go to `Manage > Authentication > Add platform`.
  
### Android Setup - Azure portal

- For Android, You need to provide `package name` and release `signature hash`.
  - To generate a signature hash in Flutter, use the below command:
  
    ```Bash
    keytool -exportcert -alias androidreleasekey -keystore app/upload-keystore.jks | openssl sha1 -binary | openssl base64
    ```

  - Make sure you have release `keystore` file placed inside `/app` folder.
  - Only one signature hash is required because it maps with `AndroidManifest.xml`.
  
### iOS Setup - Azure portal

- For iOS, You need to provide only `Bundle ID`.

  ![iOS Redirect URI](/Screenshots/iOS-Redirect-URI.png)

That's it for the Azure portal configuration.

---

Please follow the platform configuration ⬇️ before jump to the `Dart` code.

## Android Configuration

- This plugin supports fully customization as you can give configuration `JSON` that will be used in authentication.
- Follow the below steps to complete Android configuration.

### Creating MSAL Config JSON

- Create one `msal_config.json` in `/assets` folder and copy the JSON from [Microsoft default configuration file].
- Now add the `redirect_uri` in the above created JSON as below:

  ```JSON
  "redirect_uri": "msauth://<APP_PACKAGE_NAME>/<BASE64_ENCODED_PACKAGE_SIGNATURE>",
  ```

- You can directly copy the `Redirect URI` from Azure portal.

  ![Android Redirect URI](/Screenshots/Azure-Android-Redirect-URI.png)

### Setup authentication middleware (Optional)

- Set broker authentication (authenticate user by [Microsoft Authenticator App])

  ```JSON
  "broker_redirect_uri_registered": true
  ```

  - If Authenticator app is not installed on a device, `authorization_user_agent` will be used as a auth middleware.

- Authenticate using Browser

  ```JSON
  "broker_redirect_uri_registered": false,
  "authorization_user_agent": "BROWSER"
  ```

- Authenticate using WebView

  ```JSON
  "broker_redirect_uri_registered": false,
  "authorization_user_agent": "WEBVIEW"
  ```

- To learn more about configuring JSON, follow [Android MSAL configuration].

### Add Activity in AndroidManifest.xml

- Add another activity inside `<application>` tag.
- This is only needed if you want to use `Browser` as a auth middleware.

  ```XML
  <activity android:name="com.microsoft.identity.client.BrowserTabActivity">
      <intent-filter>
          <action android:name="android.intent.action.VIEW" />

          <category android:name="android.intent.category.DEFAULT" />
          <category android:name="android.intent.category.BROWSABLE" />

          <data
              android:host="com.example.msal_auth_example"
              android:path="/<BASE64_ENCODED_PACKAGE_SIGNATURE>"
              android:scheme="msauth" />
      </intent-filter>
  </activity>
  ```
- Replace `host` by your app's package name and `path` by the `base64` signature hash that is generated above.

## iOS Configuration

- Add a new keychain group to your project capabilities. The keychain group should be `com.microsoft.adalcache`. [Configuring your project to use MSAL]

- If you need to application's redirect URI scheme & `LSApplicationQueriesSchemes` to allow making calls to [Microsoft Authenticator] if installed, the below modifications will also be made to your `Info.plist` file. 
  
### `Info.plist` Modification

```Plist
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLSchemes</key>
    <array>
    	<string>msauth.$(PRODUCT_BUNDLE_IDENTIFIER)</string>
    </array>
  </dict>
</array>

<key>LSApplicationQueriesSchemes</key>
<array>
  <string>msauthv2</string>
  <string>msauthv3</string>
</array>
```

### `ios/Runner/AppDelegate.swift` file changes

- To get a correct callback behvior and no endless Loading on iOS the content of the AppDelegate.swift should be extended.

```AppDelegate.swift
import Flutter
import UIKit
import MSAL

@main
@objc class AppDelegate: FlutterAppDelegate {
  override func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
  ) -> Bool {
    GeneratedPluginRegistrant.register(with: self)
    return super.application(application, didFinishLaunchingWithOptions: launchOptions)
  }

  override func application(_ app: UIApplication, open url: URL, options: [UIApplication.OpenURLOptionsKey : Any] = [:]) -> Bool {
    return MSALPublicClientApplication.handleMSALResponse(url, sourceApplication: options[UIApplication.OpenURLOptionsKey.sourceApplication] as? String)
  }
}
```

## Code Implementation 👨‍💻

- This section contains writing `Dart` code to setup a `MSAL` application in `Flutter` and get auth token.

### Setup MSAL Application

```Dart
final msalAuth = await MsalAuth.createPublicClientApplication(
  clientId: '<MICROSOFT_CLIENT_ID>',
  scopes: <String>[
    'https://graph.microsoft.com/user.read',
    // Add other scopes here if required.
  ],
  loginHint: '<EMAIL ID (Optional)>'
  androidConfig: AndroidConfig(
    configFilePath: 'assets/msal_config.json',
    tenantId: '<MICROSOFT_TENANT_ID (Optional)>',
  ),
  iosConfig: IosConfig(
    authority: 'https://login.microsoftonline.com/<MICROSOFT_TENANT_ID>/oauth2/v2.0/authorize',
    // Change auth middleware if you need.
    authMiddleware: AuthMiddleware.msAuthenticator,
    tenantType: TenantType.entraIDAndMicrosoftAccount,
  ),
);
```
- To modify value of `authority` in `iOS`, follow [Configure iOS authority].

- in `iOS`, if middleware is `AuthMiddleware.msAuthenticator` and `Authenticator` app is not installed on a device, It will use `Safari Browser` for authentication.

- By default login will be attempted against an Entra ID (formerly Azure Active Directory) tenant which also supports Microsoft Account logins. To login to an Azure AD B2C tenant instead, set the `tenantType` value to `TenantType.azureADB2C`.

### Get Auth Token (Login to Microsoft account)

- This code is responsible to open Microsoft login page in given middleware and provide token on successful login.

```Dart
final user = await msalAuth.acquireToken();
log('User data: ${user?.toJson()}');
```

### Get Auth Token by Silent Call 🔇 (When expired)

- Before using auth token, You must check for the token expiry time. You can do it by accessing `tokenExpiresOn` property from `MsalUser` object.

```Dart
if (msalUser.tokenExpiresOn <= DateTime.now().millisecondsSinceEpoch) {
  final user = await msalAuth.acquireTokenSilent();
  log('User data: ${user?.toJson()}');
}
```

- This will generate a new token without opening Microsoft login page. However, this method can open the login page if `MSALUiRequiredException` occurs.
- You can learn more about [MSAL exceptions].

---

Follow [example] code for more details on implementation.


[Azure Portal]: https://portal.azure.com/
[Microsoft default configuration file]: https://learn.microsoft.com/en-in/entra/identity-platform/msal-configuration#the-default-msal-configuration-file
[Microsoft Authenticator App]: https://play.google.com/store/apps/details?id=com.azure.authenticator
[Android MSAL configuration]: https://learn.microsoft.com/en-in/entra/identity-platform/msal-configuration
[Microsoft Authenticator]: https://apps.apple.com/us/app/microsoft-authenticator/id983156458
[Configure iOS authority]: https://learn.microsoft.com/en-us/entra/msal/objc/configure-authority#change-the-default-authority
[MSAL exceptions]: https://learn.microsoft.com/en-us/entra/msal/dotnet/advanced/exceptions/msal-error-handling
[example]: https://pub.dev/packages/msal_auth/example
[Configuring your project to use MSAL]: https://learn.microsoft.com/en-us/entra/msal/objc/install-and-configure-msal#configuring-your-project-to-use-msal
