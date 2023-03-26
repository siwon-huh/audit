## 임나라

## 미구현된 부분: Repay, Accured Interest

## 1. False Liquidation Threshold

### 설명

문제 코드: DreamAcademyLending.sol 내 liquidate함수, 111번 줄

-   Liquidation threshold를 계산하는 과정이 잘못되었다.

```java
uint256 price = _borrow[user] * USDCPrice / ETHPrice;
require(price > _depositETH[user] * 75 / 100, "no liquidate");
```

-   빌린 USDC의 가치가 담보가치의 75% 초과인 경우 청산을 시작한다.
-   부등호의 방향이 반대로 되어야 한다.

### 파급력

High

-   산정 이유
    -   담보가치가 급격히 떨어지지 않는 한 누구의 자산이든 청산을 요구할 수 있다.
    -   DEX의 효용성을 매우 떨어뜨리며, 공격의 접근성이 아주 높다.

### 해결 방안

-   부등호의 방향을 바꿔야 한다.

```java
require(price <= _depositETH[user] * 75 / 100, "no liquidate");
```
