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
    - DEX 서비스 자체의 효용성이 떨어진다.
- 공격 시나리오
    - 첫 유동성 공급자가 공급한 두 토큰의 곱이 1000000으로 나누어떨어지지 않는 경우 무조건 문제가 발생한다. 이 경우 LP토큰 발행량의 1000 wei 아래의 값은 모두 버려진다.
    - 다만 이러한 문제가 발생했을 경우 DEX 유동성 공급자들이 유동성을 회수할 때 공급한 양보다 약간 더 적은 양을 받게 된다.
    - DEX 서비스 자체에 문제가 생기기 때문에 Medium를 부여했다.
        
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

### 파급력

Critical

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
      Attacker LP2 remove: 900000000000000000
      Attacker LP2 remove: 900000000000000000 ether);
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

## 임나라

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

## 최영현

## 1. No Imbalance Check on Add Liquidity

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
    - 아래 코드에서 TokenY는 1 ether만 공급하지만 LP토큰으로 회수하는 양은 667.3.. ether이다.
    
    ```jsx
    function testAddLiquidity() external {
      uint firstLPReturn = dex.addLiquidity(500 ether, 1000 ether, 0);
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
    [FAIL. Reason: Assertion failed.] testAddLiquidity1() (gas: 230413)
    Logs:
      firstLPReturn: 500000000000000000000000
      secondLPReturn: 1000000000000000000000000
      second LP remove: 1000000000000000000000
      second LP remove: 667333333333333333333
      Error: a == b not satisfied [uint]
            Left: 667333333333333333333
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

## 김지우

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
