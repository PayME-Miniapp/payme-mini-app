**Tiếng Việt** | [English](README.en.md)

# payme-mini-app

Thư viện React Native để mở PayME Miniapp (ví, thanh toán, nạp, rút, chuyển tiền, eKYC) trong ứng dụng của bạn trên iOS và Android.

Phiên bản hiện tại: **3.0.1**. Thư viện đã kèm SDK native (iOS `PayME-Miniapp` 0.8.3, Android `native-android` 0.8.4), ứng dụng không cần khai báo thêm.

## Mục lục

1. [Yêu cầu](#yêu-cầu)
2. [Cài đặt](#cài-đặt)
3. [Cấu hình ứng dụng](#cấu-hình-ứng-dụng)
4. [Bắt đầu nhanh](#bắt-đầu-nhanh)
5. [identityToken](#identitytoken)
6. [API](#api)
7. [Mã lỗi](#mã-lỗi)
8. [Nâng cấp từ 2.x](#nâng-cấp-từ-2x)
9. [Hỗ trợ](#hỗ-trợ)

## Yêu cầu

- React Native 0.71 trở lên.
- iOS 13.0 trở lên (hoặc mức tối thiểu của React Native bạn dùng, nếu cao hơn), build bằng Xcode 26.2 trở lên. Build bằng Xcode 27 cần thêm đoạn `post_install` ở [Cài đặt iOS](#ios).
- Android `minSdkVersion` 26, `compileSdkVersion` 37.
- Bộ key `appId`, `publicKey`, `privateKey` do PayME cấp.
- Server của bạn gọi được API cấp `identityToken` của PayME.

## Cài đặt

```sh
npm install payme-mini-app@^3.0.1
```

### iOS

Thêm vào `ios/Podfile`:

```ruby
use_frameworks! :linkage => :static

target 'YourApp' do
  # ... use_react_native!(...) giữ nguyên

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
    # react_native_post_install(...) giữ nguyên
    installer.pods_project.targets.each do |target|
      target.build_configurations.each do |config|
        if $dynamic_framework.include?(target.name)
          config.build_settings['BUILD_LIBRARY_FOR_DISTRIBUTION'] = 'YES'
        end
        # Nâng pod có iOS tối thiểu thấp hơn ứng dụng (Xcode 27 báo lỗi nếu dưới 15.0)
        if config.build_settings['IPHONEOS_DEPLOYMENT_TARGET'].to_f < min_ios_version_supported.to_f
          config.build_settings['IPHONEOS_DEPLOYMENT_TARGET'] = min_ios_version_supported
        end
      end
    end
  end
end
```

```sh
cd ios && pod install
```

### Android

1. **JitPack.** React Native đã tự thêm `https://www.jitpack.io`. Nếu dự án tắt tính năng này (`includeJitpackRepository=false`) hoặc khai báo repository trong `settings.gradle`, thêm `maven { url 'https://www.jitpack.io' }` vào danh sách repository.
2. **Phiên bản SDK** trong `android/build.gradle`:

   ```groovy
   buildscript {
     ext {
       minSdkVersion = 26
       compileSdkVersion = 37
     }
   }
   ```

   Nếu Android Gradle Plugin báo chưa hỗ trợ `compileSdk` 37, hãy nâng AGP.
3. **ProGuard.** Nếu bản release bật `minifyEnabled`, thêm vào `android/app/proguard-rules.pro`:

   ```proguard
   -keep class vn.kalapa.ekyc.** { *; }
   -keepclassmembers,allowobfuscation class * {
       @com.google.gson.annotations.SerializedName <fields>;
   }
   ```

## Cấu hình ứng dụng

### iOS

Thêm vào `Info.plist` (sửa câu mô tả theo ứng dụng của bạn):

```xml
<!-- Mở app ngân hàng khi nạp tiền -->
<key>LSApplicationQueriesSchemes</key>
<array>
  <string>vcbpartner</string>
  <string>bidv.smartbanking.partner</string>
</array>
<key>NSCameraUsageDescription</key>
<string>Ứng dụng cần dùng máy ảnh để định danh và quét mã thanh toán</string>
<key>NSPhotoLibraryUsageDescription</key>
<string>Ứng dụng cần truy cập thư viện ảnh để chọn mã thanh toán</string>
<key>NSPhotoLibraryAddUsageDescription</key>
<string>Ứng dụng cần lưu mã thanh toán vào thư viện ảnh</string>
<key>NSContactsUsageDescription</key>
<string>Cho phép truy cập danh bạ để chọn nhanh số điện thoại</string>
<key>NSFaceIDUsageDescription</key>
<string>Dùng Face ID để xác thực nhanh và an toàn</string>
<!-- Đọc chip CCCD khi eKYC -->
<key>NFCReaderUsageDescription</key>
<string>Đọc chip căn cước công dân để định danh</string>
<key>com.apple.developer.nfc.readersession.iso7816.select-identifiers</key>
<array>
  <string>A0000002471001</string>
  <string>A0000002472001</string>
  <string>00000000000000</string>
</array>
<!-- Miniapp chạy trên web server cục bộ (template React Native đã có sẵn) -->
<key>NSAppTransportSecurity</key>
<dict>
  <key>NSAllowsLocalNetworking</key>
  <true/>
</dict>
```

Trong Xcode, vào target ứng dụng → **Signing & Capabilities** → **+ Capability** → thêm **Near Field Communication Tag Reading**.

![Near Field Communication Tag Reading](documents/capabilities2.png)

### Android

Không cần khai báo quyền. Các quyền cần thiết đã có sẵn trong SDK, Miniapp tự xin quyền khi cần.

## Bắt đầu nhanh

```tsx
import { useEffect } from 'react';
import { addErrorListener, addResponseListener, init, open } from 'payme-mini-app';

export function usePayME() {
  useEffect(() => {
    // Đăng ký listener trước init/open: trên iOS, sự kiện phát khi chưa có listener sẽ bị bỏ.
    const subs = [
      addResponseListener((e) => console.log('PayME response', e.action)),
      addErrorListener((e) => console.log('PayME error', e.action, e.code, e.description)),
    ];
    init('<APP_ID>', '<PUBLIC_KEY>', '<PRIVATE_KEY>', 'vi', 'SANDBOX');
    return () => subs.forEach((s) => s.remove());
  }, []);
}

// identityToken do server của bạn lấy từ PayME
export function openWallet(identityToken: string) {
  open('modal', { action: 'OPEN', identityToken });
}
```

> Không đưa key thật vào mã nguồn công khai. `publicKey` và `privateKey` viết trên một dòng, bỏ ký tự xuống dòng (ví dụ `-----BEGIN PUBLIC KEY-----MFww...-----END PUBLIC KEY-----`).

## identityToken

`identityToken` là chuỗi PayME cấp cho từng người dùng của bạn. Ứng dụng truyền chuỗi này mỗi lần mở Miniapp.

1. Ứng dụng gọi server của bạn.
2. Server của bạn gọi API `identityToken/init` của PayME với số điện thoại của người dùng (liên hệ PayME để nhận tài liệu API). Không gọi API này từ ứng dụng.
3. Ứng dụng truyền `identityToken` nhận được vào `open`, `getBalance` hoặc `getAccountInfo`.

| Action / hàm | `identityToken` |
| --- | --- |
| `PAYME` | Không dùng (người dùng tự đăng nhập) |
| `PAYMENT` | Tuỳ chọn. Có thì người dùng trả được bằng ví PayME; không có thì thanh toán như khách vãng lai (thẻ, QR) |
| Các action còn lại, `getBalance`, `getAccountInfo` | Bắt buộc |

- Chuỗi có hiệu lực 15 phút, dùng lại được trong thời gian đó.
- Sai hoặc hết hạn: `onError` trả `401` (Miniapp đóng nếu đang mở). Lấy chuỗi mới rồi gọi lại.
- Chỉ giữ chuỗi trong bộ nhớ: không lưu, không ghi log.

## API

Mọi hàm trả `void`. Kết quả và lỗi trả về qua sự kiện `onResponse` và `onError`.

### init

```ts
init(appId, publicKey, privateKey, locale?, env?, mode?)
```

| Tham số | Mô tả |
| --- | --- |
| `appId` | Mã ứng dụng do PayME cấp |
| `publicKey` | Public key của PayME |
| `privateKey` | Private key của ứng dụng |
| `locale` | `'vi'` (mặc định) hoặc `'en'` |
| `env` | `'PRODUCTION'` (mặc định), `'SANDBOX'` hoặc `'STAGING'` |
| `mode` | Thường không cần truyền. Mặc định `'miniapp_sandbox'` khi `env` là `SANDBOX`, còn lại `'miniapp_product'` |

Gọi `init` một lần, trước mọi hàm khác. Gọi hàm khác khi chưa `init` sẽ nhận lỗi `PAYME_NOT_INITIALIZED`.

### open

```ts
open(type, { action, identityToken?, data? })
```

`type`: `'screen'` (toàn màn hình) hoặc `'modal'` (trượt từ dưới lên).

| `action` | Chức năng | `data` |
| --- | --- | --- |
| `OPEN` | Trang chủ ví (chưa kích hoạt thì mở luồng kích hoạt) | Không có |
| `KYC` | Định danh (eKYC) | Không có |
| `PAY` | Thanh toán đơn hàng của bạn | [`PayMEPaymentParam`](#paymepaymentparam-pay) |
| `PAYMENT` | Thanh toán giao dịch đã tạo qua API PayME | [`PayMEPaymentDirectParam`](#paymepaymentdirectparam-payment) |
| `TRANSFER_QR` | Chuyển tiền tới tài khoản ngân hàng | [`PayMETransferQRParam`](#paymetransferqrparam-transfer_qr) |
| `SERVICE` | Thanh toán dịch vụ (điện, nước, thẻ…) | [`PayMEServiceParam`](#paymeserviceparam-service) |
| `DEPOSIT`, `WITHDRAW`, `TRANSFER` | Nạp, rút, chuyển tiền | [`PayMEPayParam`](#paymepayparam-deposit-withdraw-transfer) |
| `PAYME` | Toàn bộ ví PayME, người dùng tự đăng nhập | Không có |

```ts
open('screen', {
  action: 'PAY',
  identityToken,
  data: { transactionId: 'ORDER_123', amount: 10000, note: 'Đơn hàng 123' },
});
open('modal', { action: 'PAYMENT', data: { transaction: '<TRANSACTION>' } }); // khách vãng lai
open('screen', { action: 'SERVICE', identityToken, data: { service: 'POWE' } });
open('screen', { action: 'DEPOSIT', identityToken, data: { amount: 50000 } });
```

Tham số sai (thiếu `data`, thiếu trường bắt buộc, `amount` không phải số nguyên, `action` không hợp lệ) thì Miniapp không mở và `onError` trả `INVALID_PARAM`.

`amount` là số nguyên VND. Các tham số `data` dưới đây đều có thêm hai trường tuỳ chọn: `extraData` (dữ liệu riêng của bạn, trả lại khi giao dịch thành công) và `isShowResult` (hiện màn hình kết quả của PayME, mặc định `true`).

#### PayMEPaymentParam (PAY)

| Trường | Bắt buộc | Mô tả |
| --- | --- | --- |
| `transactionId` | Có | Mã giao dịch của bạn |
| `amount` | Có | Số tiền |
| `ipnUrl` | Không | URL nhận thông báo trạng thái giao dịch |
| `note` | Không | Ghi chú |

#### PayMEPaymentDirectParam (PAYMENT)

| Trường | Bắt buộc | Mô tả |
| --- | --- | --- |
| `transaction` | Có | Mã giao dịch PayME trả về khi bạn tạo giao dịch |

#### PayMETransferQRParam (TRANSFER_QR)

| Trường | Bắt buộc | Mô tả |
| --- | --- | --- |
| `amount` | Có | Số tiền (`0` để người dùng tự nhập) |
| `bankNumber` | Có | Số tài khoản hoặc số thẻ nhận |
| `swiftCode` | Có | Mã SWIFT của ngân hàng nhận |
| `cardHolder` | Có | Tên chủ tài khoản |
| `note` | Không | Ghi chú |
| `partnerTransaction` | Không | Mã giao dịch của bạn |

#### PayMEServiceParam (SERVICE)

| Trường | Bắt buộc | Mô tả |
| --- | --- | --- |
| `service` | Có | `POWE` (điện), `WATE` (nước), `ADSL` (internet), `TIVI` (truyền hình), `PPMB` (trả sau), `MOBILE_CARD` (thẻ điện thoại), `MOBILE_TOPUP` (nạp điện thoại), `GAME_CARD` (thẻ game) |
| `isBackToApp` | Không | Quay về ứng dụng của bạn sau khi xong (mặc định `true`) |

#### PayMEPayParam (DEPOSIT, WITHDRAW, TRANSFER)

| Trường | Bắt buộc | Mô tả |
| --- | --- | --- |
| `amount` | Không | Số tiền điền sẵn |
| `description` | Không | Mô tả giao dịch |
| `isBackToApp` | Không | Quay về ứng dụng của bạn sau khi xong (mặc định `true`) |

`data` bắt buộc với ba action này; truyền `{}` nếu không có trường nào.

### getBalance, getAccountInfo

```ts
getBalance(identityToken)
getAccountInfo(identityToken)
```

Kết quả trả qua `onResponse`:

- `GET_BALANCE`: `linked` (đã liên kết ví chưa), `balance` (có khi `linked` = `true`).
- `GET_ACCOUNT_INFO`: thông tin tài khoản ví.

### setLanguage, close

- `setLanguage('vi' | 'en')`: đổi ngôn ngữ Miniapp.
- `close()`: đóng Miniapp đang mở.

### Sự kiện

```ts
const errorSub = addErrorListener(({ action, code, description }) => { /* ... */ });
const responseSub = addResponseListener((data) => { /* data.action, ... */ });
// Khi không dùng nữa:
errorSub.remove();
responseSub.remove();
```

| Sự kiện | Nội dung | Khi nào |
| --- | --- | --- |
| `onResponse` | `{ action, ...dữ liệu }` | Giao dịch thành công, hoặc có kết quả `getBalance` / `getAccountInfo` |
| `onError` | `{ action, code, description }` | Có lỗi hoặc người dùng đóng Miniapp |

Hằng `PayMEErrorCode` chứa các mã lỗi thường gặp để so sánh, ví dụ `PayMEErrorCode.USER_CANCEL`.

### Mở và đóng Miniapp

- Mỗi lúc chỉ mở một Miniapp. Gọi `open` khi đang có Miniapp mở sẽ nhận `MINIAPP_ALREADY_OPEN`.
- Khi Miniapp đóng, bạn nhận kết quả đúng một lần: `onResponse`, `onError`, hoặc `USER_CANCEL` khi người dùng đóng. Có thể mở lại ngay trong callback này.
- Android: nút back đóng popup, rồi lùi trang trong Miniapp; hết trang thì đóng Miniapp và báo `USER_CANCEL`.
- `close()`: iOS báo `USER_CANCEL`, Android không báo.

## Mã lỗi

| `code` | Ý nghĩa | Cách xử lý |
| --- | --- | --- |
| `401` | `identityToken` sai hoặc hết hạn | Lấy `identityToken` mới rồi gọi lại |
| `IDENTITY_TOKEN_REQUIRED` | Thiếu `identityToken` | Truyền `identityToken` |
| `INVALID_PARAM` | Tham số `open` sai | Sửa tham số theo bảng ở [open](#open) |
| `PAYME_NOT_INITIALIZED` | Chưa gọi `init` | Gọi `init` trước |
| `USER_CANCEL` | Người dùng đóng Miniapp | Không phải lỗi |
| `MINIAPP_ALREADY_OPEN` | Đang có Miniapp mở | Đợi Miniapp đóng, hoặc gọi `close()` trước |
| `MINIAPP_CANNOT_PRESENT` | Không hiển thị được Miniapp | Gọi `open` khi ứng dụng đang ở foreground |
| Mã khác | Lỗi mạng, máy chủ hoặc nghiệp vụ | Hiển thị `description`, thử lại; liên hệ PayME nếu lặp lại |

## Nâng cấp từ 2.x

Bản 3.x thay số điện thoại bằng `identityToken` và không tương thích ngược.

| 2.x | 3.x |
| --- | --- |
| `open(type, { action, phone, data })` | `open(type, { action, identityToken, data })` |
| `open('screen', { action: 'PAYME', phone })` | `open('screen', { action: 'PAYME' })` |
| `getBalance(phone)` | `getBalance(identityToken)` |
| `getAccountInfo(phone)` | `getAccountInfo(identityToken)` |

1. Server: tích hợp API `identityToken/init` (liên hệ PayME).
2. `npm install payme-mini-app@^3.0.1`, rồi `cd ios && pod install`.
3. Thay mọi `phone` bằng `identityToken`; bỏ `phone` ở action `PAYME`.
4. Xử lý lỗi `401` và `IDENTITY_TOKEN_REQUIRED`.
5. Build iOS bằng Xcode 26.2 trở lên.

## Hỗ trợ

PayME Technology Corporation, email [hotro@payme.vn](mailto:hotro@payme.vn). Bộ key, tài liệu API `identityToken/init` và API tạo giao dịch `PAYMENT`: liên hệ PayME.

Danh sách phiên bản: [npm `payme-mini-app`](https://www.npmjs.com/package/payme-mini-app?activeTab=versions).
