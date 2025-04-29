
✅ 1. Ethereum types.Transaction 구조체 설명

Go-Ethereum(Geth)의 types.Transaction은 실제 Ethereum 블록체인 상의 트랜잭션을 표현하는 구조체입니다.

🔍 주요 필드
```go
type Transaction struct {
    data txdata
    // 기타 내부 필드
}
```
하지만 일반적으로는 tx.To(), tx.Value(), tx.Data(), tx.GasPrice() 등 getter 메서드를 통해 접근합니다.

⸻

📘 주요 메서드 및 설명

| 메서드 | 설명 |
|---|---|
| tx.Hash() | 트랜잭션 해시 |
| tx.To() | 수신 주소 (컨트랙트 생성 시 nil) |
| tx.Value() | 이더 전송량 (wei 단위) |
| tx.Data() | 입력 데이터 (보통 함수 호출 정보) |
| tx.Gas() | Gas limit |
| tx.GasPrice() | 트랜잭션 gas price (legacy) |
| tx.GasFeeCap() | maxFeePerGas (EIP-1559, optional) |
| tx.GasTipCap() | maxPriorityFeePerGas (EIP-1559, optional) |
| tx.ChainId() | 체인 ID |
| tx.Nonce() | Nonce |
| tx.Type() | 0x0, 0x1, 0x2 |



⸻

✅ 2. Golang 코드: 새 블록 생성 시 모든 트랜잭션 조회 및 정보 출력/저장

📦 전제
	•	Geth 또는 Infura와 WebSocket 연결
	•	각 블록마다 트랜잭션을 순회
	•	트랜잭션 요약 정보를 출력 (Mongo 저장 등도 쉽게 확장 가능)

⸻

▶️ 전체 코드 예시
```go
package main

import (
    "context"
    "fmt"
    "log"
    "math/big"

    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/core/types"
    "github.com/ethereum/go-ethereum/ethclient"
)

func main() {
    client, err := ethclient.Dial("wss://mainnet.infura.io/ws/v3/YOUR_PROJECT_ID")
    if err != nil {
        log.Fatal("WebSocket 연결 실패:", err)
    }

    // 새 블록을 구독
    headers := make(chan *types.Header)
    sub, err := client.SubscribeNewHead(context.Background(), headers)
    if err != nil {
        log.Fatal("블록 구독 실패:", err)
    }

    fmt.Println("새 블록 구독 시작...")

    for {
        select {
        case err := <-sub.Err():
            log.Println("Subscription 오류:", err)

        case header := <-headers:
            fmt.Printf("\n\n📦 블록 #%v (Hash: %s)\n", header.Number.String(), header.Hash().Hex())

            block, err := client.BlockByHash(context.Background(), header.Hash())
            if err != nil {
                log.Println("블록 조회 실패:", err)
                continue
            }

            for _, tx := range block.Transactions() {
                from, err := types.Sender(types.LatestSignerForChainID(tx.ChainId()), tx)
                if err != nil {
                    log.Println("Sender 추출 실패:", err)
                    continue
                }

                to := tx.To()
                toAddr := "컨트랙트 생성"
                if to != nil {
                    toAddr = to.Hex()
                }

                fmt.Printf("🔹 Tx: %s\n", tx.Hash().Hex())
                fmt.Printf("    From: %s\n", from.Hex())
                fmt.Printf("    To:   %s\n", toAddr)
                fmt.Printf("    Value: %s wei\n", tx.Value().String())
                fmt.Printf("    Gas: %d @ price: %s\n", tx.Gas(), tx.GasPrice().String())
                fmt.Printf("    Nonce: %d, Type: %d\n", tx.Nonce(), tx.Type())
                fmt.Println("    Data:", tx.Data())
            }
        }
    }
}
```


⸻

✅ 출력 예시

📦 블록 #19374123 (Hash: 0xabc...)
```
🔹 Tx: 0x9ef...
    From: 0x123...
    To:   0xdef...
    Value: 1000000000000000000 wei
    Gas: 21000 @ price: 23000000000
    Nonce: 12, Type: 2
    Data: []
```


⸻

🧠 확장 아이디어

| 항목 | 내용 |
|---|---|
| MongoDB 저장 | JSON 변환 후 직접 DB에 저장 |
| 특정 주소 필터 | 관심 있는 from 또는 to 주소만 추적 |
| 컨트랙트 호출 분석 | tx.Data()를 ABI로 파싱하여 메서드 디코딩 |
| 이벤트 로그 수신 | client.TransactionReceipt() + logs 디코딩 |



⸻

✅ 마무리 요약

