# 18-MagicNumber

日期: 22/08/2024
难易: 超纲
漏洞小结: Opcodes相关
简单明了: 不超过10个操作码生成合约

# 问题描述

To solve this level, you only need to provide the Ethernaut with a `Solver`, a contract that responds to `whatIsTheMeaningOfLife()` with the right number.

Easy right? Well... there's a catch.

The solver's code needs to be really tiny. Really reaaaaaallly tiny. Like freakin' really really itty-bitty tiny: 10 opcodes at most.

Hint: Perhaps its time to leave the comfort of the Solidity compiler momentarily, and build this one by hand O_o. That's right: Raw EVM bytecode.

Good luck!

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MagicNum {
    address public solver;

    constructor() {}

    function setSolver(address _solver) public {
        solver = _solver;
    }

    /*
    ____________/\\\_______/\\\\\\\\\_____        
     __________/\\\\\_____/\\\///////\\\___       
      ________/\\\/\\\____\///______\//\\\__      
       ______/\\\/\/\\\______________/\\\/___     
        ____/\\\/__\/\\\___________/\\\//_____    
         __/\\\\\\\\\\\\\\\\_____/\\\//________   
          _\///////////\\\//____/\\\/___________  
           ___________\/\\\_____/\\\\\\\\\\\\\\\_ 
            ___________\///_____\///////////////__
    */
}
```

# 攻击思路

又是一个超越认知的题目，就当做学习吧

参照小白的思考，还好有靠GPT了

https://solidity-by-example.org/app/simple-bytecode-contract/ 

**Simple Bytecode Contract**也给了例子，基本上和解法一样，只是return的值不同

# 攻击方式

```solidity
contract AttackMagicNum{
    constructor(MagicNum target){
        bytes memory bytecode = hex"69602a60005260206000f3600052600a6016f3";
        address addr;

        assembly {
            // create(value, offset, size)
            addr := create(0, add(bytecode, 0x20), 0x13)
        }
        require(addr != address(0));

        target.setSolver(addr);
    }
}
```

# Ethernuat总结

Congratulations! If you solved this level, consider yourself a Master of the Universe.

Go ahead and pierce a random object in the room with your Magnum look. Now, try to move it from afar; Your telekinesis habilities might have just started working.

# 小白的思考

这个有点底层了，类似以前做溢出的攻击一样

看了https://solidity-by-example.org/app/simple-bytecode-contract/ 以及GPT的回答，等到有必要研究底层代码的时候再研究吧

目前要分清runtime code以及bytecode的区别