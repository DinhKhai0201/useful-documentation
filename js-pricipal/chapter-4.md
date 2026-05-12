### Chương 5 – Bất đồng bộ (async) ở mức sâu

#### 5.1. Event loop, macrotask và microtask

JavaScript chạy trên một thread nhưng vẫn xử lý được nhiều tác vụ bất đồng bộ nhờ **event loop** phối hợp giữa call stack, macrotask queue và microtask queue.[web:64][web:74]

- **Call stack**: nơi chạy code đồng bộ, mỗi lần gọi function sẽ push một frame vào stack và pop ra khi return.
- **Macrotask queue**: chứa các task lớn như `setTimeout`, `setInterval`, I/O, DOM events.
- **Microtask queue**: chứa callback của Promise, `queueMicrotask`, `MutationObserver` – luôn được xử lý **trước** macrotask tiếp theo.[web:64][web:74]

Chu trình event loop cơ bản:
1. Chạy code đồng bộ trong call stack cho đến khi rỗng.
2. Xử lý **tất cả** microtask đang chờ.
3. Cập nhật layout/paint nếu cần.
4. Lấy **một** macrotask từ queue và chạy.
5. Lặp lại.

Ví dụ minh hoạ thứ tự:

```js
console.log('A');

setTimeout(() => console.log('B (timeout)'), 0);

Promise.resolve().then(() => console.log('C (promise)'));

console.log('D');
// Kết quả: A, D, C (promise), B (timeout)
```

Ở đây, `C` là microtask nên được chạy trước macrotask `setTimeout`.[web:64][web:74]

#### 5.2. `setTimeout`, `setInterval`, `requestAnimationFrame`

- `setTimeout(fn, delay)` xếp `fn` vào macrotask queue sau ít nhất `delay` ms (không đảm bảo chính xác tuyệt đối).
- `setInterval(fn, delay)` lặp lại `fn` theo chu kỳ; nếu `fn` chạy lâu hơn `delay` có thể gây chồng chéo, nên ưu tiên tự dùng `setTimeout` lồng nhau để kiểm soát tốt hơn.
- `requestAnimationFrame(fn)` gọi `fn` trước frame render tiếp theo, đồng bộ với refresh rate của màn hình – thích hợp cho animation, tránh giật.[web:26]

Ví dụ tự cài interval an toàn bằng `setTimeout`:

```js
function safeInterval(fn, delay) {
  let cancelled = false;

  function tick() {
    if (cancelled) return;
    const start = performance.now();
    fn();
    const elapsed = performance.now() - start;
    setTimeout(tick, Math.max(0, delay - elapsed));
  }

  setTimeout(tick, delay);

  return () => {
    cancelled = true;
  };
}

// usage
const cancel = safeInterval(() => {
  console.log('Tick at', Date.now());
}, 1000);

// cancel() khi không cần nữa
```

#### 5.3. Promise, chain và combinators

`Promise` là abstraction cho kết quả tương lai của một tác vụ async, với 3 trạng thái: `pending`, `fulfilled`, `rejected`.[web:29]  Promise chain giúp bạn nối các bước async một cách có cấu trúc.

```js
function fetchJson(url) {
  return fetch(url).then((res) => {
    if (!res.ok) throw new Error('Request failed');
    return res.json();
  });
}

fetchJson('/api/user')
  .then((user) => fetchJson('/api/posts?userId=' + user.id))
  .then((posts) => {
    console.log('Posts:', posts);
  })
  .catch((err) => {
    console.error('Error:', err);
  });
```

Các combinator quan trọng:
- `Promise.all([a, b, c])`: chờ tất cả fulfilled, nếu bất kỳ promise reject thì reject ngay.
- `Promise.race([a, b])`: resolve/reject với kết quả promise kết thúc sớm nhất.
- `Promise.any([a, b])`: resolve với promise đầu tiên fulfilled, chỉ reject nếu tất cả đều reject (ES2021).[web:29]

#### 5.4. `async/await`, concurrency vs sequential

`async/await` là syntax sugar trên Promise, giúp code async trông như sync nhưng vẫn không block thread.[web:70][web:75]

```js
async function getUserWithPosts() {
  const user = await fetchJson('/api/user');
  const posts = await fetchJson('/api/posts?userId=' + user.id);
  return { user, posts };
}
```

Code trên chạy **tuần tự**: request thứ hai chỉ bắt đầu sau khi request thứ nhất hoàn thành. Để tận dụng concurrency, bạn có thể:

```js
async function getDashboardData() {
  const userPromise = fetchJson('/api/user');
  const notificationsPromise = fetchJson('/api/notifications');

  const [user, notifications] = await Promise.all([
    userPromise,
    notificationsPromise,
  ]);

  return { user, notifications };
}
```

Ở đây hai request chạy song song (concurrent) trên I/O layer, dù JS vẫn single‑thread. Đây là pattern rất quan trọng để tối ưu latency API call.[web:65][web:70]

