### Chương 21 – Kiến trúc dApp

#### 21.1. Các layer chính trong dApp

Một dApp production thường được chia thành các layer rõ ràng:

- **UI (React/Next.js)**: form, table, chart, navigation, xử lý input, hiển thị state.
- **Web3 client (ethers.js / viem / wagmi)**: kết nối RPC, đọc state on-chain, gửi transaction, decode event.[web:66][web:71]
- **Smart contract (Solidity)**: core business logic (lending, AMM, NFT, auction…).
- **Indexer / data layer (The Graph / custom indexer)**: index event/log thành bảng dễ query (GraphQL/REST), phục vụ dashboard, lịch sử, leaderboard.[web:67][web:72]

Việc tách layer giúp bạn đổi RPC, đổi protocol, hoặc đổi indexer mà không phải rewrite toàn bộ frontend.

#### 21.2. Flow giao dịch từ UI tới on-chain và ngược lại

Flow điển hình cho một action như “Open Position”:

1. User nhập dữ liệu (token, amount, leverage…) trên UI.
2. Frontend validate, tính toán preview (health factor, fee, slippage) – có thể dùng indexer hoặc contract `view`.
3. Frontend gọi web3 client (ethers/viem) để tạo transaction tới function contract tương ứng.[web:66][web:71]
4. Wallet (MetaMask, WalletConnect, smart wallet) bật popup cho user confirm.
5. Transaction được broadcast, sau đó mined, contract emit events.
6. Indexer hoặc frontend lắng nghe events/log rồi cập nhật UI (positions, PnL, lịch sử giao dịch).

Ở level Principal, bạn phải đảm bảo toàn flow này:
- Có **retry / error handling** rõ ràng.
- Có state: `idle → pending signature → pending tx → confirmed → failed`.
- UI không “đợi mù” mà luôn hiển thị rõ trạng thái.

#### 21.3. Tổ chức code dApp trong frontend

Gợi ý cấu trúc thư mục:

```text
src/
  app/
    positions/
      page.tsx           # UI chính
  features/
    web3/
      onchainClient.ts   # đọc/ghi contract
      hooks/
        useWallet.ts     # kết nối ví, chainId
        useTxFlow.ts     # state transaction (pending, success, failed)
    positions/
      api/
        indexerClient.ts # đọc positions từ indexer / The Graph
      hooks/
        usePositions.ts
        useOpenPosition.ts
      components/
        PositionTable.tsx
        OpenPositionForm.tsx
```

Tách `web3` (on-chain) ra khỏi `positions` giúp bạn tái sử dụng client cho nhiều module khác (farm, borrow, LP…) và có thể test từng phần riêng lẻ.

---

### Chương 22 – Smart contract integration từ JavaScript

#### 22.1. Concepts: provider, signer, contract

Với **ethers.js**, có 3 khái niệm cốt lõi:[web:66][web:71]

- **Provider**: nguồn đọc dữ liệu từ chain (RPC HTTP/WebSocket, Infura, Alchemy,…).
- **Signer**: entity có private key, dùng để ký transaction / message (wallet người dùng, server signer,…).
- **Contract**: object đại diện cho một contract cụ thể, biết ABI, address và provider/signer; cho phép bạn gọi `read` (call) và `write` (send transaction).[web:66]

Ví dụ khởi tạo (ethers v5):

```js
import { ethers } from 'ethers';
import ERC20_ABI from './abis/ERC20.json';

const provider = new ethers.providers.JsonRpcProvider(process.env.NEXT_PUBLIC_RPC_URL);
const signer = provider.getSigner(); // nếu chạy trong context có injected provider (MetaMask)

const usdc = new ethers.Contract(USDC_ADDRESS, ERC20_ABI, signer);
```

#### 22.2. Đọc state: `call` functions

Các function `view/pure` trong Solidity tương ứng với **call**: không tốn gas, không tạo transaction.

```js
async function getBalance(address) {
  const raw = await usdc.balanceOf(address);
  return Number(ethers.utils.formatUnits(raw, 6));
}
```

Trong dApp, bạn nên wrap các call này trong hooks có cache (React Query/TanStack Query hoặc wagmi) thay vì gọi thẳng trong component mỗi lần render.

#### 22.3. Gửi transaction: write functions, gas, nonce

Các functions non‑view sẽ tạo transaction: cần signer, gas estimation, handle pending/mined:

