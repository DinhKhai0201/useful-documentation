# Web Frontend Deep Dive – Bản Thực Chiến (React + Next.js)

> Mục tiêu: Sau tài liệu này, bạn **hiểu được cơ chế nền tảng** (JS, CSS, Browser, Framework) và **biết áp dụng** vào ứng dụng React/Next.js thực tế (SPA + SSR/SSG/Streaming + SSE).
>
> Cấu trúc mỗi chủ đề:
> - Lý thuyết lõi (mental model)
> - Liên hệ tới React/Next
> - Ví dụ code (tối thiểu 1 ví dụ nhỏ + 1 ví dụ thực tế)
> - Lỗi thường gặp
> - Bài tập thực chiến / checklist

---

## 0. Cách học với tài liệu này

1. Đọc qua **lý thuyết lõi** để có mental model.
2. Chạy và sửa **ví dụ nhỏ** (copy vào sandbox/Vite app).
3. Áp dụng **bài tập thực chiến** vào chính project của bạn (hoặc project nhỏ mới).
4. Cuối cùng, dùng **checklist** để review code của chính bạn.

---

## 1. JavaScript Core Mechanics

### 1.1 Runtime model & Event Loop: JS không "đa luồng", nhưng môi trường thì có

**Mental model:**

- JS engine chỉ có **một call stack** → luôn chạy **một đoạn code** tại một thời điểm.
- Bất cứ thứ gì async (timers, fetch, DOM events, message channel, …) là do **Web APIs** của browser hoặc APIs của Node.
- Các công việc async sau khi hoàn thành sẽ xếp vào:
  - **Task queue (macrotask queue)**: `setTimeout`, `setInterval`, DOM events, `postMessage`, v.v.
  - **Microtask queue**: `Promise.then/catch/finally`, `queueMicrotask`, `MutationObserver`.[cite:59][cite:56]
- **Event loop** chọn công việc từ các hàng đợi này, đảm bảo:
  1. Chạy xong 1 task trên call stack.
  2. Xử lý **tất cả microtask** hiện có.
  3. Cho browser cơ hội **layout + paint** frame tiếp theo.

**Ví dụ (thứ tự log):**

```js
console.log('A');

setTimeout(() => console.log('B - macrotask'), 0);

Promise.resolve()
  .then(() => console.log('C - microtask 1'))
  .then(() => console.log('D - microtask 2'));

console.log('E');
```

Thứ tự: `A, E, C, D, B`.

Giải thích:
- Script hiện tại là **1 macrotask**.
- `setTimeout` đặt callback vào task queue.
- `Promise.then` tạo microtask. Theo MDN, microtask được xử lý sau khi call stack rỗng, **trước** khi lấy task mới từ queue.[cite:59][cite:54]

**Liên hệ React/Next:**

- Mọi thứ trong React (render, effect, setState) vẫn chạy trên single thread.
- Khi bạn `await` trong event handler, control được trả về event loop; nếu phía sau có long task, UI có thể bị block, ảnh hưởng **INP**.
- Khi hiểu event loop, bạn hiểu:
  - Vì sao logs từ `useEffect` / `setTimeout` / `Promise.then` xuất hiện theo thứ tự lạ.
  - Khi nào nên dùng microtask (`queueMicrotask`) vs task (`setTimeout`) để chờ DOM update/hydration.

**Bài tập:**

1. Tự viết vài ví dụ nhỏ trộn `setTimeout`, `Promise`, `queueMicrotask`, đoán thứ tự log → chạy thử.
2. Trong 1 component React, thử:
   - `useEffect(() => console.log('effect'))`
   - thêm `Promise.resolve().then(() => console.log('microtask'))`
   - thêm `setTimeout(..., 0)`
   - Quan sát thứ tự log khi component mount.

---

### 1.2 Closures: tại sao stale closure phá state trong React

**Định nghĩa chặt:**

Closure xảy ra khi một function **ghi nhớ** lexical environment (biến trong scope cha) tại thời điểm nó được tạo, và còn được dùng sau khi scope cha đã kết thúc.

Ví dụ:

```js
function makeCounter() {
  let n = 0;
  return () => {
    n++;
    return n;
  };
}

const c1 = makeCounter();
console.log(c1()); // 1
console.log(c1()); // 2
```

`c1` là closure “giữ” biến `n`.

**Trong React function component:**

Mỗi lần render, bạn có:

```tsx
function Counter() {
  const [count, setCount] = useState(0);

  console.log('render with count =', count);

  function handleClick() {
    setCount(count + 1);
  }

  return <button onClick={handleClick}>{count}</button>;
}
```

- Mỗi render tạo **phiên bản mới** của `handleClick`, và mỗi phiên bản "đóng băng" `count` tại thời điểm render đó.
- Khi click nhiều lần **trên cùng 1 render**, React sẽ lên lịch nhiều update, nhưng với concurrent features, không đảm bảo tất cả được xử lý ngay.

