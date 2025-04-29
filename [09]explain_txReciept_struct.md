좋습니다.  
`eth_getTransactionReceipt`는 Ethereum에서 트랜잭션이 블록에 포함된 후의 결과를 요약해주는 **표준 JSON-RPC 메서드**입니다.

여기서 반환되는 **Transaction Receipt 객체**는 트랜잭션 실행 결과를 구조화하여 담고 있습니다.

---

# ✅ txReceipt (Transaction Receipt)의 전체 구조

### 예시 (`eth_getTransactionReceipt` 호출 결과)

```json
{
  "blockHash": "0xabc...",
  "blockNumber": "0x5BAD55",
  "contractAddress": "0x123...",      // 생성된 컨트랙트 주소 (create일 경우만)
  "cumulativeGasUsed": "0x5208",
  "effectiveGasPrice": "0x77359400",  // EIP-1559 가스 전략 이후 추가
  "from": "0xabc123...",
  "gasUsed": "0x5208",
  "logs": [
    {
      "address": "0xcontract...",
      "topics": [...],
      "data": "0x...",
      "logIndex": "0x0",
      "transactionIndex": "0x1",
      "blockHash": "...",
      "blockNumber": "0x...",
      "transactionHash": "0x..."
    },
    ...
  ],
  "logsBloom": "0x...",               // 2048비트 블룸필터
  "status": "0x1",                    // 0x1=성공, 0x0=실패
  "to": "0xrecipient...",
  "transactionHash": "0xabc...",
  "transactionIndex": "0x0",
  "type": "0x2"                       // 트랜잭션 유형 (legacy=0x0, EIP-1559=0x2 등)
}
```

---

# 🧠 필드별 상세 설명

| 필드 | 설명 |
|------|------|
| `blockHash` | 해당 트랜잭션이 포함된 블록의 해시 |
| `blockNumber` | 트랜잭션이 포함된 블록 번호 |
| `transactionHash` | 트랜잭션 자체의 해시 |
| `transactionIndex` | 블록 내에서 트랜잭션의 위치 인덱스 |
| `from` | 트랜잭션을 보낸 주소 |
| `to` | 트랜잭션의 수신 주소 (컨트랙트 생성인 경우 `null`) |
| `contractAddress` | **새 컨트랙트 생성 트랜잭션일 경우에만** 이 필드가 있음. 생성된 컨트랙트 주소 |
| `gasUsed` | 이 트랜잭션 단독으로 소비한 가스 |
| `cumulativeGasUsed` | 해당 블록 내에서 이 트랜잭션까지의 누적 가스 사용량 |
| `effectiveGasPrice` | 실제로 적용된 gas price (EIP-1559 적용 시 baseFee + tip 포함) |
| `status` | 실행 성공 여부 (`0x1` = 성공, `0x0` = 실패) |
| `logs` | 이벤트 로그 배열. 컨트랙트에서 `emit`한 모든 이벤트 기록 |
| `logsBloom` | 로그 이벤트를 빠르게 검색하기 위한 블룸 필터 |
| `type` | 트랜잭션 타입 (0x0 = legacy, 0x1 = access list, 0x2 = EIP-1559) |

---

# 🔎 중요 필드 요약

### ✅ 실패 여부 판단: `status`

```json
"status": "0x0"
```

- `0x1` → 성공
- `0x0` → 실패 (하지만 reason 문자열은 없음)

---

### ✅ 컨트랙트 생성 판단: `contractAddress`

```json
"to": null,
"contractAddress": "0xabc..."
```

- `to`가 `null`이고 `contractAddress`가 존재하면 컨트랙트 생성 트랜잭션

---

### ✅ 이벤트 분석: `logs`

```json
"logs": [
  {
    "address": "0xTokenContract",
    "topics": [
      "0xddf252ad...transfer event sig",
      "0x0000...from",
      "0x0000...to"
    ],
    "data": "0x0000...000a"  // 전송량 등
  }
]
```

- `topics[0]`: 이벤트 시그니처 Keccak 해시 (예: `Transfer(address,address,uint256)`)
- 이후 topic: indexed 파라미터
- `data`: 나머지 non-indexed 파라미터

---

### ✅ 가스 분석

- `gasUsed`: 이 트랜잭션만 사용한 가스
- `cumulativeGasUsed`: 블록 내 누적 (가스 리밋 근접 분석 가능)

