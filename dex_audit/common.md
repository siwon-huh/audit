## 공통 문제

## 1. Error on calculating initial LP token volume

### 설명

문제 코드: Dex.sol addLiquidity 내 sqrt()

sqrt 사용으로 인한 오차가 발생하고, 이로 인해 공급한 토큰과 LP토큰으로 회수할 수 있는 토큰의 양에 오차가 생긴다.

### 파급력

Informational

-   산정 이유
    -   오차의 크기가 매우 작기 때문에 실질적인 문제는 거의 없다.

### 해결방안

-   LP토큰의 첫 발행시에만 발생하는 문제로, 첫 발행시에 토큰의 양을 적게 하면 오차를 줄일 수 있다.
-   Solidity 자체의 문제로, 완전한 해결이 불가능하다.
