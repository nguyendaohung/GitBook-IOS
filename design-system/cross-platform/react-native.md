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

## React Native chuyên sâu

### Kiến trúc và render

React component render từ props/state, reconciliation theo identity/key. JS logic chạy trên JS runtime; native UI render qua platform. Legacy architecture bridge serialize messages, New Architecture dùng JSI/Fabric/TurboModules theo version. Không đồng nhất mọi app với New Architecture; kiểm tra package/react-native version, Hermes và native module support.

| Native iOS | React Native |
| --- | --- |
| UIViewController/SwiftUI View | React component |
| Auto Layout/SwiftUI layout | Flexbox (Yoga) |
| UITableView/UICollectionView | FlatList/SectionList |
| async/await Swift | Promise/async-await JS |
| URLSession/Alamofire | fetch/Axios |
| Native delegate | Props/callback/event/native module |

State immutable update, không mutate array/object trực tiếp. useEffect dependency đúng, cleanup khi unmount/query đổi; useMemo/useCallback chỉ khi profiling cần. FlatList keyExtractor stable ID, virtualization, getItemLayout nếu size fixed, tránh nested list cùng hướng.

### Axios instance và API state

```javascript
import axios from 'axios';

export const api = axios.create({
  baseURL: 'https://example.com/api',
  timeout: 10000,
});

export async function loadStores(signal) {
  const response = await api.get('/stores', {signal});
  if (!Array.isArray(response.data)) {
    throw new Error('Invalid response');
  }
  return response.data;
}
```

Request interceptor gắn token lấy từ secure storage/session, response interceptor map 401/403/5xx. Refresh single-flight, replay một lần, không gọi refresh bằng interceptor lặp. AbortController cancel khi component unmount/query đổi; phân biệt canceled, no response, HTTP status và schema error. Không lưu token trong AsyncStorage plaintext; dùng Keychain/secure storage library phù hợp.

```jsx
import React, {useEffect, useState} from 'react';
import {FlatList, Text, ActivityIndicator, Button} from 'react-native';
import {loadStores} from './api';

export function StoreList() {
  const [state, setState] = useState({kind: 'loading', items: []});
  const [reload, setReload] = useState(0);

  useEffect(() => {
    const controller = new AbortController();
    let active = true;
    setState({kind: 'loading', items: []});
    loadStores(controller.signal)
      .then(items => {
        if (active) setState({kind: 'loaded', items});
      })
      .catch(() => {
        if (active && !controller.signal.aborted) {
          setState({kind: 'error', items: []});
        }
      });
    return () => {
      active = false;
      controller.abort();
    };
  }, [reload]);

  if (state.kind === 'loading') return <ActivityIndicator />;
  if (state.kind === 'error') {
    return <Button title="Retry" onPress={() => setReload(n => n + 1)} />;
  }
  return <FlatList data={state.items}
    keyExtractor={item => String(item.id)}
    renderItem={({item}) => <Text>{item.name}</Text>}
    ListEmptyComponent={<Text>No stores</Text>} />;
}
```

Endpoint illustrative; production validate item schema, pagination, retry/error message. active flag tránh stale response ngay cả cancellation không dừng server. Stable key theo store ID. Props read-only; state update functional khi phụ thuộc previous value.

### Native integration và environment

iOS permissions, APNs device token, location manager, foreground/background event qua native module/library. Không giả định push SDK; configure signing/APNs environment theo scheme. Dev/staging/prod baseURL và bundle ID riêng nếu cần; .env public config không giữ secret. Metro bundler debug JS, Xcode native crash, source maps/dSYM đúng build; profile JS thread/native main thread. Native SDK version compatibility với React Native architecture.

**Q: useEffect vs lifecycle VC?** Effect đồng bộ side effect theo dependency, cleanup trước rerun/unmount; không ánh xạ một-một viewDidLoad.

**Q: FlatList lag?** Stable keys, virtualization, image cache/downsample, giảm expensive rerender; đo JS/native.

**Bài tập:** Store list loading/empty/error/pagination, Axios cancel, test staging config và permission denied.
