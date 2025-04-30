nonReentrant는 Solidity에서 재진입 공격(Reentrancy Attack)을 방지하기 위해 사용하는 **함수 수정자(modifier)**입니다. 이 modifier는 특히 call, send, transfer 등 외부 계약을 호출할 때 중요한 보안 장치입니다.

⸻

🔒 1. 재진입 공격이란?

외부 컨트랙트 호출 후 상태 변경 전에 제어권을 빼앗기면, 외부 컨트랙트가 다시 호출되어 상태가 꼬일 수 있는 취약점입니다.
```solidity
// 취약한 코드 예시
function withdraw() public {
    uint256 amount = balances[msg.sender];
    (bool sent, ) = msg.sender.call{value: amount}(""); // 외부 호출 먼저
    require(sent, "Failed to send Ether");
    balances[msg.sender] = 0; // 나중에 상태 변경 (❌)
}
```


⸻

✅ 2. nonReentrant 적용 예시 (OpenZeppelin)

📁 contracts/security/ReentrancyGuard.sol
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

abstract contract ReentrancyGuard {
    // 상태 값 상수
    uint256 private constant _NOT_ENTERED = 1;
    uint256 private constant _ENTERED = 2;

    // 현재 상태 저장
    uint256 private _status;

    constructor() {
        _status = _NOT_ENTERED;
    }

    // modifier 정의
    modifier nonReentrant() {
        // 재진입 여부 확인
        require(_status != _ENTERED, "ReentrancyGuard: reentrant call");

        // 진입 표시
        _status = _ENTERED;

        _; // 함수 본문 실행

        // 상태 복구
        _status = _NOT_ENTERED;
    }
}
```


⸻

🔍 3. 작동 방식 분석
```solidity
modifier nonReentrant() {
    require(_status != _ENTERED, "ReentrancyGuard: reentrant call");
    _status = _ENTERED;
    _;
    _status = _NOT_ENTERED;
}
```
| 단계 | 설명 |
|---|---|
| 1. require | 이미 _ENTERED 상태면 예외 발생 → 재진입 차단 |
| 2. _status = _ENTERED | 재진입 방지 상태로 변경 |
| 3. _ | 실제 함수 본문 실행 |
| 4. _status = _NOT_ENTERED | 종료 후 상태 복원 |



⸻

📌 4. 사용 예
```solidity
contract Bank is ReentrancyGuard {
    mapping(address => uint256) public balances;

    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw(uint256 amount) public nonReentrant {
        require(balances[msg.sender] >= amount, "Insufficient funds");

        balances[msg.sender] -= amount;

        (bool sent, ) = msg.sender.call{value: amount}("");
        require(sent, "Failed to send Ether");
    }
}
```


⸻

✅ 요약

| 항목 | 설명 |
|---|---|
| 목적 | 재진입 공격 방지 |
| 핵심 로직 | 함수 실행 중 재호출 차단 (_status 상태 값) |
| 위치 | 함수 수정자(modifier)로 적용 |
| 라이브러리 | OpenZeppelin ReentrancyGuard.sol 사용 권장 |



⸻

⸻

✅ 목표
	•	ReentrancyGuard를 사용한 보안 로직 포함
	•	UUPS 업그레이드 가능한 구조로 작성
	•	OpenZeppelin의 UUPSUpgradeable, ReentrancyGuardUpgradeable 사용

⸻

🧱 1. 사용 라이브러리
```solidity
// 주요 OpenZeppelin import
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/security/ReentrancyGuardUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
```


⸻

🧪 2. UUPS + Reentrancy 방지 컨트랙트 예제
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/security/ReentrancyGuardUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";

contract SecureBank is
    Initializable,
    UUPSUpgradeable,
    ReentrancyGuardUpgradeable,
    OwnableUpgradeable
{
    mapping(address => uint256) private balances;

    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers(); // proxy-safe constructor
    }

    function initialize() public initializer {
        __Ownable_init();
        __UUPSUpgradeable_init();
        __ReentrancyGuard_init();
    }

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw(uint256 amount) external nonReentrant {
        require(balances[msg.sender] >= amount, "Insufficient balance");

        balances[msg.sender] -= amount;
        (bool sent, ) = msg.sender.call{value: amount}("");
        require(sent, "Withdraw failed");
    }

    function getBalance(address user) external view returns (uint256) {
        return balances[user];
    }

    // UUPS 업그레이드 권한 제어
    function _authorizeUpgrade(address newImplementation) internal override onlyOwner {}
}
```


⸻

📦 3. 배포 및 업그레이드 흐름 요약 (Hardhat 기준)
	1.	배포
```solidity
const Bank = await ethers.getContractFactory("SecureBank");
const proxy = await upgrades.deployProxy(Bank, [], { initializer: "initialize" });
```
	2.	업그레이드
```solidity
const NewBank = await ethers.getContractFactory("SecureBankV2");
await upgrades.upgradeProxy(proxy.address, NewBank);
```


⸻

🧩 핵심 포인트

| 항목 | 설명 |
|---|---|
| initializer() | 생성자 대신 초기화용 함수, 업그레이드 safe |
| ReentrancyGuardUpgradeable | 재진입 방지 기능 제공 |
| UUPSUpgradeable | UUPS 패턴으로 업그레이드 지원 |
| _authorizeUpgrade() | onlyOwner로 업그레이드 권한 제한 |
| __X_init() | 각 부모의 초기화 함수 호출 필수 |



⸻
