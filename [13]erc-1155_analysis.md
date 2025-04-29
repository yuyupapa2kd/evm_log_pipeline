
⸻

✅ 1. ERC-1155 개요

| 항목 | 설명 |
|---|---|
| 표준 명칭 | ERC-1155 |
| 목적 | 다중 토큰 표준: 하나의 컨트랙트로 NFT와 FT 모두 관리 가능 |
| 대표 사용처 | 게임 아이템, 미술 작품, 대량 배포 NFT 등 |



⸻

📦 핵심 특징
	•	대량 발행, 대량 전송 지원 → gas 절감
	•	tokenId마다 독립적 의미 (e.g., NFT, FT 혼합)
	•	balanceOf(address, id) → 수량 기반 보유량 체크

⸻

✅ 2. ERC-1155 주요 함수
```solidity
interface IERC1155 {
    function balanceOf(address account, uint256 id) external view returns (uint256);
    function safeTransferFrom(address from, address to, uint256 id, uint256 amount, bytes calldata data) external;
    function safeBatchTransferFrom(address from, address to, uint256[] calldata ids, uint256[] calldata amounts, bytes calldata data) external;

    function setApprovalForAll(address operator, bool approved) external;
    function isApprovedForAll(address account, address operator) external view returns (bool);

    event TransferSingle(address indexed operator, address indexed from, address indexed to, uint256 id, uint256 value);
    event TransferBatch(address indexed operator, address indexed from, address indexed to, uint256[] ids, uint256[] values);
}
```


⸻

✅ 3. ERC-1155 vs ERC-721 비교

| 항목 | ERC-721 | ERC-1155 |
|---|---|---|
| 토큰 ID | 고유 (1개당 1개) | ID별로 수량 관리 가능 |
| 전송 방식 | transferFrom() | safeTransferFrom() |
| 배치 처리 | 불가 | 가능 (safeBatchTransferFrom) |
| NFT 전용 | O | X (NFT+FT 혼합 가능) |
| 게임·대량 배포 | 비효율 | 매우 효율적 (gas 절약) |



⸻

✅ 4. ERC-1155 기반 마켓플레이스 기본 구조

📦 거래 단위: (tokenAddress, tokenId, amount)
```solidity
struct ERC1155Order {
    address offerer;
    address token;
    uint256 tokenId;
    uint256 amount;
    uint256 price; // ETH 기준
    uint256 salt;
}
```


⸻

🧠 주문 흐름 (Seaport 스타일)
```plaintext
[판매자]
   ↓
createOrder() → EIP-712 구조체 서명
   ↓ (off-chain 전송)
[구매자]
   ↓
fulfillOrder(order, signature)
   ↓
컨트랙트:
   - NFT 전송 (safeTransferFrom)
   - ETH 수령인에게 지급
```


⸻

✅ 5. Solidity 기반 Minimal 마켓플레이스 (ERC1155 대응)
```solidity
function fulfillOrder(Order calldata order, bytes calldata signature) external payable {
    require(msg.value == order.price, "Wrong ETH");

    // 서명 검증 생략 (EIP-712 적용 시 사용)
    require(!usedOrders[hash], "Already used");

    IERC1155(order.token).safeTransferFrom(order.offerer, msg.sender, order.tokenId, order.amount, "");

    payable(order.offerer).transfer(order.price);
    usedOrders[hash] = true;

    emit OrderFulfilled(msg.sender, hash);
}
```


⸻

✅ 6. Golang 또는 JS에서 EIP-712 서명 예시

구조 요약
```go
{
  "types": {
    "Order": [
      { "name": "offerer", "type": "address" },
      { "name": "token", "type": "address" },
      { "name": "tokenId", "type": "uint256" },
      { "name": "amount", "type": "uint256" },
      { "name": "price", "type": "uint256" },
      { "name": "salt", "type": "uint256" }
    ]
  }
}
```
서명자는 위 구조를 서명한 후 오프체인 전송하고, 구매자는 이를 컨트랙트에 전달하여 실행합니다.

⸻

✅ 7. 확장 가능 요소

| 항목 | 설명 |
|---|---|
| 대량 오더 | ERC1155BatchOrder 구조 확장 가능 |
| 조건부 수락 | allowlist / 조건부 구매 등 zone 설계 가능 |
| 크로스체인 미러링 | NFT 미러된 상태에서도 주문 집행 가능 |
| 수수료 | feeRecipient, platformFee 추가 가능 |



⸻

✅ 결론 요약

| 항목 | 설명 |
|---|---|
| ERC-1155 | NFT + FT를 모두 처리할 수 있는 다중 토큰 표준 |
| 거래 방식 | (token, id, amount, price)를 기준으로 주문 생성 |
| Seaport 스타일 적용 | 오더를 오프체인 서명, 온체인에서 검증 및 실행 |
| 확장성 | 게임, 대량 NFT 발행, 오퍼 기반 거래에 매우 적합 |



⸻



