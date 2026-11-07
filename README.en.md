[Tiếng Việt](README.md) | **English**

# payme-mini-app

React Native library for embedding PayME Miniapp (e-wallet, payments, top-up, withdrawal, transfers, eKYC) in a partner app on iOS and Android.

| Package | Version |
| --- | --- |
| npm `payme-mini-app` | **3.0.1** (compatible with 3.0.0; breaking change from 2.x) |
| iOS pod `PayME-Miniapp` | 0.8.3 |
| Android `com.github.PayME-Miniapp:native-android` | 0.8.4 (SDK 1.0.24) |

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
npm install payme-mini-app@^3.0.1
# or
yarn add payme-mini-app@^3.0.1
```

### iOS

The library pulls in the `PayME-Miniapp` 0.8.3 pod. Add the following to `ios/Podfile` (same as the example app):

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

The SDK does not need **Background Modes** (Background fetch): Miniapp updates are downloaded while opening, not in the background.

### Android: permissions

Nothing to declare. The SDK manifest contains the permissions below and Gradle merges them into your app: `INTERNET`, `ACCESS_NETWORK_STATE`, `ACCESS_WIFI_STATE`, `CAMERA`, `READ_CONTACTS`, `NFC`, `READ_EXTERNAL_STORAGE`, `WRITE_EXTERNAL_STORAGE`, `ACCESS_LOCAL_NETWORK`. The Miniapp requests runtime permissions (camera, contacts) when it needs them.

## Quick start

```tsx
import { useEffect } from 'react';
import { addErrorListener, addResponseListener, init, open } from 'payme-mini-app';

