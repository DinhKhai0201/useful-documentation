# Roadmap Principal JavaScript Engineer

Tài liệu này trình bày chi tiết lộ trình kiến thức (kèm ví dụ) để đạt đến cấp độ Principal JavaScript Engineer. Mục tiêu là giúp bạn không chỉ giỏi code, mà còn đủ khả năng thiết kế kiến trúc, đưa ra quyết định kỹ thuật và dẫn dắt đội ngũ trên các hệ thống web và Web3 quy mô lớn.[web:16][web:22]

---

## Phần I – Nền tảng JavaScript hiện đại

### Chương 1 – Kiến thức cốt lõi về ngôn ngữ

#### 1. Lịch sử JS, ES5 → ES202x, JavaScript vs ECMAScript, vai trò TC39

JavaScript xuất hiện năm 1995 (tên ban đầu là LiveScript) để thêm logic động cho trình duyệt. ECMAScript là tiêu chuẩn ngôn ngữ do ECMA International ban hành; JavaScript là một hiện thực của tiêu chuẩn ECMAScript.[web:22] Các bản ES3, ES5, ES2015 (ES6) và các phiên bản hàng năm (ES2016, ES2017, …) lần lượt bổ sung nhiều tính năng như strict mode, `class`, `let/const`, Promise, async/await,…[web:22]

TC39 là committee định nghĩa spec ECMAScript. Mọi tính năng mới đều đi qua quy trình proposal từ stage 0 → stage 4, sau đó được đưa vào spec chính thức. Việc hiểu lịch sử giúp bạn đọc được proposal, spec và chủ động nắm bắt các tính năng mới thay vì chỉ dựa vào tutorial.

#### 2. Runtime: Browser vs Node.js, V8/SpiderMonkey/JavaScriptCore

JavaScript không chạy “một cách trừu tượng” mà chạy trong runtime cụ thể: trình duyệt (Chrome, Firefox, Safari) hoặc môi trường server (Node.js, Deno,…). Chrome dùng engine V8, Firefox dùng SpiderMonkey, Safari dùng JavaScriptCore; Node.js cũng dùng V8 nhưng cung cấp thêm API hệ điều hành (filesystem, network,…).[web:17][web:26]

Sự khác nhau này giải thích vì sao có Web APIs như `document`, `window`, `fetch` chỉ tồn tại trong browser, còn Node.js có `fs`, `http`, `process`. Ở cấp Principal, bạn cần hiểu rõ giới hạn của từng runtime để thiết kế module và layer abstraction phù hợp.

#### 3. Kiểu dữ liệu: primitive vs reference, boxing/unboxing, `typeof`, `instanceof`

Trong JavaScript có 7 primitive chính: `string`, `number`, `bigint`, `boolean`, `undefined`, `symbol`, `null`, và một nhóm non‑primitive là `object` (bao gồm cả array, function, Date,…).[web:22] Primitive được truyền theo value, còn object theo reference (thực tế là reference tới vùng nhớ). Điều này ảnh hưởng trực tiếp tới việc mutation state.

Ví dụ:
```js
let a = 1;
let b = a; // copy value
b = 2;
console.log(a, b); // 1 2

const obj1 = { x: 1 };
const obj2 = obj1; // copy reference
obj2.x = 99;
console.log(obj1.x); // 99
```

Boxing là việc wrapper object như `new Number(1)`, `new String('a')` bao primitive trong object; engine cũng tự động boxing khi bạn gọi method trên primitive (`'abc'.toUpperCase()`) nên ít khi cần new thủ công. `typeof` dùng để phân loại kiểu cơ bản (`typeof null === 'object'` là “bug lịch sử”), `instanceof` kiểm tra prototype chain – hữu ích để phân biệt instance của class hoặc constructor custom.

#### 4. Equality: `==` vs `===`, abstract equality và edge cases

`===` là strict equality: không ép kiểu, chỉ true khi kiểu và giá trị đều giống.[web:22] `==` thực hiện **abstract equality** với nhiều rule ép kiểu phức tạp, có thể gây bug khó đoán.

