# Karaoke – Utaya và Power Apps Planning

### Karaoke – Utaya

**CV:** React Native, JavaScript, Axios, Notifications; setup development/staging, maintenance, performance, location-based push, làm việc khách Nhật qua phiên dịch.

**Flow tham khảo:** environment config → Axios API → component state/list stores → location permission → nearby store/notification → deep link. Không suy ra SDK geofencing hoặc FCM.

**Q: Staging vs production?** Scheme/build config + baseURL riêng, token/APNs environment đúng, không đưa secret vào client.

**Q: useEffect cleanup?** Hủy request/subscription lúc unmount hoặc dependency đổi để tránh stale update.

**Q: Location push hoạt động?** Phân biệt backend gửi APNs và local region trigger; permission/background/system policy. Nêu cơ chế thật đã dùng.

[React Native](../cross-platform/react-native.md) · [Notifications](../ios-system-frameworks/notifications.md)

### Power Apps Planning

**CV:** supply planning/quantity/storage, maintain, design docs, database/UI design.

**Flow tham khảo:** form nhập vật liệu → validation → data source CRUD → list/gallery → filter theo planning; role/security và conflict tại server.

**Q: Patch vs SubmitForm?** Patch thay đổi record trực tiếp, SubmitForm dùng form validation/lifecycle.

**Q: Delegation?** Non-delegable formula có thể chỉ xử lý subset; chọn server-side delegable query.

[Power Apps](../cross-platform/power-apps.md)

**Chuẩn bị:** Xác nhận data source, loại app, formula đã dùng, staging config, công việc trực tiếp và bug thật. CV ghi limited experience, không cần nhận ownership ngoài thực tế.
