## 서지혜

## 1. Initial Liquidity Under 1 ether Gets Rounded Down

### 설명

문제 코드: Dex.sol addLiquidity 함수, 67번째 줄

```java
function addLiquidity ... {
	...
	if(totalSupply() == 0){
    lpAmount = tokenXAmount * tokenYAmount / _decimal;
  }
	...
}
```

첫 LP 토큰의 발행 시 두 토큰의 양의 곱을 decimal(10의 18승)으로 나누는데, 이 과정에서 decimal 이하의 값이 버려지는 현상이 발생한다.

### 파급력

Low

- 산정 이유
    - 첫 유동성 공급자만 손해를 본다.
- 공격 시나리오
    
    ```java
    function testAddLiquidity3() external {
      uint firstLPReturn = dex.addLiquidity(10**9, 10**8, 0);
      emit log_named_uint("firstLPReturn", firstLPReturn);
    
      uint secondLPReturn = dex.addLiquidity(1 ether, 0.1 ether, 0);
      emit log_named_uint("secondLPReturn", secondLPReturn);
    
      (uint tx, uint ty) = dex.removeLiquidity(secondLPReturn, 0, 0);
      emit log_named_uint("second LP remove", tx);
      emit log_named_uint("second LP remove", ty);
    
      assertEq(tx, 1 ether);
      assertEq(ty, 0.1 ether);
    }
    ```
    
    ```java
    [FAIL. Reason: Assertion failed.] testAddLiquidity3() (gas: 192172)
    Logs:
      firstLPReturn: 0
      secondLPReturn: 100000000000000000
      second LP remove: 1000000001000000000
      second LP remove: 100000000100000000
      Error: a == b not satisfied [uint]
            Left: 1000000001000000000
           Right: 1000000000000000000
      Error: a == b not satisfied [uint]
            Left: 100000000100000000
           Right: 100000000000000000
    ```
    
    - 첫 유동성 공급 시 두 토큰의 곱이 1 ether 아래이기 때문에 모두 버려져 0이 되는 것을 볼 수 있다.
- 공격 난이도
    - 공격자를 요하지 않는 내부 로직 문제이다.

### 해결 방안

- 해당 로직은 두 토큰의 양을 곱하면서 단위 (ether)가 제곱되는 현상을 해소하고자 만들어진 것으로 보인다. 직접적인 나눗셈으로 인한 버림을 피하기 위해 초기 토큰 발행 시 제곱근을 적용하는 등의 방식이 적용될 수 있다.

## 2. LPTokenAmount Dependent to Token X

### 설명

문제 코드: Dex.sol addLiquidity 함수, 71번째 줄

```java
function addLiquidity ... {
	...
	else{
  require(_decimal * tokenXAmount / tokenYAmount == _decimal * _amountX / _amountY, "amount breaks the pool ratio");
  lpAmount = totalSupply() * tokenXAmount / _amountX;
  }
	...
}
```

require 구문에서 tokenX와 Y의 교환비가 극단적으로 차이나는 경우 (1 : 1 ether 이상) rounding이 발생해 검사를 우회할 수 있고, 유동성 풀의 balance를 깨는 공급이 가능해진다. 또한 이러한 조건을 거친 뒤 발행하는 LPToken의 양이 X의 공급량에 의존한다. 따라서 X를 많이 공급하고 Y를 정상적인 비율보다 적게 공급하더라도 발급받은 LPToken으로 X의 양에 비례하는 Y를 회수할 수 있다.

### 파급력

Medium

- 산정 이유
    - 실질적인 공격이 발생할 가능성이 매우 적지만, 발생시 유동성 풀에 매우 큰 영향을 미칠 수 있다.
- 공격 시나리오
    
    ```java
    function testAddLiquidity3() external {
      uint firstLPReturn = dex.addLiquidity(1, 10 ether, 0);
      emit log_named_uint("firstLPReturn", firstLPReturn);
    
      uint secondLPReturn = dex.addLiquidity(20, 50 ether, 0);
      emit log_named_uint("secondLPReturn", secondLPReturn);
    
      (uint tx, uint ty) = dex.removeLiquidity(secondLPReturn, 0, 0);
      emit log_named_uint("second LP remove", tx);
      emit log_named_uint("second LP remove", ty);
    
      assertEq(tx, 20);
      assertEq(ty, 50 ether);
    }
    ```
    
    ```java
    [FAIL. Reason: Assertion failed.] testAddLiquidity3() (gas: 239052)
    Logs:
      firstLPReturn: 10
      secondLPReturn: 200
      second LP remove: 20
      second LP remove: 57142857142857142857
      Error: a == b not satisfied [uint]
            Left: 57142857142857142857
           Right: 50000000000000000000
    ```
    
    - require 구문에서 좌항과 우항 모두 0의 값을 가지므로 balance가 맞지 않은 값이 통과할 수 있다. 또한 초기에 설정해놓은 1 : 10 ether의 값을 깨고 20 : 50 ether로 유동성을 공급할 수 있으며, 이를 이용해 발급한 LP토큰을 회수할 시 공급한 양보다 더 많은 양을 회수할 수 있다.
- 공격 난이도
    - 공격자가 첫 유동성 공급자가 공급한 수량을 어느정도 알고 있어야 공격에 성공할 수 있다는 점에서 현실적으로 어려움이 크다.

### 해결방안

require 구문 검사 시 나눗셈을 지양하고 decimal과의 곱 또한 제거하는 것이 좋다.

```java
require(tokenXAmount * _amountY == _amountX * tokenYAmount, "amount breaks the pool ratio");
```
