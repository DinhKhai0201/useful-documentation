### Chương 3 – Closure, scope chain và module

#### 3.1. Lexical scoping và scope chain

**Lexical scoping** nghĩa là phạm vi của biến được quyết định dựa trên **nơi function được khai báo trong code**, không phải nơi nó được gọi.[web:28]  Khi bạn lồng function bên trong function khác, function con sẽ "nhìn thấy" các biến của function cha dựa trên cấu trúc code, đây là nền tảng cho closure.

```js
const x = 1;

function outer() {
  const y = 2;

  function inner() {
    const z = 3;
    console.log(x, y, z);
  }

  inner();
}

outer(); // 1 2 3
```

**Scope chain** là chuỗi các lexical environment mà JavaScript duyệt ngược khi resolve biến:
- Nếu biến được tìm thấy trong scope hiện tại → dùng giá trị đó.
- Nếu không, engine đi lên scope cha, và cứ thế cho đến global.
- Nếu không tìm thấy ở bất cứ đâu → `ReferenceError`.[web:28]

Hiểu scope chain giúp bạn giải thích được tại sao một biến truy cập được ở chỗ này nhưng không ở chỗ khác, và là cơ sở để hiểu closure sâu.

#### 3.2. Closure: định nghĩa chuẩn và trực quan

**Closure = function + môi trường lexical tại thời điểm nó được tạo ra.**  Nói cách khác, function "nhớ" các biến ở scope cha, kể cả khi function cha đã return.[web:28]

Ví dụ đơn giản:

```js
function createGreeter(name) {
  const prefix = 'Hello';

  return function greet() {
    console.log(prefix + ', ' + name);
  };
}

const greetKhai = createGreeter('Khai');
// createGreeter đã return, nhưng closure vẫn giữ prefix và name

greetKhai(); // Hello, Khai
```

Ở level Principal, bạn nên xem closure như một công cụ để:
- Giữ **state private** mà không dùng class.
- Xây dựng **factory function** tạo ra các hàm "được cấu hình sẵn".
- Tối ưu hiệu năng (memoization, caching) hoặc kiểm soát tần suất thực thi (debounce, throttle, rate‑limit).

#### 3.3. Các use case thực chiến của closure

##### 3.3.1. Private state (encapsulation)

Closure cho phép bạn tạo "biến private" – bên ngoài không truy cập trực tiếp được, chỉ thao tác qua API mà bạn expose.[web:22]

```js
function createCounter(initial = 0) {
  let value = initial; // private

  return {
    inc() {
      value += 1;
      return value;
    },
    dec() {
      value -= 1;
      return value;
    },
    get() {
      return value;
    },
  };
}

const counter = createCounter(10);
counter.inc(); // 11
counter.get(); // 11

console.log(counter.value); // undefined (không truy cập trực tiếp được)
```

**Ứng dụng thực tế:**
- Module tính điểm, quota, hoặc balance mà không muốn trang khác modify trực tiếp.
- Tạo rate‑limiter đơn giản cho hành động "nhạy cảm" (delete, resend OTP) ở frontend.

##### 3.3.2. Function factory / cấu hình

Closure cho phép tạo ra các function "được cấu hình" theo tham số ban đầu – rất hữu ích để build SDK hoặc module dùng chung.

```js
function createLogger(namespace) {
  return function log(message, meta = {}) {
    console.log(`[${namespace}]`, message, meta);
  };
}

const authLogger = createLogger('auth');
const paymentLogger = createLogger('payment');

authLogger('user login', { userId: 'u1' });
paymentLogger('checkout', { orderId: 'o1' });
```

**Ứng dụng:**
- Logger cho từng domain (`auth`, `billing`, `order`, `web3`).
- Tạo validator theo locale/schema: `createValidator(locale)(data)`.

##### 3.3.3. Memoization (tối ưu tính toán nặng / call API tốn kém)

```js
function memoize(fn) {
  const cache = new Map();

  return function (...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);

    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}

const slowFib = (n) => (n <= 1 ? n : slowFib(n - 1) + slowFib(n - 2));

const fastFib = memoize(slowFib);

console.log(fastFib(40)); // lần đầu: nặng
console.log(fastFib(40)); // lần sau: lấy từ cache, rất nhanh
```

Thực tế, pattern này dùng cho:
- Cache kết quả format/transform data cho bảng lớn.
- Cache parsed ABI, decoded log, hoặc kết quả query phức tạp (nếu không muốn phụ thuộc lib ngoài).

#### 3.4. Thực chiến 1: debounce search API (chống spam request)