---

### ✅ 트랜잭션 타입

- `type: "0x0"` → Legacy
- `type: "0x2"` → EIP-1559 (baseFee, maxFee, priorityFee 도 존재 가능)

---

# ⚠️ txReceipt의 한계

| 항목 | 지원 여부 |
|---|---|
| 트랜잭션 revert 이유 (require 실패 등) | ❌ 없음 |
| 내부 호출 (`call`, `delegatecall`) | ❌ 없음 |
| opcode-level trace | ❌ 없음 |
| stack, memory 상태 | ❌ 없음 |

➡️ 그래서 **debug.traceTransaction()** 을 병행해야 합니다.

---

# ✅ 요약 정리표

| 필드 | 용도 | 중요도 |
|---|---|---|
| `status` | 성공/실패 여부 | ⭐️⭐️⭐️⭐️⭐️ |
| `gasUsed` | 개별 가스 소비량 | ⭐️⭐️⭐️⭐️ |
| `logs` | emit된 이벤트 | ⭐️⭐️⭐️⭐️ |
| `contractAddress` | 컨트랙트 생성 여부 | ⭐️⭐️⭐️ |
| `effectiveGasPrice` | 실제 가스 단가 (EIP-1559) | ⭐️⭐️ |
| `logsBloom` | 로그 필터링용 | ⭐️ |
| `cumulativeGasUsed` | 블록 내 가스 누적 | ⭐️ |

---

**EIP-1559 (타입 2)** 트랜잭션의 `eth_getTransactionReceipt` 구조를 중심으로 다시 설명 
특히 **타입 2 트랜잭션에서 추가된 필드들**에 주목

---

# ✅ 타입 2 (EIP-1559) 트랜잭션이란?

| 항목 | 설명 |
|------|------|
| 트랜잭션 유형 | `type: 0x2` |
| 특징 | `maxFeePerGas`, `maxPriorityFeePerGas`, `effectiveGasPrice` 지원 |
| 도입 시점 | London Hardfork 이후 (2021년 8월) |
| 목적 | 가스 시장의 예측 가능성 향상, 수수료 절감

---

# ✅ 타입 2 트랜잭션 Receipt 예시

```json
{
  "blockHash": "0xabc...",
  "blockNumber": "0x10D4F",
  "contractAddress": null,
  "cumulativeGasUsed": "0x5208",
  "effectiveGasPrice": "0x77359400",   // EIP-1559에서 추가됨
  "from": "0xabc123...",
  "gasUsed": "0x5208",
  "logs": [],
  "logsBloom": "0x...",
  "status": "0x1",
  "to": "0xdef456...",
  "transactionHash": "0xabc...",
  "transactionIndex": "0x0",
  "type": "0x2"
}
```

---

# 🧠 필드 설명 (특히 타입 2 중심으로)

| 필드 | 설명 | 타입 2 관련 여부 |
|------|------|------------------|
| `type` | 트랜잭션 유형: `0x2` (EIP-1559) | ✅ 핵심 |
| `effectiveGasPrice` | 실제 적용된 가스 단가 (baseFee + priorityFee) | ✅ 핵심 |
| `gasUsed` | 이 트랜잭션 하나가 소모한 가스 | 공통 |
| `cumulativeGasUsed` | 블록 내 누적 가스 소모량 | 공통 |
| `from`, `to` | 송신자 / 수신자 주소 | 공통 |
| `contractAddress` | 컨트랙트 생성 시 생성된 주소 (그 외에는 null) | 공통 |
| `status` | `0x1`: 성공 / `0x0`: 실패 | 공통 |
| `logs`, `logsBloom` | emit된 이벤트, 필터용 bloom | 공통 |

---

## ✅ `effectiveGasPrice`란?

- EIP-1559 트랜잭션에서는 사용자가 `maxFeePerGas`와 `maxPriorityFeePerGas`를 설정합니다.
- 실제 블록에 포함될 때 적용되는 **baseFee**는 블록마다 정해지므로,  
  최종적으로 이 트랜잭션에 적용된 실제 gas price가 이 필드에 기록됩니다.

### 계산 방식

```
effectiveGasPrice = min(maxFeePerGas, baseFee + maxPriorityFeePerGas)
```