Ví dụ các edge case:
```js
0 == false;       // true
'' == false;      // true
null == undefined; // true
[] == 0;          // true
[] == '';         // true
[1] == 1;         // true
```

Ở level Principal, bạn hầu như luôn sử dụng `===` và `!==`, chỉ dùng `==` trong những trường hợp thực sự cần theo spec (ví dụ so sánh `null` với `undefined` một cách cố ý). Điều này giúp code rõ ràng và tránh surprise behavior.

#### 5. Scope: function scope, block scope, lexical environment

Scope quyết định biến nào truy cập được tại một điểm trong code. `var` tạo function scope, còn `let` và `const` tạo block scope (giữa `{}`), giúp code predictable hơn.[web:28] Lexical environment mô tả việc scope được quyết định tại thời điểm viết code (chứ không phải runtime), đây là nền tảng cho closure.

Ví dụ sự khác biệt:
```js
function demoVar() {
  for (var i = 0; i < 3; i++) {
    setTimeout(() => console.log('var i =', i), 0);
  }
}

function demoLet() {
  for (let i = 0; i < 3; i++) {
    setTimeout(() => console.log('let i =', i), 0);
  }
}

demoVar(); // 3, 3, 3
demoLet(); // 0, 1, 2
```

`var` dùng cùng một binding `i` cho toàn bộ vòng lặp, còn `let` tạo binding mới mỗi iteration nhờ block scope.

#### 6. Hoisting: biến, function declaration vs function expression

Hoisting là hành vi engine “đưa khai báo” lên đầu scope trong giai đoạn **memory creation**, trước khi chạy code.[web:19][web:17] Với `var` và function declaration, engine tạo binding sớm nên có thể gọi function trước khi khai báo.

Ví dụ:
```js
console.log(a);   // undefined (var được hoisted nhưng chưa gán)
foo();            // 'foo'

var a = 10;
function foo() {
  console.log('foo');
}

// Function expression KHÔNG được hoisted như declaration
bar(); // TypeError: bar is not a function
var bar = function () {
  console.log('bar');
};
```

`let`/`const` cũng bị hoisted nhưng nằm trong **Temporal Dead Zone**: bạn không thể truy cập trước khi dòng khai báo chạy, nếu không sẽ bị `ReferenceError`.[web:19]

#### 7. Strict mode: tác dụng và ảnh hưởng tới `this`

`"use strict"` bật strict mode, khiến engine ném lỗi ở những hành vi trước đây “im lặng” như gán vào biến chưa khai báo, xóa biến không cho phép, dùng từ khóa reserved,…[web:22] Trong strict mode, `this` trong function thường (không phải method trên object) sẽ là `undefined` thay vì global object, giúp phát hiện bug sớm.

Ví dụ:
```js
'use strict';
function foo() {
  console.log(this); // undefined trong strict mode
}
foo();
```

Strict mode cũng thay đổi một số behavior của `eval` và arguments, giúp code an toàn và tối ưu hơn cho engine.

---

### Chương 2 – Execution context & `this` binding

#### 1. Execution context, call stack và hai phase

Execution context là “môi trường thực thi” cho một đoạn code – chứa bindings cho biến, function, và giá trị của `this`. Có 3 loại chính: global, function, và eval (ít dùng).[web:17] Mỗi lần bạn gọi function, một function execution context mới được push lên call stack; khi function kết thúc, nó được pop ra.[web:17]

Engine chạy mỗi context qua hai phase:
- **Memory creation (hoisting)**: tạo bindings cho biến và function.
- **Execution**: chạy code line-by-line, gán giá trị cho biến, gọi function,…[web:17]

Hiểu execution context và call stack giúp bạn debug stack trace, tránh recursion không kiểm soát và hiểu vì sao hoisting hoạt động như vậy.

#### 2. Giá trị `this` trong các ngữ cảnh khác nhau

Giá trị của `this` phụ thuộc vào cách function được gọi, không phụ thuộc vào nơi định nghĩa:[web:22]