**Stale closure trong effect/timer:**

```tsx
function Countdown() {
  const [count, setCount] = useState(10);

  useEffect(() => {
    const id = setInterval(() => {
      // ❌ count ở đây là của render đầu tiên
      setCount(count - 1);
    }, 1000);
    return () => clearInterval(id);
  }, []);

  return <div>{count}</div>;
}
```

Sau 1s, `count` sẽ luôn tính từ giá trị ban đầu (10) vì closure.

**Cách đúng:** dùng functional update hoặc ref:

```tsx
useEffect(() => {
  const id = setInterval(() => {
    setCount(prev => prev - 1); // ✅ prev luôn là state mới nhất
  }, 1000);
  return () => clearInterval(id);
}, []);
```

Hoặc:

```tsx
const countRef = useRef(10);
const [count, setCount] = useState(countRef.current);

useEffect(() => {
  const id = setInterval(() => {
    countRef.current -= 1;
    setCount(countRef.current);
  }, 1000);
  return () => clearInterval(id);
}, []);
```

**Checklist tránh stale closure trong React:**

- Với state update phụ thuộc vào state cũ → dùng **functional update**.
- Với effect có dependency array `[]` nhưng dùng state/props →
  - Hoặc chuyển sang dùng ref / functional update.
  - Hoặc thêm vào dependency array (khi thật sự cần re-run).

**Bài tập:**

- Viết custom hook `useInterval` chuẩn, tránh stale closure.
- Viết `useEvent` (như React 19/`useEffectEvent`) để tạo callback luôn thấy state mới nhất.

---

### 1.3 `this` binding: lỗi kinh điển khi tách method khỏi object

**Quy tắc ưu tiên (từ cao xuống thấp):**

1. **`new` binding** – nếu function được gọi bằng `new`, `this` là object mới.
2. **Explicit binding** – `.call`, `.apply`, `.bind`.
3. **Implicit binding** – gọi qua object: `obj.method() → this = obj`.
4. **Default binding** – trong strict mode: `this = undefined` nếu không match case nào.
5. **Arrow function** – không có `this` riêng, nó "mượn" `this` từ scope ngoài (lexical this).

Ví dụ bug khi destructure:

```js
const user = {
  name: 'Alice',
  say() { console.log(this.name); }
};

const say = user.say;
say(); // undefined (strict mode)

user.say(); // 'Alice'
```

Trong React:

- Class components thường phải `this.handle = this.handle.bind(this)` hoặc dùng arrow class fields: `handle = () => { ... }`.
- Trong function component, biến mất nhu cầu `this`, nhưng bạn vẫn nên biết để debug thư viện cũ/class.

**Bài tập:**

- Viết một class nhỏ với method dùng `this`, thử gọi sai cách và sửa bằng `bind`.

---

### 1.4 Prototypes: tại sao mọi thứ đều có `toString`, array có `map`

**Cơ chế:**

- Mỗi object có link ẩn `[[Prototype]]`.
- Khi truy cập `obj.prop`:
  1. Tìm trên chính `obj`.
  2. Nếu không có, tìm lên `obj.[[Prototype]]` (prototypal chain).

Ví dụ đơn giản:

```js
const base = { greet() { return 'hi'; } };
const o = Object.create(base);

console.log(o.greet()); // 'hi'
```

`o.__proto__ === base` (không nên dùng trực tiếp, chỉ để minh họa).

Trong `class`:

```js
class Person {
  constructor(name) {
    this.name = name;
  }
  sayHi() {
    return `Hi, I'm ${this.name}`;
  }
}

const p = new Person('A');
console.log(p.sayHi());
```

Internally, `sayHi` được lưu trên `Person.prototype`, các instance share chung method.

**Liên hệ React:**

- React component class cũng chỉ là function + prototype methods; hiểu điều này giúp đọc code các thư viện cũ dễ hơn.

---

### 1.5 Async patterns & tránh waterfall

**Tư duy:**

- `async/await` là sugar trên `Promise` + microtask queue.
- Lỗi phổ biến không phải ở syntax mà ở **pattern** – gọi API theo kiểu tuần tự không cần thiết.

**Ví dụ xấu:**

```ts
async function loadPage() {
  const user = await fetchUser();
  const posts = await fetchPosts(user.id); // nếu thực sự không phụ thuộc có thể song song
}
```

**Ví dụ đúng (song song):**

```ts
async function loadPage(userId: string) {
  const [user, posts] = await Promise.all([
    fetchUser(userId),
    fetchPosts(userId),
  ]);
}
```

**Trong Next.js App Router:**

```tsx
// app/page.tsx

