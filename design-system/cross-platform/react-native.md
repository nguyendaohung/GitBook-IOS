# React Native, JavaScript và Axios

### Định nghĩa

React Native dùng JavaScript/TypeScript và React để dựng native UI. Component nhận props (input) và state (dữ liệu thay đổi); Hooks quản lý state/effect; Flexbox layout. JS thread xử lý logic, UI native main thread; kiến trúc cũ Bridge, New Architecture dùng JSI/Fabric/TurboModules (tùy version dự án).

```javascript
import React, {useEffect, useState} from 'react';
import {Text} from 'react-native';
import axios from 'axios';

export default function Stores() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    const controller = new AbortController();
    axios.get('https://example.com/stores', {signal: controller.signal})
      .then(response => setCount(response.data.length))
      .catch(error => {
        if (!axios.isCancel(error)) {
          // Update error state / safe logging.
        }
      });
    return () => controller.abort();
  }, []);
  return <Text>{count}</Text>;
}
```

### Kiến thức cần nắm

* ES6 const/let, closure, destructuring, map/filter/reduce, Promise, async/await, try/catch; event loop microtask/task. State update có thể batched, dùng functional update khi phụ thuộc state trước.
* useEffect dependency và cleanup, useMemo/useCallback chỉ tối ưu khi cần, FlatList stable key/virtualization, tránh render nặng.
* Axios instance baseURL, timeout, interceptor, cancellation, 401 refresh single-flight, error.response vs error.request; environment dev/staging/prod qua build config, không nhúng secret.
* Native module/bridge tùy architecture; iOS permissions/APNs, foreground/background handling, location permission. Nearby notification có thể backend geofence hoặc local location trigger; CV không ghi library.
* Debug JS/native crash, Metro bundler, Xcode schemes, staging QA. Kiểm tra package/version, không giả định New Architecture đã dùng.

### Q&A

**Q: Props vs State?** Props từ parent, read-only; state do component/owner quản lý, update gây render.

**Q: useEffect chạy khi nào?** Sau render commit, dependency đổi và cleanup trước rerun/unmount; development Strict Mode có thể chạy thêm setup/cleanup.

**Q: Axios interceptor để làm gì?** Gắn token, normalize error, refresh/retry có giới hạn; tránh infinite 401 loop.

**Liên hệ:** Karaoke – Utaya: maintenance, staging, location-based push, JavaScript/Axios/React Native.
