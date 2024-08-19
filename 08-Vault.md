# 08-Vault

日期: 19/08/2024
漏洞小结: 修饰词问题
简单明了: private是可调用范围，并不是无法读取
难易: 简单

# 问题描述

Unlock the vault to pass the level!

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Vault {
    bool public locked;
    bytes32 private password;

    constructor(bytes32 _password) {
        locked = true;
        password = _password;
    }

    function unlock(bytes32 _password) public {
        if (password == _password) {
            locked = false;
        }
    }
}
```

# 攻击思路

这道题目也非常一目了然，如果知道一些开发细节就能明白问题所在

题目中的password用了private的修饰词，感觉上是不可以被调用查看

但是实际上private这个修饰词只是限制了调用范围，而实际上它的内容是可以public review的

# 攻击方式

1. 直接调用web console来获取hex代码下面的值
    
    ```bash
    > await web3.eth.getStorageAt(0xec55eEBF0506d7865E50CB873C8eD38be6048842, 1)
    
    < '0x412076657279207374726f6e67207365637265742070617373776f7264203a29'
    ```
    
2. 直接把上面的hex形式的值输入unlock即可，转换成ascii的话，明文密码是`'A very strong secret password :)’`
    
    ```bash
    > await web3.utils.hexToAscii('0x412076657279207374726f6e67207365637265742070617373776f7264203a29')
    
    < 'A very strong secret password :)'
    ```
    

# Ethernuat总结

It's important to remember that marking a variable as private only prevents other contracts from accessing it. State variables marked as private and local variables are still publicly accessible.

To ensure that data is private, it needs to be encrypted before being put onto the blockchain. In this scenario, the decryption key should never be sent on-chain, as it will then be visible to anyone who looks for it. [zk-SNARKs](https://blog.ethereum.org/2016/12/05/zksnarks-in-a-nutshell/) provide a way to determine whether someone possesses a secret parameter, without ever having to reveal the parameter.

# 小白的思考

这个题很直白，但是在思考的过程中发现了一些比较有趣的事情

因为拿到hex形式的密码后，第一反应就把它转成了ascii，并且很自然地提交了，然后失败。

后来发现需要提交的是byte32的形式，也就是依然是hexi

1. 可以直接提交从slot1里面读取的hex（成功）
2. 从ascii在转换回去byte32，然后发现值竟然和原来的不一样
    1. 理论上hex→ascii→hex，只是做了一个转化，应该是一样的才对

### 和GPT了解了一下，学习了一些东西：

你提到的情况是很关键的一个点，确实，如果先使用 `hexToAscii` 转换为 ASCII 字符串，然后再用 `asciiToHex`，得出的结果可能与原来的十六进制值不一致。这是因为在这两个转换过程中，数据格式和填充方式发生了变化。

### 问题分析

### 原因：

1. **填充与编码**：
    - `web3.utils.hexToAscii` 将十六进制转换为 ASCII 时，只考虑将可读字符转换为字符串，而忽略了填充的空字节（通常是 `0x00`）。
    - 当你用 `asciiToHex` 进行逆向转换时，工具可能会自动填充不足的字节到 32 字节，这样得到的结果与原始的 `bytes32` 会有差异。
2. **自动补零**：
    - 原始的 `bytes32` 在内部存储中可能包含了多余的 `0x00` 填充字节，这些空字节在字符串转换过程中被忽略或被处理成不同的格式。

### 理解 `bytes32` 的实际存储

`bytes32` 是一个定长的 32 字节数组，因此即使字符串本身没有填满 32 字节，系统仍然会自动补齐。例如，原始密码可能在后面有一些隐式的 `0x00` 填充，而这些填充在 ASCII 转换中被忽略掉了。