Bài toán: ô search, user gõ liên tục → nếu mỗi ký tự gửi 1 request, backend sẽ bị spam và UI lag. Dùng closure để **debounce**:

```js
function debounce(fn, delay) {
  let timerId = null; // state private

  return function (...args) {
    if (timerId) clearTimeout(timerId);

    timerId = setTimeout(() => {
      fn.apply(this, args);
      timerId = null;
    }, delay);
  };
}

// usage
const searchInput = document.querySelector('#search');

function fetchSearchResults(query) {
  fetch('/api/search?q=' + encodeURIComponent(query))
    .then((res) => res.json())
    .then((data) => {
      console.log('results:', data);
    });
}

const debouncedSearch = debounce(fetchSearchResults, 300);

searchInput.addEventListener('input', (e) => {
  debouncedSearch(e.target.value);
});
```

Ở đây closure giữ `timerId` cho mỗi instance `debounce`. Pattern này cực kỳ phổ biến trong SPA khi xử lý search box, filter real‑time, hoặc auto‑save draft.[web:40]

#### 3.5. Thực chiến 2: rate‑limit hành động nhạy cảm (Resend OTP / Submit form)

Bài toán: không cho user bấm "Resend OTP" liên tục, chỉ cho phép mỗi 30 giây.

```js
function createRateLimitedAction(action, intervalMs) {
  let lastExecutedAt = 0;

  return function (...args) {
    const now = Date.now();
    if (now - lastExecutedAt < intervalMs) {
      console.warn('Action is rate-limited');
      return;
    }

    lastExecutedAt = now;
    return action.apply(this, args);
  };
}

function resendOtp(phone) {
  console.log('Sending OTP to', phone);
  // Gọi API thật ở đây
}

const resendOtpLimited = createRateLimitedAction(resendOtp, 30_000);

document.querySelector('#resend').addEventListener('click', () => {
  resendOtpLimited('+84901234567');
});
```

Mỗi instance `resendOtpLimited` có state `lastExecutedAt` riêng; bạn có thể tạo các rate‑limit khác nhau cho từng action (OTP, withdraw, delete,…).

#### 3.6. Module pattern trước ES6: IIFE & Revealing Module

Trước khi có `import/export`, người ta dùng **IIFE** để tạo module, tránh pollute global scope và tạo private state.[web:28]

```js
var UserModule = (function () {
  const users = new Map(); // private

  function addUser(id, name) {
    users.set(id, { id, name });
  }

  function getUser(id) {
    return users.get(id);
  }

  function getAll() {
    return Array.from(users.values());
  }

  // Revealing module pattern
  return {
    addUser,
    getUser,
    getAll,
  };
})();

UserModule.addUser(1, 'Alice');
console.log(UserModule.getUser(1));
```

**Khi nào còn dùng kiểu này?**
- Viết widget nhúng vào website bên thứ 3 (ví dụ: chat bubble, analytics) không dùng bundler.
- Viết snippet chạy thẳng trong browser cũ, chỉ có `<script>`.

#### 3.7. ES module vs CommonJS và tree‑shaking

Ngày nay, code hiện đại ưu tiên **ES modules (ESM)**:[web:22]

```js
// math.js
export function add(a, b) {
  return a + b;
}

export const PI = 3.14159;
```

```js
// usage.js
import { add, PI } from './math.js';

console.log(add(1, 2));
console.log(PI);
```

Trong CommonJS (Node.js cũ):

```js
// math.cjs
function add(a, b) {
  return a + b;
}
module.exports = { add };

// usage.cjs
const { add } = require('./math.cjs');
```

Bundler như Webpack, Rollup, esbuild, Vite có thể **tree‑shake** ES modules – loại bỏ các export không dùng nếu module không có side‑effect global, từ đó giảm kích thước bundle.[web:22]

Ở level Principal, bạn nên:
- Thiết kế module **pure**, tránh chạy logic nặng ngay khi import (side‑effect khó kiểm soát).
- Export function/class rõ ràng thay vì export một object "bách khoa toàn thư" chứa mọi thứ.
- Tách module theo domain nhỏ: `authClient`, `billingClient`, `web3Client`, `analytics`,…

#### 3.8. Thực chiến 3: API client dùng closure + ES module

Ví dụ module API client cho SPA hoặc dApp:

