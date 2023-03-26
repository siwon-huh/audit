## 서준원

## 1. Lending Under Ether-level Produces Interest Error

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

-   산정 이유
    -   DEX의 효용성을 다소 떨어뜨리며, 1 ether 미만의 값은 모두 영향을 받는다.
    -   소수점 단위의 대출은 현실에서 비교적 자주 일어날 수 있다.
-   공격 시나리오

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

    -   테스트케이스에 있던 ether를 0.1 ether로 바꾸어 테스트해보았다.
    -   계산과정에서 작은 단위의 수가 버려지며 오차를 발생시켰다.
    -   원래대로라면 나왔어야 했을 이자의 약 20%만 발생했으며, 10\*\*10 단위로 설정할 시 이자 지급이 이루어지지 않는다.

-   공격 난이도
    -   비교적 빈번하게 일어날 수 있으며, 공격자가 따로 존재하기 보다는 컨트랙트 자체의 문제점이다.

### 해결 방안

-   Interest modifier 설정 과정에서 잦은 나눗셈을 최대한 지양하여야 한다.
 
