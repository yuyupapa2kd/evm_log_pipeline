
# ✅ 요약 비교표

| 항목 | `eth_getTransactionReceipt` | `debug_traceTransaction` |
|---|---|---|
| 정보 수준 | **요약(summary)** | **실행 상세(trace)** |
| Revert 확인 | ✅ 가능 (status == 0) | ✅ 가능 + 왜 실패했는지까지 추적 |
| Revert Reason 추출 | ❌ 불가능 (표준 로그 없음) | ✅ 가능 (memory에서 직접 디코딩) |
| 가스 소비량 | ✅ receipt에 포함 | ✅ 더 상세한 per-opcode 수준 |
| 실행 흐름 | ❌ 없음 | ✅ PC, opcode, stack, memory 전부 추적 가능 |
| 비용(성능) | ✅ 빠름 (Light) | ❌ 느림 (Heavy, CPU 부담) |
| 실행 환경 요구 | 퍼블릭 RPC 가능 | ⚠️ 로컬 Geth full node 필요 (debug 모듈) |

---

# 📘 1. `eth_getTransactionReceipt`: 기본적인 요약 정보 조회

### ✅ 장점

- 퍼블릭/Infura RPC에서 바로 가능
- `status == 0` 여부로 실패인지 확인 가능
- `gasUsed`, `logs`, `contractAddress`, `blockNumber`, `from/to`, `logsBloom` 등 구조화된 정보 제공
- 속도 빠름, 시스템 부하 거의 없음

### ❌ 단점

- **왜 실패했는지**에 대한 정보는 없음
- `revert()` 안의 문자열(reason)은 ABI 로그에 포함되지 않음
- 내부 호출(`delegatecall`, `call`, `create`)에 대한 정보 없음

✅ **요약**: 실패 여부 확인에는 좋지만, **원인 파악은 불가능**

---

# 🧠 2. `debug_traceTransaction`: EVM 명령어 단위 실행 추적

### ✅ 장점

- **EVM opcode 단위 실행 흐름** 제공
- **stack / memory 상태**까지 제공됨
- `REVERT` 명령 실행 여부를 통해 정확한 실패 지점 확인 가능
- **revert reason** 문자열도 ABI 디코딩 가능
- 내부 호출 (`CALL`, `DELEGATECALL`)도 trace에서 모두 보여줌
- 가스 소비도 **명령어 단위(gasCost)**로 확인 가능

### ❌ 단점

- **느림 + 부하 큼** → 수십~수백건 분석 시 성능 이슈
- 퍼블릭 RPC에서는 사용 불가 (Geth 노드 필요)
- 일부 trace 옵션은 특정 클라이언트(Geth, Erigon)에서만 지원됨
- 실행 시 디버그 트랜잭션 캐시가 안되어 있으면 오래 걸림

✅ **요약**: 트랜잭션 실패 **원인 파악**에 **필수적 도구**, 단 성능 비용 존재

---

# ✅ 사용 목적별 권장 도구

| 목적 | 권장 방식 |
|---|---|
| 단순히 실패했는지 확인 | `eth_getTransactionReceipt` |
| 어떤 이유로 revert 되었는지 파악 | `debug_traceTransaction` + memory decode |
| 사용자 오류 피드백 제공 | `debug_traceTransaction` + `revert reason` 추출 |
| 실패 트랜잭션 통계 분석 (대량) | `txReceipt` + 샘플링된 `trace` 병행 |
| 스마트컨트랙트 디버깅 | **항상** `debug.traceTransaction`

---

# ✅ 예시 비교

### 예: 아래와 같은 Solidity 함수가 있다고 가정

```solidity
function withdraw(uint256 amount) public {
    require(balance[msg.sender] >= amount, "Insufficient balance");
    ...
}
```

### 트랜잭션 실패 시

- **`txReceipt`**
  - status == 0
  - logs == []
  - revert 이유는 알 수 없음

- **`debug.traceTransaction`**
  - 마지막 opcode: `REVERT`
  - memory:
    ```hex
    08c379a0...0000000b496e73756666696369656e742062616c616e6365
    ```
  - ABI 디코딩: `"Insufficient balance"` ← 실패 사유 복원 가능

---

# 🧠 실제 프로젝트 운영 팁

| 상황 | 권장 전략 |
|---|---|
| 실시간 운영 서비스 | `txReceipt`로 1차 필터링 → 일부만 `trace` |
| 디버깅/개발 | 100% `debug.traceTransaction` 활용 |
| 대량 데이터 적재 | Mongo 등에 `txReceipt`, `trace` 분리 저장 |
| 리소스 줄이기 | Geth `--cache`, `--tracer=callTracer` 옵션 활용

---

# ✅ 결론 요약

| 항목 | Receipt | Trace |
|---|---|---|
| 실패 여부 확인 | ✅ | ✅ |
| 실패 이유 추적 | ❌ | ✅ |
| 성능 | 빠름 | 무거움 |
| RPC 제한 | 퍼블릭 가능 | 로컬 전용 |
| 디버깅/로깅 | ❌ | ✅ |
| 자동화 적합성 | 높음 | 제한적 (성능 고려 필요) |

---