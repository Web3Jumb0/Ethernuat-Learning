# 15-Naught Coin

日期: 21/08/2024
难易: 中等
漏洞小结: 绕过transfer
简单明了: ERC20除了transfer还可以approve+transferfrom

# 问题描述

NaughtCoin is an ERC20 token and you're already holding all of them. The catch is that you'll only be able to transfer them after a 10 year lockout period. Can you figure out how to get them out to another address so that you can transfer them freely? Complete this level by getting your token balance to 0.

Things that might help

- The [ERC20](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md) Spec
- The [OpenZeppelin](https://github.com/OpenZeppelin/zeppelin-solidity/tree/master/contracts) codebase

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";

contract NaughtCoin is ERC20 {
    // string public constant name = 'NaughtCoin';
    // string public constant symbol = '0x0';
    // uint public constant decimals = 18;
    uint256 public timeLock = block.timestamp + 10 * 365 days;
    uint256 public INITIAL_SUPPLY;
    address public player;

    constructor(address _player) ERC20("NaughtCoin", "0x0") {
        player = _player;
        INITIAL_SUPPLY = 1000000 * (10 ** uint256(decimals()));
        // _totalSupply = INITIAL_SUPPLY;
        // _balances[player] = INITIAL_SUPPLY;
        _mint(player, INITIAL_SUPPLY);
        emit Transfer(address(0), player, INITIAL_SUPPLY);
    }

    function transfer(address _to, uint256 _value) public override lockTokens returns (bool) {
        super.transfer(_to, _value);
    }

    // Prevent the initial owner from transferring tokens until the timelock has passed
    modifier lockTokens() {
        if (msg.sender == player) {
            require(block.timestamp > timeLock);
            _;
        } else {
            _;
        }
    }
}
```

# 攻击思路

要熟悉原生最基础的erc20，这里只有对transfer函数做了timelock，但是对approve以及`transferFrom`函数都没有做，所以我们可以先`approve`我们自己，然后transfer from我们自己到随便一个地址，然后我们的账户就归零了

# 攻击方式

1. 复制合约到remix，修改solidity的库，让compile没有问题
2. 部署合约，调用ethernuat上面的实例，我们可以看到很多functions
    1. 即便是代码里面没有显示的（override的），也会显示出来
        1. 有`approve`和`transferFrom`
    2. 我们自己`approve`自己所有的额度
    3. 我们用`approve`的额度去做`transferFrom`，`transferFrom`没有timelock，所以可以随便转出去

# Ethernuat总结

When using code that's not your own, it's a good idea to familiarize yourself with it to get a good understanding of how everything fits together. This can be particularly important when there are multiple levels of imports (your imports have imports) or when you are implementing authorization controls, e.g. when you're allowing or disallowing people from doing things. In this example, a developer might scan through the code and think that `transfer` is the only way to move tokens around, low and behold there are other ways of performing the same operation with a different implementation.

# 小白的思考

这道题属于知道就是知道，不知道就是不知道

1. transfer函数不是唯一move token的方式
2. 刚开始的想法是绕过timelock，因为只有if有require，但是else没有。
    1. 明显是对erc20不熟悉，因为调用transfer的时候是从调用者那边扣除token的
    2. 如果用攻击合约调用一定会失败，因为攻击合约压根没有这个token的balance
3. 去compile的时候发现库不对，想了一会
    1. 其实大胆地把[erc20的库](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol)替换掉就好