```js
async function deposit(collateralAmount) {
  const amount = ethers.utils.parseUnits(collateralAmount.toString(), 6);

  // 1. Approve nếu cần
  const txApprove = await usdc.approve(LENDING_POOL_ADDRESS, amount);
  await txApprove.wait(); // chờ mined

  // 2. Gọi deposit trên lending pool
  const lendingPool = new ethers.Contract(LENDING_POOL_ADDRESS, LENDING_POOL_ABI, signer);

  const tx = await lendingPool.deposit(USDC_ADDRESS, amount);
  console.log('tx hash', tx.hash);

  const receipt = await tx.wait();
  console.log('confirmed in block', receipt.blockNumber);
}
```

Best practices:
- Luôn hiển thị rõ: `pending signature`, `pending on-chain`, `confirmed / failed`.
- Lưu hash + link explorer cho user.
- Không block UI trong khi `await tx.wait()`, dùng state để hiển thị loader.

#### 22.4. Thực chiến: `onchainClient` cho lending pool

```js
// onchainClient.ts
import { ethers } from 'ethers';
import LENDING_POOL_ABI from './abis/LendingPool.json';
import ERC20_ABI from './abis/ERC20.json';

export function createOnchainClient({ rpcUrl, lendingPoolAddress, signer }) {
  const provider = new ethers.providers.JsonRpcProvider(rpcUrl);
  const _signer = signer ?? provider.getSigner();

  const lendingPool = new ethers.Contract(lendingPoolAddress, LENDING_POOL_ABI, _signer);

  async function getPosition(user) {
    const [collateral, debt] = await Promise.all([
      lendingPool.collateralOf(user),
      lendingPool.debtOf(user),
    ]);
    return {
      collateralUSD: Number(ethers.utils.formatUnits(collateral, 18)),
      debtUSD: Number(ethers.utils.formatUnits(debt, 18)),
    };
  }

  async function deposit(collateralToken, amount) {
    const erc20 = new ethers.Contract(collateralToken, ERC20_ABI, _signer);
    const parsed = ethers.utils.parseUnits(amount.toString(), 18);

    const txApprove = await erc20.approve(lendingPoolAddress, parsed);
    await txApprove.wait();

    const tx = await lendingPool.deposit(collateralToken, parsed);
    return tx.hash;
  }

  async function repay(collateralToken, amount) {
    const parsed = ethers.utils.parseUnits(amount.toString(), 18);
    const tx = await lendingPool.repay(collateralToken, parsed);
    return tx.hash;
  }

  return {
    getPosition,
    deposit,
    repay,
  };
}
```

Module này có thể được inject vào hook React (`useOpenPosition`, `useRepay`) để thuận tiện cho UI và testing.

---

### Chương 23 – Indexing, subgraph và data layer

#### 23.1. Vì sao cần indexer (không chỉ `contract.events`)

Truy vấn trực tiếp từ chain (scan event qua RPC) chậm và phức tạp khi bạn cần:

- Lọc theo nhiều điều kiện (user, token, thời gian, status…).
- Aggregate (PnL, volume, TVL, leaderboard…).
- Pagination, sorting phức tạp.

Giải pháp: dùng **The Graph** (subgraph) hoặc tự build **custom indexer** bằng Node.js + DB.[web:67][web:72]

#### 23.2. Subgraph với The Graph

Subgraph mô tả cách index data từ một hoặc nhiều contract:
- Khai báo schema GraphQL.
- Mapping từ event/log sang entity.
- Deploy lên network của The Graph.[web:72]

Ví dụ schema đơn giản cho lending:

```graphql
# schema.graphql
type Position @entity {
  id: ID!
  user: Bytes!
  collateralToken: Bytes!
  collateralAmount: BigInt!
  debtAmount: BigInt!
  openedAt: BigInt!
}
```

Mapping handler (AssemblyScript):

```ts
// mapping.ts
export function handlePositionOpened(event: PositionOpened): void {
  const id = event.transaction.hash.toHex() + '-' + event.logIndex.toString();
  const position = new Position(id);
  position.user = event.params.user;
  position.collateralToken = event.params.collateralToken;
  position.collateralAmount = event.params.collateralAmount;
  position.debtAmount = event.params.debtAmount;
  position.openedAt = event.block.timestamp;
  position.save();
}
```

Frontend query bằng GraphQL:

```graphql
query Positions($user: Bytes!) {
  positions(where: { user: $user }, orderBy: openedAt, orderDirection: desc) {
    id
    collateralToken
    collateralAmount
    debtAmount
    openedAt
  }
}
```

The Graph được nhiều protocol DeFi/NFT lớn dùng để phục vụ hàng triệu query mỗi ngày.[web:67][web:72]

#### 23.3. Custom indexer với Node.js + DB

Khi bạn muốn toàn quyền kiểm soát hoặc protocol chưa có subgraph, bạn có thể:

