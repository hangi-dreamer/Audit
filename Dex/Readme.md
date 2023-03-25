## 1. transfer함수에 require 등의 검사가 없는것

### 설명
[DEX.sol의 105-108행](https://github.com/koor00t/DEX_solidity/blob/431117c3ead6170dfb8c0d4dce4819ce4d25c113/src/Dex.sol#L105-L108)
function transfer(address to, uint256 lpAmount) external returns (bool); 함수의 구현부분에서 해당 함수를 admin 또는 제한된 일부 address만 실행 할 수 있도록 해야하는데, 그 처리가 부재하여 누구나 lp토큰을 mint 할 수 있는 취약점 입니다.

### 파급력
심각도는 Critical로써 **정상적이지 않은** lp토큰이 이론상 uint256 max 값만큼 mint 될 수 있어, 해당 비정상적으로 mint된 토큰으로 다른사람들이 공급한 유동성 토큰을 전부 탈취 할 수 있습니다.

### 해결방안
함수 구현부 최상단에 아래와 같은(비슷한) 추가하면 됩니다.

```
require(_owner == msg.sender, "onlyOnwner");
```

혹은 해당함수에서 mint 하는게 아닌 이미 mint된 상태이고 그게 컨트랙트가 가지고 있는상태에서

```
transfer(msg.sender, to, amount);
```

위와 같은 형태로 구현된 경우엔 require 구문이 없더라도 호출한 계정의 lp토큰이 보내지도록 구현되어있으므로 공격이 아님.

### 해당 취약점 발견된 목록
* [koor00t/DEX_solidity](https://github.com/koor00t/DEX_solidity/blob/431117c3ead6170dfb8c0d4dce4819ce4d25c113/src/Dex.sol#L105-L108)
* [Gamj4tang/DEX_solidity](https://github.com/Gamj4tang/DEX_solidity/blob/869b700f6d3bfcc259095831030df0b5aad3f322/src/Dex.sol#L172-L175)
* [kimziwu/DEX_solidity](https://github.com/kimziwu/DEX_solidity/blob/49c002dec844ced9bdae62d5a01c0d641899e211/src/Dex.sol#L162-L165)
* [hangi-dreamer/Dex_solidity](https://github.com/hangi-dreamer/Dex_solidity/blob/89a39ccd2d03346ae6ad8bfc1a35cf7a478f0593/src/Dex.sol#L159-L166)


## 2. burn의 부재로 인한 무한 유동성 회수 가능한것.

### 설명
[Dex.sol의 138-157행](https://github.com/hangi-dreamer/Dex_solidity/blob/89a39ccd2d03346ae6ad8bfc1a35cf7a478f0593/src/Dex.sol#L138-L157)에서 유동성을 회수할때 제출한 lp토큰만큼 적절한 수량의 토큰 x,y를 반환해주고 해당 제출된 lp토큰은 burn 되어야하는데 burn구문이 없어서, 공격자는 컨트랙트가 소유중인, 다른사람들이 공급한 유동성 토큰 x,y를 전부 탈취 할 수 있다.

### 파급력
심각도는 Critical로써 아주 적은량의 유동성을 공급했더라도 해당 lp토큰으로 dex의 모든 유동성을 탈취할 수 있다.

### 해결방안
유동성 해제 함수의 구현부에 burn 을 넣어주고, 추가적으로 유동성 해제 이전의 balance와 이후의 balance의 계산이 맞는지 검사한다면 추가적인 안전을 도모할 수 있을꺼같다.

### 해당 취약점 발견된 목록
* [hangi-dreamer/Dex_solidity](https://github.com/hangi-dreamer/Dex_solidity/blob/89a39ccd2d03346ae6ad8bfc1a35cf7a478f0593/src/Dex.sol#L138-L157)

## 3. 유동성 회수 이후 토큰 x,y 반환 하지 않음

### 설명
[Dex.sol의 138-157행](https://github.com/hangi-dreamer/Dex_solidity/blob/89a39ccd2d03346ae6ad8bfc1a35cf7a478f0593/src/Dex.sol#L138-L157)에서 msg.sender가 제출한 LPTokenAmount에 대한 적절한 토큰 x,y를 반환해야하는데, 함수 구현부에 사용자에게 해당 토큰을 전송하는 부분이 없어서, 많은 유동성을 공급했다면 해당 유동성은 영원히 회수하지 못하는 점으로 스캠앱에 토큰을 탈취당한것으로 볼 수 있다.

### 파급력
심각도는 Information 유동성 회수기능을 확인하지않고 많은량의 유동성을 공급했다면 해당 유동성은 다시는 회수하지 못하므로 전부 빼앗기게 되는것으로써 공격자에 의한 공격은 아니지만 일반사용자 관점에서 스캠앱으로 인식 될 수 있음.

### 해당 취약점 발견된 목록
* [hangi-dreamer/Dex_solidity](https://github.com/hangi-dreamer/Dex_solidity/blob/89a39ccd2d03346ae6ad8bfc1a35cf7a478f0593/src/Dex.sol#L138-L157)

## 4. 유동성 공급시 페어 중에서 특정 토큰 기준으로 lp토큰 발급

### 설명
[Dex.sol 54행](https://github.com/hyeon777/DEX_Solidity/blob/2ae38134e37923d5331fd1bae046dbabf0a83d2d/src/Dex.sol#L54)에 lp토큰이 발행되어 있는경우의 else 분기를 탈때 X 토큰 기준으로 lp 토큰이 발급되어서, 공격자가 y토큰은 아주 적은량만 유동성 공급을 하게되어도 x토큰 기준으로 lp토큰이 발급되어 추후 유동성제거 할때 공격자가 공급한것보다 많은량의 y토큰을 회수 할 수 있습니다.

### 파급력
심각도는 Critical로써 공격자가 어느정도의 x토큰만 있으면 유동성 공급, 회수를 반복적으로 수행하면서 Dex의 y토큰을 탈취 할 수 있습니다.

### 해결방안
유동성 공급 함수에서 양쪽 비율이 안맞아도, 유동성을 공급 할 수 있게 구현한다면 페어중에서 적게 공급한 토큰을 기준으로 lp토큰을 발급하여야합니다.

혹은 비율이 맞지 않은 유동성 공급은 받지 않아야합니다.

### 해당 취약점 발견된 목록
* [hyeon777/DEX_Solidity](https://github.com/hyeon777/DEX_Solidity/blob/2ae38134e37923d5331fd1bae046dbabf0a83d2d/src/Dex.sol#L54)
* [jun4n/DEX_solidity](https://github.com/jun4n/DEX_solidity/blob/08dc46f0f3b54633bef050802f8faa7ee0ef45dd/src/Dex.sol#L66)
* [dlanaraa/DEX_solidity](https://github.com/dlanaraa/DEX_solidity/blob/766dc1dca3cc92362959044d9555800fca5c153e/src/dex.sol#L92)
*

## 5. 유동성 공급시 발급된 lp토큰과 다른 수량의 페어를 공급함.

### 설명
[Dex.sol 72-79행](https://github.com/Gamj4tang/DEX_solidity/blob/869b700f6d3bfcc259095831030df0b5aad3f322/src/Dex.sol#L72-L79)에 보면 유동성 공급시 다른 비율로 유동성을 공급하려고 할때, 적게 제출한 토큰 기준으로 lp토큰을 발급하도록 구현(72행)했을때 해당 lp토큰에 맞는 토큰 x,y만 dex에 전달해야하는데 사용자가 제출한 그대로를 유동성으로 공급함(78-79행).

### 파급력
심각도는 Information으로 올바르지 않은 비율로 유동성을 공급하려고 시도하고 그중에 작은 수량의 토큰에 맞춰서 lp토큰을 발행해줬는데, 유동성 풀로 들어간건 제출을 시도한 만큼이므로 추후 유동성 회수시 특정 토큰의 양이 현저하게 부족한 현상이 생겨서 스캠앱으로 인식 할 수 있다.

### 해결방안
발행된 lp토큰만큼만 x, y토큰 모두 유동성 공급되도록 해야합니다.

### 해당 취약점 발견 목록
* [Gamj4tang/DEX_solidity](https://github.com/Gamj4tang/DEX_solidity/blob/869b700f6d3bfcc259095831030df0b5aad3f322/src/Dex.sol#L72-L79)
* [jw-dream/DEX_solidity](https://github.com/jw-dream/DEX_solidity/blob/1af89959aa9c624e7da651a3a752d1627fff7f9e/Dex_solidity/src/Dex.sol#L62-L67)
* [kimziwu/DEX_solidity](https://github.com/kimziwu/DEX_solidity/blob/49c002dec844ced9bdae62d5a01c0d641899e211/src/Dex.sol#L116-L125)
* [jt-dream/Dex_solidity](https://github.com/jt-dream/Dex_solidity/blob/a08ea83a3335588d868bcf01114de3a57cb0cada/src/Dex.sol#L40-L48)
