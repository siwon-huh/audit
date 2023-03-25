## 이성휘

## 1. Improper Implementation of Liquidity

### 설명

LP토큰을 발행하고 회수하는 공식이 제대로 설계되어 있지 않다.

addLiquidity, removeLiquidity가 LP토큰에 대한 지분을 올바르게 반환하지 않는다.

구현된 로직에 따르면 LP토큰의 발행량이 토큰의 양보다는 유동성 공급 순서에 더 dependent하며, 마지막에 addLiquidity를 수행한 사람이 가장 많은 지분을 가져간다.

### 파급력

Critical

-   산정 이유
    -   유동성을 공급하는 순간 DEX가 제 기능을 하지 못하고 망가진다.
    -   공급자 간의 토큰량이 차이가 많을수록 LP토큰으로 회수할 수 있는 토큰과 실제 공급한 토큰 사이의 오차가 더 커진다.
-   공격 시나리오
    ```jsx
    function testAddLiquidity() external {
      uint firstLPReturn = dex.addLiquidity(5000 ether, 1000 ether, 0);
      emit log_named_uint("firstLPReturn", firstLPReturn);

      uint secondLPReturn = dex.addLiquidity(1000 ether, 200 ether, 0);
      emit log_named_uint("secondLPReturn", secondLPReturn);

      uint thirdLPReturn = dex.addLiquidity(10 ether, 2 ether, 0);
      emit log_named_uint("thirdLPReturn", thirdLPReturn);

      (uint tx, uint ty) = dex.removeLiquidity(secondLPReturn, 0, 0);
      emit log_named_uint("second LP remove", tx);
      emit log_named_uint("second LP remove", ty);

      (uint tx2, uint ty2) = dex.removeLiquidity(thirdLPReturn, 0, 0);
      emit log_named_uint("third LP remove", tx2);
      emit log_named_uint("third LP remove", ty2);

      assertEq(tx, 1000 ether);
      assertEq(ty, 200 ether);
      assertEq(tx2, 10 ether);
      assertEq(ty2, 2 ether);
    }
    ```
    ```jsx
    [FAIL. Reason: Assertion failed.] testAddLiquidity() (gas: 648904)
    Logs:
      firstLPReturn: 2236067977499789696409
      secondLPReturn: 11180339887498948482045
      thirdLPReturn: 6708203932499369089227000
      second LP remove: 9996673320026600000
      second LP remove: 1999334664005300000
      third LP remove: 6010000000000000000000
      third LP remove: 1202000000000000000000
      Error: a == b not satisfied [uint]
            Left: 9996673320026600000
           Right: 1000000000000000000000
      Error: a == b not satisfied [uint]
            Left: 1999334664005300000
           Right: 200000000000000000000
      Error: a == b not satisfied [uint]
            Left: 6010000000000000000000
           Right: 10000000000000000000
      Error: a == b not satisfied [uint]
            Left: 1202000000000000000000
           Right: 2000000000000000000
    ```
    -   두 공급자 모두 자신이 넣은 것과는 다른 양의 토큰을 회수하였다. 전체 유동성의 20%를 공급한 사람은 공급한 토큰의 1%만 회수하였고, 가장 적은 양을 공급한 사람은 공급량의 6000%를 회수하였다.
-   공격 난이도
    -   유동성 공급자 모두 공격을 수행할 수 있지만, 가장 마지막에 유동성을 공급한 사람이 거의 모든 지분을 가져갈 수 있어 경쟁적일 수 있다.

### 해결 방안

-   addLiquidity, removeLiquidity 함수를 로직에 맞게 다시 구현해야한다.

## 2. No LP Token Minted

### 설명

문제 코드: Dex.sol addLiquidity 함수

유동성 공급 및 회수에 실제 LP 토큰이 사용되는 것이 아닌 내부 전역변수의 state로 관리된다.

### 파급력

Informational

-   산정 이유
    -   외부 공격이 아닌 컨트랙트 자체의 문제점이다.
    -   유동성 공급을 하더라도 사용자에게 돌아오는 이득이 전혀 없다.
    -   사용자에게 실질적인 이득이 없으니 DEX의 효용성이 떨어지는 문제이다.

### 해결 방법

-   addLiquidity에서 LPToken_balances 등의 배열로 토큰의 수량을 저장하기보다는 mint, burn 등의 함수를 통해 실제로 토큰이 발급되고 회수되는 로직을 구현해야 한다.