#### 5.5. Race condition, cancellation pattern (`AbortController`)

**Race condition** trong frontend thường xuất hiện khi nhiều request độc lập cùng cập nhật chung một state, ví dụ:
- User gõ search, mỗi ký tự gửi một request; response của query cũ đến trễ hơn query mới nhưng lại overwrite kết quả mới.

Pattern xử lý:
- Gắn `requestId` và chỉ set state nếu response có id mới nhất.
- Hoặc dùng `AbortController` để **cancel** request cũ.[web:26]

Ví dụ với `AbortController`:

```js
let controller;

async function search(query) {
  if (controller) controller.abort();
  controller = new AbortController();

  try {
    const res = await fetch('/api/search?q=' + encodeURIComponent(query), {
      signal: controller.signal,
    });
    if (!res.ok) throw new Error('Request failed');
    const data = await res.json();
    renderResults(data);
  } catch (err) {
    if (err.name === 'AbortError') {
      // bị cancel vì query mới, bỏ qua
      return;
    }
    console.error('Search error:', err);
  }
}
```

#### 5.6. Thực chiến: tự cài `promiseAll`

```js
function promiseAll(iterable) {
  return new Promise((resolve, reject) => {
    const promises = Array.from(iterable);
    const results = [];
    let completed = 0;

    if (promises.length === 0) {
      resolve([]);
      return;
    }

    promises.forEach((p, index) => {
      Promise.resolve(p)
        .then((value) => {
          results[index] = value;
          completed += 1;
          if (completed === promises.length) {
            resolve(results);
          }
        })
        .catch(reject); // fail fast
    });
  });
}

// usage
promiseAll([
  Promise.resolve(1),
  new Promise((res) => setTimeout(() => res(2), 100)),
]).then(console.log); // [1, 2]
```

Bài tập: tự cài `promiseRace`, `promiseAny` dựa trên đặc tả behavior của chúng.

---

### Chương 6 – Kiến trúc type với TypeScript

#### 6.1. Vì sao Principal cần nghĩ bằng type

Ở hệ thống lớn, JS thuần rất dễ phát sinh bug do sai shape dữ liệu, refactor field mà quên update, hoặc contract API bị phá vỡ. TypeScript cho phép bạn "đưa logic domain lên level type":
- Model hoá entity, event, payload, config bằng type.
- Ép compiler bắt lỗi khi contract thay đổi.
- Tạo API dev‑friendly: autocomplete, type‑safe, ít cần đọc docs.[web:18][web:21][web:24]

Ở cấp Principal, bạn không chỉ viết TS cho bản thân mà còn **thiết kế loại type patterns** mà cả team sẽ reuse.

#### 6.2. Ôn nhanh: union, intersection, literal, enum

```ts
// Union: nhiều trạng thái khả dĩ
 type Status = 'idle' | 'loading' | 'success' | 'error';

// Intersection: kết hợp nhiều đặc điểm
interface HasId { id: string; }
interface HasTimestamps {
  createdAt: Date;
  updatedAt: Date;
}

 type Entity = HasId & HasTimestamps;

// Enum hoặc literal union
 enum Role {
  User = 'user',
  Admin = 'admin',
}
```

Union + literal type rất hữu ích để model hoá state machine (VD: order status, loan status,…).[web:21]

#### 6.3. Generics, utility types và hàm generic thực chiến

```ts
function identity<T>(value: T): T {
  return value;
}

const a = identity(1);      // T = number
const b = identity('hello'); // T = string

interface ApiResponse<T> {
  data: T;
  error?: string;
}

// Utility types
interface User {
  id: string;
  name: string;
  email?: string;
}

 type UserPatch = Partial<User>;
 type UserRequired = Required<User>;
 type UserPreview = Pick<User, 'id' | 'name'>;
 type UserWithoutEmail = Omit<User, 'email'>;
```

**Thực chiến**: viết hàm generic xử lý nhiều loại entity khác nhau:

```ts
function findById<T extends { id: string }>(items: T[], id: string): T | undefined {
  return items.find((item) => item.id === id);
}
```

#### 6.4. Conditional types, mapped types, `infer`

Conditional types cho phép viết "if/else" trên type:[web:18][web:24]

```ts
 type IsPromise<T> = T extends Promise<any> ? true : false;

 type A = IsPromise<Promise<number>>; // true
 type B = IsPromise<string>;          // false
```

`infer` dùng để trích xuất type nội bộ:

```ts
 type Awaited<T> = T extends Promise<infer U> ? U : T;

 type R1 = Awaited<Promise<string>>; // string
 type R2 = Awaited<number>;          // number
```

Mapped types biến đổi đồng loạt properties của một type:

```ts
 type ReadonlyDeep<T> = {
  readonly [K in keyof T]: T[K];
};

 type Optional<T> = {
  [K in keyof T]?: T[K];
};
```