| 항목 | 설명 |
|---|---|
| types.Transaction | 이더리움 트랜잭션의 핵심 구조체 |
| 주요 필드 | From, To, Value, Gas, Type, Data 등 |
| 실시간 추적 | SubscribeNewHead로 새 블록을 모니터링 |
| 저장 처리	| 로그 → MongoDB 또는 CSV로 저장 가능 |



⸻

➡️ 이어서 tx.Data() 안의 내용을 실제로 ABI로 파싱해서 어떤 함수가 호출된 건지 추출하는 예시
⸻

✅ 목표
	•	tx.Data()에서 함수 시그니처 추출
	•	해당 함수 이름 + 파라미터 값 디코딩
	•	Solidity에서 작성한 함수의 ABI 기반으로 파싱

⸻

📦 예시 컨트랙트 (Solidity 기준)
```solidity
contract Token {
    function transfer(address to, uint256 amount) public returns (bool);
}
```


⸻

🧠 calldata 구조 (tx.Data)
```text
[0:4]     → 함수 선택자 (function selector, 4바이트)
[4:...]   → 인코딩된 파라미터 (ABI encoding)
```
예시:
```text
a9059cbb000000000000000000000000d8dA6BF26964aF9D7eEd9e03E53415D37aA96045
00000000000000000000000000000000000000000000000000000000000003e8
```
	•	a9059cbb → transfer(address,uint256) 해시 앞 4바이트
	•	나머지: 인자 두 개 (주소, uint256)

⸻

✅ Golang으로 ABI 파싱 및 함수 디코딩

1. ABI 정의
```go
const erc20ABI = `[{
    "constant": false,
    "inputs": [
        {"name": "to", "type": "address"},
        {"name": "value", "type": "uint256"}
    ],
    "name": "transfer",
    "outputs": [{"name": "", "type": "bool"}],
    "type": "function"
}]`
```
2. Go 코드 예시
```go
package main

import (
    "encoding/hex"
    "fmt"
    "log"
    "strings"

    "github.com/ethereum/go-ethereum/accounts/abi"
    "github.com/ethereum/go-ethereum/common"
)

func main() {
    dataHex := "0xa9059cbb000000000000000000000000d8dA6BF26964aF9D7eEd9e03E53415D37aA9604500000000000000000000000000000000000000000000000000000000000003e8"
    data, err := hex.DecodeString(strings.TrimPrefix(dataHex, "0x"))
    if err != nil {
        log.Fatal("hex decode error:", err)
    }

    parsedABI, err := abi.JSON(strings.NewReader(erc20ABI))
    if err != nil {
        log.Fatal("ABI parse error:", err)
    }

    method, err := parsedABI.MethodById(data[:4])
    if err != nil {
        log.Fatal("method lookup error:", err)
    }

    fmt.Println("호출된 함수 이름:", method.Name)

    args := make(map[string]interface{})
    err = method.Inputs.UnpackIntoMap(args, data[4:])
    if err != nil {
        log.Fatal("파라미터 디코딩 실패:", err)
    }

    for name, value := range args {
        fmt.Printf(" - %s: %v\n", name, value)
    }
}
```


⸻

✅ 실행 결과
```plaintext
호출된 함수 이름: transfer
 - to: 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
 - value: 1000
```


⸻

✅ 요약

| 단계 | 설명 |
|---|---|
| 1. data[:4] | 함수 시그니처 추출 (selector) |
| 2. abi.MethodById() | ABI에서 함수 매칭 |
| 3. abi.UnpackIntoMap() | 인자 디코딩 |
| 4. 출력 | 함수명 + 각 파라미터 값 |



⸻

✅ 실전 팁
	•	여러 함수가 존재하는 컨트랙트라면 전체 ABI를 통째로 가져와야 합니다.
	•	eth_getCode로 address에 배포된 바이트코드 → ABI 매핑하려면 미리 ABI 수집해둬야 합니다 (Etherscan API 등 사용 가능).
	•	unknown 함수 selector는 트랜잭션 디코더 DB (like 4byte.directory)를 통해 추정 가능

⸻


이번에는 Golang으로 블록 전체를 순회하며 트랜잭션의 tx.Data()를 ABI 기반으로 자동 디코딩하는 구조를 제공합니다.
이 구조는 특정 컨트랙트의 ABI만 있으면, 트랜잭션이 어떤 함수 호출인지 자동으로 추적하고 디코딩할 수 있습니다.

⸻

✅ 목표

| 항목 | 설명 |
|---|---|
| 입력 | 블록 헤더 구독 → 해당 블록의 모든 트랜잭션 순회 |
| 디코딩 대상 | tx.Data() (calldata) |
| 사용 도구 | Go + geth (github.com/ethereum/go-ethereum) |
| ABI 사용 | 컨트랙트 ABI (ERC20, ERC721, 사용자 정의 등) |
| 결과 | 호출된 함수명 및 파라미터를 콘솔 또는 MongoDB 등에 저장 |



