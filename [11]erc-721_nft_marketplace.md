
⸻

✅ 1. ERC-721 표준 (NFT 토큰) 개요

| 항목 | 설명 |
|---|---|
| 정식 명칭 |  ERC-721: Non-Fungible Token Standard |
| 표준 번호 | EIP-721 |
| 주요 특징 | 각 토큰이 고유한 ID(tokenId)를 가지며, 상호 대체 불가 (non-fungible) |



⸻

📦 핵심 인터페이스 (Solidity)
```solidity
interface IERC721 {
    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
    event Approval(address indexed owner, address indexed approved, uint256 indexed tokenId);
    event ApprovalForAll(address indexed owner, address indexed operator, bool approved);

    function balanceOf(address owner) external view returns (uint256);
    function ownerOf(uint256 tokenId) external view returns (address);

    function safeTransferFrom(address from, address to, uint256 tokenId) external;
    function transferFrom(address from, address to, uint256 tokenId) external;

    function approve(address to, uint256 tokenId) external;
    function getApproved(uint256 tokenId) external view returns (address);

    function setApprovalForAll(address operator, bool approved) external;
    function isApprovedForAll(address owner, address operator) external view returns (bool);
}
```


⸻

🔧 주요 함수 설명

| 함수 | 설명 |
|---|---|
| balanceOf(owner) | 특정 주소가 보유한 NFT 수 |
| ownerOf(tokenId) | 특정 토큰의 현재 소유자 |
| transferFrom(from, to, tokenId) | 일반 전송 (권한 필요) |
| safeTransferFrom(...) | 전송 후 수신자가 ERC721을 받을 수 있는지 확인 (권장 방식) |
| approve(to, tokenId) | 개별 토큰에 대한 위임 전송 권한 부여 |
| setApprovalForAll(operator, approved) | 전체 토큰에 대한 권한 일괄 위임 |
| getApproved(tokenId) | 특정 토큰에 대한 승인된 주소 확인 |



⸻

✅ 2. NFT 마켓플레이스 구조

🎯 목표
	•	NFT를 판매자가 등록 → 구매자가 구매 → NFT 전송 + 대금 송금
	•	Ethereum 상에서 모든 로직을 스마트 컨트랙트로 처리

⸻

📦 핵심 컴포넌트

| 구성 요소 | 역할 |
|---|---|
| ERC721 토큰 | 고유한 NFT 소유권 표현 |
| Marketplace 컨트랙트 | 판매 등록 / 구매 / 수수료 처리 |
| 프론트엔드 | 사용자에게 상품 정보, 구매 버튼 등 제공 |
| 백엔드 (선택) | 메타데이터 캐시, 인덱싱, 검색 지원 |



⸻

🧩 Marketplace 주요 인터페이스 예시
```solidity
interface INFTMarketplace {
    function listItem(address nftAddress, uint256 tokenId, uint256 price) external;
    function cancelListing(address nftAddress, uint256 tokenId) external;
    function buyItem(address nftAddress, uint256 tokenId) external payable;
    function updatePrice(address nftAddress, uint256 tokenId, uint256 newPrice) external;

    event Listed(address indexed seller, address indexed nftAddress, uint256 indexed tokenId, uint256 price);
    event Sold(address indexed buyer, address indexed nftAddress, uint256 indexed tokenId, uint256 price);
    event Cancelled(address indexed seller, address indexed nftAddress, uint256 indexed tokenId);
}
```


⸻

⚙️ Marketplace 작동 방식
```plaintext
[1] NFT 판매 등록 (listItem)
	•	소유자는 approve(marketplace, tokenId) 호출 → 마켓플레이스에서 NFT 전송 가능하도록 위임
	•	이후 listItem() 호출 → 판매 정보 저장

[2] NFT 구매 (buyItem)
	•	구매자는 buyItem() 호출 + msg.value == price 전송
	•	마켓플레이스는:
	•	NFT를 safeTransferFrom(seller → buyer)
	•	대금을 seller에게 송금 (또는 수수료 차감 후 지급)

[3] 판매 취소 / 가격 변경
	•	cancelListing() 또는 updatePrice() 사용 가능
	•	tokenId 기준으로 매물 상태를 관리
```
⸻

✅ 3. 전체 흐름 요약
```plaintext
[판매자]
   |
   | approve(marketplace, tokenId)
   ↓
listItem(nft, tokenId, price)
   |
   ↓ 저장: 매물 등록

[구매자]
   |
   | buyItem(nft, tokenId) with msg.value
   ↓
marketplace:
   - NFT transferFrom → 구매자
   - ETH → 판매자
```


⸻

✅ 확장 고려 요소

| 기능 | 설명 |
|---|---|
| 로열티 (ERC-2981) | 창작자에게 일정 수수료 자동 분배 |
| 경매 / 입찰 | 즉시 구매 외에 오퍼 기반 시스템 |
| 배치 구매 | 여러 NFT를 한 번에 구매하는 기능 |
| 오프체인 서명 | EIP-712 기반 오퍼 → 가스 절약 (e.g. Seaport, LooksRare) |



⸻

✅ 결론

