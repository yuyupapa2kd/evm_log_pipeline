Solidity의 integer overflow/underflow 처리는 버전 0.8.0에서 중대한 변화가 있었습니다. 이 변화는 보안성과 코드 간결성 측면에서 매우 중요합니다. 아래에서는 변화의 전후를 코드와 내부 작동 방식 기반으로 상세하게 설명합니다.

⸻

📌 1. Solidity 0.8.0 이전 (예: 0.7.x)

overflow/underflow 검사 ❌: 수동으로 관리해야 했음

🔐 예시: 오버플로우 발생
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.7.6;

contract OverflowExamplePre080 {
    function overflow() public pure returns (uint8) {
        uint8 x = 255;
        return x + 1; // 🔥 0 반환됨 (silent overflow)
    }
}
```
⚠️ 설명
	•	uint8의 최대값은 255
	•	x + 1 → 256이지만, 8비트에서 wrap-around되어 0이 됨
	•	오류 발생 ❌, 예외 없이 조용히 진행

⸻

🔐 SafeMath 사용 예시 (OpenZeppelin)
```solidity
import "@openzeppelin/contracts/math/SafeMath.sol";

contract SafeExample {
    using SafeMath for uint256;

    function safeAdd(uint256 a, uint256 b) public pure returns (uint256) {
        return a.add(b); // 내부에서 overflow 검사 수행
    }
}
```
내부 구현 (SafeMath.sol)
```solidity
function add(uint256 a, uint256 b) internal pure returns (uint256) {
    uint256 c = a + b;
    require(c >= a, "SafeMath: addition overflow"); // ⛔ 수동 검사
    return c;
}
```


⸻

📌 2. Solidity 0.8.0 이후 (예: 0.8.0+)

overflow/underflow 검사 ✅: 자동 내장됨

🔐 예시: 오버플로우 발생
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract OverflowExamplePost080 {
    function overflow() public pure returns (uint8) {
        uint8 x = 255;
        return x + 1; // 🔥 예외 발생: arithmetic overflow
    }
}
```
⚙️ 내부 동작 (Solidity Compiler)
	•	Solidity는 내부적으로 checked arithmetic을 적용
	•	add, sub, mul, div 등의 연산은 자동으로 예외 (Panic(uint256))를 발생시킴

🔍 예외 메시지
```text
transact to OverflowExamplePost080.overflow errored: 
VM error: revert.
revert: arithmetic overflow
```


⸻

🔧 Solidity 내부 구현 방식 (EVM opcode 기반)
	•	Solidity 0.8.x 이후, 컴파일러가 ADD, SUB, MUL 등의 명령어에 대해 중간에 overflow 검사 코드 삽입
	•	revert(0x11) 형태의 Panic 코드로 예외 발생
```text
0x11 = Panic(uint256) - Arithmetic overflow or underflow
```


⸻

📎 3. 원한다면 예전 방식처럼 만들 수 있는 방법: unchecked
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract UncheckedExample {
    function uncheckedAdd(uint8 x) public pure returns (uint8) {
        unchecked {
            return x + 1; // ⚠️ wrap-around 허용됨
        }
    }
}
```
✅ 결과
	•	x=255일 때 → 256 % 256 = 0
	•	오버플로우가 발생해도 예외 없이 wrap-around

⸻

📌 요약 비교

| 항목 | 0.7.x 이전 | 0.8.0 이후 |
|---|---|---|
| 기본 연산 오버플로우 | ❌ 예외 발생 안함 | ✅ 자동 예외 발생 |
| SafeMath 필요 여부 | 필요함 | 필요 없음 |
| unchecked 키워드 | 없음 | 있음 (오버플로우 무시 시 사용) |
| 예외 타입 | require로 직접 처리 | Panic(uint256) 자동 발생 |



⸻

✅ 결론
	•	Solidity 0.8.0 이상에서는 더 이상 SafeMath 라이브러리가 필수가 아닙니다
	•	unchecked {} 블록을 사용하면 wrap-around 동작을 명시적으로 허용할 수 있습니다 (가스 최적화에 사용 가능)
	•	EVM에서 이 변화는 가스는 조금 더 들지만, 보안은 획기적으로 강화된 결과를 가져옵니다

⸻