⸻
```text
📦 사전 준비
	1.	ABI 파일을 JSON 문자열로 준비 (Solidity 컴파일 시 artifacts/MyContract.json에서 추출 가능)
	2.	추적 대상 컨트랙트 주소를 알고 있어야 함
```
⸻

🧠 핵심 구조 요약
```plaintext
[블록 헤더 수신]
     ↓
[BlockByHash → tx 리스트]
     ↓
[tx.Data() 추출]
     ↓
[ABI → method selector 매칭]
     ↓
[Unpack → 파라미터 디코딩]
     ↓
[출력 or 저장]
```


⸻

🧩 전체 코드 예시 (자동 ABI 디코딩 포함)
```go
package main

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
🔧 ABI 정의
```go
const erc20ABI = `[{
    "constant":false,
    "inputs":[
        {"name":"to","type":"address"},
        {"name":"value","type":"uint256"}
    ],
    "name":"transfer",
    "outputs":[{"name":"","type":"bool"}],
    "type":"function"
}]`
```


⸻

🚀 메인 함수
```go
func main() {
    client, err := ethclient.Dial("wss://mainnet.infura.io/ws/v3/YOUR_INFURA_KEY")
    if err != nil {
        log.Fatal("WebSocket 연결 실패:", err)
    }

    contractAddr := common.HexToAddress("0xdAC17F958D2ee523a2206206994597C13D831ec7") // 예: USDT

    parsedABI, err := abi.JSON(strings.NewReader(erc20ABI))
    if err != nil {
        log.Fatal("ABI 파싱 실패:", err)
    }

    headers := make(chan *types.Header)
    sub, err := client.SubscribeNewHead(context.Background(), headers)
    if err != nil {
        log.Fatal("블록 구독 실패:", err)
    }

    fmt.Println("블록 구독 시작...")

    for {
        select {
        case err := <-sub.Err():
            log.Println("Subscription 오류:", err)

        case header := <-headers:
            block, err := client.BlockByHash(context.Background(), header.Hash())
            if err != nil {
                log.Println("블록 조회 실패:", err)
                continue
            }

            fmt.Printf("📦 블록 #%s (%d개 트랜잭션)\n", block.Number().String(), len(block.Transactions()))

            for _, tx := range block.Transactions() {
                to := tx.To()
                if to == nil || *to != contractAddr {
                    continue // ABI 대상이 아니면 스킵
                }

                if len(tx.Data()) < 4 {
                    continue // 함수 호출이 아님
                }

                methodID := tx.Data()[:4]
                method, err := parsedABI.MethodById(methodID)
                if err != nil {
                    fmt.Println("⚠️ 알 수 없는 함수 호출:", hex.EncodeToString(methodID))
                    continue
                }

                args := make(map[string]interface{})
                err = method.Inputs.UnpackIntoMap(args, tx.Data()[4:])
                if err != nil {
                    log.Println("파라미터 디코딩 실패:", err)
                    continue
                }

                from, _ := types.Sender(types.LatestSignerForChainID(tx.ChainId()), tx)

                fmt.Printf("🔹 Tx %s: %s 호출\n", tx.Hash().Hex(), method.Name)
                fmt.Printf("    From: %s\n", from.Hex())
                for k, v := range args {
                    fmt.Printf("    %s: %v\n", k, v)
                }
            }
        }
    }
}
```


⸻

✅ 출력 예시
```text
📦 블록 #19887431 (25개 트랜잭션)
🔹 Tx 0xabcd...: transfer 호출
    From: 0x123...
    to: 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
    value: 1000
```


⸻

🧠 확장 아이디어

| 확장 기능 | 설명 |
|---|---|
| MongoDB 저장 | 디코딩 결과를 DB에 저장하여 분석 및 검색 |
| 여러 컨트랙트 지원 | ABI 맵을 map[common.Address]abi.ABI 형태로 관리 |
| fallback, delegatecall 추적 | trace 활용 (debug_traceTransaction) |
| 비동기화 처리 | 고루틴과 채널로 처리 속도 개선 가능 |



⸻

✅ 정리 요약

| 항목 | 설명 |
|---|---|
| tx.Data()[:4] | 함수 selector (4 bytes) |
| abi.MethodById() | selector → 함수 이름 매핑 |
| abi.UnpackIntoMap() | 파라미터 디코딩 |
| 필터링 대상 | 특정 컨트랙트 주소만 추적 가능 |
| 사용처 | 디버깅, 거래 추적, 블록 모니터링, 통계 등 |



⸻

✅ 목표 요약
	1.	Fallback 함수 및 delegatecall 추적
	2.	여러 컨트랙트 주소에 대해 ABI를 매핑하고 자동 디코딩하는 구조

⸻

📦 사전 구성
	•	Geth Full Node 필요 (fallback 및 delegatecall 추적 위해 debug_traceTransaction 사용)
	•	ABI 파일은 JSON 문자열로 미리 확보 (여러 컨트랙트용)

⸻

🧩 1. ABI 맵핑 구조 정의
```go
// address 별 ABI 매핑
var abiMap = map[common.Address]abi.ABI{}

