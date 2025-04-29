
---

# ✅ 1. 실패 트랜잭션 발생용 Solidity 예제

먼저 이 **실패하는 컨트랙트**를 배포할 거예요.

```solidity
// contracts/FailExample.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract FailExample {
    function fail() public pure {
        require(false, "You shall not pass!");
    }
}
```

**요약:**
- `fail()` 함수를 부르면 무조건 **revert**가 발생합니다.
- revert 메세지는 `"You shall not pass!"`

---

# ✅ 2. Hardhat으로 배포 준비

터미널에서:

```bash
npx hardhat compile
```

그다음 Geth Dev 모드에 연결해서:

```bash
npx hardhat run scripts/deploy.js --network localhost
```

`scripts/deploy.js` 예제:

```javascript
async function main() {
  const FailExample = await ethers.getContractFactory("FailExample");
  const failExample = await FailExample.deploy();
  await failExample.deployed();

  console.log(`FailExample deployed to: ${failExample.address}`);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

---

# ✅ 3. 실패하는 트랜잭션 만들기

하드햇 콘솔에서 바로:

```bash
npx hardhat console --network localhost
```

들어간 다음:

```javascript
const FailExample = await ethers.getContractFactory("FailExample");
const failExample = await FailExample.attach("컨트랙트_주소");

await failExample.fail();
```

이걸 실행하면 **트랜잭션 실패**가 발생합니다.

**(try/catch로 감싸지 않으면 터미널에 에러 던져지게 됩니다.)**

---

# ✅ 4. Geth 콘솔에서 트랜잭션 추적

1. 실패한 트랜잭션의 **hash(txHash)** 를 복사합니다.
2. Geth 콘솔로 가서:

```javascript
debug.traceTransaction("0x트랜잭션해시")
```

실행합니다.

---

# ✅ 5. `debug.traceTransaction` 결과에서 분석할 포인트

- **맨 마지막 structLog**를 보면 `op: "REVERT"`가 있을 것
- **stack**에 2개 값이 있을 것 (offset, length)
- **memory**에 revert 이유(You shall not pass!)가 ABI 인코딩 되어 있을 것

### 메모리 해석하는 방법:

memory는 ABI 포맷을 따르므로 대략 구조는 이렇습니다:

| 위치 | 내용 |
|:---|:---|
| 0x00-0x03 | function selector: `0x08c379a0` (Error(string)) |
| 0x04-0x23 | offset (0x20) |
| 0x24-0x43 | string length |
| 0x44-    | string 내용 ("You shall not pass!")

이걸 읽으면  
"아! revert한 이유가 `You shall not pass!` 였구나" 를 확인할 수 있습니다.

---

# 🚀 여기까지 하면

- **트랜잭션이 왜 실패했는지**
- **어떤 메세지가 revert로 보내졌는지**
- **어디서 EVM 명령어가 멈췄는지**

를 **완벽하게** 파악할 수 있습니다.

---

# ✅ 정리

| 단계 | 할 일 |
|---|---|
| 컨트랙트 배포 | `FailExample` |
| 실패 트랜잭션 발생 | `fail()` 호출 |
| debug.traceTransaction으로 추적 | REVERT, stack, memory 분석 |
| memory ABI 디코딩 | revert reason 추출 |

---