# seaport market place
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC1155/IERC1155.sol";
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract MinimalERC1155Marketplace is ReentrancyGuard {
    using ECDSA for bytes32;

    struct ERC1155Order {
        address offerer;
        address token;
        uint256 tokenId;
        uint256 amount;
        uint256 price;
        uint256 salt;
    }

    bytes32 public constant ORDER_TYPEHASH = keccak256(
        "ERC1155Order(address offerer,address token,uint256 tokenId,uint256 amount,uint256 price,uint256 salt)"
    );

    mapping(bytes32 => bool) public usedOrders;

    event OrderFulfilled(address indexed buyer, bytes32 indexed orderHash);

    function fulfillOrder(ERC1155Order calldata order, bytes calldata signature) external payable nonReentrant {
        bytes32 orderHash = keccak256(abi.encode(
            ORDER_TYPEHASH,
            order.offerer,
            order.token,
            order.tokenId,
            order.amount,
            order.price,
            order.salt
        ));

        require(!usedOrders[orderHash], "Order already used");
        require(orderHash.toEthSignedMessageHash().recover(signature) == order.offerer, "Invalid signature");
        require(msg.value == order.price, "Incorrect ETH sent");

        usedOrders[orderHash] = true;

        // Transfer ETH to seller
        payable(order.offerer).transfer(order.price);

        // Transfer ERC1155 token to buyer
        IERC1155(order.token).safeTransferFrom(order.offerer, msg.sender, order.tokenId, order.amount, "");

        emit OrderFulfilled(msg.sender, orderHash);
    }
}
```

ERC1155Order 기반의 Solidity 마켓플레이스 구현은 완료했습니다 (상단 코드캔버스 참고).
이제 Golang 기반 서명 생성 코드를 설명드리겠습니다.

⸻

✅ Golang: ERC1155Order 서명 생성 (EIP-191, 간소화 버전)

📦 전제 사항
	•	서명 방식: eth_sign (EIP-191) 또는 eth_signTypedData_v4 (EIP-712)
	•	이번 예시는 간단화를 위해 EIP-712 대신 Keccak256 + Sign 방식 사용

⸻

🧩 Go 코드 예시
```go
package main

import (
    "crypto/ecdsa"
    "encoding/hex"
    "fmt"
    "log"
    "math/big"

    "github.com/ethereum/go-ethereum/accounts/abi"
    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/crypto"
)

func main() {
    // 샘플 개인키 (테스트 전용)
    privateKeyHex := "4c0883a69102937d6231471b5dbb6204fe5129617082791592b805385b3ffec7"
    privateKey, err := crypto.HexToECDSA(privateKeyHex)
    if err != nil {
        log.Fatal("Invalid private key:", err)
    }

    // 주문 정보
    offerer := common.HexToAddress("0x90f8bf6A479f320ead074411a4B0e7944Ea8c9C1")
    token := common.HexToAddress("0xABCDEFabcdefABCDEFabcdefabcdefABCDEFabcdef")
    tokenId := big.NewInt(1)
    amount := big.NewInt(10)
    price := big.NewInt(5e17) // 0.5 ETH
    salt := big.NewInt(123456789)

    // 오더 해시 (Solidity encodePacked와 대응되는 Keccak256)
    packed, err := abi.Arguments{
        {Type: abi.AddressTy}, {Type: abi.AddressTy}, {Type: abi.UintTy},
        {Type: abi.UintTy}, {Type: abi.UintTy}, {Type: abi.UintTy},
    }.Pack(offerer, token, tokenId, amount, price, salt)
    if err != nil {
        log.Fatal("ABI pack 실패:", err)
    }

    orderHash := crypto.Keccak256Hash(packed)

    // 서명 (EIP-191 스타일)
    sig, err := crypto.Sign(orderHash.Bytes(), privateKey)
    if err != nil {
        log.Fatal("Sign 실패:", err)
    }

    fmt.Println("🧾 ERC1155Order 해시:", orderHash.Hex())
    fmt.Println("🔏 서명:", hex.EncodeToString(sig))
    fmt.Println("🧍‍♂️ 서명자:", offerer.Hex())
}
```


⸻

✅ 실행 결과 예시
```text
🧾 ERC1155Order 해시: 0xabc123...de
🔏 서명: 0xf3c6...1b
🧍‍♂️ 서명자: 0x90f8bf6A479f320ead074411a4B0e7944Ea8c9C1
```


⸻

📌 보충 설명

| 항목 | 설명 |
|---|---|
| abi.Arguments{} | Solidity의 abi.encodePacked(...) 대체용 |
| crypto.Keccak256Hash() | EVM의 keccak256 해시와 동일 |
| crypto.Sign() | ECDSA 서명 → Solidity ecrecover로 검증 가능 |
| 주의 | 실제 프로덕션에서는 EIP-712 (domain separator 포함) 사용 권장 |



⸻

✅ 다음으로:
	•	해당 서명 결과를 Solidity 컨트랙트에서 검증 (ECDSA.recover)
	•	EIP-712 기반 서명 완전 구현
	•	ERC1155BatchOrder 확장

원하시는 방향 알려주시면 이어서 도와드릴게요!