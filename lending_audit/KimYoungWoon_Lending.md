## 김영운

## 미구현된 부분: 없음

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

-   amount를 계산하는 과정에서 1 ether의 상수를 나눈 뒤 곱해준다. 이 과정에서 1 ether아래의 값은 모두 버려진다.
-   1 ether 미만의 수수료는 버려진 채로 withdraw를 수행한다. 따라서 1 ether 미만의 수수료를 청구하는 유동성 공급자에 대해서는 수수료를 전혀 지급하지 않는다.
-   1 ether 미만의 유동성을 공급한 공급자는 원금과 이자의 합이 1 ether 이상이 될때까지 원금을 출금할 수 없으며, 수수료 또한 원금+수수료가 2 ether 이상이 되기 전까지 한동안 버려지게 된다.

### 파급력

Medium

-   산정 이유
    -   DEX의 효용성을 낮추는 문제이고, 피해 규모가 유동성 공급 토큰의 가치에 따라 달라질 수 있으므로 를 부여했다.
-   공격 시나리오
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
    -   10 ether의 USDC를 유동성으로 공급하고, 0.1 ether USDC 만큼의 이자를 요청해도 이를 지급하지 않는다.
    -   또한
-   공격 난이도
    -   유동성을 공급한 모두에게 해당되며, 1 ether 수수료 및 원금은 회수가 불가능해진다.

### 해결 방안

-   amount를 계산하는 과정에서 `WAD`를 배제해야 한다.

## 2. No Partial USDC Withdraw Implemented

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

-   산정 이유
    -   손해보는 사람이 없으며, 기능상의 구현 부족으로 판단된다.
 
### 해결 방안

-   amount 값을 구할 때 getAccruedSupplyAmount의 결과값을 amount / 현재까지의 usdc 토큰 공급량으로 비율을 계산해주어야 한다.