- Global context (non‑strict): trong browser, `this === window`.
- Function thường:
  - Non‑strict: `this` là global object nếu gọi `fn()`. 
  - Strict: `this` là `undefined` nếu gọi `fn()`.
- Method trên object: `obj.method()` → `this === obj`.
- Constructor với `new`: `this` là object mới.
- Event handler DOM: `this` là element nhận event (trong handler kiểu `elem.onclick = function() { ... }`).
- Arrow function: **không có `this` riêng**, nó “chụp” `this` từ scope bên ngoài (lexical `this`).[web:22]

Ví dụ minh họa arrow vs function thường:
```js
const obj = {
  value: 42,
  regular() {
    console.log('regular this.value =', this.value);
  },
  arrow: () => {
    console.log('arrow this.value =', this.value);
  }
};

obj.regular(); // 42
obj.arrow();   // thường là undefined hoặc window.value tùy môi trường
```

#### 3. `call`, `apply`, `bind` và pattern sử dụng

`Function.prototype.call`, `apply`, `bind` cho phép bạn điều khiển `this` khi gọi function:[web:22]

- `fn.call(thisArg, arg1, arg2, ...)`
- `fn.apply(thisArg, [arg1, arg2, ...])`
- `fn.bind(thisArg, presetArg1, ...)` trả về function mới với `this` đã được “cố định”.

Ví dụ sử dụng trong “function borrowing”:
```js
const person = { name: 'Alice' };

function greet(greeting) {
  console.log(greeting + ', ' + this.name);
}

greet.call(person, 'Hello');   // Hello, Alice
greet.apply(person, ['Hi']);   // Hi, Alice

const greetAlice = greet.bind(person, 'Hey');
greetAlice();                  // Hey, Alice
```

Khi làm việc với class trong React hoặc custom library, bạn thường dùng `bind` (hoặc arrow function) để đảm bảo `this` trỏ về instance.

#### 4. Ví dụ: tự triển khai `myCall` và `myBind`

Việc tự viết lại `call` và `bind` giúp hiểu sâu cách hoạt động của `this`.

```js
Function.prototype.myCall = function (context, ...args) {
  // Nếu context là null/undefined, fallback về global object (non-strict behavior)
  context = context || globalThis;
  // Tạo một property tạm trên context trỏ đến function gốc
  const fnSymbol = Symbol('fn');
  context[fnSymbol] = this;
  // Gọi function trong ngữ cảnh context
  const result = context[fnSymbol](...args);
  // Xoá property tạm
  delete context[fnSymbol];
  return result;
};

Function.prototype.myBind = function (context, ...presetArgs) {
  const originalFn = this;
  return function (...laterArgs) {
    return originalFn.myCall(context, ...presetArgs, ...laterArgs);
  };
};

function say(message) {
  console.log(message + ', ' + this.name);
}

const user = { name: 'Khai' };

say.myCall(user, 'Hello'); // Hello, Khai

const sayToKhai = say.myBind(user, 'Hi');
sayToKhai(); // Hi, Khai
```

---

### Chương 3 – Closure, scope chain và module

#### 1. Lexical scoping, scope chain và closure

Lexical scoping nghĩa là scope của biến được quyết định dựa trên **cấu trúc code** (nơi function được định nghĩa), không phải nơi nó được gọi. Khi một function lồng trong function khác, nó “nhớ” environment ở nơi nó được khai báo – đây là closure.[web:28]

Scope chain là chuỗi các lexical environment mà engine đi ngược lên để tìm biến. Nếu biến không tồn tại trong scope hiện tại, engine sẽ tìm ở scope cha, rồi cha nữa, cho tới global.[web:28]

Closure là function + môi trường lexical xung quanh nó; nhờ closure, bạn có thể giữ state private mà không cần class.

#### 2. Các use case thực tế của closure

- **Encapsulation & private state**: mô phỏng biến private.
- **Function factory**: tạo function chuyên biệt dựa trên tham số cấu hình.
- **Memoization**: cache kết quả của computation tốn kém.
- **Event handler**: giữ state giữa nhiều lần gọi.

