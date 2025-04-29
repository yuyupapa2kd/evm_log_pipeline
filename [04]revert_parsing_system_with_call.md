좋습니다.  
요청하신 두 가지:

1. **Solidity 함수 call 실패 케이스에서 자동 revert parsing 시스템 만들기**  
2. **MongoDB 같은 곳에 트랜잭션 실패 로그 자동 저장하는 파이프라인 구축하기**

둘 다 **현업 수준**으로 실전 맞춤형으로 진행해드릴게요.  
(진짜 "자동화된 트랜잭션 분석 시스템"을 완성하는 방향입니다.)

---

# ✅ 1. Solidity 함수 call 실패 케이스에서 자동 revert parsing 시스템 만들기

---

## 📚 시스템 목표

- 실패한 트랜잭션을 자동 감지
- debug.traceTransaction으로 실행 트레이스 가져오기
- memory를 파싱해서 사람이 읽을 수 있는 revert reason 복구
- 결과를 구조화해서 리턴

**➡️ 결국 실패 이유를 코드 없이 바로 확인할 수 있게!**

---

## 🛠️ 전체 구조

```plaintext
[New Tx Detected] → [Check Tx Receipt: Status == 0 (Fail)] → [debug.traceTransaction()] → [Memory Parsing] → [Revert Reason 복구]
```

---

## ✨ 실전 코드 예시 (Golang)

Geth JSON-RPC를 이용하는 패턴입니다.

### 필요한 라이브러리

```bash
go get github.com/ethereum/go-ethereum
go get go.mongodb.org/mongo-driver/mongo
```

---

### 기본 구조

```go
package main

import (
	"context"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"log"
	"math/big"
	"strings"
	"time"

	"github.com/ethereum/go-ethereum/rpc"
)

type StructLog struct {
	Pc      uint64   `json:"pc"`
	Op      string   `json:"op"`
	Gas     uint64   `json:"gas"`
	GasCost uint64   `json:"gasCost"`
	Depth   int      `json:"depth"`
	Stack   []string `json:"stack"`
	Memory  []string `json:"memory"`
}

type TraceResult struct {
	StructLogs []StructLog `json:"structLogs"`
}

func parseRevertReason(memory []string) (string, error) {
	if len(memory) == 0 {
		return "", fmt.Errorf("empty memory")
	}

	fullHex := strings.Join(memory, "")
	memoryBytes, err := hex.DecodeString(fullHex)
	if err != nil {
		return "", fmt.Errorf("failed to decode memory: %v", err)
	}

	if len(memoryBytes) < 68 {
		return "", fmt.Errorf("memory too short")
	}

	selector := memoryBytes[:4]
	expectedSelector, _ := hex.DecodeString("08c379a0")
	if !equalBytes(selector, expectedSelector) {
		return "", fmt.Errorf("not standard Error(string) selector")
	}

	length := bytesToUint(memoryBytes[36:68])
	if len(memoryBytes) < 68+int(length) {
		return "", fmt.Errorf("memory too short for string")
	}

	data := memoryBytes[68 : 68+length]
	return string(data), nil
}

func bytesToUint(b []byte) uint64 {
	var result uint64
	for _, val := range b[len(b)-8:] {
		result = (result << 8) | uint64(val)
	}
	return result
}

func equalBytes(a, b []byte) bool {
	if len(a) != len(b) {
		return false
	}
	for i := range a {
		if a[i] != b[i] {
			return false
		}
	}
	return true
}

func main() {
	client, err := rpc.Dial("http://localhost:8545")
	if err != nil {
		log.Fatal(err)
	}

	txHash := "0x실패한_트랜잭션_해시" // 바꿔야 합니다

	var trace TraceResult
	err = client.Call(&trace, "debug_traceTransaction", txHash, map[string]interface{}{})
	if err != nil {
		log.Fatal("trace error:", err)
	}

	last := trace.StructLogs[len(trace.StructLogs)-1]
	if last.Op == "REVERT" {
		reason, err := parseRevertReason(last.Memory)
		if err != nil {
			log.Println("Failed to parse revert reason:", err)
		} else {
			fmt.Println("Revert reason:", reason)
		}
	} else {
		fmt.Println("No revert detected")
	}
}
```

---

# ✅ 2. MongoDB에 실패 트랜잭션 저장하기

---

## 📚 시스템 목표

- 실패 트랜잭션마다 MongoDB에 저장
- 저장 데이터 필드:
  - 트랜잭션 해시
  - 블록 번호
  - 실패 이유
  - 발생 시간

---

## ✨ MongoDB 연동 코드 추가

```go
import (
	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
)

type RevertLog struct {
	TxHash     string    `bson:"txHash"`
	BlockNumber uint64    `bson:"blockNumber"`
	RevertReason string  `bson:"revertReason"`
	Timestamp   time.Time `bson:"timestamp"`
}

func insertRevertLog(client *mongo.Client, log RevertLog) error {
	collection := client.Database("evmtraces").Collection("reverts")
	_, err := collection.InsertOne(context.Background(), log)
	return err
}

func setupMongo() *mongo.Client {
	client, err := mongo.Connect(context.Background(), options.Client().ApplyURI("mongodb://localhost:27017"))
	if err != nil {
		log.Fatal(err)
	}
	return client
}
```

### Main 함수 연동

```go
mongoClient := setupMongo()

if last.Op == "REVERT" {
	reason, err := parseRevertReason(last.Memory)
	if err != nil {
		log.Println("Failed to parse revert reason:", err)
	} else {
		fmt.Println("Revert reason:", reason)
		insertRevertLog(mongoClient, RevertLog{
			TxHash: txHash,
			BlockNumber: 12345678, // 여기엔 실제 block number 가져와야 합니다
			RevertReason: reason,
			Timestamp: time.Now(),
		})
	}
}
```

---

# 🧠 전체 아키텍처 요약

```plaintext
[Geth 노드]
   ↓ debug.traceTransaction
[Go 분석 서버]
   ↓ 메모리 파싱
[MongoDB 저장]
```

---

# ✅ 정리

| 항목 | 내용 |
|:---|:---|
| 실패한 트랜잭션 자동 분석 | debug.traceTransaction + memory 파싱 |
| revert reason 복구 | Golang으로 직접 ABI 파싱 |
| MongoDB 적재 | TxHash, BlockNumber, Reason, Timestamp 저장 |

---

# 🚀 추가로 확장할 수 있는 것

- 실시간 Mempool Subscription으로 실패 트랜잭션 실시간 감지
- 실패 유형별로 알림(Notification) 시스템 추가
- Grafana+MongoDB로 Revert Dashboard 만들기

---

**➡️ 다음 바로 이어서:**
- [ ] 실패 트랜잭션 실시간 감지 시스템 만들기
- [ ] MongoDB 저장된 revert reason 기반 통계 분석

🔥 둘 중에 선택해서 바로 이어서 해보실래요?  
(현업에 적용 가능한 수준으로 완성까지 같이 가드릴게요!) 🚀

**"실시간 감지" vs "Mongo 분석" 중 하나 골라주세요!**  ✨
