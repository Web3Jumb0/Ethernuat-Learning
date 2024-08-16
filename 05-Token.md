# 05-Token

日期: 16/08/2024
漏洞小结: 溢出问题
简单明了: 版本旧导致的向下溢出问题
难易: 简单

# 问题描述

You are given 20 tokens to start with and you will beat the level if you somehow manage to get your hands on any additional tokens. Preferably a very large amount of tokens.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Token {
    mapping(address => uint256) balances;
    uint256 public totalSupply;

    constructor(uint256 _initialSupply) public {
        balances[msg.sender] = totalSupply = _initialSupply;
    }

    function transfer(address _to, uint256 _value) public returns (bool) {
        require(balances[msg.sender] - _value >= 0);
        balances[msg.sender] -= _value;
        balances[_to] += _value;
        return true;
    }

    function balanceOf(address _owner) public view returns (uint256 balance) {
        return balances[_owner];
    }
}
```

# 问题分析

非常经典的整数溢出问题，尤其是涉及到 Solidity 0.6.0 版本前的代码。在这个版本之前，Solidity的uint256不会自动抛出整数溢出和下溢错误。

```solidity
require(balances[msg.sender] - _value >= 0);
```

看似在确保转账金额 _value 不超过调用者的余额，但实际上这是一个减法操作，而减法操作在 Solidity 0.6.0 之前是不会抛出溢出错误的。如果转移一个大于余额的 _value，减法操作可能会导致整数下溢，产生一个非常大的数值！

# 攻击方式

直接调用`transfer`函数

```solidity
直接调用transfer，给victim合约打21个代币即可：contract.transfer("0x478f3476358Eb166Cb7adE4666d04fbdDB56C407",21)
```

成功后会获得一个是非巨大的balance

# 小白的思考

刚开始看的时候没看出来什么问题，觉得一切都非常的ok，逻辑合理，但是版本是0.6.0非常亮眼，估计是跟版本相关的。

查了一下，结合相关SafeMath这个库（他为啥不用），才想到了这个问题。

和GPT求证了一下：

*在 Solidity 0.6.0 之前，uint256 类型的数字是不会自动抛出溢出或下溢错误的。*

*如果两个 uint256 数字 A 和 B 满足 A < B，那么 A - B 的结果本来应该是一个负数。但由于 uint256 是无符号的整数，它只能存储非负数。因此，当出现负数时，它会“溢出”到一个极大的正数。*

*在 Solidity 0.6.0 之前的版本中，A - B 的结果会按照这种“环绕”方式进行处理：*

```
•	假设 A = 5，B = 10，那么 5 - 10 的结果会变成 2^256 - 5，也就是一个非常大的数值（相当于 2 的 256 次方减 5）。
```

*这种机制被称为“下溢”，会使得小于零的计算结果变成一个极大的数。这就是为什么在这个题目中你可以通过传入大于自己余额的 _value，让 balances[msg.sender] 下溢，从而获得一个巨额的代币余额。*

*从 Solidity 0.8.0 开始，编译器会默认开启溢出检查。如果出现溢出或下溢，代码会自动抛出错误，防止这种漏洞。*

## 进一步思考

我自学开发是从0.8.0开始的，此版本已经改进了非常多的问题，甚至都不需要用safeMath来进行加减乘除

> 在 Solidity 0.8.0 之后，SafeMath 的使用变得不再必要，直接进行加减乘除操作是安全的。
> 

所以上面这些问题，在遇到的时候，如果逻辑上没有问题，那就需要从版本上入手了。跟最早的时候传统安全打靶机一样一样，都是找到版本搜索漏洞。

但是今后这种相关的肯定会与来越少，编译器也不允许低级版本。

把它当做一种知识和经验来学习吧。