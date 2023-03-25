## 임나라

## 1. No Imbalance Check on Add Liquidity(2)

### 설명

문제 코드: addLiquidity 함수 전체

유동성 공급 시 유동성 풀 내의 토큰 비율을 검사하지 않는다.

### 파급력

Critical

-   산정 이유
    -   Imbalance하게 유동성을 공급한 모두가 손해를 보며, 기존 유동성 공급자들은 그만큼의 이익을 나눠갖는다.
    -   Frontend에서 서비스 공급자가 실제 유동성 풀의 비율과 다르게 보여준 뒤, 유동성 공급자가 손해를 보도록 유도할 수 있다.
-   공격 시나리오
    -   Imbalance한 유동성 공급이 가능하다.
    -   하지만 유동성 공급 이후 발행된 LP토큰으로 removeLiquidity를 수행하면 더 적은 토큰을 기준으로 LP토큰이 발행된다.
    ```jsx
    function testAddLiquidity2() external {
      uint firstLPReturn = dex.addLiquidity(1000 ether, 1000 ether, 0);
      emit log_named_uint("firstLPReturn", firstLPReturn);

      uint secondLPReturn = dex.addLiquidity(1 ether, 1000 ether, 0);
      emit log_named_uint("secondLPReturn", secondLPReturn);

      (uint tx, uint ty) = dex.removeLiquidity(secondLPReturn, 0, 0);
      emit log_named_uint("second LP remove", tx);
      emit log_named_uint("second LP remove", ty);

      assertEq(tx, 1 ether);
      assertEq(ty, 1000 ether);
    }
    ```
    ```jsx
    [FAIL. Reason: Assertion failed.] testAddLiquidity2() (gas: 274175)
    Logs:
      firstLPReturn: 1000000000000000000000
      secondLPReturn: 1000000000000000000
      second LP remove: 1000000000000000000
      second LP remove: 1998001998001998001
      Error: a == b not satisfied [uint]
            Left: 1998001998001998001
           Right: 1000000000000000000000
    ```
    -   공격 난이도
        -   누구나 접근 가능하므로, 매우 쉽다.
    -   발생할 수 있는 피해
        -   imbalance하게 유동성 공급을 시도한 사람은 공급한 유동성의 상당량을 회수할 수 없게 된다.

### 해결 방안

-   addLiquidity 함수 실행시 유동성 풀에 있는 토큰의 비율을 검사하는 코드가 필요하다.

## 2. Swap over tolerance

### 설명

문제 코드: Dex.sol swap 함수

swap의 규모가 커질 경우 0.01%의 tolerance를 벗어난 거래가 발생한다.

### 파급력

Informational

-   산정 이유
    -   외부 공격이 아닌 컨트랙트 자체의 문제점이다.
    -   swap의 규모가 매우 클 때만 발생한다.
    -   swap 최대치인 60000 ether일 경우에도 0.05%의 fee를 발생시켰다.
-   공격 시나리오
    ```jsx
    function testSwap1() external {
      dex.addLiquidity(3000 ether, 4000 ether, 0);
      dex.addLiquidity(30000 ether * 2, 40000 ether * 2, 0);

      // x -> y
      uint output = dex.swap(60000 ether, 0, 0);

      uint poolAmountX = 60000 ether + 3000 ether;
      uint poolAmountY = 80000 ether + 4000 ether;

      int expectedOutput = -(int(poolAmountX * poolAmountY) / int(poolAmountX + 60000 ether)) + int(poolAmountY);
      expectedOutput = expectedOutput * 999 / 1000; // 0.1% fee
      uint uExpectedOutput = uint(expectedOutput);

      emit log_named_int("expected output", expectedOutput);
      emit log_named_uint("real output", output);

      bool success = output <= (uExpectedOutput * 10001 / 10000) && output >= (uExpectedOutput * 9999 / 10000); // allow 0.01%;
      assertTrue(success, "Swap test fail 1; expected != return");
    }
    ```
    ```jsx
    [FAIL. Reason: Assertion failed.] testSwap1() (gas: 245828)
    Logs:
      expected output: 40934634146341463414634
      real output: 40954612005856515373352
      Error: Swap test fail 1; expected != return
      Error: Assertion Failed
    ```
    -   매우 큰 양의 토큰을 스왑시키면 0.01%를 넘어가는 수수료가 발생한다.
-   공격 난이도
    -   공격자를 요하지 않는 내부 로직 문제이다.

### 해결 방법

-   기존 컨트랙트는 스왑할 토큰의 양에서 0.01%를 제한 만큼을 스왑시킨 결과값을 사용하고 있다.
-   스왑할 토큰의 양을 그대로 두고 나온 값에서 0.01%를 제한 로직으로 바꾸면 해결된다.
