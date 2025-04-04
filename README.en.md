[![en](https://img.shields.io/badge/lang-en-red.svg)](https://github.com/PayME-Miniapp/payme-mini-app/blob/main/README.en.md)
[![vi](https://img.shields.io/badge/lang-vi-green.svg)](https://github.com/PayME-Miniapp/payme-mini-app/blob/main/README.md)

# payme-mini-app

[Installation](#installation)

- [Install Library](#install-library)
- [Android](#android)
- [iOS](#ios)

[Usage](#usage)

# Installation

## Install Library

npm:

```javascript
npm install payme-mini-app
```

yarn:

```javascript
yarn add payme-mini-app
```

## Android

**Add maven jitpack.io**

Update the project's build.gradle file

```kotlin
allprojects {
  repositories {
    ...
    maven {
        url "https://jitpack.io"
    }
 }
}
```

## Android

⚠️ Miniapp supports Android version ≥ 26 and targetSdk level ≥ 33 due to the NFC feature.

⚠️ If you declared minifyEnabled = false. You should also define this in your proguard-rules.pro:

```
-keep class vn.kalapa.ekyc.**{*;}
-keep class com.fis.ekyc.**{*;}
-keep class com.fis.nfc.**{*;}
-keep class retrofit2.**{*;}

-keepclassmembers,allowobfuscation class * {
    @com.google.gson.annotations.SerializedName <fields>;
}
```

## iOS

⚠️ Miniapp supports iOS 13+.

Add this line to Podfile:

```swift
use_frameworks! :linkage => :static
```

Add to target:

```swift
$dynamic_framework = ['PayMEMiniApp', 'CryptoSwift', 'SwiftyRSA', 'GCDWebServer', 'NSLogger', 'lottie-ios', 'SwiftyJSON', 'ZIPFoundation', 'Mixpanel-swift']

post_install do |installer|
  installer.pods_project.targets.each do |target|
    target.build_configurations.each do |config|
      if $dynamic_framework.include?(target.name)
        config.build_settings['BUILD_LIBRARY_FOR_DISTRIBUTION'] = 'YES'
        config.build_settings['IPHONEOS_DEPLOYMENT_TARGET'] = '13.0'
      end
    end
  end
end

pre_install do |installer|
  installer.pod_targets.each do |pod|
    if $dynamic_framework.include?(pod.name)
      def pod.build_type;
       Pod::BuildType.dynamic_framework
      end
    end
  end
end
```

## Set up the application to be compatible with PayMEMiniApp

### Info.plist

Update the Info.plist with the keys below to ensure PayMEMiniApp can function properly

⚠️ From version 0.9.0, NFC permission is required to supports NFC scanning.

```swift
Queried URL Schemes
Privacy - Camera Usage Description
Privacy - Photo Library Usage Description
Privacy - Photo Library Additions Usage Description
Privacy - Contacts Usage Description
Privacy - NFC Scan Usage Description
ISO7816 application identifiers for NFC Tag Reader Session
```

Raw Keys version:

```swift
LSApplicationQueriesSchemes
NSCameraUsageDescription
NSPhotoLibraryUsageDescription
NSPhotoLibraryAddUsageDescription
NSContactsUsageDescription
NFCReaderUsageDescription
com.apple.developer.nfc.readersession.iso7816.select-identifiers
```

Explanation:

```text
- LSApplicationQueriesSchemes: Declare URL schemes for depositing money via deeplink of VCB bank
- NSCameraUsageDescription: Permission to take photos when using the KYC feature.
- NSPhotoLibraryUsageDescription: Permission to access photos in the library when using the QR Code download feature.
- NSPhotoLibraryAddUsageDescription: Permission to access photos in the library when using the QR Code download feature.
- NSContactsUsageDescription: Permission to access contacts when using the feature to top up phone credit for subscribers in the contacts list.
- NFCReaderUsageDescription: Permission to access the device's NFC when using the eKYC feature via NFC
- com.apple.developer.nfc.readersession.iso7816.select-identifiers: ISO7816 application identifiers for NFC Tag Reader Session
```

Example Info.plist:

```
<key>LSApplicationQueriesSchemes</key>
<array>
  <string>vcbpartner</string>
  <string>bidv.smartbanking.partner</string>
</array>
<key>NSCameraUsageDescription</key>
<string>Chúng tôi cần dùng máy ảnh để sử dụng cho việc định danh và đọc mã vạch thanh toán</string>
<key>NSContactsUsageDescription</key>
<string>Cho phép truy cập Danh bạ để chọn nhanh số điện thoại của bạn bè</string>
<key>NSPhotoLibraryAddUsageDescription</key>
<string>Chúng tôi cần sử dụng thư viện ảnh của bạn để hiển thị ảnh trong ứng dụng hoặc chọn mã vạch thanh toán</string>
<key>NSPhotoLibraryUsageDescription</key>
<string>Chúng tôi cần sử dụng thư viện ảnh của bạn để hiển thị ảnh trong ứng dụng hoặc chọn mã vạch thanh toán</string>
<key>NFCReaderUsageDescription</key>
<string>eKYC cần sử dụng NFC</string>
<key>com.apple.developer.nfc.readersession.iso7816.select-identifiers</key>
<array>
    <string>A0000002471001</string>
    <string>A0000002472001</string>
    <string>00000000000000</string>
</array>
```

### Add Capabilities

In XCode, select your app in Targets -> Signing & Capabilities -> Click on the "+" in the upper right corner to open the window to add capability for the app

⚠️ From version 0.9.0, add the Near Field Communication Tag Reading capability to support NFC scanning.

Find and select "Near Field Communication Tag Reading".

![img_1.png](documents/capabilities2.png)

Find and select "Background Modes", turn on the option "Background Fetch"

![img_1.png](documents/capabilities1.png)

![img.png](documents/background_fetch.png)

### Set PayMEMiniApp Version Update

Add the following line to AppDelegate:

- Objective-C:

```objective-c
- (void)application:(UIApplication *)application handleEventsForBackgroundURLSession:(NSString *)identifier completionHandler:(void (^)(void))completionHandler { completionHandler(); }
```

- Swift:

```swift
import UIKit
import PayMEMiniApp

@main
class AppDelegate: UIResponder, UIApplicationDelegate {
...
func application(_ application: UIApplication, handleEventsForBackgroundURLSession identifier: String, completionHandler: @escaping () -> Void) {
   completionHandler()
}
...
}
```

# Usage

## Initialize PayMEMiniApp

### Set up the key set from PayME

The key set includes: appId, publicKey, privateKey. Contact PayME for guidance.

### Initialize Miniapp

<strong>_Initializing this will also start checking and downloading the PayMEMiniApp version, so it is recommended to initialize it as early as possible when running the app._<strong>

| **Parameter**  | **Required** | **Data Type**                         |
| -------------- | ------------ | ------------------------------------- |
| **appId**      | Yes          | String                                |
| **publicKey**  | Yes          | String                                |
| **privateKey** | Yes          | String                                |
| **locale**     | No           | String (from version 0.5.3 and above) |
| **env**        | No           | String                                |
| **mode**       | No           | String                                |

Note:

- appId: Each partner integrating PayME Miniapp will be provided with a unique appId. (Note: The value of the appId is taken from the x-api-client field on the dashboard)
- publicKey, privateKey: a key pair generated when registering as a partner with PayME.
- locale: Initialization language for PayMEMiniApp (vi, en).
- env: Initialization environment for PayMEMiniApp (PRODUCTION, SANDBOX).
- mode: PayMEMiniApp usage mode (miniapp_sandbox, miniapp_product).

```javascript
import { init } from "payme-mini-app";

init(appId, publicKey, privateKey, locale, env, mode);
```

⚠️ Note: delete "\n" and newline in the publicKey + privateKey pair. See the example code below for details

Example:

```javascript
init(
    appId: "559163930378",
    publicKey: """
    -----BEGIN PUBLIC KEY-----
    MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBAK0RONYVPYn/3IWloU83Qy16hKNHhlCx
    gTJay6/rERk8tmsMbILLzTYW7H9WOqN2gS0s0ymD+3TxP+q+MxEp0qECAwEAAQ==
    -----END PUBLIC KEY-----
    """,
    privateKey: """
    -----BEGIN RSA PRIVATE KEY-----
    MIIBOgIBAAJBAMXIuvTT8Z5U/AqyFvBbDApQ2STLm9Ca2nmu2pxqwhrhN+80mOLb
    MzbQDRCNpro6S61d34A7cEIX/5gxxrAaVAkCAwEAAQJAfzB70e/uJHTgdHxcNgtG
    7edaDMiHFhpPPwtL+GTLGH70yhFDs2eIXFHLY/wfRRcxzwGyGOyvXlGbDjsMFdpn
    lQIhAPIoUVsADDfI4KNZEKHaJRVAmz2D0xdiB6R716HA7A0XAiEA0RcPxHzYLhVp
    +adoGpJBq7e87BzQrVBJQFSOg8Kim98CIQCYmynyFEye1zwiFR3zMfuOsiFjGfFs
    2f2A/f69VEwuTwIgFN/3jAdm0dsDdJBZHWYCtnEmpHAQCW2dkpWekNsKvwMCIGXm
    rg+mppNNZQx6+6Swsp8L8Hgc+HikKy02Okijjw0W
    -----END RSA PRIVATE KEY-----
    """,
    locale: "vi",
    env: "SANDBOX",
    mode: "miniapp_sandbox"
)
```

### Set up listeners:

Use this function to set up listeners for onResponse or onError events thrown during Miniapp operations

```javascript
import { PayMEMiniAppEvents } from "payme-mini-app";

PayMEMiniAppEvents.addListener("onResponse", listener);
PayMEMiniAppEvents.addListener("onError", listener);
```

| **Parameter**  | **Required** | **Data Type**                            |
| -------------- | ------------ | ---------------------------------------- |
| **onResponse** | No           | (ActionOpenMiniApp, JSONObject?) -> Unit |
| **onError**    | No           | (ActionOpenMiniApp, PayMEError?) -> Unit |

Note:

- onResponse: The onResponse event is triggered when a Miniapp action is completed (e.g., successful payment). This event is triggered with the action that caused the event and a JSONObject containing additional data.
- onError: The onError event is triggered when an error occurs during Miniapp operation. This event is triggered with the action being performed and a PayMEError containing additional information about the error.

Details of data types

**ActionOpenMiniApp:** (Miniapp action)

| **Value**       | **Explanation**                                                                                                                                                   |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **OPEN**        | If the PayME wallet account has not been activated, open the activation interface; if it has already been activated, open the home interface of the PayME wallet. |
| **PAY**         | Open the payment interface for an order                                                                                                                           |
| **GET_BALANCE** | Get the PayME wallet balance                                                                                                                                      |
| **SERVICE**     | Open the service payment interface                                                                                                                                |
| **DEPOSIT**     | Open the deposit interface                                                                                                                                        |
| **WITHDRAW**    | Open the withdrawal interface                                                                                                                                     |
| **TRANSFER**    | Open the money transfer interface                                                                                                                                 |
| **KYC**         | Open the KYC interface                                                                                                                                            |

**PayMEError:** (Error during Miniapp operation)
| **Attribute** | **Data Type** | **Explanation** |
|-------------------|-----------------------------------------|-------------------------------------------------------------------------------|
| **type** | enum "MiniApp", "UserCancel", "Network" | Error group: Miniapp error, user closes Miniapp or network connection error |  
| **code** | String | Error code |
| **description** | String | Error description |

### Function openMiniApp:

Partners use this function to open the PayME Miniapp interface after initialization

```javascript
import { open } from 'payme-mini-app'

open(
  openType: OpenMiniAppType = OpenMiniAppType.screen,
  openMiniAppData: OpenMiniAppDataInterface
)
```

| **Parameter**       | **Required** | **Data Type**            | **Explanation**                                                               |
| ------------------- | ------------ | ------------------------ | ----------------------------------------------------------------------------- |
| **openType**        | Yes          | enum "screen", "modal"   | Open the Miniapp in full-screen or modal interface sliding from the bottom up |
| **openMiniAppData** | Yes          | OpenMiniAppDataInterface | Additional information depending on the type of action                        |

Details of OpenMiniAppData:

**OPEN:** Partners use this action when they want to open the PayME wallet home page to use PayME's utility services. If the PayME account is not activated, activate it; if already activated, open the PayME wallet home page

| **Attribute** | **Required** | **Data Type** | **Explanation**             |
| ------------- | ------------ | ------------- | --------------------------- |
| **phone**     | Yes          | String        | Phone number of the account |

Example:

```javascript
open("screen", {
  action: "OPEN",
  phone: "0123456789",
});
```

**PAY:** Partners use this action when they want to open the payment interface of the Miniapp
| **Attribute** | **Required** | **Data Type** | **Explanation** |
|---------------------|--------------|------------------|--------------------------------------------|
| **phone** | Yes | String | Phone number of the account |
| **data** | Yes | PaymentData | Additional information for payment |

Details of PaymentData:
| **Attribute** | **Required** | **Data Type** | **Explanation** |
|---------------------|--------------|------------------|--------------------------------------------------------------------------------------|
| **transactionId** | Yes | String | Transaction ID |  
| **amount** | Yes | Int | Total transaction amount |  
| **note** | No | String | Transaction note |
| **ipnUrl** | No | String | Path for PayME server ipn to check when the transaction progresses (success/failure) |
| **extraData** | No | Object | Additional data specified by the partner, this data will be returned to the partner upon successful payment. (from version 0.6.6 and above) |
| **isShowResult** | No | Boolean | Does PayME display the result screen? (Default: true) |

Example:

```javascript
open("screen", {
  action: "PAY",
  phone: "0123456789",
  data: {
    amount: 10000,
    note: "Thanh toán cho đơn hàng abc",
    transactionId: "123",
    ipnUrl: "www.google.com",
  },
});
```

**PAYMENT:** Partners use this action when they want to open the payment interface of the Miniapp

- The steps of retrieving a list of methods and generating transaction code will be carried out in the partner's application. (For details, please contact PayME)

  - Utilize API connection to fetch the list of methods.
  - Then select a method, call the transaction creation API to retrieve the transaction.

| **Attribute** | **Required** | **Data Type**     | **Explanation**                              |
| ------------- | ------------ | ----------------- | -------------------------------------------- |
| **phone**     | No           | String            | Phone number of the account                  |
| **data**      | Yes          | PaymentDirectData | Additional information to facilitate payment |

Details of PaymentDirectData:
| **Attribute** | **Required** | **Data Type** | **Explanation** |
|---------------------|--------------|------------------|--------------------------------------------------------------------------------------|
| **transaction** | Yes | String | Transaction code |
| **extraData** | No | Object | Additional data specified by the partner, this data will be returned to the partner upon successful payment. (from version 0.6.6 and above) |
| **isShowResult** | No | Boolean | Does PayME display the result screen? (Default: true) |

Example:

```javascript
open("screen", {
  action: "PAYMENT",
  data: {
    transaction: "123456",
  },
});
```

**TRANSFER_QR:** Partners use this action when they want to open the payment interface of the Miniapp (from version 0.6.2 and above)

- This action is used to transfer money from PayME wallet to the destination account provided by the partner.

| **Attribute** | **Required** | **Data Type**  | **Explanation**                              |
| ------------- | ------------ | -------------- | -------------------------------------------- |
| **data**      | Yes          | TransferQRData | Additional information to facilitate payment |

Details of PaymentDirectData:
| **Attribute** | **Required** | **Data Type** | **Explanation** |
|---------------------|--------------|------------------|--------------------------------------------------------------------------------------|
| **amount** | Yes | Int | Transaction amount (if no amount is provided, pass 0) |
| **bankNumber** | Yes | String | Bank account number |
| **swiftCode** | Yes | String | Bank's swiftCode |
| **cardHolder** | Yes | String | Cardholder's name |
| **partnerTransaction** | No | String | Partner's Transaction (from version 0.6.8 and above) |
| **note** | No | String | Transaction note |
| **extraData** | No | Object | Additional data specified by the partner, this data will be returned to the partner upon successful payment. (from version 0.6.6 and above) |
| **isShowResult** | No | Boolean | CShould PayME's result screen be displayed? (Default: true) |

Example:

```javascript
open("screen", {
  action: "TRANSFER_QR",
  data: {
    amount: 20000,
    bankNumber: "9704000000000018",
    swiftCode: "SBITVNVX",
    cardHolder: "Nguyen Van A",
    note: "Test",
  },
});
```

**DEPOSIT:** Partners use this action when they want to open the interface to deposit money into the PayME wallet
| **Attribute** | **Required** | **Data Type** | **Explanation** |
|---------------------|--------------|--------------------------------------|----------------------------------|
| **phone** | Yes | String | Phone number of the account |
| **additionalData** | Yes | DepositWithdrawTransferData | Additional information for deposit |

Example:

```javascript
open("screen", {
  action: "DEPOSIT",
  phone: "0123456789",
  data: {
    amount: 10000,
  },
});
```

**WITHDRAW:** Partners use this action when they want to open the interface to withdraw money from the PayME wallet
| **Attribute** | **Required** | **Data Type** | **Explanation** |
|---------------------|--------------|--------------------------------------|----------------------------------|
| **phone** | Yes | String | Phone number of the account |
| **additionalData** | Yes | DepositWithdrawTransferData | Additional information for withdraw |

Example:

```javascript
open("screen", {
  action: "WITHDRAW",
  phone: "0123456789",
  data: {
    amount: 10000,
  },
});
```

**TRANSFER:** Partners use this action when they want to open the money transfer interface
| **Attribute** | **Required** | **Data Type** | **Explanation** |
|---------------------|--------------|--------------------------------------|----------------------------------|
| **phone** | Yes | String | Phone number of the account |
| **additionalData** | Yes | DepositWithdrawTransferData | Additional information for transfer |

Example:

```javascript
open("screen", {
  action: "TRANSFER",
  phone: "0123456789",
  data: {
    amount: 10000,
  },
});
```

Details of DepositWithdrawTransferData:
| **Attribute** | **Required** | **Data Type** | **Explanation** |
|---------------------|--------------|------------------|-------------------------------------------------------|
| **description** | No | String | Transaction description |  
| **amount** | No | Int | Total transaction amount |
| **extraData** | No | Object | Additional data specified by the partner, this data will be returned to the partner upon successful payment. (from version 0.6.6 and above) |
| **isShowResult** | No | Boolean | Does PayME display the result screen? (Default: true) |

**KYC:** Partners use this action when they want to open the KYC interface
| **Attribute** | **Required** | **Data Type** | **Explanation** |
|---------------------|--------------|--------------------------------------|----------------------------------|
| **phone** | Yes | String | Phone number of the account |

Example:

```javascript
open("screen", {
  action: "KYC",
  phone: "0123456789",
});
```

**SERVICE:** Partners use this action when they want to open the service payment interface
| **Attribute** | **Required** | **Data Type** | **Explanation** |
|--------------------|--------------|---------------|--------------------------------|
| **phone** | Yes | String | Phone number of the account |
| **additionalData** | Yes | ServiceData | Additional information for service |

Details of ServiceData:
| **Attribute** | **Required** | **Data Type** | **Explanation** |
|---------------|--------------|---------------|-----------------|
| **code** | No | String | Service code |
| **extraData** | No | Object | Additional data specified by the partner, this data will be returned to the partner upon successful payment. (from version 0.6.6 and above) |
| **isShowResult** | No | Boolean | Does PayME display the result screen? (Default: true) |

List of service codes:

| **Value**        | **Explanation** |
| ---------------- | --------------- |
| **POWE**         | Electricity     |
| **WATE**         | Water           |
| **ADSL**         | Internet        |
| **TIVI**         | Television      |
| **PPMB**         | Postpaid phone  |
| **MOBILE_CARD**  | Phone card      |
| **MOBILE_TOPUP** | Top up mobile   |
| **GAME_CARD**    | Game card       |

Example:

```javascript
open("screen", {
  action: "SERVICE",
  phone: "0123456789",
  data: {
    code: "POWE",
  },
});
```

### Function getBalance

Partners use this function to retrieve the balance information of a PayME account. The result will be returned in the onResponse event with the action GET_BALANCE.

```javascript
import { getBalance } from 'payme-mini-app'

getBalance(phone: string)
```

| **Attribute** | **Required** | **Data Type** | **Explanation**                                                   |
| ------------- | ------------ | ------------- | ----------------------------------------------------------------- |
| **phone**     | Yes          | String        | The phone number of the account for which to retrieve the balance |

### Function getAccountInfo

Partners use this function to retrieve information about a PayME account. The result will be returned in the onResponse event with the action GET_ACCOUNT_INFO.

```javascript
import { getAccountInfo } from 'payme-mini-app'

getAccountInfo(phone: String)
```

| **Attribute** | **Required** | **Data Type** | **Explanation**                                                   |
| ------------- | ------------ | ------------- | ----------------------------------------------------------------- |
| **phone**     | Yes          | String        | The phone number of the account for which to retrieve the balance |

### Function setLanguage (from version 0.5.3 and above)

Partners use this function to switch language.

```javascript
import { setLanguage } from 'payme-mini-app'

setLanguage(language: String)
```

| **Attribute** | **Required** | **Data Type** | **Explanation**                     |
| ------------- | ------------ | ------------- | ----------------------------------- |
| **language**  | Yes          | String        | The language to switch to (vi, en). |

### Function close (from version 0.7.0 and above)

Partners use this function to close the mini-app.

```javascript
import { close } from "payme-mini-app";

close();
```
