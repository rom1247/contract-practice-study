# meme币介绍

meme币是指一种基于互联网“meme”（模因、网络梗、表情包或流行文化）的加密货币。它通常起源于幽默、讽刺或病毒式传播的网络现象，而不是像比特币或以太坊那样有坚实的技术创新或实际应用场景。Meme币的核心价值来自于社区共识、社交媒体炒作和文化共鸣，而不是基本面（如实用功能或技术优势）。

为什么要叫meme币这个名字？
叫“meme币”的原因直接源于“meme”这个词：

“Meme”最早由生物学家Richard Dawkins在1976年提出，指文化传播的单位（如想法、行为、风格），类似于基因的复制传播。在互联网时代，meme特指快速病毒式传播的幽默内容，比如表情包、搞笑图、梗视频。
第一个真正流行的meme币是狗狗币（Dogecoin），于2013年由两位工程师作为玩笑创建，基于热门的“Doge”柴犬meme（一张搞笑柴犬配破英文的图片），最初是为了讽刺比特币等“严肃”加密货币的投机热。
Dogecoin的意外爆火证明：一个纯靠网络meme和社区幽默驱动的币也能获得巨大关注和市值。这开启了meme币时代，后续如SHIB（柴犬币）、PEPE等都模仿这种模式。
因此，“meme币”这个名字强调了其本质：源于互联网meme文化，通过模仿、分享和病毒传播来获受欢迎，而不是技术或实用价值。

简单说，叫meme币就是因为它们像互联网meme一样——好玩、易传播、靠热度活着。投资需谨慎，大多是高风险投机品！如果想看经典例子，狗狗币就是起源鼻祖。

## meme币机制

核心机制通常通过智能合约（smart contract）实现，一般使用使用ERC20实现

### 交易税

买入税：用户从交易所购买代币时，扣除一定的比例
卖出税：用户从交易所卖出代币时，扣除一定的比例

```solidity
uint256 taxAmount = amount * taxRate / 100;
uint256 sendAmount = amount - taxAmount;
super._transfer(sender, address(this), taxAmount); // 收税
super._transfer(sender, recipient, sendAmount); // 转账

```

税收用途

- 流动性池增值：部分税收自动添加到 Uniswap/SushiSwap 等的流动性池。
- 燃烧：直接把一部分币烧掉，减少总量。
- 奖励持币人：例如 Reflection 机制，将部分税收按持币比例分配给现有持币人。
- 项目运营：用于团队资金或社区活动。

其他税：

| 税类型           | 作用         |
| ------------- | ---------- |
| Buy Tax       | 买入时扣       |
| Sell Tax      | 卖出时扣 |
| Transfer Tax  | 普通转账也扣     |
| LP Tax        | 注入流动性      |
| Burn Tax      | 直接销毁       |
| Treasury Tax  | 进金库        |
| Marketing Tax | 给项目方钱包     |

### 黑白名单

黑名单直接禁止某些地址交易，用途：防机器人，防MEV等。
白名单直接允许某些地址交易，用途：预售，内侧，免税等。

```solidity
mapping(address => bool) public blackList;
mapping(address => bool) public whiteList;

function _transfer(address sender, address recipient, uint256 amount) internal override {
    require(!blackList[sender] && !blackList[recipient], "Address banned");
    if(!whiteList[sender] && !whiteList[recipient]){
        uint256 tax = amount * taxRate / 100;
        super._transfer(sender, address(this), tax);
        amount -= tax;
    }
    super._transfer(sender, recipient, amount);
}

```

### 持币量控制

单次最大交易量（MaxTx）：防止单笔交易大量控制。
单地址持币上限（MaxWallet）：防止单地址大量持币，避免大户屯币，保持社区分散持有。
动态限制：根据链上活动动态调整，比如启动期严格限制，成熟期放宽。

```solidity
uint256 public maxWallet = totalSupply / 50; // 每个地址最多持有2%
uint256 public maxTx = totalSupply / 100;    // 单笔交易最多1%

function _beforeTokenTransfer(address from, address to, uint256 amount) internal override {
    require(amount <= maxTx, "Exceeds max tx limit");
    require(balanceOf(to) + amount <= maxWallet, "Exceeds max wallet limit");
}

```

### 治理机制

治理就是让社区参与决策，如：代币投票、提案等。
