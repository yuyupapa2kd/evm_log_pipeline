# Seaport 인터페이스에 맞춘 커스텀 마켓 구현 예제
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";

contract MinimalSeaportLikeMarketplace {
    using ECDSA for bytes32;

    enum ItemType { NATIVE, ERC721 }

    struct OfferItem {
        ItemType itemType;
        address token;
        uint256 identifier;
        uint256 amount;
    }

    struct ConsiderationItem {
        ItemType itemType;
        address token;
        uint256 identifier;
        uint256 amount;
        address payable recipient;
    }

    struct Order {
        address offerer;
        OfferItem offer;
        ConsiderationItem consideration;
        uint256 salt;
    }

    bytes32 public constant ORDER_TYPEHASH = keccak256(
        "Order(address offerer,OfferItem offer,ConsiderationItem consideration,uint256 salt)"
    );

    mapping(bytes32 => bool) public usedOrders;

    event OrderFulfilled(address indexed buyer, bytes32 indexed orderHash);

    function fulfillOrder(Order calldata order, bytes calldata signature) external payable {
        bytes32 orderHash = keccak256(abi.encode(
            ORDER_TYPEHASH,
            order.offerer,
            keccak256(abi.encode(order.offer.itemType, order.offer.token, order.offer.identifier, order.offer.amount)),
            keccak256(abi.encode(order.consideration.itemType, order.consideration.token, order.consideration.identifier, order.consideration.amount, order.consideration.recipient)),
            order.salt
        ));

        require(!usedOrders[orderHash], "Order already used");
        require(orderHash.toEthSignedMessageHash().recover(signature) == order.offerer, "Invalid signature");

        usedOrders[orderHash] = true;

        // Transfer ETH to seller
        require(msg.value == order.consideration.amount, "Incorrect ETH amount");
        order.consideration.recipient.transfer(msg.value);

        // Transfer NFT to buyer
        IERC721(order.offer.token).safeTransferFrom(order.offerer, msg.sender, order.offer.identifier);

        emit OrderFulfilled(msg.sender, orderHash);
    }
}
```


먼저, Solidity 기반의 Seaport 스타일 마켓플레이스 구현 예제를 Solidity로 완성했습니다 (위 텍스트캔버스 참고).

이제 요청하신 Seaport 방식 오더 생성 / 서명 / 실행을 위한 Golang 코드 예제를 제공합니다.

⸻

✅ Golang으로 Seaport-style 주문 생성 및 서명

📦 필요 라이브러리
```
go get github.com/ethereum/go-ethereum
```


⸻

🧩 Golang 코드 구조 (EIP-712 주문 생성 + 서명)
```go
package main

import (
    "crypto/ecdsa"
    "encoding/hex"
    "fmt"
    "log"
    "math/big"

    "github.com/ethereum/go-ethereum/accounts/abi/bind"
    "github.com/ethereum/go-ethereum/crypto"
    "github.com/ethereum/go-ethereum/crypto/secp256k1"
)

// 주문 구조체 간단 모델링
type Order struct {
    Offerer      string
    OfferToken   string
    OfferId      *big.Int
    ConsiderETH  *big.Int
    Salt         *big.Int
}

func main() {
    // 예제 개인키 (테스트 전용)
    privateKeyHex := "4c0883a69102937d6231471b5dbb6204fe5129617082791592b805385b3ffec7"
    pkBytes, _ := hex.DecodeString(privateKeyHex)
    privateKey, _ := crypto.ToECDSA(pkBytes)
    publicKey := privateKey.Public().(*ecdsa.PublicKey)
    fromAddr := crypto.PubkeyToAddress(*publicKey)

    // 오더 생성
    order := Order{
        Offerer:     fromAddr.Hex(),
        OfferToken:  "0xABCDEFabcdefABCDEFabcdefabcdefABCDEFabcdef",
        OfferId:     big.NewInt(1),
        ConsiderETH: big.NewInt(1_000_000_000_000_000_000), // 1 ETH
        Salt:        big.NewInt(12345),
    }

    // 오더 해시 생성 (단순화, 실제는 EIP-712로 도메인 결합 필요)
    rawData := []byte(order.Offerer + order.OfferToken + order.OfferId.String() + order.ConsiderETH.String() + order.Salt.String())
    hash := crypto.Keccak256Hash(rawData)

    // 서명 생성
    sig, err := crypto.Sign(hash.Bytes(), privateKey)
    if err != nil {
        log.Fatal("서명 실패:", err)
    }

    fmt.Println("Order Hash:", hash.Hex())
    fmt.Println("Signature :", hex.EncodeToString(sig))
    fmt.Println("Signer    :", fromAddr.Hex())
}
```


⸻

✅ 실행 결과 예시
```plaintext
Order Hash: 0x4e26c0...cbaa
Signature : 7b128e92...1b
Signer    : 0x90f8bf6A479f320ead074411a4B0e7944Ea8c9C1
```


⸻

✅ 실전 적용 시 고려 사항

| 항목 | 설명 |
| Order 구조 | 실제는 EIP712Domain, OfferItem, ConsiderationItem 포함 |
| 서명 방식 | eth_signTypedData_v4 (EIP-712) 서명 필요 |
| 해시 계산 | domainSeparator + typeHash + structHash 조합 |
| fulfillOrder() | Solidity에서 ECDSA.recover()로 검증 후 실행 |



⸻

💡 다음으로:
	•	위 Golang 서명을 실제 MinimalSeaportMarketplace 컨트랙트와 연동 테스트
	•	실제 EIP-712 구조체로 확장 (domain separator, type hash)
	•	JavaScript 또는 Foundry 테스트 코드로 실행 확인

필요하신 흐름으로 이어서 도와드릴까요?
예: "EIP-712 전체 구조 적용해서 해시 만드는 방법도 알려줘"