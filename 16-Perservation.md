# 16-Perservation

日期: 21/08/2024
难易: 困难
漏洞小结: delegate
简单明了: 利用delegate调用自创函数

# 问题描述

This contract utilizes a library to store two different times for two different timezones. The constructor creates two instances of the library for each time to be stored.

The goal of this level is for you to claim ownership of the instance you are given.

Things that might help

- Look into Solidity's documentation on the `delegatecall` low level function, how it works, how it can be used to delegate operations to on-chain. libraries, and what implications it has on execution scope.
- Understanding what it means for `delegatecall` to be context-preserving.
- Understanding how storage variables are stored and accessed.
- Understanding how casting works between different data types.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Preservation {
    // public library contracts
    address public timeZone1Library;
    address public timeZone2Library;
    address public owner;
    uint256 storedTime;
    // Sets the function signature for delegatecall
    bytes4 constant setTimeSignature = bytes4(keccak256("setTime(uint256)"));

    constructor(address _timeZone1LibraryAddress, address _timeZone2LibraryAddress) {
        timeZone1Library = _timeZone1LibraryAddress;
        timeZone2Library = _timeZone2LibraryAddress;
        owner = msg.sender;
    }

    // set the time for timezone 1
    function setFirstTime(uint256 _timeStamp) public {
        timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
    }

    // set the time for timezone 2
    function setSecondTime(uint256 _timeStamp) public {
        timeZone2Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
    }
}

// Simple library contract to set the time
contract LibraryContract {
    // stores a timestamp
    uint256 storedTime;

    function setTime(uint256 _time) public {
        storedTime = _time;
    }
}
```

# 攻击思路

这道题目相比较别的算是比较难的

- 要claim owner但是只有constructor里面有owner，所以只能通过别的方式去实现owner的转变了
- 很明显是delegatecall是突破点
    - 调用LibraryContract设置时间，会调用setTime，但是值拿回来之后会赋值给slot0，而不是slot3的storedTime，问题出在这
- 我们设计一个攻击合约，里面有和Preservation合约对应的变量
    - 确保owner的slot可以对应上
- 然后设置一个setup函数，signature要和调用的一样，但是内容是owner=msg.sender
- 部署这个攻击函数，活得地址，地址长度为40 bytes，然后paddingleft成64
- 作为变量传入setFirstTime函数，这样就可以把slot0设置成我们的攻击函数
    - 即便他的本意是set storedTime的时间
- 在执行一次setFirstTime函数，现在目标函数已经设置成我们的供给函数，并且里面也有setup会被调用，就可以实现owner=msg.sender

# 攻击方式

1. 部署一个恶意合约（攻击合约），这个合约包含一个伪造的 `setTime` 函数，该函数会将 `Preservation` 的 `owner` 地址修改为攻击者的地址。
2. 使用 `Preservation` 合约中的 `setFirstTime` 函数，并将 `timeZone1Library` 地址改为攻击合约的地址。
3. 再次调用 `setFirstTime`，这次会执行你的恶意合约的 `setTime` 函数，将 `owner` 改为你的地址。

# Ethernuat总结

As the previous level, `delegate` mentions, the use of `delegatecall` to call libraries can be risky. This is particularly true for contract libraries that have their own state. This example demonstrates why the `library` keyword should be used for building libraries, as it prevents the libraries from storing and accessing state variables.

# 小白的思考

涉及到很多细节，首先是GPT的笔记，可以记录一下，详细介绍了delegatecall

## delegatecall

### `delegatecall` 的工作原理

`delegatecall` 是一种低级别调用，它在执行时有几个关键特点：

1. **调用目标合约的代码**，但在**调用者合约的上下文中**执行。
2. 目标合约的存储不会被修改，而是调用者合约的存储会被修改。
3. 目标合约的函数需要有相同的函数签名，否则调用将失败。

换句话说，`delegatecall` 允许一个合约去调用另一个合约的函数，但所有的存储操作都发生在调用者（即原始合约）中。这就解释了为什么两个独立的合约可以通过 `delegatecall` 产生影响。

### 挑战中的问题

你看到的 `Preservation` 合约中确实没有直接使用 `LibraryContract`，但 `LibraryContract` 的代码在逻辑上被“借用”了。看一下这个逻辑：

```solidity
solidityCopy code
function setFirstTime(uint256 _timeStamp) public {
    timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
}

```

这里使用了 `delegatecall`，其中 `timeZone1Library` 是一个合约地址，它指向了一个合约，比如说 `LibraryContract`。当调用这个函数时，`delegatecall` 实际上是在当前合约（`Preservation`）的上下文中执行 `LibraryContract` 的 `setTime` 函数。由于存储布局不同，本应更新 `LibraryContract` 中的 `storedTime` 的操作，现在会更新 `Preservation` 合约中的某个存储变量。

### 存储布局冲突的关键点

在 `Preservation` 合约中，变量按以下顺序存储：

1. `timeZone1Library` - slot 0
2. `timeZone2Library` - slot 1
3. `owner` - slot 2
4. `storedTime` - slot 3

而在 `LibraryContract` 中，只有一个变量 `storedTime`，它存储在 slot 0。当 `delegatecall` 执行 `LibraryContract.setTime` 时，它会试图更新 slot 0，但由于在 `Preservation` 合约的上下文中，slot 0 是 `timeZone1Library` 的位置，这意味着你实际上可以通过调用这个函数来更改 `Preservation` 中的 `timeZone1Library` 地址。

## 一些type转换的小技巧

- padd成64，左边都是0 - `web3.utils.padLeft(attackAddress,64)`