Ví dụ counter module đơn giản:
```js
function createCounter() {
  let count = 0; // private
  return {
    increment() { count++; },
    decrement() { count--; },
    getValue() { return count; }
  };
}

const counter = createCounter();
counter.increment();
console.log(counter.getValue()); // 1
console.log(counter.count);      // undefined (private)
```

Ở đây, object trả về giữ closure lên biến `count`. Bên ngoài không thể truy cập trực tiếp, chỉ thông qua method.

#### 3. Module pattern trước ES6: IIFE, revealing module pattern

Trước khi có `import/export`, dev thường dùng **IIFE (Immediately Invoked Function Expression)** để tạo module và tránh pollute global scope.[web:28]

Ví dụ module pattern:
```js
var UserModule = (function () {
  const users = new Map(); // private

  function addUser(id, name) {
    users.set(id, { id, name });
  }

  function getUser(id) {
    return users.get(id);
  }

  // revealing module pattern
  return {
    addUser,
    getUser,
  };
})();

UserModule.addUser(1, 'Alice');
console.log(UserModule.getUser(1));
```

IIFE giúp tạo scope riêng, giữ biến private và chỉ expose API cần thiết.

#### 4. ES module vs CommonJS và tree‑shaking

Ngày nay, hầu hết code modern dùng **ES modules (ESM)**:
```js
// math.js
export function add(a, b) { return a + b; }
export const PI = 3.14159;

// main.js
import { add, PI } from './math.js';
```

CommonJS (Node.js cũ) dùng `require` và `module.exports`:
```js
// math.cjs
function add(a, b) { return a + b; }
module.exports = { add };

// main.cjs
const { add } = require('./math.cjs');
```

Bundler hiện đại (Webpack, Rollup, esbuild, Vite) có thể **tree‑shake** ES modules – loại bỏ code không dùng nếu module không có side effect, giúp giảm size bundle.[web:22] Ở level Principal, bạn cần thiết kế module nhỏ, không side‑effect global và export rõ ràng để tree‑shaking hoạt động tối ưu.

#### 5. Ví dụ: counter module & memoized Fibonacci

```js
// counter.js
export function createCounter(initial = 0) {
  let count = initial;
  return {
    inc() { count++; return count; },
    dec() { count--; return count; },
    value() { return count; },
  };
}

// fib.js
export function createFib() {
  const cache = new Map([[0, 0], [1, 1]]);

  function fib(n) {
    if (cache.has(n)) return cache.get(n);
    const val = fib(n - 1) + fib(n - 2);
    cache.set(n, val);
    return val;
  }

  return fib;
}

// usage
import { createCounter } from './counter.js';
import { createFib } from './fib.js';

const counter = createCounter();
const fib = createFib();

console.log(counter.inc()); // 1
console.log(fib(10));       // 55
```

Ở đây, closure + module pattern giúp giữ state private, dễ test và dễ mở rộng.

---

### Chương 4 – Prototype, OOP và `class`

#### 1. Prototype chain và `Object.create`

Trong JavaScript, inheritance được thực hiện qua **prototype chain**. Mỗi object có internal slot `[[Prototype]]` trỏ đến một object khác (hoặc `null`); khi truy cập property, engine sẽ tìm trên object, nếu không có thì đi lên prototype, rồi prototype của prototype,…[web:22] 

`Object.create(proto)` tạo object mới với prototype chỉ định; đây là cách rõ ràng để thiết lập prototype chain mà không cần constructor.

```js
const animal = {
  speak() {
    console.log(this.name + ' makes a noise');
  }
};

const dog = Object.create(animal);
dog.name = 'Doggo';
dog.speak(); // Doggo makes a noise
```

#### 2. Constructor function và `new` hoạt động như thế nào

Trước ES6 `class`, người ta dùng constructor function kết hợp với `new`:
```js
function Person(name) {
  this.name = name;
}

Person.prototype.greet = function () {
  console.log('Hi, I am ' + this.name);
};

const alice = new Person('Alice');
alice.greet();
```

