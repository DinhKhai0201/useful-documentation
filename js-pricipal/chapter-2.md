## Phần II – Hệ sinh thái Frontend (Browser + SPA)

### Chương 7 – DOM, BOM và hiệu năng rendering

#### 1. DOM tree, CSSOM, render tree và critical rendering path

Khi trình duyệt nhận HTML, nó parse thành **DOM tree** (mỗi thẻ là một node). Sau đó CSS được parse thành **CSSOM** (cây đại diện cho các rule CSS). Từ DOM + CSSOM, browser xây **render tree** – chỉ bao gồm các node hiển thị được, có đầy đủ thông tin style cần thiết.[web:37][web:46]

Critical Rendering Path là chuỗi bước: parse HTML → xây DOM → parse CSS → xây CSSOM → kết hợp thành render tree → layout (tính toán vị trí, kích thước) → paint (vẽ pixel). Bất cứ thao tác nào chạm mạnh vào DOM/CSS đều có thể ảnh hưởng tới layout và paint.[web:46]

#### 2. Reflow vs repaint và layout thrashing

- **Reflow (layout)**: khi browser phải tính toán lại layout – vị trí, kích thước của elements. Việc này xảy ra khi thêm/bớt nodes, đổi kích thước, margin, padding, font size, v.v.[web:37][web:46]
- **Repaint**: khi browser chỉ cần vẽ lại pixel mà không phải tính toán layout, ví dụ đổi màu, background, visibility.[web:37][web:46]

Reflow thường tốn kém hơn repaint vì nó có thể ảnh hưởng cả subtree hoặc toàn bộ document. **Layout thrashing** xảy ra khi code liên tục đọc layout (như `offsetHeight`) xen kẽ với việc ghi style, khiến browser phải reflow nhiều lần thay vì gộp lại.[web:37][web:40]

Best practice:
- Batch DOM writes / reads: đọc tất cả layout trước, rồi mới ghi style.
- Dùng `class` để thay đổi nhiều thuộc tính cùng lúc.
- Sử dụng `DocumentFragment` khi thêm nhiều phần tử.[web:37][web:40]

Ví dụ xấu (gây layout thrashing):
```js
for (let i = 0; i < items.length; i++) {
  const height = element.offsetHeight; // read
  element.style.height = (height + 10) + 'px'; // write
}
```

Ví dụ tốt (batch read → write):
```js
const heights = items.map(() => element.offsetHeight); // read all

heights.forEach(h => {
  element.style.height = (h + 10) + 'px'; // then write
});
```

#### 3. Event bubbling/capturing và event delegation

Sự kiện DOM diễn ra theo 3 phase: capturing (từ root xuống target), target, bubbling (từ target lên root). Mặc định hầu hết event handlers được đăng ký cho phase bubbling.[web:37]

**Event delegation**: thay vì gắn listener cho từng phần tử con, bạn gắn một listener lên phần tử cha, sau đó kiểm tra `event.target` để phân biệt phần tử nào được click. Điều này giảm số lượng listener, giảm memory footprint, và hoạt động tốt cho list lớn hoặc nội dung dynamic.[web:43]

Ví dụ delegation cho list lớn:
```js
const list = document.querySelector('#todo-list');

list.addEventListener('click', (event) => {
  const item = event.target.closest('li[data-id]');
  if (!item) return;

  const id = item.dataset.id;
  console.log('Clicked item id:', id);
});
```

#### 4. Web APIs quan trọng: Fetch, History, Storage, WebSocket, Web Worker

- **Fetch API**: thay thế XMLHttpRequest, cung cấp Promise‑based API cho HTTP request.
- **History API**: thao tác history stack (`pushState`, `replaceState`) – nền tảng của router SPA.
- **Storage**: `localStorage`, `sessionStorage`, `IndexedDB` để lưu dữ liệu phía client.
- **WebSocket**: giao tiếp realtime hai chiều, dùng cho chat, notification, orderbook.
- **Web Worker**: chạy JS trong background thread, tránh block UI khi xử lý nặng.[web:40]

