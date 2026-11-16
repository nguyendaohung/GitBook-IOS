# ⌚ Apple Watch Development

* [WatchConnectivity và WCSession](watch-connectivity.md)
* [PhotoKit, PhotosUI và Watch Faces](photo-watch-faces.md)

Nhóm kiến thức cho Watch Faces – Photo Gallery, phân biệt đồng bộ với cài mặt đồng hồ.

## Câu hỏi phỏng vấn tổng quan

**Q: WatchConnectivity vs CloudKit/REST?** WCSession peer sync companion, REST/backend hoặc CloudKit sync authoritative tùy kiến trúc; chọn theo latency/offline/data ownership.

**Q: PhotoKit vs PhotosUI?** PhotoKit query library có authorization, PhotosUI picker user chọn asset với quyền tối thiểu.

**Q: Watch face vs complication?** Face system-managed, import/share theo public API; complication WidgetKit timeline slot, không arbitrary third-party face.