💡 이 값이 곧 `gasUsed * effectiveGasPrice`로 **총 수수료 비용**을 계산하는 데 사용됩니다.

---

## ✅ 타입 2 트랜잭션에 없는 필드?

- `maxFeePerGas` 및 `maxPriorityFeePerGas`는 **Receipt에는 포함되지 않습니다.**
  → 이 값은 **`eth_getTransactionByHash`** 호출을 통해 조회해야 합니다.

---

## ✅ 타입 2과 타입 0 Receipt 구조 비교

| 필드 | 타입 0 (legacy) | 타입 2 (EIP-1559) |
|------|------------------|------------------|
| `type` | 없음 or `"0x0"` | `"0x2"` |
| `effectiveGasPrice` | 없음 | ✅ 있음 |
| `gasPrice` | Transaction에 있음 | Transaction에는 `maxFeePerGas`와 `priorityFeePerGas`로 분리됨 |
| 기타 구조 | 동일 | 동일 |

---

# ✅ 실제 분석 시 주의할 점

| 항목 | 설명 |
|------|------|
| `type == 0x2` 여부 확인 | type 필드가 명시적이면 EIP-1559 적용 트랜잭션 |
| 실제 수수료 계산 | `gasUsed * effectiveGasPrice` |
| 가스 전략 분석 | Transaction 본문에서 `maxFeePerGas`, `priorityFeePerGas`도 함께 확인 필요 |

---

# ✅ 예시: 타입 2 트랜잭션 총 수수료 계산

```js
gasUsed = 21000 (가스)
effectiveGasPrice = 0x77359400 = 2,000,000,000 wei (2 gwei)

→ 수수료 = 21000 * 2,000,000,000 = 42,000,000,000,000 wei
         = 0.000042 ETH
```

---

# ✅ 결론 요약

| 항목 | 설명 |
|------|------|
| Receipt 구조 자체는 타입 0/2 모두 유사 |
| 타입 2 전용 필드 | `type: "0x2"`, `effectiveGasPrice` |
| 수수료 계산 | `gasUsed * effectiveGasPrice` |
| 전체 수수료 전략 분석 | `eth_getTransactionByHash`와 조합 필요 |

---

---

# ✅ 목차

1. [`logs`] 필드의 구조 및 디코딩 방법  
2. [`logsBloom`]의 역할 및 동작 방식  
3. 🔍 실제 예시: ERC20 `Transfer` 이벤트 디코딩  
4. 📌 보안 및 디버깅에서의 활용법

---

## ✅ 1. `logs` 필드 구조 및 디코딩 방법

### `logs`는 이벤트 로그 배열이며, 각각 다음의 구조를 가집니다:

```json
{
  "address": "0xContractAddress",
  "topics": [
    "0xddf252ad..."      // topic[0]: event signature hash
    "0x000...from",      // topic[1]: indexed param 1
    "0x000...to"         // topic[2]: indexed param 2
  ],
  "data": "0x000...00a", // non-indexed params
  "blockNumber": "...",
  "logIndex": "...",
  ...
}
```

---

### 🧠 필드별 의미

| 필드 | 설명 |
|------|------|
| `address` | 이벤트를 발생시킨 컨트랙트 주소 |
| `topics[0]` | 이벤트 시그니처의 keccak256 해시 (`event Transfer(address,address,uint256)` 등) |
| `topics[1..n]` | indexed 인자들 (address, uint256 등) |
| `data` | non-indexed 인자들이 ABI-encoded 된 값들 |

---

## 🔎 디코딩 단계

### ① 시그니처 찾기

- `topics[0]` = `keccak256("Transfer(address,address,uint256)")`  
  → `0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a66e9c28b33`

### ② indexed 파라미터 디코딩

- 주소 타입인 경우: 마지막 40자리만 읽으면 됨 (20바이트)
- 예: `0x0000000000000000000000005aeda56215b167893e80b4fe645ba6d5bab767de`  
  → 주소: `0x5aeda56215b167893e80b4fe645ba6d5bab767de`

### ③ data 파라미터 디코딩

- 여러 개의 non-indexed 파라미터가 있으면 32바이트씩 나뉘어 있음
- 예: `0x00000000000000000000000000000000000000000000000000000000000003e8`  
  → `1000` (10진수)

---

### 🛠️ 디코딩 도구

