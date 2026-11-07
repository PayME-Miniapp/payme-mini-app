[Tiếng Việt](README.md) | **English**

# payme-mini-app

React Native library for embedding PayME Miniapp (e-wallet, payments, top-up, withdrawal, transfers, eKYC) in a partner app on iOS and Android.

| Package | Version |
| --- | --- |
| npm `payme-mini-app` | **3.0.0** (breaking change from 2.x) |
| iOS pod `PayME-Miniapp` | 0.8.1 |
| Android `com.github.PayME-Miniapp:native-android` | 0.8.2 (SDK 1.0.24) |

Both native SDKs are declared by the library; your app does not add them directly.

## Table of contents

1. [Requirements](#requirements)
2. [Installation](#installation)
3. [App configuration](#app-configuration)
4. [Quick start](#quick-start)
5. [identityToken](#identitytoken)
6. [API reference](#api-reference)
7. [Error codes](#error-codes)
8. [Upgrading from 2.x](#upgrading-from-2x)
9. [Version history and support](#version-history-and-support)

## Requirements

| Item | Requirement |
| --- | --- |
| React Native | 0.71 or later. The example app is tested with React Native 0.85 |
| Architecture | The library is a bridge (legacy) native module. It also runs with the New Architecture enabled, through React Native's interop layer (the example app has it enabled) |
| iOS | iOS 13.0 or later (or your React Native version's minimum, if higher). **Xcode 26.2 or later** |
| Android | `minSdkVersion` 26, `compileSdkVersion` 37 |
| Keys | `appId`, `publicKey`, `privateKey`, issued by PayME for each app |
| Partner server | Able to call PayME's `identityToken` API (see [identityToken](#identitytoken)) |

## Installation

```sh
npm install payme-mini-app@^3.0.0
# or
yarn add payme-mini-app@^3.0.0
```

### iOS

The library pulls in the `PayME-Miniapp` 0.8.1 pod. Add the following to `ios/Podfile` (same as the example app):

```ruby
use_frameworks! :linkage => :static

target 'YourApp' do
  # ... use_react_native!(...) as generated

  $dynamic_framework = [
    'PayMEMiniApp', 'CryptoSwift', 'SwiftyRSA', 'GCDWebServer', 'NSLogger', 'lottie-ios',
    'SwiftyJSON', 'ZIPFoundation', 'Mixpanel-swift', 'jsonlogic', 'json-enum', 'MixpanelSwiftCommon'
  ]

  pre_install do |installer|
    installer.pod_targets.each do |pod|
      if $dynamic_framework.include?(pod.name)
        def pod.build_type
          Pod::BuildType.dynamic_framework
        end
      end
    end
  end

  post_install do |installer|
    # react_native_post_install(...) as generated
    installer.pods_project.targets.each do |target|
      target.build_configurations.each do |config|
        if $dynamic_framework.include?(target.name)
          config.build_settings['BUILD_LIBRARY_FOR_DISTRIBUTION'] = 'YES'
          config.build_settings['IPHONEOS_DEPLOYMENT_TARGET'] = '13.0'
        end
      end
    end
  end
end
```

Then run:

```sh
cd ios && pod install
```

### Android

**1. JitPack.** The Android SDK is published on JitPack. The React Native Gradle Plugin adds `https://www.jitpack.io` to every project unless you turn it off with `includeJitpackRepository=false`. If you turned it off, or your project declares repositories in `settings.gradle` with `RepositoriesMode.FAIL_ON_PROJECT_REPOS`, add it yourself:

```groovy
// android/build.gradle
allprojects {
  repositories {
    maven { url 'https://www.jitpack.io' }
  }
}
```

**2. SDK levels.** The Android SDK requires `minSdkVersion` 26 and `compileSdkVersion` 37 (the React Native 0.85 template defaults to 24 and 36):

```groovy
// android/build.gradle
buildscript {
  ext {
    minSdkVersion = 26
    compileSdkVersion = 37
    targetSdkVersion = 36
  }
}
```

If your Android Gradle Plugin reports that `compileSdk` 37 is not supported, upgrade AGP (the example app uses AGP 9.1.1).

**3. R8/ProGuard.** If your release build sets `minifyEnabled true`, add to `android/app/proguard-rules.pro`:

```proguard
-keep class vn.kalapa.ekyc.** { *; }
-keepclassmembers,allowobfuscation class * {
    @com.google.gson.annotations.SerializedName <fields>;
}
```

## App configuration

### iOS: Info.plist

| Key | Purpose |
| --- | --- |
| `LSApplicationQueriesSchemes` | Open banking apps for top-up via deeplink |
| `NSCameraUsageDescription` | Capture ID documents and face for eKYC; scan QR codes |
| `NSPhotoLibraryUsageDescription` | Pick a QR code image from the library |
| `NSPhotoLibraryAddUsageDescription` | Save a QR code image to the library |
| `NSContactsUsageDescription` | Pick a phone number from contacts (mobile top-up) |
| `NSFaceIDUsageDescription` | Authenticate with Face ID |
| `NFCReaderUsageDescription` | Read the chip of the Vietnamese citizen ID card over NFC for eKYC |
| `com.apple.developer.nfc.readersession.iso7816.select-identifiers` | ISO 7816 application IDs of the ID card chip |
| `NSAppTransportSecurity` → `NSAllowsLocalNetworking` | The Miniapp runs on a local web server (`http://localhost`). The React Native template already enables it. `NSAllowsArbitraryLoads` is not needed |

```xml
<key>LSApplicationQueriesSchemes</key>
<array>
  <string>vcbpartner</string>
  <string>bidv.smartbanking.partner</string>
</array>
<key>NSCameraUsageDescription</key>
<string>The camera is used for identity verification and scanning payment codes</string>
<key>NSPhotoLibraryUsageDescription</key>
<string>Photo library access is used to pick payment codes</string>
<key>NSPhotoLibraryAddUsageDescription</key>
<string>Photo library access is used to save payment codes</string>
<key>NSContactsUsageDescription</key>
<string>Contacts access lets you pick a phone number quickly</string>
<key>NSFaceIDUsageDescription</key>
<string>Face ID is used for fast and secure authentication</string>
<key>NFCReaderUsageDescription</key>
<string>NFC is used to read your ID card chip for identity verification</string>
<key>com.apple.developer.nfc.readersession.iso7816.select-identifiers</key>
<array>
  <string>A0000002471001</string>
  <string>A0000002472001</string>
  <string>00000000000000</string>
</array>
<key>NSAppTransportSecurity</key>
<dict>
  <key>NSAllowsArbitraryLoads</key>
  <false/>
  <key>NSAllowsLocalNetworking</key>
  <true/>
</dict>
```

### iOS: Capabilities

In Xcode, select your app target → **Signing & Capabilities** → **+ Capability**.

1. Add **Near Field Communication Tag Reading** (to read the ID card chip during eKYC).

   ![Near Field Communication Tag Reading](documents/capabilities2.png)

2. Add **Background Modes** and enable **Background fetch** (PayME's reference configuration).

   ![Background Modes](documents/capabilities1.png)

   ![Background fetch](documents/background_fetch.png)

### Android: permissions

Nothing to declare. The SDK manifest contains the permissions below and Gradle merges them into your app: `INTERNET`, `ACCESS_NETWORK_STATE`, `ACCESS_WIFI_STATE`, `CAMERA`, `READ_CONTACTS`, `NFC`, `READ_EXTERNAL_STORAGE`, `WRITE_EXTERNAL_STORAGE`, `ACCESS_LOCAL_NETWORK`. The Miniapp requests runtime permissions (camera, contacts) when it needs them.

## Quick start

```tsx
import { useEffect } from 'react';
import { init, open, PayMEMiniAppEvents } from 'payme-mini-app';

export function usePayME() {
  useEffect(() => {
    // Call once, after the first screen has mounted (Android needs the current Activity).
    init('<APP_ID>', '<PUBLIC_KEY>', '<PRIVATE_KEY>', 'vi', 'SANDBOX', 'miniapp_sandbox');
    const subs = [
      PayMEMiniAppEvents.addListener('onResponse', (e) => console.log('PayME response', e)),
      PayMEMiniAppEvents.addListener('onError', (e) => console.log('PayME error', e)),
    ];
    return () => subs.forEach((s) => s.remove());
  }, []);
}

// identityToken comes from your server, which gets it from PayME. See identityToken.
export function openWallet(identityToken: string) {
  open('modal', { action: 'OPEN', identityToken });
}
```

> Never put real keys in public source code, READMEs or screenshots. Write `publicKey` and `privateKey` on a single line without line breaks (for example `-----BEGIN PUBLIC KEY-----MFww...-----END PUBLIC KEY-----`).

## identityToken

Since 3.0.0 the library no longer accepts a phone number. Each time you open the Miniapp for a user, the app passes an `identityToken`: a string signed by the PayME server for that user and bound to your `appId`.

**Three-step flow**

1. Your app calls your server (with the user already authenticated).
2. Your server calls PayME's `identityToken/init` API (server to server) with the user's phone number and receives an `identityToken`. Contact PayME for the API documentation. Never call this API from the app.
3. Your app passes the `identityToken` to `open`, `getBalance` or `getAccountInfo`.

| Action / function | `identityToken` |
| --- | --- |
| `OPEN`, `KYC`, `PAY`, `TRANSFER_QR`, `SERVICE`, `DEPOSIT`, `WITHDRAW`, `TRANSFER`, `getBalance`, `getAccountInfo` | Required |
| `PAYMENT` | Optional. With it, the user can pay from the PayME wallet. Without it, the user pays as a guest (card, QR) |
| `PAYME` | Not used. The user enters a phone number and signs in with a PIN |

**Lifetime**

- A token is valid for 15 minutes and can be reused any number of times within that window.
- Invalid or expired: the Miniapp closes (if open) and `onError` reports `code` `"401"`. Get a new token from your server and try again.
- Empty string for a required action/function: the Miniapp does not open and `onError` reports `code` `"IDENTITY_TOKEN_REQUIRED"`.

**Security**

- Keep the `identityToken` in memory only: do not log it, persist it (AsyncStorage, MMKV, Keychain, files) or put it in URLs or analytics.
- Never ship signing keys or `identityToken/init` credentials in the app.

## API reference

```ts
import {
  init, open, getBalance, getAccountInfo, setLanguage, close, PayMEMiniAppEvents,
} from 'payme-mini-app';
```

Every function returns `void`. Results and errors arrive asynchronously through [`PayMEMiniAppEvents`](#paymeminiappevents).

### init

```ts
declare function init(
  appId: string,
  publicKey: string,
  privateKey: string,
  locale?: 'vi' | 'en',                              // default 'vi'
  env?: 'PRODUCTION' | 'STAGING' | 'SANDBOX',        // default 'PRODUCTION'
  mode?: 'miniapp_sandbox' | 'miniapp_product',      // default 'miniapp_product'
): void;
```

| Parameter | Required | Description |
| --- | --- | --- |
| `appId` | Yes | App ID issued by PayME (the `x-api-client` value on the dashboard) |
| `publicKey` | Yes | PayME public key, on a single line |
| `privateKey` | Yes | Your app's private key, on a single line |
| `locale` | No | UI language |
| `env` | No | PayME server environment |
| `mode` | No | `miniapp_sandbox` with `SANDBOX`, `miniapp_product` with `PRODUCTION` |

Notes:

- Call `init` once, as early as possible (the SDK checks for and downloads Miniapp updates at initialization), and **before any other function**. On iOS, calling `open`, `getBalance`, `getAccountInfo`, `setLanguage` or `close` before `init` crashes the app.
- Android needs the current Activity: call it in a `useEffect` of your root component, not at module top level. If there is no Activity yet, later calls are silently ignored.
- Register listeners before calling `open`, `getBalance` or `getAccountInfo`.

### open

```ts
declare function open(type: 'screen' | 'modal', param: PayMEOpeningParam): void;
```

| Parameter | Description |
| --- | --- |
| `type` | `screen`: full screen. `modal`: slides up from the bottom |
| `param` | Object with `action`, `identityToken` (see the table in [identityToken](#identitytoken)) and `data` (depends on the action) |

`PayMEOpeningParam` is not exported by the package. If you need the type, derive it with `Parameters`:

```ts
import { open } from 'payme-mini-app';

type PayMEOpeningParam = Parameters<typeof open>[1];
```

**Actions**

| `action` | Purpose | `identityToken` | `data` |
| --- | --- | --- | --- |
| `OPEN` | Open the wallet home; starts activation if the wallet is not activated yet | Required | None |
| `KYC` | Open identity verification (eKYC) | Required | None |
| `PAY` | Pay a partner order | Required | [`PaymentParam`](#paymentparam-pay) |
| `PAYMENT` | Pay a transaction the partner has already created | Optional | [`PaymentDirectParam`](#paymentdirectparam-payment) |
| `TRANSFER_QR` | Transfer from the PayME wallet to a bank account given by the partner | Required | [`TransferQRParam`](#transferqrparam-transfer_qr) |
| `SERVICE` | Pay a bill or service (electricity, water, cards…) | Required | [`ServiceParam`](#serviceparam-service) |
| `DEPOSIT` | Top up the wallet | Required | [`PayParam`](#payparam-deposit-withdraw-transfer) |
| `WITHDRAW` | Withdraw from the wallet | Required | [`PayParam`](#payparam-deposit-withdraw-transfer) |
| `TRANSFER` | Transfer money | Required | [`PayParam`](#payparam-deposit-withdraw-transfer) |
| `PAYME` | Open the full PayME wallet; the user signs in | Not used | None |

`amount` is an integer in VND. `extraData` is a `Record<string, string | object>`: partner data returned to you when the transaction succeeds. `isShowResult` and `isBackToApp` default to `true` on both platforms.

#### PaymentParam (PAY)

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `transactionId` | `string` | Yes | Partner transaction ID |
| `amount` | `number` | Yes | Amount |
| `ipnUrl` | `string` | Yes | URL that receives IPN callbacks when the transaction changes state. Pass `''` if unused |
| `note` | `string` | No | Note |
| `extraData` | `Record<string, string \| object>` | No | Partner data |
| `isShowResult` | `boolean` | No | Show PayME's result screen |

#### PaymentDirectParam (PAYMENT)

The partner fetches the payment methods and creates the transaction through PayME's API (contact PayME), then passes the transaction code it received.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `transaction` | `string` | Yes | Transaction code returned by PayME when the transaction was created |
| `extraData` | `Record<string, string \| object>` | No | Partner data |
| `isShowResult` | `boolean` | No | Show PayME's result screen |

#### TransferQRParam (TRANSFER_QR)

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `amount` | `number` | Yes | Amount (pass `0` to let the user enter it) |
| `bankNumber` | `string` | Yes | Destination account or card number |
| `swiftCode` | `string` | Yes | SWIFT code of the destination bank |
| `cardHolder` | `string` | Yes | Account holder name |
| `note` | `string` | No | Note |
| `partnerTransaction` | `string` | No | Partner transaction ID |
| `extraData` | `Record<string, string \| object>` | No | Partner data |
| `isShowResult` | `boolean` | No | Show PayME's result screen |

#### ServiceParam (SERVICE)

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `service` | `string` | Yes | Service code (table below) |
| `extraData` | `Record<string, string \| object>` | No | Partner data |
| `isBackToApp` | `boolean` | No | Return to the partner app when done |
| `isShowResult` | `boolean` | No | Show PayME's result screen |

| Service code | Service |
| --- | --- |
| `POWE` | Electricity |
| `WATE` | Water |
| `ADSL` | Internet |
| `TIVI` | Television |
| `PPMB` | Postpaid mobile |
| `MOBILE_CARD` | Mobile scratch card |
| `MOBILE_TOPUP` | Mobile top-up |
| `GAME_CARD` | Game card |

#### PayParam (DEPOSIT, WITHDRAW, TRANSFER)

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `amount` | `number` | No | Prefilled amount |
| `description` | `string` | No | Transaction description |
| `extraData` | `Record<string, string \| object>` | No | Partner data |
| `isBackToApp` | `boolean` | No | Return to the partner app when done |
| `isShowResult` | `boolean` | No | Show PayME's result screen |

`data` is required for these three actions; pass `{}` if you have no fields to set.

**Examples**

```ts
import { open } from 'payme-mini-app';

declare const identityToken: string; // from your server

open('screen', {
  action: 'PAY',
  identityToken,
  data: { transactionId: 'ORDER_123', amount: 10000, ipnUrl: 'https://partner.example.com/ipn', note: 'Order 123' },
});

open('modal', { action: 'PAYMENT', data: { transaction: '<TRANSACTION>' } }); // guest checkout
open('modal', { action: 'PAYMENT', identityToken, data: { transaction: '<TRANSACTION>' } }); // wallet allowed

open('modal', {
  action: 'TRANSFER_QR',
  identityToken,
  data: { amount: 200000, bankNumber: '<BANK_NUMBER>', swiftCode: '<SWIFT_CODE>', cardHolder: 'NGUYEN VAN A' },
});

open('screen', { action: 'SERVICE', identityToken, data: { service: 'POWE' } });
open('screen', { action: 'DEPOSIT', identityToken, data: { amount: 50000 } });
open('screen', { action: 'KYC', identityToken });
open('screen', { action: 'PAYME' });
```

**Parameter validation**

- iOS: a missing required `data` field (or a wrong type, such as a non-integer `amount`) for `PAY`, `PAYMENT`, `TRANSFER_QR` or `SERVICE`, or an unknown `action`: the Miniapp does not open and `onError` reports `code` `"INVALID_PARAM"`.
- Android: the library does not report `INVALID_PARAM`; missing optional fields get defaults (`note`, `ipnUrl`, `description` = `''`; `isShowResult`, `isBackToApp` = `true`). Rely on the TypeScript types to always pass the required fields.

### getBalance

```ts
declare function getBalance(identityToken: string): void;
```

Gets the user's wallet balance. The result arrives in `onResponse` with `action` `"GET_BALANCE"`:

| Field | Type | Description |
| --- | --- | --- |
| `linked` | `boolean` | Whether the user has linked the wallet to your app |
| `balance` | `number` | Balance (only when `linked` is `true`) |
| `message` | `string` | Server message |

Invalid or expired `identityToken`: `onError` with `code` `"401"`. Empty string: `"IDENTITY_TOKEN_REQUIRED"`.

### getAccountInfo

```ts
declare function getAccountInfo(identityToken: string): void;
```

Gets the wallet account information. The result arrives in `onResponse` with `action` `"GET_ACCOUNT_INFO"`; the other fields are the `accountInfo` object returned by the PayME server (includes `accountId`). Account not found: `onError` (`code` `"OTHER"` on Android, `"SERVER"` on iOS). `identityToken` errors are the same as for `getBalance`.

### setLanguage

```ts
declare function setLanguage(language: string): void;
```

Changes the Miniapp language after `init`. `'en'` selects English; any other value is treated as `'vi'`.

### close

```ts
declare function close(): void;
```

Closes the open Miniapp.

### PayMEMiniAppEvents

A `NativeEventEmitter` with two events. `addListener` returns a subscription; call `remove()` when you no longer need it.

| Event | Payload | When |
| --- | --- | --- |
| `onResponse` | `{ action: string, ...data }` | An action completes (for example a successful payment), or a `getBalance` / `getAccountInfo` result is ready |
| `onError` | `{ action: string, code: string, description: string }` | An error occurs or the user closes the Miniapp |

`action` is one of `PAYME`, `OPEN`, `KYC`, `PAY`, `PAYMENT`, `TRANSFER_QR`, `SERVICE`, `DEPOSIT`, `WITHDRAW`, `TRANSFER`, `GET_BALANCE`, `GET_ACCOUNT_INFO`.

Platform differences:

| | iOS | Android |
| --- | --- | --- |
| `onResponse` | Payload has `action` plus all data returned by the SDK | Same as iOS; no event is emitted when the SDK returns no data |
| `onError` | Adds `INVALID_PARAM` (from the library's parameter check) | No `INVALID_PARAM` |
| Listeners | Events emitted while no listener is registered are dropped | Events are always emitted |

The library does not export payload types; declare them in your app:

```ts
import { PayMEMiniAppEvents } from 'payme-mini-app';

type PayMEErrorEvent = { action: string; code: string; description: string };
type PayMEResponseEvent = { action: string; [key: string]: unknown };

const errorSub = PayMEMiniAppEvents.addListener('onError', (event) => {
  const { action, code, description } = event as PayMEErrorEvent;
  if (code === '401') {
    // identityToken expired: get a new one from your server and open again
  }
  console.warn(action, code, description);
});

const responseSub = PayMEMiniAppEvents.addListener('onResponse', (event) => {
  const data = event as PayMEResponseEvent;
  if (data.action === 'GET_BALANCE') {
    console.log(data.linked, data.balance);
  }
});

// On unmount:
errorSub.remove();
responseSub.remove();
```

## Error codes

`onError` reports `code` as a string.

| `code` | Platform | Meaning | What to do |
| --- | --- | --- | --- |
| `401` | iOS, Android | `identityToken` is invalid or expired. The Miniapp closes if open | Get a new `identityToken` from your server and retry |
| `IDENTITY_TOKEN_REQUIRED` | iOS, Android | `identityToken` missing (empty string) for a required action/function. The Miniapp does not open | Obtain an `identityToken` before calling |
| `INVALID_PARAM` | iOS | Unknown `action`, or `data` is missing a required field / has a wrong type | Fix the parameters per the [action table](#open) |
| `USER_CANCEL` | iOS, Android | The user closed the Miniapp | Not an error; update your UI if needed |
| `SYSTEM_CANCEL` | iOS | The Miniapp could not start on the device | Retry; contact PayME if it persists |
| `CONNECTION_LOST` | iOS, Android | Network connection lost | Check connectivity and retry |
| `TIMED_OUT` | iOS, Android | Server request timed out | Retry |
| `NO_RESPONSE` | iOS, Android | Server did not respond | Retry later |
| `ENCODE_FAILED` | iOS, Android | Request encryption failed | Check `publicKey`, `privateKey`, `appId` and `env` |
| `DECODE_FAILED` | iOS, Android | Response decryption failed | Check your keys and `env` |
| `SERVER` | iOS | Server returned an error (see `description`) | Show `description` or retry |
| `SERVER_ERROR` | Android | Server returned an error | Retry later |
| `OTHER` | iOS, Android | Other error (for example account not found in `getAccountInfo`) | Show `description` |
| Other codes | iOS, Android | Business errors reported by the Miniapp | Show `description`; contact PayME to look up a code |

## Upgrading from 2.x

3.0.0 replaces the phone number with `identityToken` and moves the native SDKs from 0.7.x to 0.8.x. It is not backward compatible.

| 2.x | 3.0.0 |
| --- | --- |
| `open(type, { action, phone, data })` | `open(type, { action, identityToken, data })` |
| `open('screen', { action: 'PAYME', phone })` | `open('screen', { action: 'PAYME' })` |
| `open(type, { action: 'PAYMENT', phone?, data })` | `open(type, { action: 'PAYMENT', identityToken?, data })` |
| `getBalance(phone)` | `getBalance(identityToken)` |
| `getAccountInfo(phone)` | `getAccountInfo(identityToken)` |
| iOS `PayME-Miniapp` 0.7.6 | iOS `PayME-Miniapp` 0.8.1, Xcode 26.2 or later |
| Android `native-android` 0.7.2 | Android `native-android` 0.8.2 (SDK 1.0.24) |
| — | Handle `onError` codes `"401"` and `"IDENTITY_TOKEN_REQUIRED"` |

Steps:

1. Server: integrate PayME's `identityToken/init` API (contact PayME for the documentation and to enable it for your `appId`).
2. App: `npm install payme-mini-app@^3.0.0`, then `cd ios && pod install`.
3. Replace every `phone` with `identityToken`; drop `phone` from the `PAYME` action.
4. Handle `401` (fetch a new token) and `IDENTITY_TOKEN_REQUIRED`.
5. Build iOS with Xcode 26.2 or later.

`init`, `setLanguage`, `close`, `PayMEMiniAppEvents` and the `data` shape of each action are unchanged.

## Version history and support

| Version | Main changes |
| --- | --- |
| 3.0.0 | `identityToken` replaces the phone number; iOS `PayME-Miniapp` 0.8.1, Android `native-android` 0.8.2 (SDK 1.0.24) |
| 2.0.0 | iOS `PayME-Miniapp` 0.7.6, Android `native-android` 0.7.2; phone-number identification |

All versions: [npm `payme-mini-app`](https://www.npmjs.com/package/payme-mini-app?activeTab=versions).

Support: PayME Technology Corporation, [hotro@payme.vn](mailto:hotro@payme.vn), [payme.vn](https://payme.vn/web/home). Documentation issues: [GitHub Issues](https://github.com/PayME-Miniapp/payme-mini-app/issues). Keys, the `identityToken/init` API documentation and the `PAYMENT` transaction API: contact PayME.