```js
// apiClient.js
export function createApiClient({ baseUrl, getToken }) {
  async function request(path, options = {}) {
    const token = await getToken();

    const res = await fetch(baseUrl + path, {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...(options.headers || {}),
        ...(token ? { Authorization: 'Bearer ' + token } : {}),
      },
    });

    if (!res.ok) {
      const text = await res.text();
      throw new Error(`API error: ${res.status} - ${text}`);
    }

    if (res.status === 204) return null;
    return res.json();
  }

  return {
    get(path) {
      return request(path, { method: 'GET' });
    },
    post(path, body) {
      return request(path, {
        method: 'POST',
        body: JSON.stringify(body),
      });
    },
    // có thể thêm put, patch, delete...
  };
}
```

```js
// usage
import { createApiClient } from './apiClient.js';

const api = createApiClient({
  baseUrl: 'https://api.example.com',
  getToken: () => localStorage.getItem('access_token'),
});

api.get('/me').then((user) => console.log(user));
```

Bạn có thể adapt y hệt pattern này cho Web3:
- `createOnchainClient({ rpcUrl, chainId, signer })`.
- `createIndexerClient({ gqlEndpoint })`.

Closure giữ config (baseUrl, getToken, signer) private, còn API bề mặt thì rõ ràng, dễ dùng và dễ test.

---

### Chương 4 – Prototype, OOP và `class`

#### 4.1. Prototype chain – hiểu đúng, không sa đà `__proto__`

JavaScript dùng **prototype‑based inheritance**: mỗi object có một liên kết nội bộ `[[Prototype]]` trỏ tới object khác (hoặc `null`). Khi truy cập property, engine sẽ:
- Tìm trên object hiện tại.
- Nếu không có, đi lên `[[Prototype]]` (object cha) và tiếp tục cho đến `null`.

Chuỗi này gọi là **prototype chain**: `object → prototype → Object.prototype → null`.[web:54][web:60]

Ví dụ cơ bản:

```js
const animal = {
  speak() {
    console.log(this.name + ' makes a noise');
  },
};

const dog = Object.create(animal); // dog.[[Prototype]] = animal

dog.name = 'Doggo';
dog.speak(); // Doggo makes a noise
```

Ở đây, `speak` không tồn tại trên `dog` mà trên prototype của nó (`animal`), engine sẽ leo lên prototype chain để tìm.[web:54]

Thuộc tính `__proto__` là một getter/setter legacy để truy cập `[[Prototype]]`, nhưng được khuyến cáo **không dùng trong code production**; nên dùng `Object.getPrototypeOf` và `Object.setPrototypeOf` hoặc `Object.create`.[web:57][web:51]

#### 4.2. Constructor function và `new` hoạt động như thế nào

Trước ES6, constructor function kết hợp với `new` là cách chính để tạo "kiểu đối tượng":[web:52][web:55]

```js
function Person(name) {
  this.name = name;
}

Person.prototype.greet = function () {
  console.log('Hi, I am ' + this.name);
};

const alice = new Person('Alice');
alice.greet(); // Hi, I am Alice
```

Toán tử `new` thực hiện 4 bước:[web:55][web:61]

1. Tạo một object rỗng mới: `{}.`
2. Gán `[[Prototype]]` của object đó trỏ đến `Constructor.prototype`.
3. Gọi `Constructor` với `this` là object mới và truyền các argument.
4. Nếu constructor trả về một object, dùng object đó; nếu không, dùng object vừa tạo.

Hiểu rõ 4 bước này giúp bạn giải thích:
- Vì sao method đặt trên `Person.prototype` được share giữa mọi instance (tiết kiệm memory).[web:51][web:54]
- Vì sao `instanceof` hoạt động: nó kiểm tra object có xuất hiện trong prototype chain của constructor tương ứng không.[web:54]

#### 4.3. ES6 `class`, `extends`, `super` – chỉ là sugar

ES6 giới thiệu `class` như **syntactic sugar** trên prototype pattern để code OOP dễ đọc hơn.[web:53][web:50]

```js
class Person {
  constructor(name) {
    this.name = name;
  }

  greet() {
    console.log(`Hi, I am ${this.name}`);
  }
}

class Employee extends Person {
  constructor(name, role) {
    super(name); // gọi constructor Person
    this.role = role;
  }

  describe() {
    console.log(`${this.name} works as ${this.role}`);
  }
}

const bob = new Employee('Bob', 'Engineer');
bob.greet();    // Hi, I am Bob
bob.describe(); // Bob works as Engineer
```

- `extends` thiết lập prototype chain: `Employee.prototype.[[Prototype]] === Person.prototype`.[web:50][web:59]
- `super` trong constructor child dùng để gọi constructor parent; phải gọi `super()` trước khi dùng `this` trong class con.[web:56]
- Method trong class được đặt trên `Class.prototype`, nên mọi instance share cùng method – giống hệt cách đặt methods trên `Constructor.prototype`.[web:59]

