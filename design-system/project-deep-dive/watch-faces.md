# Watch Faces – Project Deep Dive

### CV

Gallery hàng nghìn thiết kế, preview/category/search/favorites, photo customization, install flow, iPhone–Watch sync, subscription/remote content/notification/onboarding, image caching/localization. Swift, UIKit, WatchConnectivity, PhotoKit, StoreKit, REST, Auto Layout.

### Flow tham khảo

Remote catalog → pagination → stable ID → thumbnail/cache → preview → user photo picker/crop → compatible system watch face install flow. Favorites/settings → WCSession applicationContext; file asset → transferFile; events → transferUserInfo; interactive request → sendMessage nếu reachable.

[WatchConnectivity](../apple-watch/watch-connectivity.md) · [PhotoKit và Watch Faces](../apple-watch/photo-watch-faces.md) · [Image Performance](../ui-+-autolayout/snapkit-image-performance.md)

### Edge cases

* Watch không paired/installed/reachable, activation pending, đổi watch, duplicate/out-of-order transfer.
* iCloud photo chưa download, permission limited, memory spike, cell reuse, cache version.
* Watch face compatibility/model/watchOS, user cancel/install error; không tự tạo native face tùy ý.
* Subscription refund/expiry, offline entitlement policy, remote content update, localized pricing.

### Q&A

**Q: sendMessage vs applicationContext?** Interactive reachable vs latest state giao bất đồng bộ.

**Q: Tại sao ảnh nhỏ file vẫn tốn RAM?** Decode pixel RGBA, downsample trước, cache bounded.

**Q: Có cài watch face tự động không?** System public API/compatibility và user flow giới hạn; nêu đúng cách thực tế.

**Chuẩn bị:** Xác nhận watchOS version, install API, sync types, image pipeline, StoreKit version, lỗi production và kết quả thật.
