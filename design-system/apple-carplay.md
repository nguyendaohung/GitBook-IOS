# 🚘 APPLE CARPLAY

## Requesting CarPlay Entitlements: [https://developer.apple.com/documentation/carplay/](https://developer.apple.com/documentation/carplay/)

## CARPLAY DOSC: [https://developer.apple.com/documentation/carplay](https://developer.apple.com/documentation/carplay)

## CarPlay: kiến thức nền tảng

CarPlay đưa một số loại ứng dụng được Apple hỗ trợ lên màn hình xe qua template hệ thống; không phải mirror tùy ý mọi UIView. Cần entitlement phù hợp category và tuân thủ driver safety. Dùng CPTemplateApplicationScene/CPTemplateApplicationSceneDelegate, CPInterfaceController, các template như CPListTemplate/CPMapTemplate tùy category/entitlement.

```swift
import CarPlay

final class CarSceneDelegate: UIResponder, CPTemplateApplicationSceneDelegate {
    func templateApplicationScene(_ templateApplicationScene: CPTemplateApplicationScene,
                                  didConnect interfaceController: CPInterfaceController) {
        let item = CPListItem(text: "Favorites", detailText: nil)
        let section = CPListSection(items: [item])
        let template = CPListTemplate(title: "Catalog", sections: [section])
        interfaceController.setRootTemplate(template, animated: false, completion: nil)
    }
}
```

Ví dụ skeleton cần UIKit import, scene manifest, entitlement và category hỗ trợ list template; không phải mọi app có quyền dùng CarPlay. Khi disconnect cleanup scene resources, không giữ interfaceController cũ. Test CarPlay simulator/device phù hợp, không thiết kế tương tác phức tạp gây mất tập trung.

**Q: CarPlay có render UIView tùy ý?** Không, chủ yếu system templates và category-specific APIs.

**Q: Entitlement để làm gì?** Cấp quyền category/feature được Apple duyệt; signing và provisioning phải khớp.

**Q: CarPlay lifecycle khác app lifecycle?** CarPlay là scene riêng, connect/disconnect và interface controller riêng; share domain/service, không share VC.