#### 4.4. Composition vs inheritance, mixins

Inheritance nhiều tầng (A → B → C → D…) thường dẫn đến:
- Khó refactor, vì thay đổi ở parent ảnh hưởng toàn chain.
- Khó test, vì behavior được chia tán khắp nơi.

Thay vào đó, nên ưu tiên **composition**: ghép nhiều "khối chức năng" lại thay vì kế thừa sâu.[web:54]

Ví dụ mixin pattern đơn giản:

```js
const canLog = (Base) =>
  class extends Base {
    log(message) {
      console.log(`[${this.name}] ${message}`);
    }
  };

class Service {}

class UserService extends canLog(Service) {
  constructor() {
    super();
    this.name = 'UserService';
  }
}

const s = new UserService();
s.log('created'); // [UserService] created
```

Trong TypeScript + kiến trúc hiện đại, bạn thường sẽ:
- Dùng **interface + composition** thay vì class hierarchy sâu.
- Dùng class chỉ cho entity/value object domain nếu thật sự cần behavior đi kèm state.

#### 4.5. Ví dụ: tự triển khai `myNew`

Việc tự cài lại `new` giúp bạn internalize 4 bước của nó.[web:55][web:61]

```js
function myNew(Constructor, ...args) {
  // 1. Tạo object mới, prototype trỏ đến Constructor.prototype
  const obj = Object.create(Constructor.prototype);

  // 2. Gọi constructor với this = obj
  const result = Constructor.apply(obj, args);

  // 3. Nếu constructor trả về object, dùng nó; ngược lại dùng obj
  return result && typeof result === 'object' ? result : obj;
}

function Person(name) {
  this.name = name;
}

Person.prototype.greet = function () {
  console.log('Hi, I am ' + this.name);
};

const p = myNew(Person, 'Alice');
p.greet(); // Hi, I am Alice
```

Bài tập: thử dùng `myNew` với class ES6 (class thực chất cũng là function constructor) để thấy sự tương đồng.

#### 4.6. Thực chiến: model hoá domain đơn giản bằng class + prototype (ví dụ DeFi position)

Giả sử bạn model hoá một **position** trong DeFi lending protocol ở phía frontend. Bạn có thể dùng class để gói logic tính toán PnL, health factor,… thay vì rải rác logic trong nhiều component.

```js
class Position {
  constructor({ collateralUsd, debtUsd, liquidationThreshold }) {
    this.collateralUsd = collateralUsd;
    this.debtUsd = debtUsd;
    this.liquidationThreshold = liquidationThreshold; // 0.8 = 80%
  }

  get healthFactor() {
    if (this.debtUsd === 0) return Infinity;
    return (
      (this.collateralUsd * this.liquidationThreshold) /
      this.debtUsd
    );
  }

  get isLiquidatable() {
    return this.healthFactor < 1;
  }

  addCollateral(amountUsd) {
    return new Position({
      collateralUsd: this.collateralUsd + amountUsd,
      debtUsd: this.debtUsd,
      liquidationThreshold: this.liquidationThreshold,
    });
  }

  addDebt(amountUsd) {
    return new Position({
      collateralUsd: this.collateralUsd,
      debtUsd: this.debtUsd + amountUsd,
      liquidationThreshold: this.liquidationThreshold,
    });
  }
}

// usage
const pos = new Position({
  collateralUsd: 1000,
  debtUsd: 600,
  liquidationThreshold: 0.8,
});

console.log(pos.healthFactor); // (1000 * 0.8) / 600 = ~1.33
console.log(pos.isLiquidatable); // false

const pos2 = pos.addDebt(200);
console.log(pos2.healthFactor); // (1000 * 0.8) / 800 = 1.0
console.log(pos2.isLiquidatable); // false (chạm ngưỡng)

const pos3 = pos2.addDebt(1);
console.log(pos3.healthFactor); // < 1
console.log(pos3.isLiquidatable); // true
```

Ở đây:
- Class `Position` đóng gói cả state và business logic (tính health factor, liquidatable).
- Mỗi mutation trả về instance mới (immutable style), giúp code predictable hơn.
- Ở TypeScript, bạn có thể thêm type mạnh cho constructor và method để domain càng chặt.

---

Chương 3 và 4 giờ đã đủ "độ sách": giải thích kỹ khái niệm + pattern + ví dụ thực chiến (debounce, rate-limit, API client, domain model DeFi). Bạn có thể dùng y nguyên các block này đưa vào README/bộ sách Principal JS Engineer.