Ví dụ Web Worker đơn giản:
```js
// worker.js
self.onmessage = function (e) {
  const n = e.data;
  let sum = 0;
  for (let i = 0; i < n; i++) sum += i;
  self.postMessage(sum);
};

// main.js
const worker = new Worker('worker.js');
worker.postMessage(1e8);
worker.onmessage = (e) => {
  console.log('Sum:', e.data);
};
```

#### 5. Ví dụ: delegation + tối ưu scroll event với passive listener

```js
const list = document.querySelector('#messages');

// Tối ưu scroll: dùng passive để browser không phải chờ handler
window.addEventListener('scroll', () => {
  // cập nhật lazy UI, nhưng tránh đo layout quá nhiều
}, { passive: true });

// Event delegation cho click
list.addEventListener('click', (event) => {
  const item = event.target.closest('.message-item');
  if (!item) return;

  console.log('Clicked message id =', item.dataset.id);
});
```

`passive: true` giúp browser biết rằng handler sẽ không gọi `preventDefault()`, nên có thể tối ưu scroll hiệu năng hơn.[web:40]

---

### Chương 8 – Kiến trúc SPA với React / Next.js

#### 1. Tư duy component-based và one-way data flow

Trong React, UI được tổ chức thành các **component** nhỏ, mỗi component nhận `props` và có thể có `state` riêng. Dữ liệu chảy một chiều: parent → child qua props, giúp luồng dữ liệu dễ theo dõi và debug.[web:42]

Component có thể là:
- Presentational (UI thuần): nhận props, không quản lý state domain.
- Container (smart component): fetch data, quản lý state, truyền xuống presentational.

Tư duy này giúp code base tách biệt rõ ràng giữa phần hiển thị và phần logic.

#### 2. Hooks core: `useState`, `useEffect`, `useContext`, `useMemo`, `useCallback`, `useRef`

- `useState` thêm state cho function component.
- `useEffect` đồng bộ với hệ thống bên ngoài: fetch data, subscribe event, thao tác DOM,…[web:36][web:45]
- `useContext` chia sẻ state/global config mà không cần prop drilling.
- `useMemo` memoize giá trị tính toán tốn kém.
- `useCallback` memoize function để tránh re-render con không cần thiết.
- `useRef` giữ một giá trị qua nhiều lần render mà không trigger re-render (thường dùng với DOM node hoặc mutable value).[web:42][web:39]

Ví dụ:
```jsx
import { useState, useEffect, useMemo, useCallback } from 'react';

function UserList({ initialFilter }) {
  const [users, setUsers] = useState([]);
  const [filter, setFilter] = useState(initialFilter);

  useEffect(() => {
    let cancelled = false;
    fetch('/api/users')
      .then((res) => res.json())
      .then((data) => {
        if (!cancelled) setUsers(data);
      });
    return () => { cancelled = true; };
  }, []);

  const filtered = useMemo(
    () => users.filter((u) => u.name.includes(filter)),
    [users, filter]
  );

  const handleChange = useCallback((e) => {
    setFilter(e.target.value);
  }, []);

  return (
    <div>
      <input value={filter} onChange={handleChange} />
      <ul>
        {filtered.map((u) => (
          <li key={u.id}>{u.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

#### 3. State management: Context, Redux Toolkit, Zustand, query state

Ở app nhỏ, bạn có thể dùng local state + Context. Ở scale lớn, cần tách rõ:
- **UI state**: modal open/close, active tab.
- **Server/cache state**: dữ liệu từ API, nên dùng thư viện như React Query/TanStack Query.
- **Global app state**: auth, user info, feature flags, có thể dùng Redux Toolkit, Zustand, Recoil,…[web:42]

Pattern hay: dùng React Query cho data fetching/cache (server state), Redux/Zustand cho domain state phức tạp, Context cho config/global nhỏ.

#### 4. Next.js: routing, SSR/SSG/ISR, server components

Next.js cung cấp:
- File‑based routing.
- Render mode: SSR (server‑side rendering), SSG (static site generation), ISR (incremental static regeneration).
- API routes cho backend nhẹ.
- App Router (từ Next 13+): server components, layout nested, streaming.

Lựa chọn mode render dựa trên:
- SEO cần HTML sẵn → SSR/SSG.
- Dashboard bảo mật, content cá nhân → CSR hoặc kết hợp.

Ví dụ page SSR đơn giản (kiểu pages router):
```js
export async function getServerSideProps() {
  const res = await fetch('https://api.example.com/posts');
  const posts = await res.json();
  return { props: { posts } };
}

