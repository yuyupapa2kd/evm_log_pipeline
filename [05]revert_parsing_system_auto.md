✅ 목표: 실패 트랜잭션 실시간 감지 시스템 구축

| 항목 | 내용 |
|---|---|
| 실시간	| 블록 생성될 때마다 모든 트랜잭션을 확인 |
| 실패 탐지 | status == 0 (fail) 인 트랜잭션만 추출 |
|후처리 | debug.traceTransaction으로 reason 복구 → MongoDB 저장 |

---

# 📚 전체 시스템 흐름
```plaintext
[Geth WebSocket 연결]
   ↓ (newHeads 구독)
[새 블록 도착]
   ↓
[블록 내 모든 트랜잭션 조회]
   ↓
[Tx Receipt: status == 0 체크]
   ↓
[debug.traceTransaction 호출]
   ↓
[revert reason 파싱]
   ↓
[MongoDB에 저장]

```


# 🛠️ 실전용 Golang 코드
1. WebSocket Provider로 Geth 연결
```go
import (
	"github.com/ethereum/go-ethereum/ethclient"
)

func setupWSClient() *ethclient.Client {
	client, err := ethclient.Dial("ws://localhost:8546")
	if err != nil {
		log.Fatal("WebSocket 연결 실패:", err)
	}
	return client
}

```

ws://localhost:8546 로 WebSocket 연결합니다.
(http://localhost:8545 는 안됩니다. 꼭 ws://)

# 2. 새로운 블록을 구독 (newHeads)
```go
import (
	"context"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/event"
)

func subscribeNewHeads(client *ethclient.Client) (event.Subscription, chan *types.Header) {
	headers := make(chan *types.Header)
	sub, err := client.SubscribeNewHead(context.Background(), headers)
	if err != nil {
		log.Fatal("블록 구독 실패:", err)
	}
	return sub, headers
}
```


# 3. 블록 안의 트랜잭션들 가져오기
```go
import (
	"github.com/ethereum/go-ethereum/common"
)

func fetchBlockTransactions(client *ethclient.Client, blockHash common.Hash) ([]*types.Transaction, error) {
	block, err := client.BlockByHash(context.Background(), blockHash)
	if err != nil {
		return nil, err
	}
	return block.Transactions(), nil
}
```

# 4. 각 트랜잭션의 성공/실패 체크
```go
func isTxFailed(client *ethclient.Client, txHash common.Hash) (bool, error) {
	receipt, err := client.TransactionReceipt(context.Background(), txHash)
	if err != nil {
		return false, err
	}
	return receipt.Status == 0, nil
}
```
- receipt.Status == 1 ➔ 성공

- receipt.Status == 0 ➔ 실패

# 5. 실패 트랜잭션 처리
앞서 만든 debug.traceTransaction, parseRevertReason, insertMongo 함수를 그대로 사용합니다.

(조합하면 됩니다.)

# 6. 전체 Main 흐름
```go
func main() {
	wsClient := setupWSClient()
	sub, headers := subscribeNewHeads(wsClient)

	for {
		select {
		case err := <-sub.Err():
			log.Println("Subscription error:", err)
		case header := <-headers:
			fmt.Println("새 블록:", header.Number.String())

			txs, err := fetchBlockTransactions(wsClient, header.Hash())
			if err != nil {
				log.Println("블록 가져오기 실패:", err)
				continue
			}

			for _, tx := range txs {
				failed, err := isTxFailed(wsClient, tx.Hash())
				if err != nil {
					log.Println("Receipt 확인 실패:", err)
					continue
				}
				if failed {
					fmt.Println("실패한 트랜잭션 발견:", tx.Hash().Hex())

					// 여기서 debug.traceTransaction + memory 파싱 + Mongo 저장
					// (이전 답변에서 제공한 코드 이용)
				}
			}
		}
	}
}
```

# ✅ 이렇게 하면
새로운 블록이 생성되면

그 안의 트랜잭션을 모두 검사

실패한 트랜잭션만 뽑아서

자동으로 revert reason을 복구

MongoDB에 저장

➡️ 완전 자동화된 실패 감지 및 기록 시스템이 완성됩니다. 🚀