async function Page() {
  const userPromise = fetchUser();
  const postsPromise = fetchPosts();

  const [user, posts] = await Promise.all([userPromise, postsPromise]);

  return <UI user={user} posts={posts} />;
}
```

Next docs khuyến khích kiểu fetch song song để tránh waterfall khi SSR/Streaming.[cite:30][cite:36]

---

## 2. TypeScript – Lớp Type trên Runtime JS

### 2.1 Structural typing & design theo "shape"

TS dùng **structural type system**: nếu hai type có cùng cấu trúc, chúng tương thích.

Ví dụ:

```ts
type ApiUser = { id: string; name: string };
type UserModel = { id: string; name: string };

const apiUser: ApiUser = { id: '1', name: 'Alice' };
const u: UserModel = apiUser; // OK
```

**Ứng dụng:**
- Dùng type alias / interface để biểu diễn **data contract**, không gắn chặt vào class.

### 2.2 Narrowing, type guards & discriminated unions

**Discriminated union** là pattern cực hữu ích để encode state.

```ts
type Loading = { status: 'loading' };
type ErrorState = { status: 'error'; message: string };
type Success<T> = { status: 'success'; data: T };

type RemoteData<T> = Loading | ErrorState | Success<T>;

function renderUser(state: RemoteData<{ name: string }>) {
  switch (state.status) {
    case 'loading':
      return 'Loading...';
    case 'error':
      return state.message;
    case 'success':
      return state.data.name;
    default:
      const _exhaustive: never = state;
      throw new Error('Unhandled state: ' + _exhaustive);
  }
}
```

- `status` là discriminant giúp TS narrow type trong từng case.
- Dùng `never` check giúp bạn không quên handle state mới.

**Áp dụng React/Next:**

- Encode request state (loading/error/success) thành một type thay vì `boolean` rời rạc (`isLoading`, `error`, `data`).

### 2.3 Generics & inference

Generics cho phép giữ **quan hệ** giữa input và output.

```ts
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

const n = first([1, 2, 3]);     // T inferred là number
const s = first(['a', 'b']);    // T inferred là string
```

Trong React:

```tsx
interface SelectProps<T> {
  options: T[];
  getLabel: (opt: T) => string;
  value: T | null;
  onChange: (value: T | null) => void;
}

function Select<T>({ options, getLabel, value, onChange }: SelectProps<T>) {
  // ...
}
```

### 2.4 `any`, `unknown`, `never`, `satisfies`

- **`any`**: tắt kiểm tra type, tránh dùng nếu không bắt buộc.
- **`unknown`**: buộc bạn phải narrow trước khi dùng.

```ts
function parseJSON(input: string): unknown {
  return JSON.parse(input);
}

const result = parseJSON('{"x": 1}');
if (typeof result === 'object' && result && 'x' in result) {
  // safe để dùng (narrowed)
}
```

- **`never`**: biểu diễn giá trị không tồn tại, dùng để check exhaustiveness.
- **`satisfies`**: object literals thỏa interface nhưng vẫn giữ literal types.

```ts
const theme = {
  primary: '#2563eb',
  danger: '#dc2626',
} as const satisfies Record<string, string>;
```

**Bài tập:**

- Định nghĩa type cho một form multi-step (step, data theo step, trạng thái valid/invalid) dùng discriminated union.

---

## 3. Modern CSS – Tư duy Component-first

### 3.1 Flexbox vs Grid – chọn theo bài toán

**Flexbox:**

- Một trục chính (main axis): row hoặc column.
- Case điển hình: navbar, button group, card nội bộ.

**Grid:**

- Hàng + cột (2D).
- Case: layout page, dashboard, gallery.

**Ví dụ thực chiến (layout đơn giản bằng Grid + Flex):**

```css
.app-layout {
  min-height: 100vh;
  display: grid;
  grid-template-rows: auto 1fr auto;
}

.app-header,
.app-footer {
  padding: 1rem;
  background: #0f172a;
  color: white;
}

.app-main {
  display: grid;
  grid-template-columns: 250px 1fr;
}

.sidebar {
  border-right: 1px solid #e5e7eb;
}

.content {
  padding: 1rem;
}

.nav {
  display: flex;
  gap: 0.75rem;
  align-items: center;
}
```

- Header/footer/grid layout dùng Grid.
- Bên trong header, `.nav` dùng Flex.

### 3.2 Cascade layers (`@layer`): chấm dứt chiến tranh specificity

**Vấn đề:** Codebase lớn thường có:

- reset.css
- component.css
- utility.css

Nếu không kiểm soát, bạn sẽ dùng `!important` khắp nơi.

**Giải pháp `@layer`:** định nghĩa thứ tự cascade.

```css
@layer reset, base, components, utilities;

@layer reset {
  *, *::before, *::after { box-sizing: border-box; }
}

@layer base {
  body { margin: 0; font-family: system-ui, sans-serif; }
}

@layer components {
  .btn {
    padding: 0.5rem 1rem;
    border-radius: 0.375rem;
  }
}