1. Dùng **ethers.js + WebSocket provider** lắng nghe event.
2. Parse log, transform data.
3. Ghi vào DB (Postgres, MongoDB…).

```js
import { ethers } from 'ethers';
import LENDING_POOL_ABI from './abis/LendingPool.json';
import { savePositionOpened } from './db';

async function startIndexer() {
  const provider = new ethers.providers.WebSocketProvider(process.env.RPC_WS);
  const lendingPool = new ethers.Contract(LENDING_POOL_ADDRESS, LENDING_POOL_ABI, provider);

  lendingPool.on('PositionOpened', async (user, collateralToken, collateralAmount, debtAmount, event) => {
    const block = await event.getBlock();
    await savePositionOpened({
      txHash: event.transactionHash,
      logIndex: event.logIndex,
      user,
      collateralToken,
      collateralAmount: collateralAmount.toString(),
      debtAmount: debtAmount.toString(),
      blockNumber: event.blockNumber,
      timestamp: block.timestamp,
    });
  });
}

startIndexer().catch(console.error);
```

Frontend sau đó chỉ cần gọi REST/GraphQL từ DB này – pattern giống hầu hết backend “indexer” cho DeFi phức tạp.

---

### Chương 24 – Bảo mật dApp end‑to‑end

#### 24.1. Attack vectors phía frontend

Một số rủi ro phổ biến bên UI:

- Lạm dụng `window.ethereum`: dApp giả mạo yêu cầu user ký message/phê duyệt nguy hiểm.
- XSS: script bị inject, attacker đổi recipient address, RPC endpoint, payload giao dịch.
- Không xác thực input từ browser gửi về backend relay (ví dụ: order, withdrawal request).

Best practices:
- Không build dApp kiểu “nhận bất cứ JSON từ URL rồi ký/gửi raw”.
- Với action nhạy cảm (swap lớn, withdraw, transfer), luôn hiển thị **summary rõ ràng** (token, amount, address) trước khi gọi `sign` hoặc `sendTransaction`.
- Dùng CSP, sanitize user input, tránh `dangerouslySetInnerHTML` khi không cần.

#### 24.2. Backend relay và verify chữ ký (EIP‑712)

Nhiều hệ thống dùng **meta‑transaction** hoặc **off‑chain order**:

1. Frontend build object (order/intent), user ký bằng EIP‑712 (typed data).[web:68][web:73]
2. Backend nhận payload + signature, verify.
3. Backend thực thi on-chain (relayer gửi transaction) nếu hợp lệ.

EIP‑712 cho phép ký structured data, dễ hiển thị cho user hiểu, khó phishing hơn raw hex data.[web:68]

Ví dụ cấu trúc typed data (simplified):

```ts
const domain = {
  name: 'MyLendingApp',
  version: '1',
  chainId: 1,
  verifyingContract: LENDING_RELAY_ADDRESS,
};

const types = {
  OpenPosition: [
    { name: 'user', type: 'address' },
    { name: 'collateralToken', type: 'address' },
    { name: 'collateralAmount', type: 'uint256' },
    { name: 'debtToken', type: 'address' },
    { name: 'debtAmount', type: 'uint256' },
    { name: 'nonce', type: 'uint256' },
  ],
};

const message = {
  user,
  collateralToken,
  collateralAmount,
  debtToken,
  debtAmount,
  nonce,
};
```

Frontend dùng hàm `signTypedData` của wallet SDK để user ký. Backend (Node.js + ethers):

```js
import { ethers } from 'ethers';

function verifyOpenPositionSignature(message, signature) {
  const domain = { /* giống phía client */ };
  const types = { /* giống phía client */ };

  const recovered = ethers.utils.verifyTypedData(domain, types, message, signature);
  return recovered.toLowerCase() === message.user.toLowerCase();
}
```

Chỉ khi verify thành công, backend mới gửi transaction on-chain, giúp:
- Ngăn attacker forge request thay user (trừ khi chiếm private key).
- Cho phép thêm rule business (limit, KYC, risk) trước khi thực thi.[web:68][web:73]

#### 24.3. Rate limit, spam protection cho RPC/middleware

Các endpoint public thường gặp trong dApp:

- API relay: nhận signature + payload, gửi transaction.
- API indexer: trả về positions, orders, history.

Cần thêm:
- Rate limiting (theo IP, API key, user address).
- Anti‑spam: limit số transaction/phút, hoặc yêu cầu stake/fee đối với bot.
- Logging + alert để phát hiện pattern bất thường (flood, brute force).

Ở phía frontend, tránh lạm dụng một public RPC duy nhất; nên dùng RPC chuyên dụng hoặc provider aggregator, có logic fallback khi endpoint chậm/hết quota.
