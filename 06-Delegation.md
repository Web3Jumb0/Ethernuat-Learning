# 06-Delegation

日期: 16/08/2024
漏洞小结: Delegation漏洞
简单明了: 调用Delegation Call，执行另一个合约的函数，但是修改的内容还是自己的
难易: 中等

# 问题描述

The goal of this level is for you to claim ownership of the instance you are given.

Things that might help

- Look into Solidity's documentation on the `delegatecall` low level function, how it works, how it can be used to delegate operations to on-chain libraries, and what implications it has on execution scope.
- Fallback methods
- Method ids

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Delegate {
    address public owner;

    constructor(address _owner) {
        owner = _owner;
    }

    function pwn() public {
        owner = msg.sender;
    }
}

contract Delegation {
    address public owner;
    Delegate delegate;

    constructor(address _delegateAddress) {
        delegate = Delegate(_delegateAddress);
        owner = msg.sender;
    }

    fallback() external {
        (bool result,) = address(delegate).delegatecall(msg.data);
        if (result) {
            this;
        }
    }
}
```

# 思路分析

这个很容易，更改owner的函数只有在Delegate合约里面的`pwn`函数，我们需要调用Delegation里面的fallback就行

原理就是做一个low level call，把`pwn`的签名作为一个参数。去call的时候发现Delegation合约里面没有匹配的，那么就会触发fallback函数去调用`delegatecall`，并且这个pwn的签名会变成`msg.data`传过去，最后调取了Delegate合约里面的`pwn`函数，但是修改的还是Delegation合约里面的`owner`变量

# 攻击方式

1. 找到pwn函数的签名
    
    `web3.eth.abi.encodeFunctionSignature("pwn()")`
    
    输出的结果为`0xdd365b8b`
    
2. 用web console调用
    
    `await contract.sendTransaction({ data: "0xdd365b8b" });`
    
3. 用Remix调用，直接at address里面输入目标实例合约地址，然后再low level transaction那边输入pwn的签名并且发送

# 小白的思考

刚开始使用的是remix的方法，但是我漏做了一个操作，导致了一个很有趣的现象

正确的操作应该如下

1. 复制源码到remix中，compile
2. 选择Delegation合约
3. 按照攻击方式里面的第三步进行攻击

然而我漏做了第二步骤，有Delegate合约在代码中位于Delegation合约的上方，因此默认是Delegate合约。

关键是的确返回了abi给我，并且是Delegate的合约，有pwn和owner两个函数。

我直接点击了一下pwn，结果就搞定了，并且一脸懵逼。这题目虽然不难但也是这么设计的呀。

虽然后来发现我是漏做了第二步，但是非常好奇为什么我这样子也能成功，一个地址上面不应该是只有一个合约吗？难道我发现了什么神奇的攻击技巧？

## 深入研究（非常有意思）

和GPT打了几个来回，大概明白了，如下

1. **合约地址**
    - 这个地址确实是 `Delegation` 合约实例的地址。每个合约部署在链上的地址是唯一的，`Delegation` 合约在部署时会有自己的地址。
2. **错误的 ABI 导致的函数调用**
    - 当你在 Remix 中使用 `at address` 功能并选择了错误的合约 ABI（比如 `Delegate` 的 ABI），虽然地址对应的是 `Delegation` 合约，但 Remix 以为你在与 `Delegate` 合约交互。
    - 因为你选择了 `Delegate` 的 ABI，Remix 会显示 `Delegate` 合约中的函数，包括 `pwn()`。
3. **函数调用的实际效果**
    - 当你点击 `pwn()` 函数时，实际上是向 `Delegation` 合约发送了一个交易，因为地址是 `Delegation` 合约的。
    - `Delegation` 合约并没有 `pwn()` 函数，因此这个调用会触发 `Delegation` 合约的 `fallback` 函数。
    - `fallback` 函数会调用 `delegatecall`，并将 `msg.data` 传递给 `Delegate` 合约。这时候 `msg.data` 是 `0xdd365b8b`，即 `pwn()` 函数的签名。
4. **`delegatecall` 的执行**
    - `delegatecall` 会在 `Delegation` 合约的上下文中执行 `Delegate` 合约的 `pwn()` 函数。这意味着虽然 `pwn()` 函数的代码来自 `Delegate` 合约，但它实际上修改的是 `Delegation` 合约的存储（即 `owner` 变量）。
    - 结果是 `Delegation` 合约的 `owner` 被改成了调用者的地址（你的地址）。

### 总结

- **地址**：`0x8Aae4D9ae2bD7688DC814eFa54Fb557F246499D4` 是 `Delegation` 合约的地址。
- **ABI 错误**：在 Remix 中选择了 `Delegate` 的 ABI，会导致显示 `pwn()` 函数。
- **实际调用**：点击 `pwn()` 函数时，实际上是在 `Delegation` 合约上执行，因为地址指向的是 `Delegation` 合约。
- **`fallback` 和 `delegatecall`**：`Delegation` 合约的 `fallback` 函数处理了这个调用，并通过 `delegatecall` 执行了 `Delegate` 合约的 `pwn()` 函数，从而修改了 `Delegation` 合约的状态。

你对这个过程的理解是准确的！这种情况展示了 `delegatecall` 在合约调用中的强大功能，以及如何利用合约的设计模式来实现预期的功能。