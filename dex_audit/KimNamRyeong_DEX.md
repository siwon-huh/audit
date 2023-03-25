## 김남령

## 1. Remove Liquidity Does Not Work on Proper Request

### 설명

문제 코드: removeLiquidity 함수, 151번 줄

```jsx
function removeLiquidity ... {
	require(amountX >_minimumTokenXAmount && amountY>_minimumTokenYAmount, "INSUFFICIENT_LIQUIDITY_BURNED");
}
```

miniminTokenAmount를 본인이 유동성 공급한 토큰만큼으로 설정할 시 에러를 발생시킨다.

### 파급력

Informational

-   산정 이유
    -   컨트랙트의 효용성이 떨어지는 문제만 발생하기 때문에 Informational로 산정했다.

### 해결방안

-   조건문에 등호를 추가한다.

```jsx
function removeLiquidity ... {
	require(amountX >= _minimumTokenXAmount && amountY >= _minimumTokenYAmount, "INSUFFICIENT_LIQUIDITY_BURNED");
}
```

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