@layer utilities {
  .text-center { text-align: center; }
}
```

- Dù `.text-center` có specificity thấp, nó vẫn override `.btn { text-align: left }` nếu `utilities` nằm sau `components` trong khai báo layer.[cite:21][cite:18]

### 3.3 Container queries: responsive theo container, không theo viewport

**Core idea:**

- Thay vì: "khi viewport > 1024px thì card xếp 2 cột".
- Ta dùng: "khi **chính card** rộng > 420px thì xếp 2 cột".

**Cú pháp:**

1. Đánh dấu container:

```css
.card-shell {
  container-type: inline-size;
}
```

2. Dùng `@container`:

```css
.card {
  display: grid;
  grid-template-columns: 1fr;
  gap: 12px;
}

@container (min-width: 420px) {
  .card {
    grid-template-columns: 160px 1fr;
  }
}
```

MDN: Container queries cho phép style phần tử dựa trên kích cỡ container ancestor thay vì viewport.[cite:64][cite:62][cite:66]

**Trong component React:**

Chỉ cần đảm bảo wrapper có class `card-shell` và CSS trên, không phải viết JS.

### 3.4 Logical properties: chuẩn bị cho multi-language, RTL

Thay vì:

```css
.card { padding-left: 1rem; padding-right: 1rem; }
```

Dùng:

```css
.card { padding-inline: 1rem; }
```

`inline` tương ứng với hướng chữ: LTR → trái-phải, RTL → phải-trái.

Giúp:

- Dễ hỗ trợ Arabic/Hebrew.
- Suy nghĩ theo "khối" (block/inline) hơn là left/right/top/bottom.

### 3.5 `:has()` – selector quan hệ ngược

Theo MDN, `:has()` là pseudo-class chọn element nếu selector bên trong match descendant/relative selector.[cite:69][cite:63]

Ví dụ: highlight card khi có checkbox được tick.

```css
.card:has(input[type="checkbox"]:checked) {
  border-color: #22c55e;
  background: #f0fdf4;
}
```

Bạn vừa tạo **interactive style** mà không cần JS.

**Pattern mạnh:** "parent phản ứng với child".

### 3.6 Scroll-driven animations: animation gắn với scroll timeline

MDN: scroll-driven animations map tiến độ scroll vào timeline, cho phép animation phụ thuộc scroll position thay vì thời gian.[cite:11][cite:20][cite:23]

Ví dụ: progress bar theo scroll trang.

```css
@keyframes grow { from { transform: scaleX(0); } to { transform: scaleX(1); } }

.progress {
  position: fixed;
  inset: 0 0 auto;
  height: 4px;
  background: #0ea5e9;
  transform-origin: left;
  transform: scaleX(0);

  animation-name: grow;
  animation-duration: auto;
  animation-timing-function: linear;
  animation-timeline: scroll(root block);
}
```

Không cần JS `scroll` event → nhẹ hơn, tránh layout thrashing.

**Bài tập:**

- Tạo section có hình ảnh fade-in/out theo khi nó đi vào viewport dùng view-timeline.

---

## 4. Responsive & Adaptive Design

### 4.1 Mobile-first: CSS và cả thiết kế

- Bắt đầu từ luồng nội dung cho màn nhỏ (320–480px).
- Trong CSS: viết style **mặc định** cho mobile, rồi nâng cấp với `min-width`.

```css
.card-list {
  display: grid;
  gap: 1rem;
}