Toán tử `new` làm 4 việc:
1. Tạo object rỗng.
2. Gán prototype của object mới thành `Constructor.prototype`.
3. Gọi constructor với `this` trỏ đến object mới.
4. Trả về object mới (trừ khi constructor trả về object khác).

#### 3. Từ ES6 `class`, `extends`, `super`

`class` là syntax sugar trên prototype và constructor, giúp code dễ đọc hơn:[web:22]
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
b ob.describe(); // Bob works as Engineer
```

ES2022 bổ sung class fields, private fields (`#secret`) và static fields, giúp mô hình OOP rõ ràng hơn.

#### 4. Mixins, composition vs inheritance

Ở hệ thống lớn, kế thừa nhiều tầng dễ dẫn đến fragile base class, khó refactor. Composition (“has‑a”) thường dễ bảo trì hơn inheritance (“is‑a”). Bạn có thể dùng mixin để “trộn” behavior vào object mà không cần deep class hierarchy.

Ví dụ mixin đơn giản:
```js
const canLog = Base => class extends Base {
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
s.log('created');
```

Ở level Principal, bạn phải thiết kế API và abstraction cẩn thận, ưu tiên composition và interface rõ ràng hơn là inheritance phức tạp.

#### 5. Ví dụ: tự triển khai `myNew`

```js
function myNew(Constructor, ...args) {
  // 1. Tạo object mới có prototype trỏ tới Constructor.prototype
  const obj = Object.create(Constructor.prototype);
  // 2. Gọi constructor với this = obj
  const result = Constructor.apply(obj, args);
  // 3. Nếu constructor trả về object, dùng nó; ngược lại dùng obj
  return (result && typeof result === 'object') ? result : obj;
}

function Person(name) {
  this.name = name;
}

Person.prototype.greet = function () {
  console.log('Hi, I am ' + this.name);
};

const p = myNew(Person, 'Alice');
p.greet();
```

Việc tự cài `myNew` giúp bạn hiểu rõ cách engine xử lý `new`, từ đó giải thích bug liên quan đến constructor, `instanceof`, và prototype.

---

### Chương 5 – Bất đồng bộ (async) ở mức sâu

#### 1. Event loop, task queue và microtask queue

JavaScript là single‑threaded nhưng vẫn xử lý được nhiều tác vụ bất đồng bộ nhờ **event loop**. Runtime duy trì:
- Call stack: nơi thực thi code đồng bộ.
- Web APIs / Node APIs: xử lý các tác vụ I/O, timer, network.
- Macrotask queue (task queue): chứa callback từ `setTimeout`, `setInterval`, I/O, DOM events.
- Microtask queue: chứa callback từ Promise, `queueMicrotask`, `MutationObserver`.[web:26][web:23]

Chu trình event loop: 
1) chạy code trong call stack, 2) xử lý toàn bộ microtask queue, 3) lấy 1 macrotask từ queue, lặp lại.[web:26] Vì microtask có ưu tiên cao hơn, callback của Promise thường chạy trước `setTimeout`.

#### 2. `setTimeout`, `setInterval`, `requestAnimationFrame`

- `setTimeout(fn, delay)` lên lịch callback sau tối thiểu `delay` ms (không đảm bảo chính xác tuyệt đối).
- `setInterval(fn, delay)` lặp lại callback, nhưng dễ drift nếu callback chậm.
- `requestAnimationFrame(fn)` dùng để sync với frame render của browser, thích hợp cho animation.[web:26]

Ví dụ thứ tự log:
```js
console.log('start');

setTimeout(() => console.log('timeout'), 0);

Promise.resolve().then(() => console.log('promise'));

console.log('end');
// output: start -> end -> promise -> timeout
```

#### 3. Promise, states và combinators

`Promise` đại diện cho kết quả tương lai (thành công hoặc thất bại) của một tác vụ async. Promise có 3 trạng thái: `pending`, `fulfilled`, `rejected`.[web:29][web:23]