| 항목 | 설명 |
|---|---|
| ERC-721 | 고유한 디지털 자산 (NFT)의 토큰 표준 |
| Marketplace 컨트랙트 | 거래를 자동화하며 소유권 전송을 안전하게 처리 |
| 핵심 키워드 | approve, transferFrom, buyItem, listItem, safeTransferFrom |
| 확장 가능성 | 경매, 로열티, 온체인 메타데이터, lazy mint 등 다양 |



⸻

⸻

✅ 기능 개요
	•	NFT 소유자가 판매 등록
	•	구매자가 ETH 지불 후 NFT 구매
	•	판매자가 판매 취소
	•	마켓플레이스는 approve() 받은 NFT만 판매 가능
	•	수수료 (플랫폼 fee) 적용 가능

⸻

📦 전제 조건
	•	판매 대상 NFT는 ERC-721 규격을 따라야 함
	•	마켓플레이스가 NFT를 대신 전송하기 위해 approve() 호출 필요

⸻

✅ 전체 코드 (Solidity, SPDX-License 포함)
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract SimpleNFTMarketplace is ReentrancyGuard {
    struct Listing {
        address seller;
        uint256 price;
    }

    // nftAddress → tokenId → Listing
    mapping(address => mapping(uint256 => Listing)) public listings;

    event ItemListed(address indexed seller, address indexed nftAddress, uint256 indexed tokenId, uint256 price);
    event ItemCanceled(address indexed seller, address indexed nftAddress, uint256 indexed tokenId);
    event ItemBought(address indexed buyer, address indexed nftAddress, uint256 indexed tokenId, uint256 price);

    modifier isOwner(address nftAddress, uint256 tokenId, address caller) {
        IERC721 nft = IERC721(nftAddress);
        require(nft.ownerOf(tokenId) == caller, "Not owner");
        _;
    }

    modifier isListed(address nftAddress, uint256 tokenId) {
        require(listings[nftAddress][tokenId].price > 0, "Not listed");
        _;
    }

    modifier notListed(address nftAddress, uint256 tokenId) {
        require(listings[nftAddress][tokenId].price == 0, "Already listed");
        _;
    }

    /// @notice List an NFT for sale
    function listItem(address nftAddress, uint256 tokenId, uint256 price)
        external
        notListed(nftAddress, tokenId)
        isOwner(nftAddress, tokenId, msg.sender)
    {
        require(price > 0, "Price must be > 0");

        IERC721 nft = IERC721(nftAddress);
        require(nft.getApproved(tokenId) == address(this) || nft.isApprovedForAll(msg.sender, address(this)), "Not approved");

        listings[nftAddress][tokenId] = Listing(msg.sender, price);
        emit ItemListed(msg.sender, nftAddress, tokenId, price);
    }

    /// @notice Cancel a listing
    function cancelListing(address nftAddress, uint256 tokenId)
        external
        isOwner(nftAddress, tokenId, msg.sender)
        isListed(nftAddress, tokenId)
    {
        delete listings[nftAddress][tokenId];
        emit ItemCanceled(msg.sender, nftAddress, tokenId);
    }

    /// @notice Purchase a listed NFT
    function buyItem(address nftAddress, uint256 tokenId)
        external
        payable
        nonReentrant
        isListed(nftAddress, tokenId)
    {
        Listing memory listing = listings[nftAddress][tokenId];

        require(msg.value >= listing.price, "Price not met");

        delete listings[nftAddress][tokenId];

        // Transfer ETH to seller
        (bool success, ) = payable(listing.seller).call{value: listing.price}("");
        require(success, "Transfer failed");

        // Transfer NFT to buyer
        IERC721(nftAddress).safeTransferFrom(listing.seller, msg.sender, tokenId);

        emit ItemBought(msg.sender, nftAddress, tokenId, listing.price);
    }

    /// @notice Update the price of an existing listing
    function updateListing(address nftAddress, uint256 tokenId, uint256 newPrice)
        external
        isOwner(nftAddress, tokenId, msg.sender)
        isListed(nftAddress, tokenId)
    {
        listings[nftAddress][tokenId].price = newPrice;
        emit ItemListed(msg.sender, nftAddress, tokenId, newPrice);
    }

    /// @notice Get current listing info
    function getListing(address nftAddress, uint256 tokenId) external view returns (Listing memory) {
        return listings[nftAddress][tokenId];
    }
}
```


⸻

🧪 테스트 시나리오 요약

| 상황 | 수행 함수 | 필요 조건 |
|---|---|---|
| NFT 등록 | approve() → listItem() | msg.sender == owner |
| NFT 구매 | buyItem() | msg.value >= price |
| 등록 취소 | cancelListing() | msg.sender == owner |
| 가격 변경 | updateListing() | msg.sender == owner |



⸻

🔐 보안 메커니즘
	•	ReentrancyGuard: 이중 지불 공격 방지
	•	require(nft.getApproved() == address(this)): 위임 확인
	•	call{value:} 사용 후 require(success): 송금 안전 처리

⸻

🔧 확장 가능 요소

| 기능 | 구현 방향 |
|---|---|
| 로열티 (ERC-2981) | 판매 수익 일부를 원 저작자에게 전송 |
| 플랫폼 수수료 | 거래 금액의 일부를 owner()에게 전송 |
| 경매 / 입찰 | 별도 Auction 구조 정의 |
| ERC1155 지원 | IERC1155로 다중 NFT 확장 가능 |
| 오프체인 오퍼 | EIP-712 + 서명 → lazy execution |