@media (min-width: 768px) {
  .card-list {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
}

@media (min-width: 1024px) {
  .card-list {
    grid-template-columns: repeat(3, minmax(0, 1fr));
  }
}
```

### 4.2 Fluid typography với `clamp()`

```css
:root {
  --font-base: clamp(14px, 1.2vw + 10px, 18px);
}

body { font-size: var(--font-base); }

h1 {
  font-size: clamp(1.75rem, 3vw + 1rem, 2.5rem);
}
```

- Nhỏ: text không bé hơn 14px.
- To: không vượt quá 18px.
- Giữa: scale mượt theo viewport.

### 4.3 Breakpoints thông minh

- Đặt breakpoint nơi **layout gãy**, không theo kích thước device cụ thể.
- Tận dụng **container queries** để layout component linh hoạt.

**Bài tập:**

- Chọn một component hiện tại bạn đang dùng Tailwind/media-query và rewrite lại dùng container queries + `clamp()` cho typography.

---

## 5. Browser Rendering & Performance

### 5.1 Critical Rendering Path (CRP)

Pipeline đơn giản:

1. Download & parse HTML → DOM.
2. Download & parse CSS → CSSOM.
3. DOM + CSSOM → Render Tree.
4. Layout (reflow) → Paint → Composite.

Tất cả CSS blocking sẽ trì hoãn bước 3; JS sync lớn block step 1.

**Tối ưu trong React/Next:**

- Dùng SSR/SSG để gửi HTML sẵn, giảm blank screen.
- Sử dụng code splitting để tránh gửi cả app JS lớn khi chỉ cần 1 route.
- Đặt CSS critical inline (hoặc sử dụng framework đã tối ưu sẵn).

### 5.2 Reflow vs repaint vs composite

- **Reflow (layout):** thay đổi ảnh hưởng kích thước/vị trí → browser phải tính lại layout.
- **Repaint:** thay đổi màu, background.
- **Composite:** thay đổi transform/opacity trên layer GPU (thường rẻ nhất).

**Practical tips:**

- Animate bằng `transform` + `opacity`, tránh `top/left/width/height`.
- Gom các thay đổi DOM vào 1 batch (tránh đọc layout giữa các lần ghi).

### 5.3 Core Web Vitals (LCP / INP / CLS)

Google dùng 3 metrics chính để đánh giá UX.[cite:57][cite:60]

- **LCP (Largest Contentful Paint)**: thời gian element nội dung lớn nhất xuất hiện (thường là hero image/title). Mục tiêu ≤ 2.5s.[cite:22]
- **INP (Interaction to Next Paint)**: đo độ trễ của *tất cả* interactive events trong vòng đời trang, thay thế FID. Mục tiêu ≤ 200ms.[cite:16][cite:51]
- **CLS (Cumulative Layout Shift)**: tổng dịch chuyển layout không mong đợi. Mục tiêu ≤ 0.1.

**Chiến lược Next.js:**

- LCP:
  - Dùng `<Image>` của Next cho hero image, set kích thước/aspect-ratio rõ ràng.
  - Preload font critical.
  - Dùng `priority` cho ảnh trên fold.
- INP:
  - Tránh tạo mega-component render lại toàn trang mỗi khi click.
  - Tách logic nặng (sorting, filtering) ra Web Worker nếu cần.
- CLS:
  - Set `width/height` hoặc `aspect-ratio` cho mọi ảnh.
  - Reserve chỗ cho banner/ads.

### 5.4 Lazy loading, code splitting

- `loading="lazy"` cho ảnh bên dưới fold.
- `dynamic(() => import('./HeavyComponent'), { ssr: false })` trong Next cho phần chỉ render client.
- Module-level splitting: tách editor/chart library khỏi bundle chính.

### 5.5 Memoization trong React: khi nào dùng `useMemo` / `useCallback`

**Dùng khi:**

- Tính toán đắt (sorting/filtering list lớn).
- Tránh tạo object/function mới mỗi render *khi* nó làm child `React.memo` re-render quá nhiều.

**Ví dụ đúng:**

```tsx
const sorted = useMemo(
  () => items.slice().sort((a, b) => a.name.localeCompare(b.name)),
  [items]
);
```

**Sai thường gặp:**

```tsx
// Không có child memo, không dùng dependency đặc biệt → useCallback vô nghĩa
const handleClick = useCallback(() => setOpen(true), []);
```

Hãy **đo** trước bằng React Profiler / DevTools, đừng dùng memoization theo cảm tính.

---

## 6. Accessibility (A11y) – Không Thỏa Hiệp

### 6.1 Semantic HTML trước, ARIA sau

- Dùng `button`, `a`, `nav`, `main`, `header`, `footer`, `section`, `form`, `label`.
- Chỉ dùng ARIA khi HTML không có element tương đương.

Ví dụ xấu:

```jsx
<div onClick={onClick}>Submit</div>
```

Ví dụ đúng:

```jsx
<button type="button" onClick={onClick}>Submit</button>
```

Bạn được miễn phí:
- keyboard interaction (Enter/Space)
- focus outline
- semantics cho screen reader

### 6.2 Keyboard navigation & focus management (đặc biệt cho modal)

**Modal chuẩn cần:**

- Trap focus bên trong.
- Focus vào element đầu tiên khi mở.
- Đóng bằng Esc.
- Trả focus về nút mở modal khi đóng.

Trong React:

- Dùng `focus()` trong `useEffect` sau mount.
- Dùng `tabindex="-1"` cho container nếu cần focus programmatic.

### 6.3 Screen reader & color contrast

- Dùng tooling: Axe, Lighthouse để scan lỗi cơ bản.
- Màu: đảm bảo contrast theo WCAG (thường ≥ 4.5:1 cho nội dung chính).
- Dùng `aria-live="polite"` cho thông báo async (VD: save thành công, error).

**Bài tập:**

- Lấy component hiện có (VD: custom select) → kiểm tra tab navigation, screen reader (NVDA/VoiceOver), sửa cho chuẩn.

---

## 7. Client-side Security Basics

### 7.1 XSS Prevention

**XSS = chạy JavaScript của attacker trong origin của bạn.**

Chống XSS trên client:

- Không dùng `innerHTML` với data chưa sanitize.
- Trong React, tránh `dangerouslySetInnerHTML` trừ khi đã dùng thư viện sanitize.
- Encode output đúng context (HTML, attribute, URL, JS).

### 7.2 Content Security Policy (CSP)

MDN: CSP là cơ chế browser để chỉ định nguồn tài nguyên được phép load, giúp giảm rủi ro XSS bằng cách cho phép chỉ script từ nguồn tin cậy.[cite:34][cite:49]

OWASP CSP Cheat Sheet khuyến nghị dùng nonce/hash thay vì allowlist domain rộng.[cite:40]

Ví dụ header CSP đơn giản (minh họa):

```http
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'nonce-<RANDOM>'; 
  object-src 'none';
  base-uri 'none';
```

- Inline script phải có `nonce` hợp lệ mới chạy.
- Kết hợp với encode/sanitize để defense-in-depth.[cite:31][cite:28]

### 7.3 Secure cookies & token storage

- `HttpOnly`: JS không đọc được cookie (chống XSS đánh cắp token).
- `Secure`: chỉ gửi cookie qua HTTPS.
- `SameSite`: chống CSRF (Lax/Strict).

Token:
- Nếu threat model có XSS, ưu tiên lưu token trong cookie HttpOnly.
- Nếu buộc dùng localStorage: cần CSP + sanitize + audit kỹ.

### 7.4 CORS – hiểu cho đúng

- CORS là **policy của browser**, không phải cơ chế auth.
- Server config `Access-Control-Allow-Origin`, `Allow-Credentials`, `Allow-Headers` để cho phép origin cụ thể.
- Tuyệt đối tránh pattern "`Access-Control-Allow-Origin: *` + `credentials: true`".

---

## 8. Component Architecture & Composition

### 8.1 Prop drilling: mùi hôi của kiến trúc

Dấu hiệu:

- Props đi từ `App` tới `Page` → `Layout` → `Section` → `Card` chỉ để `Card` dùng.
- Khi muốn thêm 1 props nữa phải pass qua 5 nơi.

Giải pháp:

- **Composition:**

```tsx
<Card>
  <Card.Header>
    <UserAvatar />
  </Card.Header>
  <Card.Body>
    <UserDetails />
  </Card.Body>
</Card>
```

- **Context:** share state xuống dưới mà không phải pass props.
- **Store (Zustand/Jotai):** khi nhiều phần của tree cần cùng state và update liên tục.

### 8.2 Compound Components

Ví dụ `Tabs` thực chiến:

```tsx
<Tabs value={tab} onValueChange={setTab}>
  <Tabs.List>
    <Tabs.Trigger value="overview">Overview</Tabs.Trigger>
    <Tabs.Trigger value="settings">Settings</Tabs.Trigger>
  </Tabs.List>
  <Tabs.Content value="overview">...</Tabs.Content>
  <Tabs.Content value="settings">...</Tabs.Content>
</Tabs>
```

Implementation outline:

```tsx
const TabsContext = createContext<...>(...);

function TabsRoot(props) {
  const [value, setValue] = useControllableState(props);
  const ctx = useMemo(() => ({ value, setValue }), [value]);
  return (
    <TabsContext.Provider value={ctx}>
      {props.children}
    </TabsContext.Provider>
  );
}

function TabsTrigger({ value, children }) {
  const { value: active, setValue } = useContext(TabsContext);
  const selected = active === value;
  return (
    <button
      aria-selected={selected}
      role="tab"
      onClick={() => setValue(value)}
    >
      {children}
    </button>
  );
}
```

**Lợi ích:**

- API tự nhiên, gần semantics.
- Dễ thêm variant (vertical, with icons) mà không đổi core logic.

### 8.3 Từ render props → hooks → composition

- **Render props**: `<DataProvider render={data => <UI data={data} /> } />`.
- **Hooks**: `const data = useData();`, logic tách khỏi UI, linh hoạt hơn.
- **Composition**: nhiều hooks kết hợp để tạo behavior complex.

Bài tập:

- Refactor một component dùng render props sang custom hook + compound components.

---

## 9. State Management Patterns

### 9.1 Phân loại state trước khi chọn library

1. **Local UI state:** modal open/close, input value.
2. **Server cache state:** data từ API có thể refetch, stale.
3. **Global app state:** auth user, theme.
4. **URL state:** filters, pagination.

Sai lầm kinh điển: ném hết vào Redux.

### 9.2 Khi nào Context đủ, khi nào không

Context tốt cho:

- Theme, locale, feature flags.
- Auth user (id, role) *đổi ít*.

Không tốt cho:

- State update liên tục (typing, drag-drop) → re-render toàn bộ consumer.

### 9.3 Zustand / Jotai / Redux Toolkit

**Zustand:**

- Store nhỏ gọn, hooks API.
- Hợp cho state UI phức tạp (canvas, viewport, audio player, v.v.).

**Jotai:**

- Mỗi atom là state riêng; component chỉ re-render khi atom nó dùng đổi.

**Redux Toolkit:**

- Hợp với app lớn, cần:
  - time-travel debugging
  - logging middleware
  - predictable reducer logic

### 9.4 Server Components & "đẩy state về server"

Next App Router khuyến khích data fetching trong Server Components; docs đưa ra các pattern fetch song song và chia nhỏ thành các segment để tối ưu.[cite:30][cite:27][cite:36]

Lợi ích:
- Giảm lượng state data phải giữ trên client.
- Dễ tận dụng caching ở server.

---

## 10. Testing Culture

### 10.1 Testing pyramid thực tế

- **Unit tests:** thuần function/hook, không đụng DOM.
- **Integration tests (RTL):** render component thật + bấm click, nhập text.
- **E2E tests (Cypress/Playwright):** chạy trên browser thật, test full flow.
- **Visual regression:** snapshot UI.

### 10.2 React Testing Library (RTL) + Jest/Vitest

**Nguyên tắc:** test behavior qua mắt user.

```tsx
test('user can login', async () => {
  render(<LoginForm />);

  await user.type(screen.getByLabelText(/email/i), 'user@test.com');
  await user.type(screen.getByLabelText(/password/i), '123456');
  await user.click(screen.getByRole('button', { name: /login/i }));

  expect(await screen.findByText(/welcome/i)).toBeInTheDocument();
});
```

Không test "gọi đúng hook X" mà test output UI.

### 10.3 Cypress / Playwright

- Viết test cho luồng chính: login, checkout, settings.
- Sử dụng fixtures/mocking API để test deterministic.

---

## 11. Debugging & DevTools Mastery

### 11.1 Performance profiling

Chrome DevTools → Performance tab:

- Record tương tác (VD: mở modal, scroll list).
- Tìm **long tasks** (màu đỏ > 50ms).
- Xem call stack để biết function nào nặng.

React DevTools → Profiler:

- Xem component nào render nhiều.
- Gắn "why did this render" để biết props/state changing.

### 11.2 Network debugging & throttling

- Dùng các profile: Slow 3G, Fast 3G.
- Check header: `cache-control`, `etag`, `stale-while-revalidate`.
- Test offline (PWA/service worker nếu có).

### 11.3 CSS containment

`contain` cho browser biết effect layout/paint của element khá cô lập.[cite:64][cite:71]

```css
.card-list {
  contain: layout paint;
}
```

Cẩn thận: dùng sai có thể làm element không expand theo nội dung như mong muốn.

---

## 12. Build Tools & DX

### 12.1 Vite

- Dev: dùng native ESM + HMR rất nhanh.
- Prod: bundle bằng Rollup, hỗ trợ code splitting, tree-shaking.

Điểm quan trọng:
- Alias, env vars (`import.meta.env`), plugin ecosystem (SVGR, legacy, PWA).

### 12.2 Turbopack & Module Federation

- Turbopack (Next): dev bundler mới, incremental.
- Module Federation: chia app thành microfrontends; chỉ nên dùng khi team rất lớn, deploy độc lập.

### 12.3 Bundle analysis workflow

1. Build production.
2. Chạy bundle analyzer.
3. Xem:
   - thư viện nặng → thay thế hoặc lazy-load
   - duplication giữa bundles

---

## 13. API Integration & Data Fetching

### 13.1 REST, GraphQL, tRPC

- REST: resource-based, dễ cache.
- GraphQL: 1 endpoint, query shape linh hoạt.
- tRPC: type-safe TS end-to-end, phù hợp full TS stack.

### 13.2 Caching strategies

- HTTP caching (`cache-control`, `etag`).
- Client caching (React Query, SWR) với patterns như `stale-while-revalidate`.

### 13.3 Optimistic UI & error handling

Pattern:

1. Cập nhật UI trước.
2. Gửi request.
3. Nếu lỗi → rollback + show error.

Giúp UX nhanh, tốt cho INP.

### 13.4 React Server Components data fetching

Next docs mô tả patterns như:
- fetch trong Server Component
- share kết quả fetch qua `cache()`
- parallel fetch giữa đoạn UI khác nhau.[cite:30][cite:36][cite:27]

---

## 14. Framework Internals

### 14.1 React Fiber & reconciler

Repo `react-fiber-architecture` miêu tả mục tiêu: hỗ trợ incremental rendering, priority scheduling, và khả năng pause/resume/hủy render.[cite:9]

**Mental model:**

- Có 2 Fiber tree: `current` (đang hiển thị) và `workInProgress` (đang tính toán).
- Render phase: xây dựng `workInProgress` tree, tính diff.
- Commit phase: apply changes vào DOM, swap tree.[cite:2][cite:3][cite:8]

Điều này cho phép concurrent rendering, transitions.

### 14.2 Next.js App Router vs Pages Router

- Pages Router:
  - `pages/`, `getServerSideProps`, `getStaticProps`.
- App Router:
  - `app/`, React Server Components, nested layouts, streaming SSR.[cite:30][cite:36]

### 14.3 SSR / SSG / ISR trade-offs

- SSR: data mới, TTFB cao hơn.
- SSG: build-time, cực nhanh, nhưng data cũ.
- ISR: rebuild theo thời gian hoặc on-demand.

Chọn theo profile data và traffic.

### 14.4 Suspense & streaming

- Suspense dùng để chờ async resource (data, code, image) với fallback.
- Kết hợp streaming SSR → gửi shell + fallback trước, sau đó stream phần data nặng khi sẵn sàng.[cite:27][cite:30]

---

## 15. Streaming UIs với Server-Sent Events (SSE)

### 15.1 Khái niệm & khi dùng SSE

MDN: SSE là cơ chế server đẩy events một chiều xuống client thông qua kết nối HTTP dài hạn, client dùng `EventSource` API.[cite:35][cite:45]

WHATWG spec mô tả format `text/event-stream`: mỗi event là một block các dòng `field: value` kết thúc bằng dòng trống.[cite:48]

**Dùng SSE khi:**

- Cần stream log, progress, kết quả LLM token-by-token.
- Không cần channel hai chiều liên tục như WebSocket.

### 15.2 Backend SSE – ví dụ Node/Express

```js
app.get('/events', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  let i = 0;
  const timer = setInterval(() => {
    const payload = JSON.stringify({
      id: i,
      time: Date.now(),
      msg: `chunk ${i}`,
    });

    res.write(`data: ${payload}\n\n`);
    i++;
  }, 1000);

  req.on('close', () => {
    clearInterval(timer);
  });
});
```

**Lưu ý:**
- Mỗi event kết thúc bằng `\n\n`.
- Có thể dùng `event: name` để phân loại.

### 15.3 React client với `EventSource`

```tsx
import { useEffect, useState } from 'react';

