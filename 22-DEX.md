# 22-DEX

日期: 23/08/2024
难易: 困难
漏洞小结: 交换漏洞
简单明了: 来回倒，token越来越多

# 问题描述

The goal of this level is for you to hack the basic [DEX](https://en.wikipedia.org/wiki/Decentralized_exchange) contract below and steal the funds by price manipulation.

You will start with 10 tokens of `token1` and 10 of `token2`. The DEX contract starts with 100 of each token.

You will be successful in this level if you manage to drain all of at least 1 of the 2 tokens from the contract, and allow the contract to report a "bad" price of the assets.

### Quick note

Normally, when you make a swap with an ERC20 token, you have to `approve` the contract to spend your tokens for you. To keep with the syntax of the game, we've just added the `approve` method to the contract itself. So feel free to use `contract.approve(contract.address, <uint amount>)` instead of calling the tokens directly, and it will automatically approve spending the two tokens by the desired amount. Feel free to ignore the `SwappableToken` contract otherwise.

Things that might help:

- How is the price of the token calculated?
- How does the `swap` method work?
- How do you `approve` a transaction of an ERC20?
- Theres more than one way to interact with a contract!
- Remix might help
- What does "At Address" do?

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/IERC20.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import "openzeppelin-contracts-08/access/Ownable.sol";

contract Dex is Ownable {
    address public token1;
    address public token2;

    constructor() {}

    function setTokens(address _token1, address _token2) public onlyOwner {
        token1 = _token1;
        token2 = _token2;
    }

    function addLiquidity(address token_address, uint256 amount) public onlyOwner {
        IERC20(token_address).transferFrom(msg.sender, address(this), amount);
    }

    function swap(address from, address to, uint256 amount) public {
        require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
        require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
        uint256 swapAmount = getSwapPrice(from, to, amount);
        IERC20(from).transferFrom(msg.sender, address(this), amount);
        IERC20(to).approve(address(this), swapAmount);
        IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
    }

    function getSwapPrice(address from, address to, uint256 amount) public view returns (uint256) {
        return ((amount * IERC20(to).balanceOf(address(this))) / IERC20(from).balanceOf(address(this)));
    }

    function approve(address spender, uint256 amount) public {
        SwappableToken(token1).approve(msg.sender, spender, amount);
        SwappableToken(token2).approve(msg.sender, spender, amount);
    }

    function balanceOf(address token, address account) public view returns (uint256) {
        return IERC20(token).balanceOf(account);
    }
}

contract SwappableToken is ERC20 {
    address private _dex;

    constructor(address dexInstance, string memory name, string memory symbol, uint256 initialSupply)
        ERC20(name, symbol)
    {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
    }

    function approve(address owner, address spender, uint256 amount) public {
        require(owner != _dex, "InvalidApprover");
        super._approve(owner, spender, amount);
    }
}
```

# 攻击思路

思路非常得直接，就是通过交换，然后让自己手头上币越来越多，最后一次控制数量，然后把其中一种币给掏空

操作方式也很简单，但是对于现阶段的我来说，细节上面有点复杂。

# 攻击方式

- 对币进行交换
    
    ```solidity
        function getSwapPrice(address from, address to, uint256 amount) public view returns (uint256) {
            return ((amount * IERC20(to).balanceOf(address(this))) / IERC20(from).balanceOf(address(this)));
        }
    ```
    
    1. 用token1换取token2，10个全换
        1. 根据公式，玩家换到10个token2，玩家剩余0个token1，20个token2
        2. 合约剩余110个token1，90个token2
    2. 用token2换取token1，20个全换
        1. 根据规则，玩家换到24个token1，玩家剩余24个token1，0个token2
        2. 合约剩余86个token1，110个token2
    3. 用token1换取token2，24个全换
        1. 根据规则，玩家换到30个token2，玩家剩余0个token1，30个token2
        2. 合约剩余110个token1，80个token2
    4. 用token2换取token1，30个全换
        1. 根据规则，玩家换到41个token1，玩家剩余41个token1，0个token2
        2. 合约剩余69个token1，110个token2
    5. 用token1换取token2，41个全换
        1. 根据规则，玩家换到65个token2，玩家剩余0个token1，65个token2
        2. 合约剩余110个token1，45个token2
    6. 用token2换取token1，换**45个（不能全换，不然不够了，算出来45个）**
        1. 根据规则，玩家换到110个token1，玩家剩余110个token1，0个token2
        2. 合约剩余0个token1，110个token2
        3. 目标达成
- 需要把IERC20-也就是token和IDEX的函数都写出来
- 需要全部部署，在token测approve，攻击函数才能开始攻击

# Ethernuat总结

The integer math portion aside, getting prices or any sort of data from any single source is a massive attack vector in smart contracts.

You can clearly see from this example, that someone with a lot of capital could manipulate the price in one fell swoop, and cause any applications relying on it to use the wrong price.

The exchange itself is decentralized, but the price of the asset is centralized, since it comes from 1 dex. However, if we were to consider tokens that represent actual assets rather than fictitious ones, most of them would have exchange pairs in several dexes and networks. This would decrease the effect on the asset's price in case a specific dex is targeted by an attack like this.

[Oracles](https://betterprogramming.pub/what-is-a-blockchain-oracle-f5ccab8dbd72?source=friends_link&sk=d921a38466df8a9176ed8dd767d8c77d) are used to get data into and out of smart contracts.

[Chainlink Data Feeds](https://docs.chain.link/docs/get-the-latest-price) are a secure, reliable, way to get decentralized data into your smart contracts. They have a vast library of many different sources, and also offer [secure randomness](https://docs.chain.link/docs/chainlink-vrf), ability to make [any API call](https://docs.chain.link/docs/make-a-http-get-request), [modular oracle network creation](https://docs.chain.link/docs/architecture-decentralized-model), [upkeep, actions, and maintainance](https://docs.chain.link/docs/kovan-keeper-network-beta), and unlimited customization.

[Uniswap TWAP Oracles](https://docs.uniswap.org/contracts/v2/concepts/core-concepts/oracles) relies on a time weighted price model called [TWAP](https://en.wikipedia.org/wiki/Time-weighted_average_price#). While the design can be attractive, this protocol heavily depends on the liquidity of the DEX protocol, and if this is too low, prices can be easily manipulated.

Here is an example of getting the price of Bitcoin in USD from a Chainlink data feed (on the Sepolia testnet):

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.7;

import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

contract PriceConsumerV3 {
    AggregatorV3Interface internal priceFeed;

    /**
     * Network: Sepolia
     * Aggregator: BTC/USD
     * Address: 0x1b44F3514812d835EB1BDB0acB33d3fA3351Ee43
     */
    constructor() {
        priceFeed = AggregatorV3Interface(
            0x1b44F3514812d835EB1BDB0acB33d3fA3351Ee43
        );
    }

    /**
     * Returns the latest price.
     */
    function getLatestPrice() public view returns (int) {
        // prettier-ignore
        (
            /* uint80 roundID */,
            int price,
            /*uint startedAt*/,
            /*uint timeStamp*/,
            /*uint80 answeredInRound*/
        ) = priceFeed.latestRoundData();
        return price;
    }
}

```

[Try it on Remix](https://remix.ethereum.org/#url=https://docs.chain.link/samples/PriceFeeds/PriceConsumerV3.sol)

Check the Chainlink feed [page](https://data.chain.link/ethereum/mainnet/crypto-usd/btc-usd) to see that the price of Bitcoin is queried from up to 31 different sources.

You can check also, the [list](https://docs.chain.link/data-feeds/price-feeds/addresses/) all Chainlink price feeds addresses.

# 小白的思考

这个题目贼拉长，需要稳定的思考，其实没有多难，试一试就知道了

- 数学逻辑要明细，拿个笔仔细谢谢算算
- ERC20的关系要理清
    - 在transferFrom之前一定先要approve
        - 要去token下面approve，并且需要赋予给攻击合约，这样攻击才能开始
- 查询合约的某个token余额
    1. 先查看合约的实力address，然后调出token地址
    2. 然后去token地址，用erc20的标准方式`balanceOf`去查看余额