export default function Page({ posts }) {
  return (
    <ul>
      {posts.map((p) => (
        <li key={p.id}>{p.title}</li>
      ))}
    </ul>
  );
}
```

#### 5. Pattern: refactor “God component” thành nhiều component + hooks

Một “God component” là component làm quá nhiều việc: fetch data, quản lý nhiều loại state, render UI lớn. Ở level Principal, bạn thường xuyên phải chia nhỏ những component này.

Ví dụ refactor:
```jsx
// hooks/useUsers.js
import { useState, useEffect } from 'react';

export function useUsers() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    setLoading(true);
    fetch('/api/users')
      .then((res) => res.json())
      .then((data) => setUsers(data))
      .catch((e) => setError(e))
      .finally(() => setLoading(false));
  }, []);

  return { users, loading, error };
}

// components/UserTable.js
function UserTable({ users }) {
  return (
    <table>
      <tbody>
        {users.map((u) => (
          <tr key={u.id}>
            <td>{u.name}</td>
            <td>{u.email}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}

// pages/users.js
import { useUsers } from '../hooks/useUsers';

export default function UsersPage() {
  const { users, loading, error } = useUsers();

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error</p>;

  return <UserTable users={users} />;
}
```

---

### Chương 9 – Frontend architecture & design system

#### 1. Module boundaries: feature-first vs layer-first

- **Layer-first**: chia theo tầng `components/`, `pages/`, `services/`, `utils/`. Dễ hiểu lúc đầu nhưng dễ thành “thùng rác chung” khi app lớn.
- **Feature-first**: nhóm theo domain: `features/users`, `features/orders`, mỗi feature chứa UI, hooks, service, test riêng.[web:42]

Ở scale lớn, feature-first giúp team ownership rõ ràng, giảm coupling giữa domain. Thường bạn sẽ kết hợp: `features/` cho domain, `shared/` cho components/utilities dùng chung.

#### 2. Design system: typography, spacing, theming, component library

Design system định nghĩa:
- Token: màu, font, spacing, radius, shadow.
- Components cơ bản: `Button`, `Input`, `Card`, `Modal`.
- Guideline về accessibility, responsive.

Dùng Storybook hoặc tương đương để document components. Ở level Principal, bạn thường dẫn dắt việc chuẩn hoá design system, thay vì để mỗi đội tự “bịa UI”.[web:42]

Ví dụ token đơn giản (CSS variables):
```css
:root {
  --color-primary: #2563eb;
  --color-bg: #0b1120;
  --space-1: 4px;
  --space-2: 8px;
  --space-3: 12px;
}
```

#### 3. State domain vs UI state, query state (React Query)

Phân loại state giúp bạn xử lý đúng nơi:
- **UI state** (local): modal, tab, hover.
- **Domain/app state**: shopping cart, current user, filters phức tạp.
- **Server/cache state**: data từ API, nên dùng React Query/TanStack Query để xử lý cache, refetch, background refresh, stale‑while‑revalidate.[web:42]

Example React Query:
```jsx
import { useQuery } from '@tanstack/react-query';

function usePosts() {
  return useQuery({
    queryKey: ['posts'],
    queryFn: () => fetch('/api/posts').then((r) => r.json()),
    staleTime: 1000 * 60,
  });
}
```

#### 4. Error boundaries, Suspense, skeleton loading

Error Boundary cho phép bạn catch lỗi render ở component con và hiển thị UI fallback thay vì crash toàn app. Suspense cho phép declarative loading khi dùng data fetching model mới hoặc lazy loading component.

Skeleton loading tốt hơn “Loading…” vì giữ layout ổn định, giúp CLS thấp hơn (xem chương sau về Core Web Vitals).[web:41]

#### 5. Ví dụ: folder structure cho một SaaS lớn

```text
src/
  app/
    layout.tsx
    page.tsx
  features/
    users/
      components/
      hooks/
      api/
      types.ts
    billing/
      components/
      hooks/
      api/
      types.ts
  shared/
    ui/
      Button.tsx
      Input.tsx
      Modal.tsx
    lib/
      fetcher.ts
      analytics.ts
    hooks/
      useBoolean.ts
      useDebounce.ts
```

Cấu trúc như vậy giúp mapping giữa domain thực tế và code, dễ onboard dev mới, dễ refactor.

---

### Chương 10 – Hiệu năng frontend ở scale lớn

#### 1. Lighthouse metrics và Core Web Vitals

Google giới thiệu **Core Web Vitals** để đo trải nghiệm người dùng thực tế: 
- **LCP (Largest Contentful Paint)**: thời gian render nội dung lớn nhất, nên < 2.5s.
- **INP (Interaction to Next Paint)**: đo độ phản hồi tương tác, nên < 200ms.
- **CLS (Cumulative Layout Shift)**: độ ổn định layout, nên < 0.1.[web:38][web:41][web:35]

Lighthouse thêm các metric lab như TBT (Total Blocking Time) để đánh giá thời gian JS block main thread. Tối ưu TBT thường cũng cải thiện INP.[web:41]

#### 2. Code splitting, dynamic import, prefetch và caching

App SPA lớn dễ bị bundle quá to, gây time‑to‑interactive chậm. Một số kỹ thuật:
- **Code splitting**: tách code theo route/feature bằng dynamic `import()`.
- **Route‑level prefetch**: preload bundle cho trang user sắp truy cập.
- **Caching**: HTTP cache (`Cache-Control`), service worker (PWA) cho offline/fast reload.[web:41]

Ví dụ dynamic import:
```jsx
import dynamic from 'next/dynamic';

const HeavyChart = dynamic(() => import('../components/HeavyChart'), {
  ssr: false,
  loading: () => <p>Loading chart...</p>,
});
```

#### 3. Image optimization, responsive images, font loading

Hình ảnh thường là nguyên nhân chính khiến LCP chậm. Kỹ thuật:
- Dùng định dạng modern (WebP, AVIF).
- Dùng `srcset` và `sizes` để tải ảnh phù hợp device.
- Lazy‑loading cho ảnh ngoài viewport (`loading="lazy"`).
- Giới hạn số font custom, preload font critical, tránh layout shift khi font load.[web:32][web:35]

Ví dụ responsive image:
```html
<img
  src="/images/hero-800.jpg"
  srcset="/images/hero-400.jpg 400w, /images/hero-800.jpg 800w, /images/hero-1200.jpg 1200w"
  sizes="(max-width: 600px) 400px, (max-width: 900px) 800px, 1200px"
  alt="Hero"
/>
```

#### 4. Profiling React components và tránh re-render thừa

Profiler của React DevTools giúp bạn xem component nào render nhiều, tốn bao nhiêu thời gian. Các kỹ thuật giảm re-render:
- Memo hoá component (`React.memo`) khi props ít thay đổi.
- Dùng `useMemo` / `useCallback` cho computation/function nặng.
- Tránh tạo object/array mới trong JSX mỗi render nếu không cần thiết.[web:42]

Ví dụ dùng `React.memo`:
```jsx
const UserRow = React.memo(function UserRow({ user, onSelect }) {
  return (
    <tr onClick={() => onSelect(user.id)}>
      <td>{user.name}</td>
      <td>{user.email}</td>
    </tr>
  );
});
```

#### 5. Case study: tối ưu Lighthouse từ 40 → 90+

Flow thực tế:
1. Chạy Lighthouse → ghi lại LCP, CLS, TBT.
2. Phân tích coverage: loại bỏ CSS/JS không dùng, tách vendor.[web:41][web:44]
3. Tối ưu hình ảnh: resize, WebP, lazy load.
4. Giảm JS blocking: code splitting, defer script, loại bỏ thư viện thừa.
5. Fix CLS: đặt kích thước cố định cho ảnh/ads, skeleton để giữ layout.

Chỉ cần giải quyết vài điểm nghẽn lớn (ảnh hero, bundle quá to, JS blocking), bạn đã có thể nâng điểm Lighthouse đáng kể.[web:41][web:38]

---

### Chương 11 – Testing frontend

#### 1. Testing pyramid: unit, integration, E2E

Testing pyramid trên frontend thường gồm:
- **Unit test**: test hàm, component nhỏ, logic đơn lẻ.
- **Integration test**: test nhiều component/logic kết hợp (form, flow UI đơn giản).
- **E2E test**: test full flow từ góc nhìn user (login, checkout,…).

Tỉ lệ thường: nhiều unit, ít integration, rất ít E2E – vì E2E chậm và fragile hơn.

#### 2. Jest/Vitest, Testing Library, Cypress/Playwright

- **Jest/Vitest**: framework unit test nhanh cho JS/TS.
- **React Testing Library**: tập trung test behavior từ góc nhìn user (query bằng text, role,…).
- **Cypress/Playwright**: dùng cho E2E, chạy browser thật hoặc headless, có thể record video, screenshot.

Ví dụ unit/integration test với Testing Library:
```jsx
// Counter.jsx
export function Counter() {
  const [value, setValue] = useState(0);
  return (
    <div>
      <span>{value}</span>
      <button onClick={() => setValue((v) => v + 1)}>Increment</button>
    </div>
  );
}
```
```js
// Counter.test.jsx
import { render, screen, fireEvent } from '@testing-library/react';
import { Counter } from './Counter';

test('increments counter', () => {
  render(<Counter />);
  fireEvent.click(screen.getByText('Increment'));
  expect(screen.getByText('1')).toBeInTheDocument();
});
```

#### 3. Test strategy: test gì, không test gì

- Test **logic quan trọng**: validation, pricing, permission.
- Test **state machine/phần flow**: stepper, checkout, wizard.
- Không test implementation detail (class name, nội bộ hook) quá nhiều; tập trung behavior từ góc nhìn user.
- Hạn chế snapshot test lớn – dễ trở thành noise khi diff.

#### 4. Mock network, contract testing, visual regression

- Mock network bằng MSW (Mock Service Worker) để test UI mà không phụ thuộc backend thật.
- Contract testing: đảm bảo schema API không bị breaking change (kết hợp với backend tests).
- Visual regression dùng tools như Percy, Chromatic để detect layout break sau khi đổi CSS/component.

#### 5. Ví dụ: test form phức tạp

```jsx
// LoginForm.jsx
export function LoginForm({ onSubmit }) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  function handleSubmit(e) {
    e.preventDefault();
    if (!email || !password) return;
    onSubmit({ email, password });
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        placeholder="Email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
      />
      <input
        placeholder="Password"
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
      />
      <button type="submit">Login</button>
    </form>
  );
}
```
```js
// LoginForm.test.jsx
import { render, screen, fireEvent } from '@testing-library/react';
import { LoginForm } from './LoginForm';

test('calls onSubmit with email and password', () => {
  const handleSubmit = jest.fn();
  render(<LoginForm onSubmit={handleSubmit} />);

  fireEvent.change(screen.getByPlaceholderText('Email'), {
    target: { value: 'test@example.com' },
  });
  fireEvent.change(screen.getByPlaceholderText('Password'), {
    target: { value: 'secret' },
  });
  fireEvent.click(screen.getByText('Login'));

  expect(handleSubmit).toHaveBeenCalledWith({
    email: 'test@example.com',
    password: 'secret',
  });
});
```