export function LogStream() {
  const [logs, setLogs] = useState<string[]>([]);

  useEffect(() => {
    const es = new EventSource('/events');

    es.onmessage = (event) => {
      const data = JSON.parse(event.data);
      setLogs(prev => [...prev, `${data.id}: ${data.msg}`]);
    };

    es.onerror = () => {
      console.warn('SSE error');
      es.close();
    };

    return () => es.close();
  }, []);

  return (
    <div>
      {logs.map((l, idx) => (
        <div key={idx}>{l}</div>
      ))}
    </div>
  );
}
```

**Checklist production:**

- Cleanup connection khi unmount.
- Retry/backoff nếu kết nối fail.
- Nếu dùng auth cookie: đảm bảo CORS & cookies config đúng.

**Capstone idea:**

- Build "Build logs UI": backend stream logs build, frontend hiển thị live (có virtualization nếu log dài), thêm filter/search.

---

## 16. Checklists & Roadmap

### 16.1 Checklist kiến thức nền

- [ ] Hiểu event loop, phân biệt task vs microtask.[cite:59][cite:54]
- [ ] Biết closure và tránh stale closure trong React.
- [ ] Nắm structural typing, discriminated unions, generics cơ bản.
- [ ] Dùng Flexbox & Grid đúng bài toán.
- [ ] Biết dùng container queries, `:has()`, scroll-driven animations.[cite:64][cite:69][cite:11]
- [ ] Hiểu CRP, reflow/repaint, biết cơ bản Core Web Vitals.[cite:57][cite:60]
- [ ] Viết HTML semantic, biết basics ARIA, keyboard nav.
- [ ] Nắm XSS basics, CSP, CORS, secure cookies.[cite:34][cite:40]
- [ ] Biết thiết kế component theo composition + compound pattern.
- [ ] Phân loại state và chọn Context/Zustand/Jotai/Redux/Server Components phù hợp.
- [ ] Viết được unit/integration test cho component chính.
- [ ] Dùng DevTools profiler để bắt bottleneck.
- [ ] Hiểu sơ bên trong React Fiber và Next App Router.[cite:9][cite:30]
- [ ] Từng build UI dùng SSE stream.

### 16.2 Roadmap luyện trong 4–8 tuần (gợi ý ngắn)

- Tuần 1–2: JS/TS core + event loop + closures + TS discriminated union.
- Tuần 2–3: Modern CSS (Grid, Flexbox, container queries, `:has()`, scroll anim).
- Tuần 3–4: React/Next data fetching, Core Web Vitals, performance + a11y.
- Tuần 4–6: State management, component architecture, testing.
- Tuần 6–8: Framework internals (Fiber, App Router, Suspense/Streaming, SSE capstone).

---

> Gợi ý: Mỗi lần bạn build feature mới trong project thật, hãy mở lại tài liệu này, lướt qua **checklist tương ứng**, rồi review code của mình theo checklist. Sau vài vòng như vậy, hầu hết kiến thức sẽ trở thành "muscle memory" thay vì lý thuyết.
