## 1. No Partial USDC Withdraw Implemented

### 설명

문제 코드: DreamAcademyLending.sol 내 withdraw함수, 148 ~ 151번줄

```java
function withdraw (address tokenAddress, uint256 amount) ... {
	...

	amount = getAccruedSupplyAmount(tokenAddress) / WAD * WAD;
  userBalances[msg.sender].balance += amount - userBalances[msg.sender].balance;
  ERC20(usdc).transfer(msg.sender, amount);
  userBalances[msg.sender].balance -= amount;
}
```

USDC 유동성을 공급한 사람이 특정 amount만큼의 USDC 회수를 요청할 경우를 지원하지 않는다. withdraw 함수가 usdc의 주소를 인자로 받을 경우 항상 현재 시점까지의 총 이자를 지급한다.

### 파급력

Informational

-   산정 이유
    -   손해보는 사람이 없으며, 기능상의 구현 부족으로 판단된다.

### 해결 방안

-   amount 값을 구할 때 getAccruedSupplyAmount의 결과값을 amount / 현재까지의 usdc 토큰 공급량으로 비율을 계산해주어야 한다.
