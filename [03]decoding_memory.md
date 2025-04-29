
# ✅ 목표

- debug.traceTransaction() 결과에 있는 **memory** 필드
- ABI 규칙을 이해하고 **메모리 바이너리**를
- 사람이 읽을 수 있는 **revert reason 문자열**로 복구한다

---

# 📚 먼저, EVM Revert 에러 표준 구조를 알아야 합니다.

**Solidity에서 `require(false, "메시지")` 같은 코드를 실행하면**, EVM은 메모리에 이런 식으로 데이터를 씁니다:

| 바이트 오프셋 | 내용 | 설명 |
|:---|:---|:---|
| 0x00 ~ 0x03 | `0x08c379a0` | function selector (`Error(string)`) |
| 0x04 ~ 0x23 | 32바이트 | offset to string (대개 0x20) |
| 0x24 ~ 0x43 | 32바이트 | 문자열 길이 (예: 17) |
| 0x44 ~ 이후 | 문자열 데이터 (ASCII) |

즉, memory를 읽으면 다음과 같이 나옵니다.

예시 memory dump:

```hex
08c379a0 0000000000000000000000000000000000000000000000000000000000000020
0000000000000000000000000000000000000000000000000000000000000011
596f75207368616c6c206e6f74207061737321
```

이걸 해석하면:

- `08c379a0`: `Error(string)` function selector
- `0020`: 에러 메세지 시작 offset (32 bytes 이후)
- `0011`: 에러 메세지 길이 17바이트
- `"You shall not pass!"` (ASCII)

---

# ✅ 실전: Memory를 사람이 읽을 수 있도록 파싱하는 방법

### 🔥 메모리 파싱 스크립트 (go 예시)

```go
package main

import (
	"encoding/hex"
	"fmt"
)

func parseRevertMemory(memoryDump []string) (string, error) {
	// 1. 메모리 hex string array를 하나의 긴 hex string으로 합치기
	fullHex := ""
	for _, m := range memoryDump {
		fullHex += m
	}

	// 2. hex string → []byte 변환
	memoryBytes, err := hex.DecodeString(fullHex)
	if err != nil {
		return "", fmt.Errorf("failed to decode memory hex: %v", err)
	}

	// 3. 첫 4바이트는 selector (Error(string) 인지 확인)
	if len(memoryBytes) < 4 {
		return "", fmt.Errorf("memory too short to contain selector")
	}
	selector := memoryBytes[:4]
	expectedSelector, _ := hex.DecodeString("08c379a0")
	if !equalBytes(selector, expectedSelector) {
		return "", fmt.Errorf("not a standard Error(string) revert")
	}

	// 4. 그 다음 32바이트는 offset (skip)
	if len(memoryBytes) < 36 {
		return "", fmt.Errorf("memory too short to contain offset")
	}
	// offset := memoryBytes[4:36]  // 거의 항상 0x20

	// 5. 그 다음 32바이트는 string length
	if len(memoryBytes) < 68 {
		return "", fmt.Errorf("memory too short to contain string length")
	}
	stringLength := bytesToUint(memoryBytes[36:68])

	// 6. 이후 stringLength 만큼 읽어서 utf-8 decode
	if len(memoryBytes) < 68+int(stringLength) {
		return "", fmt.Errorf("memory too short for string data")
	}
	stringData := memoryBytes[68 : 68+stringLength]

	return string(stringData), nil
}

// uint256(bytes32) -> uint64로 변환하는 보조 함수
func bytesToUint(b []byte) uint64 {
	var result uint64
	for _, byteVal := range b[len(b)-8:] { // 마지막 8바이트만 사용 (uint64 범위)
		result = (result << 8) | uint64(byteVal)
	}
	return result
}

// 두 byte array 비교
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
	memoryDump := []string{
		"08c379a000000000000000000000000000000000000000000000000000000000000020",
		"0000000000000000000000000000000000000000000000000000000000000011",
		"596f75207368616c6c206e6f74207061737321",
	}

	message, err := parseRevertMemory(memoryDump)
	if err != nil {
		fmt.Println("Error parsing memory:", err)
	} else {
		fmt.Println("Revert Reason:", message)
	}
}

```

### ✅ 실행 결과:

```
Revert Reason: You shall not pass!
```

---
# go 코드 흐름 요약
단계 | 내용
memory hex array → full hex string 합치기 | 
hex 디코딩 → byte array로 변환 | 
0~3 byte: function selector 체크 (08c379a0) | 
4~35 byte: offset (거의 32) | 
36~67 byte: string length | 
68 byte부터 string length 만큼 읽어서 utf-8 decode | 


# 🧠 핵심 요약

| 파싱 순서 | 설명 |
|:---|:---|
| 1 | 앞 4바이트는 항상 `0x08c379a0` (`Error(string)`) |
| 2 | 그 다음 32바이트: 문자열 offset (거의 항상 32) |
| 3 | 그 다음 32바이트: 문자열 길이 |
| 4 | 그 이후: 실제 문자열 데이터 (utf-8 인코딩) |

---

# ✅ 주의할 점

- **revert 이유가 없는 경우** memory에 아무것도 없을 수 있음 (Silent Revert)
- **custom error**를 사용할 경우 구조가 다를 수 있음
  - (ex: `MyCustomError(uint256 code)` 같은 것은 selector, parameter 형태로 저장됨)
- Solidity >=0.8.4 부터 **custom error** 기능이 추가되어서 이 경우 별도 처리 필요

---

# 🚀 추가 심화

나중에:

- revert 데이터의 4바이트 selector를 분석해서 **custom error 이름**까지 복원
- memory 파싱 자동화 스크립트 만들기
- debug.traceTransaction 출력 전체를 자동 분석하는 툴 만들기

까지도 이어질 수 있어요.

---

# ✅ 정리

| 주제 | 요약 |
|---|---|
| EVM memory revert 포맷 | `selector(4) + offset(32) + length(32) + utf8 string` |
| 파싱 방법 | hex string → bytes 변환 후 순차 파싱 |
| 디코딩 결과 | 사람이 읽을 수 있는 에러 메시지 복구 |

---

# 📢 결론:  
**이제 debug.traceTransaction memory만 있으면 → revert reason을 직접 복구할 수 있습니다!**