- [Ethers.js](https://docs.ethers.org/v5/api/utils/abi/interface/)
- [Web3.js](https://web3js.readthedocs.io/en/v1.2.11/web3-eth-abi.html)
- Go: `github.com/ethereum/go-ethereum/accounts/abi`
- Python: `eth_abi`, `web3.py`

---

## ✅ 2. `logsBloom` 동작 방식

### 📘 정의

- `logsBloom`: 2048비트의 **블룸 필터**
- 트랜잭션 receipt에서 이벤트 로그를 빠르게 필터링하기 위한 **미리 계산된 요약값**

---

### 🔬 동작 원리

1. 이벤트의 **address**, **topics**를 keccak256 해시함
2. 해시 결과를 바탕으로 **3개의 비트 위치**를 계산
3. 해당 위치의 비트를 1로 설정 → 2048비트 로그 필터에 기록

예를 들어:
```plaintext
logsBloom: 0x0000...0010...0000
```
→ 특정 주소나 topic에 해당하는 비트가 1로 설정돼 있으면,
→ 이 로그에 해당 이벤트가 **있을 수도 있다** (확실하진 않음)

📌 블룸 필터는 **false positive는 가능, false negative는 없음**

---

### 📦 예시: Transfer 이벤트 확인

**Transfer(address,address,uint256)** 로그가 포함됐는지 필터링

1. `keccak256("Transfer(address,address,uint256)")` → hash
2. 이 hash에서 3개의 비트 위치 계산
3. `logsBloom`에 해당 비트들이 모두 1이면 → 후보 로그로 간주

---

### ✅ 활용

- Ethereum 노드는 `eth_getLogs`, `eth_newFilter` 요청 처리 시 `logsBloom`을 이용해 빠르게 후보를 거름
- Full-log scan을 방지하여 **효율적으로 로그 조회**

---

## 🔍 실제 예시: ERC20 Transfer 로그

### Solidity 이벤트 정의:

```solidity
event Transfer(address indexed from, address indexed to, uint256 value);
```

### 로그 구조 (예시):

```json
{
  "address": "0xTokenAddress",
  "topics": [
    "0xddf252ad...", // Transfer()
    "0x0000...from",
    "0x0000...to"
  ],
  "data": "0x00000000000000000000000000000000000000000000000000000000000003e8"
}
```

### 디코딩 결과:

- `from`: `0xabc...`
- `to`: `0xdef...`
- `value`: `1000` (from `data`)

---

## 📌 보안 및 디버깅 팁

| 상황 | 로그 분석 활용 |
|------|----------------|
| 컨트랙트 이벤트 발생 확인 | 특정 이벤트 topic hash로 로그 필터링 |
| 누가 호출했는지 추적 | indexed 주소 디코딩 |
| 이상 동작 여부 확인 | 이벤트 발행 순서 추적 |
| contract 없는데 logs 있음 | self-destruct된 address 사용 가능성 확인 필요

---

## ✅ 요약 정리

| 항목 | 설명 |
|------|------|
| `logs` | 이벤트 발생 내용, topic + data 구조 |
| 디코딩 방법 | topic[0] = signature hash, topic[1+] = indexed, data = non-indexed |
| `logsBloom` | 빠른 필터링을 위한 2048비트 비트마스크 |
| 디코딩 도구 | Ethers.js, Web3.py, Go-Ethereum ABI 등 |

---

⸻

✅ 목표
	•	Ethereum 트랜잭션 receipt의 logs 필드를 Go에서 디코딩
	•	예시: Transfer(address indexed from, address indexed to, uint256 value) 이벤트

⸻

📦 준비사항

먼저 필요한 패키지를 설치합니다:
```
go get github.com/ethereum/go-ethereum
```
필요 패키지:
```go
import (
    "context"
    "fmt"
    "log"
    "math/big"

    "github.com/ethereum/go-ethereum/accounts/abi"
    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/core/types"
    "github.com/ethereum/go-ethereum/ethclient"
)
```


⸻

🧠 1. Transfer 이벤트 ABI 정의
```go
const transferEventABI = `[{"anonymous":false,"inputs":[
    {"indexed":true,"internalType":"address","name":"from","type":"address"},
    {"indexed":true,"internalType":"address","name":"to","type":"address"},
    {"indexed":false,"internalType":"uint256","name":"value","type":"uint256"}
],"name":"Transfer","type":"event"}]`
```


⸻

🛠️ 2. Go 코드 전체 예시
```go
package main

import (
    "context"
    "fmt"
    "log"
    "math/big"

    "github.com/ethereum/go-ethereum/accounts/abi"
    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/core/types"
    "github.com/ethereum/go-ethereum/ethclient"
)

const transferEventABI = `[{"anonymous":false,"inputs":[
    {"indexed":true,"internalType":"address","name":"from","type":"address"},
    {"indexed":true,"internalType":"address","name":"to","type":"address"},
    {"indexed":false,"internalType":"uint256","name":"value","type":"uint256"}
],"name":"Transfer","type":"event"}]`

func main() {
    client, err := ethclient.Dial("https://mainnet.infura.io/v3/YOUR_INFURA_PROJECT_ID")
    if err != nil {
        log.Fatal(err)
    }

    txHash := common.HexToHash("0x트랜잭션해시") // 실제 트랜잭션 해시로 바꿔야 함

    receipt, err := client.TransactionReceipt(context.Background(), txHash)
    if err != nil {
        log.Fatal(err)
    }

    parsedABI, err := abi.JSON(strings.NewReader(transferEventABI))
    if err != nil {
        log.Fatal(err)
    }

    transferSigHash := parsedABI.Events["Transfer"].ID

    for _, vLog := range receipt.Logs {
        if vLog.Topics[0] == transferSigHash {
            fmt.Println("Transfer 이벤트 발견!")

            // topic[1] = from, topic[2] = to
            from := common.HexToAddress(vLog.Topics[1].Hex())
            to := common.HexToAddress(vLog.Topics[2].Hex())

            // data = value (non-indexed)
            var result struct {
                Value *big.Int
            }
            err := parsedABI.UnpackIntoInterface(&result, "Transfer", vLog.Data)
            if err != nil {
                log.Fatal(err)
            }

            fmt.Printf("From: %s\n", from.Hex())
            fmt.Printf("To: %s\n", to.Hex())
            fmt.Printf("Value: %s\n", result.Value.String())
        }
    }
}
```


⸻

✅ 실행 결과 예시
```text
Transfer 이벤트 발견!
From: 0xabc123...
To:   0xdef456...
Value: 1000000000000000000
```


⸻

📌 정리

| 항목 | 설명 |
|---|---|
| indexed 값 | topics[1..n]에서 직접 추출 (주소는 마지막 40자리) |
| non-indexed 값 | data를 abi.Unpack으로 디코딩 |
| 이벤트 필터링 | topics[0] = 이벤트 시그니처의 keccak256 해시 |



⸻

⸻

✅ 목표

| 요구 | 해결 방식 |
| 여러 이벤트 유형을 처리 | ABI 등록 후 event 시그니처에 따라 분기 처리 |
| indexed / non-indexed 자동 구분 | topics vs data 기반 분리 디코딩 |
| 코드 재사용성 | 이벤트마다 구조체 매핑, 자동 처리 |



⸻

📦 준비사항
```
go get github.com/ethereum/go-ethereum
```
```go
import (
    "context"
    "encoding/hex"
    "fmt"
    "log"
    "math/big"
    "strings"

    "github.com/ethereum/go-ethereum/accounts/abi"
    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/core/types"
    "github.com/ethereum/go-ethereum/ethclient"
)
```


⸻

🧠 예제에 사용할 이벤트 ABI 정의
```solidity
event Transfer(address indexed from, address indexed to, uint256 value);
event Approval(address indexed owner, address indexed spender, uint256 value);
```
```go
const tokenABI = `[{
  "anonymous":false,
  "inputs":[
    {"indexed":true,"internalType":"address","name":"from","type":"address"},
    {"indexed":true,"internalType":"address","name":"to","type":"address"},
    {"indexed":false,"internalType":"uint256","name":"value","type":"uint256"}
  ],
  "name":"Transfer","type":"event"
},
{
  "anonymous":false,
  "inputs":[
    {"indexed":true,"internalType":"address","name":"owner","type":"address"},
    {"indexed":true,"internalType":"address","name":"spender","type":"address"},
    {"indexed":false,"internalType":"uint256","name":"value","type":"uint256"}
  ],
  "name":"Approval","type":"event"
}]`
```


⸻

🧩 고급 디코더 전체 구조
```go
type TransferEvent struct {
    From  common.Address
    To    common.Address
    Value *big.Int
}

type ApprovalEvent struct {
    Owner   common.Address
    Spender common.Address
    Value   *big.Int
}

func decodeLogs(logs []*types.Log, contractAbi abi.ABI) {
    for _, vLog := range logs {
        if len(vLog.Topics) == 0 {
            continue
        }

        eventSig := vLog.Topics[0].Hex()

        switch eventSig {
        case contractAbi.Events["Transfer"].ID.Hex():
            var transfer TransferEvent

            // topics[1], topics[2] = From, To
            transfer.From = common.HexToAddress(vLog.Topics[1].Hex())
            transfer.To = common.HexToAddress(vLog.Topics[2].Hex())

            err := contractAbi.UnpackIntoInterface(&transfer, "Transfer", vLog.Data)
            if err != nil {
                log.Println("Transfer decode error:", err)
                continue
            }

            fmt.Printf("→ Transfer: %s → %s : %s\n", transfer.From.Hex(), transfer.To.Hex(), transfer.Value.String())

        case contractAbi.Events["Approval"].ID.Hex():
            var approval ApprovalEvent

            approval.Owner = common.HexToAddress(vLog.Topics[1].Hex())
            approval.Spender = common.HexToAddress(vLog.Topics[2].Hex())

            err := contractAbi.UnpackIntoInterface(&approval, "Approval", vLog.Data)
            if err != nil {
                log.Println("Approval decode error:", err)
                continue
            }

            fmt.Printf("→ Approval: %s → %s : %s\n", approval.Owner.Hex(), approval.Spender.Hex(), approval.Value.String())

        default:
            fmt.Println("알 수 없는 이벤트 시그니처:", eventSig)
        }
    }
}
```


⸻

🧪 실행 코드 통합 예시
```go
func main() {
    client, _ := ethclient.Dial("https://mainnet.infura.io/v3/YOUR_PROJECT_ID")
    txHash := common.HexToHash("0x...") // 대상 트랜잭션 해시

    receipt, err := client.TransactionReceipt(context.Background(), txHash)
    if err != nil {
        log.Fatal(err)
    }

    contractAbi, err := abi.JSON(strings.NewReader(tokenABI))
    if err != nil {
        log.Fatal(err)
    }

    decodeLogs(receipt.Logs, contractAbi)
}
```


⸻

✅ 구조 정리

| 구성 요소 | 역할 |
|---|---|
| TransferEvent, ApprovalEvent | 이벤트별 구조체 정의 |
| decodeLogs() | topics[0]로 이벤트 분기 + 자동 언팩 |
| contractAbi.Events["Transfer"].ID | 시그니처 자동 hash 매핑 |
| UnpackIntoInterface() | non-indexed 데이터 자동 디코딩 |



⸻

🧠 고급 팁
	•	여러 컨트랙트의 ABI를 map으로 관리할 수 있습니다.
	•	indexed가 없는 경우 topics 없이 data만 있는 로그도 처리 가능
	•	event가 anonymous이면 topics[0] 없이 디코딩 필요

⸻

✨ 확장 예시
	•	로그를 MongoDB에 저장
	•	Etherscan 없이 tx hash → 이벤트 추출
	•	DEX 로그 (Uniswap, SushiSwap 등) 자동 파싱 구조로 확장 가능

⸻

✅ 이렇게 하면 모든 ERC20/721/1155 이벤트를 자동으로 식별하고 디코딩할 수 있는 범용 구조가 완성됩니다.

⸻

이제 앞서 만든 고급 로그 디코더 구조를 기반으로,
WebSocket을 통해 Ethereum 블록체인 이벤트를 실시간으로 감지하고 디코딩하는 구조를 제공합니다. 🚀

⸻

✅ 목표

| 항목 | 내용 |
|---|---|
| 네트워크 | Ethereum (예: mainnet, testnet) |
| 연결 방식 | WebSocket (eth_subscribe, logs) |
| 기능 | 특정 컨트랙트의 이벤트를 실시간 수신 후 디코딩 |
| 언어 | Golang (go-ethereum 기반) |



⸻

🧩 아키텍처 요약
```plaintext
[Geth / Infura WebSocket]
        ↓ (logs subscribe)
[Golang WebSocket 클라이언트]
        ↓ (수신한 logs)
[abi + 디코더] → 해석
        ↓
[출력 / 저장 / Slack 알림 등 확장 가능]
```


⸻

📦 1. 의존성
```
go get github.com/ethereum/go-ethereum
```


⸻

🧠 2. 실시간 이벤트 수신 코드 (전체 구조)

▶️ main.go
```go
package main

import (
    "context"
    "fmt"
    "log"
    "math/big"
    "strings"

    "github.com/ethereum/go-ethereum"
    "github.com/ethereum/go-ethereum/accounts/abi"
    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/core/types"
    "github.com/ethereum/go-ethereum/ethclient"
)
```
▶️ ABI 정의 및 이벤트 구조체
```go
const tokenABI = `[{"anonymous":false,"inputs":[
  {"indexed":true,"internalType":"address","name":"from","type":"address"},
  {"indexed":true,"internalType":"address","name":"to","type":"address"},
  {"indexed":false,"internalType":"uint256","name":"value","type":"uint256"}
],"name":"Transfer","type":"event"}]`

type TransferEvent struct {
    From  common.Address
    To    common.Address
    Value *big.Int
}
```


⸻

🛰️ 3. WebSocket + Subscription 연결
```go
func main() {
    // WebSocket 연결 (Infura 또는 로컬 Geth)
    client, err := ethclient.Dial("wss://mainnet.infura.io/ws/v3/YOUR_INFURA_KEY")
    if err != nil {
        log.Fatal("WebSocket 연결 실패:", err)
    }

    contractAbi, err := abi.JSON(strings.NewReader(tokenABI))
    if err != nil {
        log.Fatal("ABI 파싱 실패:", err)
    }

    contractAddr := common.HexToAddress("0xdAC17F958D2ee523a2206206994597C13D831ec7") // 예: USDT

    query := ethereum.FilterQuery{
        Addresses: []common.Address{contractAddr},
    }

    logsCh := make(chan types.Log)
    sub, err := client.SubscribeFilterLogs(context.Background(), query, logsCh)
    if err != nil {
        log.Fatal("구독 실패:", err)
    }

    fmt.Println("이벤트 구독 시작!")

    for {
        select {
        case err := <-sub.Err():
            log.Println("Subscription 오류:", err)
        case vLog := <-logsCh:
            decodeLog(vLog, contractAbi)
        }
    }
}
```


⸻

🔍 4. 로그 디코딩 함수
```go
func decodeLog(vLog types.Log, contractAbi abi.ABI) {
    eventID := vLog.Topics[0].Hex()

    switch eventID {
    case contractAbi.Events["Transfer"].ID.Hex():
        var transfer TransferEvent
        transfer.From = common.HexToAddress(vLog.Topics[1].Hex())
        transfer.To = common.HexToAddress(vLog.Topics[2].Hex())

        err := contractAbi.UnpackIntoInterface(&transfer, "Transfer", vLog.Data)
        if err != nil {
            log.Println("Transfer decode 실패:", err)
            return
        }

        fmt.Printf("💸 Transfer Event: %s → %s, Value: %s\n", transfer.From.Hex(), transfer.To.Hex(), transfer.Value.String())
    default:
        fmt.Println("알 수 없는 이벤트:", eventID)
    }
}
```


⸻

✅ 실행 결과 예시
```text
이벤트 구독 시작!
💸 Transfer Event: 0xa9059cbb... → 0x8e6b1e... , Value: 1500000000000000000
💸 Transfer Event: ...
```


⸻

✨ 확장 아이디어

| 기능 | 설명|
|---|---|
| Slack / Telegram 알림 | 특정 주소 감지 시 메신저 전송 |
| MongoDB 저장 | 모든 이벤트를 실시간으로 기록 |
| 여러 컨트랙트 / 이벤트 | ABI + 이벤트 맵핑 구조 확장 |
| 다양한 이벤트 추적 | ERC721 Transfer, Swap, Mint 등 다양화 가능|



⸻

✅ 마무리 요약

| 항목 | 설명 |
|---|---|
|실시간 수신 | wss:// WebSocket + SubscribeFilterLogs |
| 이벤트 필터링 | FilterQuery로 컨트랙트 주소 제한 가능 |
| 디코딩 방식 | indexed → topics, 나머지 → data |
| 코드 구조화 | 고급 이벤트 매핑 분기 + 구조체 매핑 |



⸻