export function usePayME() {
  useEffect(() => {
    // Call once when the first screen mounts. Without mode, SANDBOX uses 'miniapp_sandbox'.
    init('<APP_ID>', '<PUBLIC_KEY>', '<PRIVATE_KEY>', 'vi', 'SANDBOX');
    const subs = [
      addResponseListener((e) => console.log('PayME response', e.action)),
      addErrorListener((e) => console.log('PayME error', e.action, e.code, e.description)),
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
  init, open, getBalance, getAccountInfo, setLanguage, close,
  addResponseListener, addErrorListener, PayMEMiniAppEvents, PayMEErrorCode,
} from 'payme-mini-app';
import type {
  PayMEOpeningParam, PayMEOpeningType, PayMEEnv, PayMEMode, PayMELocale,
  PayMEPaymentParam, PayMEPaymentDirectParam, PayMETransferQRParam, PayMEServiceParam, PayMEPayParam,
  PayMEErrorEvent, PayMEResponseEvent,
} from 'payme-mini-app';
```

Every function returns `void`. Results and errors arrive asynchronously through the `onResponse` and `onError` [events](#events).

### init

```ts
declare function init(
  appId: string,
  publicKey: string,
  privateKey: string,
  locale?: PayMELocale, // 'vi' | 'en', default 'vi'
  env?: PayMEEnv,       // 'PRODUCTION' | 'STAGING' | 'SANDBOX', default 'PRODUCTION'
  mode?: PayMEMode,     // 'miniapp_sandbox' | 'miniapp_product', default depends on env
): void;
```

| Parameter | Required | Description |
| --- | --- | --- |
| `appId` | Yes | App ID issued by PayME (the `x-api-client` value on the dashboard) |
| `publicKey` | Yes | PayME public key, on a single line |
| `privateKey` | Yes | Your app's private key, on a single line |
| `locale` | No | UI language |
| `env` | No | PayME server environment |
| `mode` | No | Miniapp update channel. When omitted: `miniapp_sandbox` for `SANDBOX`, `miniapp_product` for other environments. A value you pass is used as is |

Notes:

- Call `init` once and **before any other function**. Calling `open`, `getBalance`, `getAccountInfo`, `setLanguage` or `close` before `init` does nothing and `onError` reports `code` `"PAYME_NOT_INITIALIZED"` (both platforms; before 3.0.1, iOS crashed).
- Calling `init` again (for example to change `env`, `mode` or `locale`) does not close an open Miniapp, and `close()` still closes it. iOS: with the same `appId`/`publicKey`/`privateKey` the SDK instance is kept and only the configuration is updated; with different keys the open Miniapp is closed first.
- Android: `open` uses the current Activity (the SDK is recreated when the Activity changes, for example after rotation). With no Activity, `open` reports `PAYME_NOT_INITIALIZED`. `getBalance`, `getAccountInfo`, `setLanguage` and `close` do not need an Activity (they work while the app is in the background).
- Register listeners before calling `init`/`open`: on iOS, events emitted while no listener is registered are dropped.

### open

```ts
declare function open(type: PayMEOpeningType, param: PayMEOpeningParam): void;
```

| Parameter | Description |
| --- | --- |
| `type` | `screen`: full screen. `modal`: slides up from the bottom. Case-insensitive (`'Modal'` = `modal`); any other value is treated as `screen` |
| `param` | Object with `action`, `identityToken` (see the table in [identityToken](#identitytoken)) and `data` (depends on the action) |

`PayMEOpeningParam` is a union keyed by `action`, exported by the package together with the `data` types:

```ts
import { open, type PayMEOpeningParam } from 'payme-mini-app';

declare const identityToken: string; // from your server

const param: PayMEOpeningParam = { action: 'OPEN', identityToken };
open('modal', param);
```

**Actions**

| `action` | Purpose | `identityToken` | `data` |
| --- | --- | --- | --- |
| `OPEN` | Open the wallet home; starts activation if the wallet is not activated yet | Required | None |
| `KYC` | Open identity verification (eKYC) | Required | None |
| `PAY` | Pay a partner order | Required | [`PayMEPaymentParam`](#paymepaymentparam-pay) |
| `PAYMENT` | Pay a transaction the partner has already created | Optional | [`PayMEPaymentDirectParam`](#paymepaymentdirectparam-payment) |
| `TRANSFER_QR` | Transfer from the PayME wallet to a bank account given by the partner | Required | [`PayMETransferQRParam`](#paymetransferqrparam-transfer_qr) |
| `SERVICE` | Pay a bill or service (electricity, water, cards…) | Required | [`PayMEServiceParam`](#paymeserviceparam-service) |
| `DEPOSIT` | Top up the wallet | Required | [`PayMEPayParam`](#paymepayparam-deposit-withdraw-transfer) |
| `WITHDRAW` | Withdraw from the wallet | Required | [`PayMEPayParam`](#paymepayparam-deposit-withdraw-transfer) |
| `TRANSFER` | Transfer money | Required | [`PayMEPayParam`](#paymepayparam-deposit-withdraw-transfer) |
| `PAYME` | Open the full PayME wallet; the user signs in | Not used | None |

`amount` is an integer in VND, at most 2,147,483,647; fractional values (for example `20000.5`) are invalid (`20000.0` is accepted). `extraData` is a `Record<string, string | object>`: partner data returned to you when the transaction succeeds. `isShowResult` and `isBackToApp` accept booleans only and default to `true` on both platforms; other values (for example `0`/`1`) are ignored and the default is used.

#### PayMEPaymentParam (PAY)

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `transactionId` | `string` | Yes | Partner transaction ID |
| `amount` | `number` | Yes | Amount |
| `ipnUrl` | `string` | No | URL that receives IPN callbacks when the transaction changes state |
| `note` | `string` | No | Note |
| `extraData` | `Record<string, string \| object>` | No | Partner data |
| `isShowResult` | `boolean` | No | Show PayME's result screen |

#### PayMEPaymentDirectParam (PAYMENT)

The partner fetches the payment methods and creates the transaction through PayME's API (contact PayME), then passes the transaction code it received.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `transaction` | `string` | Yes | Transaction code returned by PayME when the transaction was created |
| `extraData` | `Record<string, string \| object>` | No | Partner data |
| `isShowResult` | `boolean` | No | Show PayME's result screen |

#### PayMETransferQRParam (TRANSFER_QR)

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

#### PayMEServiceParam (SERVICE)

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

#### PayMEPayParam (DEPOSIT, WITHDRAW, TRANSFER)

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

- Missing `data`, or a missing or wrongly typed required field for `PAY`, `PAYMENT`, `TRANSFER_QR` or `SERVICE` (including a non-integer `amount`), or an unknown `action`: the Miniapp does not open and `onError` reports `code` `"INVALID_PARAM"`. Same on iOS and Android (before 3.0.1, Android opened the PayME wallet instead).
- `DEPOSIT`, `WITHDRAW`, `TRANSFER`: `amount` is optional, but a non-integer, out-of-range or wrongly typed `amount` also reports `INVALID_PARAM`.
- Wrong JS argument types (`null`, a number instead of a string…) no longer crash the app.
- On Android, missing optional fields get defaults: `note`, `ipnUrl`, `description`, `partnerTransaction` = `''`; `isShowResult`, `isBackToApp` = `true`.

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

Closes the open Miniapp. Does nothing when no Miniapp is open.

| | iOS | Android |
| --- | --- | --- |
| How it closes | Closes the Miniapp itself, even while it is covered (camera, another screen); never dismisses one of your screens by mistake | Closes the Miniapp (`screen` and `modal`) |
| Event | `onError` `USER_CANCEL` once the Miniapp has closed | No `onError` |

### Opening, closing and the back button

- **One Miniapp at a time.** Calling `open` while a Miniapp is open (iOS: or while an update is being checked or downloaded before opening) reports `MINIAPP_ALREADY_OPEN`; the open Miniapp is not affected.
- **Reopening from the closing callback** (`USER_CANCEL`, or a result that closes the Miniapp) or right after `close()` works. iOS defers the open until the previous Miniapp has gone; Android opens immediately.
- **One closing result per session.** Each open has exactly one closing result: `USER_CANCEL`, or the `onResponse`/`onError` that closes the Miniapp. When the Miniapp is only covered (eKYC camera, sub-webview) and comes back, no `USER_CANCEL` is sent.
- **Android back button** (`screen` and `modal`): the Miniapp closes its top popup (or blocks back on a payment-in-progress screen); otherwise it goes back one page; on its first page it closes and `onError` reports `USER_CANCEL`.
- iOS: the library opens the Miniapp from the top-most screen of your app (works with scene-based apps and while an RN `Modal` is shown).

### Events

```ts
declare function addResponseListener(listener: (event: PayMEResponseEvent) => void): EmitterSubscription;
declare function addErrorListener(listener: (event: PayMEErrorEvent) => void): EmitterSubscription;
```

The library emits two events. `addResponseListener` and `addErrorListener` register typed listeners and return a subscription; call `remove()` when you no longer need it. `PayMEMiniAppEvents` (a `NativeEventEmitter`) still works: `PayMEMiniAppEvents.addListener('onResponse' | 'onError', ...)`, but its payloads are untyped.

| Event | Payload | When |
| --- | --- | --- |
| `onResponse` | `PayMEResponseEvent`: `{ action: string, ...data }` | An action completes (for example a successful payment), or a `getBalance` / `getAccountInfo` result is ready |
| `onError` | `PayMEErrorEvent`: `{ action: string, code: string, description: string }` | An error occurs or the user closes the Miniapp |

`action` is one of `PAYME`, `OPEN`, `KYC`, `PAY`, `PAYMENT`, `TRANSFER_QR`, `SERVICE`, `DEPOSIT`, `WITHDRAW`, `TRANSFER`, `GET_BALANCE`, `GET_ACCOUNT_INFO`. A `PAYME_NOT_INITIALIZED` error from `setLanguage` or `close` has `action` `SET_LANGUAGE` or `CLOSE` respectively; a `PAYME_SDK_ERROR` from `init` (Android) has `action` `INIT`.

The `PayMEErrorCode` constant lists the codes generated by the library (`NOT_INITIALIZED` = `PAYME_NOT_INITIALIZED`, `INVALID_PARAM`, `SDK_ERROR` = `PAYME_SDK_ERROR`) and common SDK codes (`IDENTITY_TOKEN_REQUIRED`, `USER_CANCEL`, `SYSTEM_CANCEL`, `MINIAPP_ALREADY_OPEN`, `MINIAPP_CANNOT_PRESENT`). See [Error codes](#error-codes) for their meaning.

Platform differences:

| | iOS | Android |
| --- | --- | --- |
| `onResponse` | Payload has `action` plus all data returned by the SDK | Same as iOS; no event is emitted when the SDK returns no data |
| Listeners | Events emitted while no listener is registered are dropped | Events are always emitted |

```ts
import { addErrorListener, addResponseListener, PayMEErrorCode } from 'payme-mini-app';

const errorSub = addErrorListener(({ action, code, description }) => {
  switch (code) {
    case '401':
      // identityToken expired: get a new one from your server and open again
      break;
    case PayMEErrorCode.NOT_INITIALIZED:
      // Not initialized (or no Activity yet for open on Android): call init, then retry
      break;
    case PayMEErrorCode.USER_CANCEL:
      // The user closed the Miniapp
      break;
    default:
      console.warn(action, code, description);
  }
});

const responseSub = addResponseListener((data) => {
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
| `INVALID_PARAM` | iOS, Android | Unknown `action`, or `data` is missing a required field / has a wrong type (including a non-integer `amount`). The Miniapp does not open | Fix the parameters per the [action table](#open) |
| `PAYME_NOT_INITIALIZED` | iOS, Android | `open`, `getBalance`, `getAccountInfo`, `setLanguage` or `close` called before `init` (Android: or `open` with no Activity yet) | Call `init` first; on Android, call `open` once an Activity exists |
| `USER_CANCEL` | iOS, Android | The user closed the Miniapp (close button, Android back on the first page, tap outside a `modal`); iOS: also after `close()`. Reported once per session | Not an error; update your UI if needed |
| `SYSTEM_CANCEL` | iOS | The Miniapp could not start on the device (for example the local web server failed to start). The Miniapp closes | Retry; contact PayME if it persists |
| `MINIAPP_ALREADY_OPEN` | iOS, Android | `open` called while a Miniapp is open. The new one does not open; the open one is not affected | Wait for the current Miniapp to close, or call `close()` and open again |
| `MINIAPP_CANNOT_PRESENT` | iOS, Android | The Miniapp cannot be shown. iOS: no screen to present from. Android: `modal` while the Activity is not a `FragmentActivity`, or the SDK failed to show it (for example after the Activity saved its state) | Call `open` while your app is in the foreground |
| `PAYME_SDK_ERROR` | Android | The SDK threw an unexpected exception in `init`, `getBalance`, `getAccountInfo`, `setLanguage` or `close`; `description` is the exception class name | Log it and report to PayME |
| `CONNECTION_LOST` | iOS, Android | Network connection lost | Check connectivity and retry |
| `TIMED_OUT` | iOS, Android | Server request timed out | Retry |
| `NO_RESPONSE` | iOS | Server did not respond | Retry later |
| `ENCODE_FAILED` | iOS | Request encryption failed | Check `publicKey`, `privateKey`, `appId` and `env` |
| `DECODE_FAILED` | iOS, Android | Response decryption failed | Check your keys and `env` |
| `SERVER` | iOS | Server returned an error (see `description`) | Show `description` or retry |
| `SERVER_ERROR` | Android | Server returned an error | Retry later |
| `OTHER` | iOS, Android | Other error (for example account not found in `getAccountInfo`) | Show `description` |
| Other codes | iOS, Android | Business errors reported by the Miniapp | Show `description`; contact PayME to look up a code |

## Upgrading from 2.x

3.0.0 replaces the phone number with `identityToken` and moves the native SDKs from 0.7.x to 0.8.x. It is not backward compatible. Upgrade straight to 3.0.1 (compatible with 3.0.0, with the fixes listed in [Version history](#version-history-and-support)).

| 2.x | 3.0.1 |
| --- | --- |
| `open(type, { action, phone, data })` | `open(type, { action, identityToken, data })` |
| `open('screen', { action: 'PAYME', phone })` | `open('screen', { action: 'PAYME' })` |
| `open(type, { action: 'PAYMENT', phone?, data })` | `open(type, { action: 'PAYMENT', identityToken?, data })` |
| `getBalance(phone)` | `getBalance(identityToken)` |
| `getAccountInfo(phone)` | `getAccountInfo(identityToken)` |
| iOS `PayME-Miniapp` 0.7.6 | iOS `PayME-Miniapp` 0.8.3, Xcode 26.2 or later |
| Android `native-android` 0.7.2 | Android `native-android` 0.8.4 (SDK 1.0.24) |
| — | Handle `onError` codes `"401"` and `"IDENTITY_TOKEN_REQUIRED"` |

Steps:

1. Server: integrate PayME's `identityToken/init` API (contact PayME for the documentation and to enable it for your `appId`).
2. App: `npm install payme-mini-app@^3.0.1`, then `cd ios && pod install`.
3. Replace every `phone` with `identityToken`; drop `phone` from the `PAYME` action.
4. Handle `401` (fetch a new token) and `IDENTITY_TOKEN_REQUIRED`.
5. Build iOS with Xcode 26.2 or later.

`init`, `setLanguage`, `close`, `PayMEMiniAppEvents` and the `data` shape of each action are unchanged (since 3.0.1, `ipnUrl` in `PAY` is optional).

## Version history and support

| Version | Main changes |
| --- | --- |
| 3.0.1 | Compatible with 3.0.0. iOS `PayME-Miniapp` 0.8.3, Android `native-android` 0.8.4 (SDK 1.0.24): `USER_CANCEL` is reported once per session; one Miniapp at a time (`MINIAPP_ALREADY_OPEN`), reopening from the closing callback works; Android: back closes a popup → goes back a page → closes the Miniapp, and the host app is no longer forced into light mode or switched to another language. `mode` defaults by `env`. `type` is case-insensitive. A non-integer `amount` is `INVALID_PARAM` for every action; `isShowResult`/`isBackToApp` accept booleans only. Android: `open` uses the current Activity, `getBalance`/`getAccountInfo` work without an Activity. iOS: opens from the top-most screen, `close()` closes the Miniapp itself, calling `init` again with the same keys keeps the instance. Exported types (`PayMEOpeningParam`, `PayMEErrorEvent`…), `PayMEErrorCode`, `addErrorListener` / `addResponseListener`; `ipnUrl` optional. `INVALID_PARAM` and `PAYME_NOT_INITIALIZED` behave the same on both platforms (previously iOS crashed before `init`, and Android ignored calls or opened the wallet when `data` was missing) |
| 3.0.0 | `identityToken` replaces the phone number; iOS `PayME-Miniapp` 0.8.1, Android `native-android` 0.8.2 (SDK 1.0.24) |
| 2.0.0 | iOS `PayME-Miniapp` 0.7.6, Android `native-android` 0.7.2; phone-number identification |

All versions: [npm `payme-mini-app`](https://www.npmjs.com/package/payme-mini-app?activeTab=versions).

Support: PayME Technology Corporation, [hotro@payme.vn](mailto:hotro@payme.vn), [payme.vn](https://payme.vn/web/home). Documentation issues: [GitHub Issues](https://github.com/PayME-Miniapp/payme-mini-app/issues). Keys, the `identityToken/init` API documentation and the `PAYMENT` transaction API: contact PayME.
