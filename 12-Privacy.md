# 12-Privacy

日期: 20/08/2024
难易: 简单
漏洞小结: Storage读取
简单明了: 了解stroage结构之后读取以及变换值

# 问题描述

The creator of this contract was careful enough to protect the sensitive areas of its storage.

Unlock this contract to beat the level.

Things that might help:

- Understanding how storage works
- Understanding how parameter parsing works
- Understanding how casting works

Tips:

- Remember that metamask is just a commodity. Use another tool if it is presenting problems. Advanced gameplay could involve using remix, or your own web3 provider.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Privacy {
    bool public locked = true;
    uint256 public ID = block.timestamp;
    uint8 private flattening = 10;
    uint8 private denomination = 255;
    uint16 private awkwardness = uint16(block.timestamp);
    bytes32[3] private data;

    constructor(bytes32[3] memory _data) {
        data = _data;
    }

    function unlock(bytes16 _key) public {
        require(_key == bytes16(data[2]));
        locked = false;
    }

    /*
    A bunch of super advanced solidity algorithms...

      ,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`
      .,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,
      *.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^         ,---/V\
      `*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.    ~|__(o.o)
      ^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'  UU  UU
    */
}
```

# 攻击思路

就和题目中的提示一样，需要了解storge是如何运作的，并且要知道private只限制了范围，没法限制查看。

我们需要的data变量在index 5的位置上，但是它是一个长度为3的string array，而目标似乎data array上面的第三个变量，所以是存在storage上面的第八个data slot（index为7）

通过下面的代码可以读取到

```bash
web3.eth.getStorageAt(contractAddress, 7)
```

然后再观察代码，unlock函数里面的require条件比对的是byte16，所以需要把byte32裁切后的值输入到unlock函数调用即可

# 攻击方式

1. ~~通过`web3.eth.getStorageAt(contractAddress, 7)`读取data[2]的值~~
2. **实际上datap[2]的值在index5上面，而不是7，详见小白的思考**
    
    ```bash
    web3.eth.getStorageAt(contractAddress, 7)
    ```
    
    得到的结果如下
    
    `'0x5c271a10dc41954476115e650b1788829b2a9a5c1d1dc182be81d405fad67304’`
    
3. 裁剪获取前16位，然后当做结果传入unlock函数即可
    
    ```bash
    await contract.unlock('0x5c271a10dc41954476115e650b178882')
    ```
    

# Ethernuat总结

Nothing in the ethereum blockchain is private. The keyword private is merely an artificial construct of the Solidity language. Web3's `getStorageAt(...)` can be used to read anything from storage. It can be tricky to read what you want though, since several optimization rules and techniques are used to compact the storage as much as possible.

It can't get much more complicated than what was exposed in this level. For more, check out this excellent article by "Darius": [How to read Ethereum contract storage](https://medium.com/aigang-network/how-to-read-ethereum-contract-storage-44252c8af925)

# 小白的思考

## 关于解题思路

因为之前做过Vault这个题，查了一下数组array是如何存储的，很自然而然得就觉得index是7，结果拿到的storage值全都是0，感觉很奇怪。

和GPT探讨了一下，感觉还是蛮神奇的：

在合约中，以下变量按顺序占据了槽位：

- `bool public locked` -> 槽位 0
- `uint256 public ID` -> 槽位 1
- `uint8 private flattening` -> 槽位 2
- `uint8 private denomination` -> 槽位 2 （与`flattening`共享同一槽位，因为`uint8`可以在同一个32字节槽位中存储多个值）
- `uint16 private awkwardness` -> 槽位 2 （与前两个`uint8`共享）
- **从槽位 3 开始是 `data` 数组。**

[存储槽的使用和布局](https://www.notion.so/7fd43fb819274277a7d558858ac7a22b?pvs=21)

## 关于slice函数

从byte32到byte16，除了手动去数数之外，还可以直接使用slice这个函数，有需要记录的点，记忆一下

`slice(0, 34)`的操作是基于十六进制的字符串处理。以下是更详细的解释：

- 以`0x`开头的字符串表示这是一个十六进制数。
- 十六进制数中每两位代表一个字节，因此`bytes32`总共有64位字符（32字节）。
- 你需要截取前16个字节（即32位字符）。包括开头的`0x`，因此`slice(0, 34)`是正确的操作：

```jsx
const key = dataSlot7.slice(0, 34); // 0x + 前16字节 = 34个字符
```