Kết hợp conditional + mapped types, bạn có thể xây các helper rất mạnh (router type-safe, form builder, query client,…).[web:21][web:24]

#### 6.5. Kiến trúc type‑safe cho API layer, event và domain

Ví dụ: định nghĩa type cho API response và sử dụng trong client:

```ts
// api-types.ts
export interface UserDto {
  id: string;
  name: string;
  email: string;
}

export interface PostDto {
  id: string;
  title: string;
  body: string;
  authorId: string;
}

// api-client.ts
async function getJson<T>(url: string, init?: RequestInit): Promise<T> {
  const res = await fetch(url, init);
  if (!res.ok) throw new Error('Request failed');
  return res.json() as Promise<T>;
}

export const Api = {
  getUser(id: string) {
    return getJson<UserDto>(`/api/users/${id}`);
  },
  getPostsByUser(userId: string) {
    return getJson<PostDto[]>(`/api/users/${userId}/posts`);
  },
};
```

Khi backend đổi field (VD: đổi `authorId` → `userId`), bạn update `PostDto` và để compiler báo hết mọi chỗ sử dụng cần sửa.

Bạn cũng có thể định nghĩa type cho event payload:

```ts
 type UserEvents =
  | { type: 'USER_CREATED'; payload: { id: string; email: string } }
  | { type: 'USER_SUSPENDED'; payload: { id: string; reason: string } };

function handleUserEvent(event: UserEvents) {
  switch (event.type) {
    case 'USER_CREATED':
      // event.payload.email có type an toàn
      break;
    case 'USER_SUSPENDED':
      // event.payload.reason
      break;
  }
}
```

#### 6.6. `Result<T, E>` pattern cho error handling

Pattern `Result` giúp bạn model hoá operations có thể fail mà không phụ thuộc hoàn toàn vào exception:

```ts
export type Ok<T> = {
  ok: true;
  value: T;
};

export type Err<E> = {
  ok: false;
  error: E;
};

export type Result<T, E> = Ok<T> | Err<E>;

export function ok<T>(value: T): Ok<T> {
  return { ok: true, value };
}

export function err<E>(error: E): Err<E> {
  return { ok: false, error };
}
```

Dùng trong API client:

```ts
async function fetchUser(id: string): Promise<Result<UserDto, 'NotFound' | 'NetworkError'>> {
  try {
    const res = await fetch(`/api/users/${id}`);
    if (res.status === 404) return err('NotFound');
    if (!res.ok) return err('NetworkError');
    const data = (await res.json()) as UserDto;
    return ok(data);
  } catch {
    return err('NetworkError');
  }
}

async function showUserProfile(id: string) {
  const result = await fetchUser(id);

  if (!result.ok) {
    if (result.error === 'NotFound') {
      // show 404 UI
    } else {
      // show generic error
    }
    return;
  }

  renderUser(result.value);
}
```

Pattern này rất hợp với codebase lớn: flow rõ ràng, dễ test, không phụ thuộc throw/catch tràn lan, và compiler đảm bảo bạn xử lý hết các nhánh `error`.

---

## Phần V – JavaScript trong Web3 (bắt đầu)

### Chương 21 – Kiến trúc dApp: từ UI tới on-chain

#### 21.1. Các layer chính trong dApp

Một dApp điển hình có các layer:
- **UI (React/Next.js)**: form, table, chart, dashboard.
- **Web3 client (ethers.js, viem, wagmi)**: đọc/ghi dữ liệu từ/to smart contract.[web:66][web:71]
- **Smart contract (Solidity)**: logic business on-chain (lending, AMM, NFT,…).
- **Indexer / data layer (The Graph, custom indexer)**: query data đã tiền xử lý qua GraphQL/REST.

Tách rõ các layer này giúp code dễ test, dễ migrate chain và dễ tối ưu hiệu năng query.[web:67][web:72]

#### 21.2. Flow giao dịch từ UI đến contract và ngược lại

Flow cơ bản khi user tương tác dApp:
1. User điền form trên UI (số lượng, token, leverage,…).
2. Frontend validate dữ liệu, show estimate gas, rate.
3. Frontend tạo transaction (call contract function) qua web3 client.
4. Wallet (MetaMask, WalletConnect, smart wallet) hiển thị popup confirm.
5. Transaction được gửi lên network, chờ mined.
6. Sau khi mined, contract emit events → indexer hoặc frontend lắng nghe → UI cập nhật state.[web:66][web:71]

Trong code JS/TS, bạn thường sẽ:
- Có `onchainClient` để wrap ethers/viem.
- Có `indexerClient` để đọc dữ liệu aggregate (positions, volume, APY,…).

Ở bước tiếp theo, mình có thể viết đầy đủ **Chương 21–24** (Web3) với ví dụ thực chiến dùng `ethers.js` + The Graph + verify chữ ký EIP-712 trên backend nếu bạn muốn đào sâu phần Web3.