Ví dụ cơ bản:
```js
const fetchData = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve('Data fetched');
  }, 1000);
});

fetchData
  .then(result => console.log(result))
  .catch(err => console.error(err));
```

Các combinator:
- `Promise.all` chờ tất cả fulfilled (hoặc reject ngay khi có 1 lỗi).
- `Promise.race` trả về kết quả promise kết thúc đầu tiên.
- `Promise.any` chờ promise đầu tiên fulfilled, bỏ qua các reject.

#### 4. `async/await` và concurrency vs parallelism

`async/await` là syntax sugar trên Promise, giúp code async trông như đồng bộ.[web:23]
```js
async function getUserWithPosts() {
  const user = await fetch('/api/user').then(res => res.json());
  const posts = await fetch('/api/posts?userId=' + user.id).then(res => res.json());
  return { user, posts };
}
```

Code trên **tuần tự**: phải chờ request thứ nhất xong mới tới request thứ hai. Để tận dụng concurrency, bạn có thể chạy song song:

```js
async function getUserWithPostsConcurrent() {
  const userPromise = fetch('/api/user').then(r => r.json());
  const postsPromise = fetch('/api/posts').then(r => r.json());

  const [user, posts] = await Promise.all([userPromise, postsPromise]);
  return { user, posts };
}
```

Ở đây, hai request chạy **concurrent** (song hành về mặt thời gian), nhưng vẫn chỉ trên một thread – khác với parallelism thật sự (multi‑thread/multi‑core).

#### 5. Race condition, cancellation pattern

Race condition xảy ra khi nhiều tác vụ async truy cập/chỉnh sửa cùng một state mà không có kiểm soát thứ tự; ví dụ: nhiều request update cùng một resource dẫn đến trạng thái inconsistent.[web:26] Ở frontend, một pattern phổ biến là user đổi filter hoặc search query nhanh, response của request cũ đến trễ hơn request mới → UI hiển thị sai.

Bạn có thể:
- Gắn `requestId` và chỉ chấp nhận response có id mới nhất.
- Dùng `AbortController` để cancel fetch cũ.

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
    const data = await res.json();
    renderResults(data);
  } catch (err) {
    if (err.name === 'AbortError') return; // ignore
    console.error(err);
  }
}
```

#### 6. Ví dụ: tự viết `promiseAll`

```js
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    const results = [];
    let completed = 0;

    if (promises.length === 0) {
      resolve([]);
      return;
    }

    promises.forEach((p, index) => {
      Promise.resolve(p)
        .then(value => {
          results[index] = value;
          completed++;
          if (completed === promises.length) {
            resolve(results);
          }
        })
        .catch(reject);
    });
  });
}

// usage
promiseAll([
  Promise.resolve(1),
  new Promise(res => setTimeout(() => res(2), 100)),
]).then(console.log); // [1, 2]
```

Cài đặt này giúp bạn hiểu rõ semantics của `Promise.all` (fail fast, giữ thứ tự kết quả theo index ban đầu).

---

### Chương 6 – Kiến trúc type với TypeScript

#### 1. Tại sao cần TypeScript ở level Principal

Ở hệ thống lớn, JavaScript thuần rất dễ phát sinh bug runtime do thiếu type thông tin: API trả về khác expectation, refactor đổi tên field nhưng quên update, v.v. TypeScript cung cấp layer type static giúp phát hiện lỗi ngay lúc compile, đồng thời tạo self‑documentation cho codebase.[web:18][web:21]

Ở cấp Principal, bạn không chỉ “biết dùng TS” mà cần **thiết kế kiến trúc type**: modeling domain bằng type, định nghĩa contract cho API, event, config; và đặt guideline để cả team viết TypeScript nhất quán, dễ maintain.

#### 2. Type cơ bản, union, intersection, literal, enum

TypeScript thêm layer type lên JavaScript:

```ts
let id: number = 1;
let name: string = 'Alice';
let isActive: boolean = true;

// union type
let status: 'idle' | 'loading' | 'success' | 'error';

