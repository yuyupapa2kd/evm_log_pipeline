
# ✅ 1. 실패하는 트랜잭션의 스택 분석 방법

## 🧠 기본 개념

트랜잭션이 실패할 때:

- EVM은 `REVERT` 명령어를 실행합니다.
- 실패 지점의 **stack**, **memory**, **storage** 상태를 보면
- **왜 실패했는지**를 알 수 있습니다.

특히:

| 항목 | 분석 포인트 |
|:---|:---|
| stack 최상단 | 보통 revert reason 코드 or 반환값 |
| memory 영역 | 에러 메세지가 들어있을 수 있음 (ABI 인코딩 형태) |
| 실행된 마지막 opcode | `REVERT` |

---

## 📋 실패 트랜잭션 분석 흐름

1. `debug.traceTransaction(txHash)`로 트랜잭션을 추적합니다.
2. 반환된 **structLogs**를 순서대로 분석합니다.
3. `op == "REVERT"` 찾습니다.
4. 그 직전 stack/memory를 확인합니다.
5. 문제 발생 지점을 찾아냅니다.

---

## 🔥 예시 분석

**structLog 마지막 부분 예시**

```json
{
  "pc": 228,
  "op": "REVERT",
  "gasCost": 0,
  "gas": 0,
  "depth": 1,
  "stack": [
    "0x40",    // memory offset
    "0x20"     // memory length
  ],
  "memory": [
    "00000020....00000100"  // encoded error message
  ]
}
```

여기서:

- stack의 첫 번째 값은 memory 위치 (offset)
- 두 번째 값은 memory에서 읽어올 길이 (length)
- memory 내용을 해석하면 → 에러 메세지가 나옵니다.

**(예: 'SafeMath: subtraction overflow' 같은 revert reason)**

---

## 🧠 더 깊은 이해

- `require(false, "message")` 이런 코드가 있다면
- 컴파일시 EVM은 `REVERT`와 함께 메모리에 **에러메시지**를 인코딩해서 넣습니다.
- 이 메모리는 **Solidity Error Standard**를 따릅니다.

(즉, memory를 디코딩하면 사람이 읽을 수 있는 에러를 복구할 수 있어요.)

---

# 🛠️ 실패 트랜잭션 분석 요령

| 단계 | 해야 할 일 |
|:---|:---|
| 1 | debug.traceTransaction으로 트랜잭션 로그 가져오기 |
| 2 | structLogs에서 REVERT 위치 찾기 |
| 3 | 해당 시점 stack, memory 살펴보기 |
| 4 | memory를 UTF-8로 디코딩해서 에러메세지 복구 |
| 5 | 실패 위치 이전의 stack, opcode들을 따라가서 원인 찾기 |

---
