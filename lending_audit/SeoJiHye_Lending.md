## 김지우

## 1. Fixed Liquidation Amount

### 설명

문제 코드: DreamAcademyLending.sol 내 liquidate함수, 137번 줄

```java
require(_etherHolders[user]._borrowAmount < 100 ether || 
amount == _etherHolders[user]._borrowAmount / 4, "only liquidating 25% possible")
```

-   Liquidation amount가 빌린 토큰의 4분의 1로 고정되어 있다.
-   Liquidation amount를 한계치로 입력하지 않으면 청산이 수행되지 않는다.

### 파급력

Low

-   산정 이유
    -   DEX의 효용성을 떨어뜨리는 문제로 Low를 부여했다.
    -   담보가치가 급락하는 상황을 가정하면 담보 청산을 통한 유동성 확보가 어려워지므로 심각한 경우 일시적으로 DEX 내 유동성이 고갈될 수 있다.

### 개선 방안

-   amount를 \_borrow[user]/4 이하로 설정하면 된다.

```java
require(_etherHolders[user]._borrowAmount < 100 ether || 
amount **<=** _etherHolders[user]._borrowAmount / 4, "only liquidating 25% possible")
```