// 초기화 예시
func initABIs() {
    abis := map[string]string{
        "0xdAC17F958D2ee523a2206206994597C13D831ec7": usdtABI,
        "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48": usdcABI,
        // 추가 가능
    }

    for addrStr, abiStr := range abis {
        addr := common.HexToAddress(addrStr)
        parsed, err := abi.JSON(strings.NewReader(abiStr))
        if err != nil {
            log.Fatalf("ABI 파싱 실패 (%s): %v", addrStr, err)
        }
        abiMap[addr] = parsed
    }
}
```


⸻

🛰️ 2. 트랜잭션 트레이싱 + fallback / delegatecall 추적
```go
func traceTx(txHash common.Hash, client *rpc.Client) {
    var traceResult map[string]interface{}

    err := client.CallContext(context.Background(), &traceResult, "debug_traceTransaction", txHash.Hex(), map[string]interface{}{
        "tracer": "callTracer",
    })
    if err != nil {
        log.Println("trace 실패:", err)
        return
    }

    // 기본 추적 결과
    fmt.Printf("🔍 Trace for %s\n", txHash.Hex())

    // 간단 파싱 예시
    if traceResult["type"] == "CALL" || traceResult["type"] == "DELEGATECALL" || traceResult["type"] == "CALLCODE" {
        to := traceResult["to"].(string)
        input := traceResult["input"].(string)
        callType := traceResult["type"].(string)
        fmt.Printf("  %s → %s (%d bytes input)\n", callType, to, len(input))

        decodeTxData(common.HexToAddress(to), input)
    }

    if traceResult["type"] == "FALLBACK" {
        fmt.Println("⚠️ fallback function 호출 감지")
    }
}
```


⸻

🧠 3. ABI 맵 기반 디코더 (공통화)
```go
func decodeTxData(to common.Address, dataHex string) {
    abiDef, ok := abiMap[to]
    if !ok {
        fmt.Println("알 수 없는 ABI 주소:", to.Hex())
        return
    }

    data, err := hex.DecodeString(strings.TrimPrefix(dataHex, "0x"))
    if err != nil || len(data) < 4 {
        fmt.Println("데이터 디코딩 실패 또는 함수 아닌 입력")
        return
    }

    method, err := abiDef.MethodById(data[:4])
    if err != nil {
        fmt.Println("ABI에서 method 매칭 실패:", err)
        return
    }

    args := make(map[string]interface{})
    if err := method.Inputs.UnpackIntoMap(args, data[4:]); err != nil {
        fmt.Println("파라미터 언팩 실패:", err)
        return
    }

    fmt.Printf("🧠 %s() 호출 → %v\n", method.Name, args)
}
```


⸻

✅ 전체 흐름 통합
```go
func main() {
    ethClient, _ := ethclient.Dial("wss://mainnet.infura.io/ws/v3/...")
    rpcClient, _ := rpc.Dial("https://mainnet.infura.io/v3/...")

    initABIs()

    headers := make(chan *types.Header)
    ethClient.SubscribeNewHead(context.Background(), headers)

    for header := range headers {
        block, _ := ethClient.BlockByHash(context.Background(), header.Hash())

        for _, tx := range block.Transactions() {
            fmt.Printf("\n⛓️ 블록 #%d 트랜잭션 %s\n", block.Number().Uint64(), tx.Hash().Hex())
            traceTx(tx.Hash(), rpcClient)
        }
    }
}
```


⸻

✅ 출력 예시
```text
⛓️ 블록 #19888888 트랜잭션 0xabc...

🔍 Trace for 0xabc...
  CALL → 0xdAC1... (68 bytes input)
🧠 transfer() 호출 → map[to:0xd8dA6BF2..., value:1000000000000]

🔍 Trace for 0xdef...
  DELEGATECALL → 0xProxyContract...
🧠 executeMetaTx() 호출 → map[user:0xabc...]
```


⸻

🔐 보안 및 실전 팁

| 항목 | 설명 |
|---|---|
| fallback 디코딩 | data.length == 0 이면 fallback 함수로 간주 |
| unknown selector | 4byte.directory + method guessing API 활용 가능 |
| proxy 패턴 추적 | delegatecall to 주소 + 실제 ABI 구분 필요 |
| trace 옵션 | callTracer, vmTracer, prestateTracer 다양하게 활용 가능 |



⸻
