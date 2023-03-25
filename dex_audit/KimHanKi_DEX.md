## 김한기

## 1. RemoveLiquidity Does Nothing

### 설명

문제 코드: removeLiquidity 함수 전체

removeLiquidity에 LPToken만큼의 tokenX와 tokenY를 되돌려주는 로직이 구현되어있지 않다.

removeLiquidity는 돌려받아야할 tokenX와 tokenY의 값만 반환한다.

### 파급력

Critical

-   산정 이유
    -   유동성을 공급한 모든 사람에게 해당하는 문제이다.
    -   유동성을 공급한 모든 사람은 LP토큰만 받고 자신의 자산을 되돌려받을 수 없다.
-   공격 시나리오
    -   유동성을 공급하고 제거하는 일반적인 경우이다.
    -   removeLiquidity의 반환값은 정상적으로 출력되나 실제 LP토큰과 X,Y토큰 모두의 balance에 반영되지 않는다.
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
-   공격 난이도
    -   DEX 설계자로 인해 발생한 문제로, 참여하는 모두가 영향을 받는다.

### 해결책

removeLiquidity 함수가 LPToken의 burn과 토큰 X,Y에 대한 transfer를 수행할 수 있도록 고쳐야 한다.

## 2. No authority check for LP mint

### 설명

문제 코드: LPT 컨트랙트 전체, transfer 함수 전체

DEX 컨트랙트와는 다른 컨트랙트를 통해 LP토큰을 민팅하였으나, 이를 transfer하는 과정에서 검증이 이루어지지 못해 누구나 LP토큰을 민팅할 수 있는 문제점이 발생했다.

### 파급력

Critical

-   산정 이유
    -   1번 문제로 인해 LP토큰의 효용성이 현재는 아예 없지만, removeLiquidity가 고쳐진 상태에서는 유동성 풀의 전체 토큰을 탈취할 수 있는 심각한 문제점이다.
    -   공격자는 별다른 조건 없이 transfer를 호출해 무한정으로 LP토큰을 발급받을 수 있다.
-   공격 시나리오
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
    -   transfer를 호출하면 DEX 컨트랙트가 LPToken을 민팅하고, 이를 임의로 지정한 주소로 보낼 수 있다.
    -   DEX가 LPT를 선언하기 때문에 LPT의 admin은 DEX이다.
    -   따라서 transfer가 누구에 의해서 호출되건 lpt는 mint를 실행해서 to 주소로 LPToken을 보낸다.
-   공격 난이도
    -   transfer 함수를 호출할 수 있는 사람은 모두 공격자가 될 수 있다.

### 해결 방안

-   transfer 함수에 자체에 대해 관리자만 내용을 실행할 수 있도록 권한을 설정해주어야 한다.

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

-   산정 이유
    -   첫 유동성 공급자, 즉 service provider가 유의하면 문제는 발생하지 않는다. 유동성 공급 과정에서 1 ether 이상을 보낸 일이 없어야 한다.
    -   문제가 발생할 가능성이 거의 없다.
-   공격 시나리오
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
    -   TokenX와 TokenY를 각각 0.9 ether씩 100번 공급하고, 이후 1 ether씩을 공급해준다.
    -   지금까지 공급된 양이 전부 첫 유동성 공급으로 반영된다.
    -   이후 공급된 유동성에 대해서는 문제 없이 반영된다.

### 해결방안

-   setPrice에서 decimal로 나누지 않아야 한다.
-   나눗셈 등 decimal이 직접적으로 사용되지 않아도 되는 부분에서의 사용을 지양해야 한다.