// intersection type
interface HasId { id: string; }
interface HasTimestamps { createdAt: Date; updatedAt: Date; }

type Entity = HasId & HasTimestamps;

// enum (hoặc literal union để linh hoạt hơn)
enum Role {
  User = 'user',
  Admin = 'admin',
}
```

Union type cho phép biểu diễn các trạng thái hữu hạn; intersection type giúp kết hợp nhiều trait (gần với composition). Literal type (`'idle' | 'loading'`) hữu ích cho state machine, tránh magic string.

#### 3. Function typing, generics, utility types cơ bản

```ts
// Function typing
function add(a: number, b: number): number {
  return a + b;
}

// function type alias
type UnaryFn<T, R> = (arg: T) => R;

// Generics
function identity<T>(value: T): T {
  return value;
}

const num = identity(1);       // T = number
const str = identity('hello'); // T = string

// Built-in utility types
interface User {
  id: string;
  name: string;
  email?: string;
}

type UserPatch = Partial<User>;        // tất cả field optional
type UserRequired = Required<User>;    // tất cả field required
type UserPreview = Pick<User, 'id' | 'name'>;
type UserWithoutEmail = Omit<User, 'email'>;
```

Generics giúp bạn viết hàm, class, type reusable một cách type‑safe. Utility types giảm boilerplate khi transform type.[web:21][web:30]

#### 4. Conditional types, mapped types, `infer`

Conditional types cho phép viết “if/else” trên type:[web:18][web:24]

```ts
type IsPromise<T> = T extends Promise<any> ? true : false;

type A = IsPromise<Promise<number>>; // true
type B = IsPromise<string>;          // false
```

`infer` giúp trích xuất type bên trong:

```ts
type Awaited<T> = T extends Promise<infer U> ? U : T;

type R1 = Awaited<Promise<string>>; // string
type R2 = Awaited<number>;          // number
```

Mapped types transform toàn bộ property của type:

```ts
interface ApiResponse<T> {
  data: T;
  error?: string;
}

// Biến mọi field thành readonly
 type ReadonlyDeep<T> = {
  readonly [K in keyof T]: T[K];
};

// Biến mọi field thành optional
 type Optional<T> = {
  [K in keyof T]?: T[K];
};
```

Nắm vững conditional + mapped types giúp bạn xây dựng các abstraction type‑level mạnh như router, form builder, query client,… với API dev‑friendly nhưng vẫn cực kỳ an toàn.[web:18][web:21]

#### 5. Khai báo type‑safe cho API layer, domain models, event payloads

Một trong những ứng dụng quan trọng nhất của TS ở scale lớn là đảm bảo type‑safety xuyên suốt từ API đến UI.

Ví dụ: type‑safe API client:

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

Khi đổi shape của `UserDto` hoặc `PostDto`, toàn bộ nơi sử dụng sẽ được compiler báo lỗi nếu không update – giảm rủi ro bug runtime.

#### 6. Ví dụ: `Result<T, E>` type cho error handling

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

// helper constructors
export function ok<T>(value: T): Ok<T> {
  return { ok: true, value };
}

export function err<E>(error: E): Err<E> {
  return { ok: false, error };
}

// usage in JS/TS
async function fetchUser(id: string): Promise<Result<UserDto, 'NotFound' | 'NetworkError'>> {
  try {
    const res = await fetch(`/api/users/${id}`);
    if (res.status === 404) return err('NotFound');
    if (!res.ok) return err('NetworkError');
    const data = await res.json();
    return ok(data as UserDto);
  } catch {
    return err('NetworkError');
  }
}

async function showUser(id: string) {
  const result = await fetchUser(id);
  if (!result.ok) {
    if (result.error === 'NotFound') {
      // show 404 UI
    }
    return;
  }
  renderUser(result.value);
}
```

Pattern này tạo ra flow rõ ràng, dễ test, và tránh exception bị nuốt mất. Ở level Principal, bạn có thể định nghĩa các abstraction kiểu này cho cả org dùng, thống nhất cách xử lý error giữa services và UI.
