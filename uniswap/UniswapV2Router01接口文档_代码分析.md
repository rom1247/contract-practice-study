# UniswapV2Router01 接口文档（含代码分析）

`UniswapV2Router01` 是 Uniswap V2 协议的外围路由合约，旨在简化与核心合约（Pair 和 Factory）的交互。它提供了添加/移除流动性、代币交换（Swap）以及价格查询等功能的便捷接口。

- 统一到期保护: 绝大多数对外方法均受 `ensure(deadline)` 修饰符保护，若 `deadline < block.timestamp` 则回退。
- WETH 交互约束: 合约仅通过 `receive()` 接受来自 WETH 合约的 ETH。

## 状态变量 (State Variables)

### factory

```solidity
address public immutable factory;
```

- 描述: Uniswap V2 工厂合约地址
- 访问方式: `router.factory()`

### WETH

```solidity
address public immutable WETH;
```

- 描述: WETH (Wrapped Ether) 合约地址
- 访问方式: `router.WETH()`

---

## 流动性管理 (Liquidity Management)

### addLiquidity

添加两个 ERC-20 代币的流动性。

```solidity
function addLiquidity(
    address tokenA,
    address tokenB,
    uint amountADesired,
    uint amountBDesired,
    uint amountAMin,
    uint amountBMin,
    address to,
    uint deadline
) external returns (uint amountA, uint amountB, uint liquidity);
```

- 参数:
  - `tokenA`: 代币 A 地址
  - `tokenB`: 代币 B 地址
  - `amountADesired`: 期望存入 A 的数量
  - `amountBDesired`: 期望存入 B 的数量
  - `amountAMin`: 最小可接受 A 数量
  - `amountBMin`: 最小可接受 B 数量
  - `to`: 流动性代币接收地址
  - `deadline`: 过期时间戳
- 返回:
  - `amountA`: 实际存入 A 的数量
  - `amountB`: 实际存入 B 的数量
  - `liquidity`: 铸造的 LP 数量
- 代码分析:
  - 调用私有 `_addLiquidity` 依据储备等比计算最优配对数量
  - 计算配对地址 `pairFor`，其实就是获取流动性合约地址，该地址在\_addLiquidity 方法使用 create2 创建的
  - 将两种代币从调用者转至 Pair
  - 由 Pair 合约铸造 LP 并转给 `to`
  - 铸造逻辑说明：
    - 首次添加（储备为 0）：
      $$liquidity = \sqrt{amountA \cdot amountB} - MINIMUM\_LIQUIDITY$$
      其中 `MINIMUM_LIQUIDITY` 会被永久锁定到零地址，用于防止价格操纵
    - 非首次添加（储备非 0）：
      $$liquidity = \min\!\left( \frac{amountA \cdot totalSupply}{reserveA},\ \frac{amountB \cdot totalSupply}{reserveB} \right)$$
      以当前储备比例为准，按两侧贡献的相对份额计算铸造数量，取较小值以保持价格不变
    - 铸造后，Pair 更新储备为最新余额，并返回本次铸造的 `liquidity`，LP 发送给 `to`
  - 变量说明:
    - `reserveA`/`reserveB`: 铸造前交易对两侧的储备
    - `amountA`/`amountB`: 本次新增的两侧存入数量
    - `totalSupply`: 当前 LP 总供应量
    - `MINIMUM_LIQUIDITY`: 最小流动性常量，被铸造后永久锁定
    - `liquidity`: 本次铸造的 LP 数量

### addLiquidityETH

添加 ERC-20 代币与 ETH 的流动性。

```solidity
function addLiquidityETH(
    address token,
    uint amountTokenDesired,
    uint amountTokenMin,
    uint amountETHMin,
    address to,
    uint deadline
) external payable returns (uint amountToken, uint amountETH, uint liquidity);
```

- 参数:
  - `token`: 代币地址
  - `amountTokenDesired`: 期望代币数量
  - `amountTokenMin`: 最小可接受代币数量
  - `amountETHMin`: 最小可接受 ETH 数量
  - `to`: LP 接收地址
  - `deadline`: 过期时间戳
- 返回:
  - `amountToken`: 实际存入代币数量
  - `amountETH`: 实际存入 ETH 数量
  - `liquidity`: 铸造的 LP 数量
- 代码分析:
  - 通过 `_addLiquidity(token, WETH, ...)` 计算最优配对
  - 计算配对地址，转入代币
  - 将 ETH 包装为 WETH 并转入 Pair
  - 在 Pair 上铸造 LP
  - 多余 ETH 原路退回（dust 退款）
  - 细节解释：
    - 先计算 `(amountToken, amountETH)`：内部调用 `_addLiquidity` 读取 `token/WETH` 储备并按比例计算最优匹配数量，保证价格不偏移
    - 将 `amountToken` 从调用者转入 Pair：`TransferHelper.safeTransferFrom(token, msg.sender, pair, amountToken)`
    - 将 `amountETH` 包装为 WETH 并转入 Pair：`IWETH(WETH).deposit{value: amountETH}()` 后 `transfer(pair, amountETH)`
    - 由 Pair 执行 `mint(to)`：按当前储备与贡献计算 LP 并发送给 `to`
    - 退款机制：若 `msg.value > amountETH`，多余的 ETH 原路退回调用者，避免多付
      $$\text{refund} = \max(0,\ \text{msg.value} - amountETH)$$
  - 数学推导（与 `_addLiquidity` 一致，仅将 B 侧替换为 WETH）：
    - 储备比例：
      $$R_{\text{token}},\ R_{\text{WETH}},\quad R_{\text{token}}:R_{\text{WETH}}$$
    - 最优配比：
      $$
      \mathrm{token}^{*} = \frac{\mathrm{WETH}_{\text{desired}} \cdot R_{\text{token}}}{R_{\text{WETH}}}
      $$
    - 条件选择：
      $$
      \begin{cases}
      (\mathrm{token}_{\text{desired}},\ \mathrm{WETH}^{*}), & \mathrm{WETH}^{*} \le \mathrm{WETH}_{\text{desired}} \\\\
      (\mathrm{token}^{*},\ \mathrm{WETH}_{\text{desired}}), & \text{otherwise}
      \end{cases}
      $$
    - 滑点保护：
      $$\mathrm{token} \ge \mathrm{token}_{\min},\ \mathrm{WETH} \ge \mathrm{ETH}_{\min}
  - 变量说明:
    - `R_token`/`R_WETH`: `token/WETH` 交易对的当前储备
    - `token_desired`/`WETH_desired`: 用户希望存入的 `token/ETH` 数量
    - `token_min`/`ETH_min`: 最小可接受数量（滑点保护）
    - `amountToken`/`amountETH`: 实际采用的两侧存入数量（来自最优匹配与最小值校验）
    - `refund`: 多余 ETH 退款数量

### removeLiquidity

移除两个 ERC-20 代币的流动性。

```solidity
function removeLiquidity(
    address tokenA,
    address tokenB,
    uint liquidity,
    uint amountAMin,
    uint amountBMin,
    address to,
    uint deadline
) external returns (uint amountA, uint amountB);
```

- 参数:
  - `tokenA`: 代币 A 地址
  - `tokenB`: 代币 B 地址
  - `liquidity`: 待销毁 LP 数量
  - `amountAMin`: 最小可接受 A 数量
  - `amountBMin`: 最小可接受 B 数量
  - `to`: 资产接收地址
  - `deadline`: 过期时间戳
- 返回:
  - `amountA`: 获得的 A 数量
  - `amountB`: 获得的 B 数量
- 代码分析:
  - 计算配对地址并将 LP 从调用者转入 Pair
  - 由 Pair `burn` 赎回两种代币
  - 根据 token 排序映射至 `(amountA, amountB)`
  - 滑点保护检查最小金额

### removeLiquidityETH

移除 ERC-20 代币与 ETH 的流动性。

```solidity
function removeLiquidityETH(
    address token,
    uint liquidity,
    uint amountTokenMin,
    uint amountETHMin,
    address to,
    uint deadline
) external returns (uint amountToken, uint amountETH);
```

- 参数:
  - `token`: 代币地址
  - `liquidity`: 待销毁 LP 数量
  - `amountTokenMin`: 最小可接受代币数量
  - `amountETHMin`: 最小可接受 ETH 数量
  - `to`: 资产接收地址
  - `deadline`: 过期时间戳
- 返回:
  - `amountToken`: 获得的代币数量
  - `amountETH`: 获得的 ETH 数量
- 代码分析:
  - 先调用 `removeLiquidity(token, WETH, ...)` 取回代币与 WETH
  - 将代币转给 `to`
  - 将 WETH 解包为 ETH 并发送给 `to`

### removeLiquidityWithPermit

使用签名移除 ERC-20/ ERC-20 流动性。

```solidity
function removeLiquidityWithPermit(
    address tokenA,
    address tokenB,
    uint liquidity,
    uint amountAMin,
    uint amountBMin,
    address to,
    uint deadline,
    bool approveMax,
    uint8 v,
    bytes32 r,
    bytes32 s
) external returns (uint amountA, uint amountB);
```

- 参数:
  - `tokenA`: 代币 A 地址
  - `tokenB`: 代币 B 地址
  - `liquidity`: 待销毁的 LP 数量
  - `amountAMin`: 最小可接受的代币 A 数量
  - `amountBMin`: 最小可接受的代币 B 数量
  - `to`: 接收赎回资产的地址
  - `deadline`: 过期时间戳（到期后交易回退）
  - `approveMax`: 是否授权最大值（`true` 则为 `uint(-1)`）
  - `v,r,s`: ECDSA 签名参数
- 返回:
  - `amountA`: 赎回得到的代币 A 数量
  - `amountB`: 赎回得到的代币 B 数量
- 代码分析:
  - 通过 `permit` 使用离线签名授权路由合约移动 LP
  - 授权数量为 `approveMax ? uint(-1) : liquidity`
  - 完成后复用 `removeLiquidity(...)` 执行赎回

### removeLiquidityETHWithPermit

使用签名移除 ERC-20/ETH 流动性。

```solidity
function removeLiquidityETHWithPermit(
    address token,
    uint liquidity,
    uint amountTokenMin,
    uint amountETHMin,
    address to,
    uint deadline,
    bool approveMax,
    uint8 v,
    bytes32 r,
    bytes32 s
) external returns (uint amountToken, uint amountETH);
```

- 参数:
  - `token`: ERC-20 代币地址
  - `liquidity`: 待销毁的 LP 数量
  - `amountTokenMin`: 最小可接受代币数量
  - `amountETHMin`: 最小可接受 ETH 数量
  - `to`: 接收赎回资产的地址
  - `deadline`: 过期时间戳
  - `approveMax`: 是否授权最大值
  - `v,r,s`: ECDSA 签名参数
- 返回:
  - `amountToken`: 赎回得到的代币数量
  - `amountETH`: 赎回得到的 ETH 数量
- 代码分析:
  - 对 `token/WETH` 对的 LP 使用 `permit` 授权
  - 调用 `removeLiquidityETH(...)` 完成赎回与 WETH 解包

---

## 代币交换 (Swapping)

### swapExactTokensForTokens

使用精确的输入数量交换尽可能多的输出代币。

```solidity
function swapExactTokensForTokens(
    uint amountIn,
    uint amountOutMin,
    address[] calldata path,
    address to,
    uint deadline
) external returns (uint[] memory amounts);
```

- 参数:
  - `amountIn`: 精确输入数量
  - `amountOutMin`: 最小输出数量
  - `path`: 交易路径（如 `[TokenA, WETH, TokenB]`）
    - **来源**: 通常由前端 SDK 根据链上 Factory 状态计算得出。
    - **有效性**: 路由合约不显式校验路径存在性；若某跳 Pair 未创建或代币不存在，后续 `getAmountsOut` 读取储备或转账时将导致交易 revert。
  - `to`: 接收地址
  - `deadline`: 过期时间戳
- 返回:
  - `amounts`: 每跳的数量数组
- 代码分析:
  - 计算各跳输出 `getAmountsOut`（若 Pair 不存在此处会失败）
  - 校验最终输出不小于最小值
  - 将输入代币转至首个配对
  - 调用私有 `_swap` 执行多跳交换

### swapTokensForExactTokens

使用尽可能少的输入交换精确的输出代币数量。

```solidity
function swapTokensForExactTokens(
    uint amountOut,
    uint amountInMax,
    address[] calldata path,
    address to,
    uint deadline
) external returns (uint[] memory amounts);
```

- 参数:
  - `amountOut`: 期望的精确输出数量
  - `amountInMax`: 输入代币的最大可用数量
  - `path`: 交换路径（代币地址数组）
  - `to`: 接收地址
  - `deadline`: 过期时间戳
- 返回:
  - `amounts`: 每一跳的输入/输出数量数组（`amounts[0]` 为实际输入）
- 代码分析:
  - 计算所需的输入数组 `getAmountsIn`
  - 校验首个输入不超过最大值
  - 将输入代币转至首个配对
  - 调用 `_swap` 执行交换

### swapExactETHForTokens

使用精确的 ETH 输入交换尽可能多的代币。

```solidity
function swapExactETHForTokens(
    uint amountOutMin,
    address[] calldata path,
    address to,
    uint deadline
) external payable returns (uint[] memory amounts);
```

- 参数:
  - `amountOutMin`: 最小可接受的最终输出代币数量
  - `path`: 交换路径（首项需为 WETH）
  - `to`: 接收地址
  - `deadline`: 过期时间戳
- 返回:
  - `amounts`: 每一跳的输入/输出数量数组（`amounts[0]` 等于 `msg.value` 包装为 WETH 后的数量）
- 代码分析:
  - 路径首项需为 WETH
  - 计算输出数组并校验最小值
  - 将 ETH 包装为 WETH 并转至首个配对
  - 调用 `_swap` 执行交换

### swapTokensForExactETH

使用尽可能少的代币输入交换精确的 ETH 输出。

```solidity
function swapTokensForExactETH(
    uint amountOut,
    uint amountInMax,
    address[] calldata path,
    address to,
    uint deadline
) external returns (uint[] memory amounts);
```

- 参数:
  - `amountOut`: 期望的精确 ETH 输出数量
  - `amountInMax`: 输入代币的最大可用数量
  - `path`: 交换路径（末项需为 WETH）
  - `to`: 接收地址
  - `deadline`: 过期时间戳
- 返回:
  - `amounts`: 每一跳的输入/输出数量数组（末项为 WETH 数量）
- 代码分析:
  - 路径末项需为 WETH
  - 计算所需输入并做最大值校验
  - 将输入转至首配对，执行 `_swap`
  - 将 WETH 解包为 ETH 并发送给 `to`

### swapExactTokensForETH

使用精确的代币输入交换尽可能多的 ETH。

```solidity
function swapExactTokensForETH(
    uint amountIn,
    uint amountOutMin,
    address[] calldata path,
    address to,
    uint deadline
) external returns (uint[] memory amounts);
```

- 参数:
  - `amountIn`: 精确的输入代币数量
  - `amountOutMin`: 最小可接受的最终 ETH 数量
  - `path`: 交换路径（末项需为 WETH）
  - `to`: 接收地址
  - `deadline`: 过期时间戳
- 返回:
  - `amounts`: 每一跳的输入/输出数量数组（末项为 WETH 数量）
- 代码分析:
  - 路径末项需为 WETH
  - 计算输出并校验最小值
  - 将输入转至首配对并执行 `_swap`
  - 解包 WETH 并发送 ETH

### swapETHForExactTokens

使用尽可能少的 ETH 输入交换精确的代币输出。

```solidity
function swapETHForExactTokens(
    uint amountOut,
    address[] calldata path,
    address to,
    uint deadline
) external payable returns (uint[] memory amounts);
```

- 参数:
  - `amountOut`: 期望的精确代币输出数量
  - `path`: 交换路径（首项需为 WETH）
  - `to`: 接收地址
  - `deadline`: 过期时间戳
- 返回:
  - `amounts`: 每一跳的输入/输出数量数组（`amounts[0]` 为包装成 WETH 的 ETH 数量）
- 代码分析:
  - 路径首项需为 WETH
  - 计算所需 ETH 并校验不超过 `msg.value`
  - 包装 ETH 为 WETH 并转至首配对
  - 执行 `_swap`，多余 ETH 退回调用者

---

## 工具函数 (Utilities)

### quote

给定资产 A 的数量和储备量，计算等值的资产 B 数量（基于当前价格，不含滑点）。

```solidity
function quote(uint amountA, uint reserveA, uint reserveB) public pure returns (uint amountB);
```

- 参数:
  - `amountA`: 输入的代币 A 数量
  - `reserveA`: 交易对中代币 A 当前储备
  - `reserveB`: 交易对中代币 B 当前储备
- 返回:
  - `amountB`: 等值计算得到的代币 B 数量（不含手续费）
- 代码分析:
  - 直接代理库函数 `UniswapV2Library.quote`
- 数学推导:
  - 基于价格比如下比例：
    $$reserveA:reserveB$$
    $$amountB = \frac{amountA \cdot reserveB}{reserveA}$$
- 变量说明:
  - `reserveA`: 交易对中 A 代币当前储备
  - `reserveB`: 交易对中 B 代币当前储备
  - `amountA`: 输入的 A 代币数量
  - `amountB`: 等值计算得到的 B 代币数量（不含手续费）

### getAmountOut

给定输入金额，计算扣除手续费后的最大输出金额。

```solidity
function getAmountOut(uint amountIn, uint reserveIn, uint reserveOut) public pure returns (uint amountOut);
```

- 参数:
  - `amountIn`: 输入的代币数量
  - `reserveIn`: 输入侧的当前储备
  - `reserveOut`: 输出侧的当前储备
- 返回:
  - `amountOut`: 扣除手续费后可获得的最大输出数量
- 代码分析:
  - 直接代理库函数 `UniswapV2Library.getAmountOut`
- 数学推导:
  - 常数乘积模型保持不变量：
    $$x \cdot y = k$$
  - 输入输出与手续费：
    $$x=reserveIn,\quad y=reserveOut,\quad f=\frac{997}{1000}$$
  - 有效输入量：
    $$\Delta x_{\text{eff}} = amountIn \cdot f$$
  - 交换不变量展开：
    $$(x + \Delta x_{\text{eff}})(y - \Delta y) = x \cdot y$$
  - 解得输出：
    $$
    = \frac{amountIn \cdot 997 \cdot reserveOut}{reserveIn \cdot 1000 + amountIn \cdot 997}
    $$
- 变量说明:
  - `x`/`reserveIn`: 交易对中输入代币侧的当前储备
  - `y`/`reserveOut`: 交易对中输出代币侧的当前储备
  - `k`: 不变量 $x \cdot y$
  - `f`: 手续费系数，0.3% 手续费对应 $f=997/1000$
  - `Δx_eff`: 有效输入数量，扣除手续费后的净输入
  - `amountIn`: 用户提供的原始输入数量
  - `amountOut`: 计算得到的最大输出数量

### getAmountIn

给定目标输出金额，计算扣除手续费后所需的最小输入金额。

```solidity
function getAmountIn(uint amountOut, uint reserveIn, uint reserveOut) public pure returns (uint amountIn);
```

- 参数:
  - `amountOut`: 期望的精确输出数量
  - `reserveIn`: 输入侧的当前储备
  - `reserveOut`: 输出侧的当前储备
- 返回:
  - `amountIn`: 需要的最小输入数量（通常向上取整）
- 代码分析:
  - 这是应该调用库函数 `getAmountIn`；当前代码调用的是 `getAmountOut`是错误的，已在 02 版本修复
  - 这与函数语义不一致，可能为实现错误或拼写失误，需注意
- 数学推导:
  - 同样基于交换不变量：
    $$(x + \Delta x_{\text{eff}})(y - amountOut) = x \cdot y$$
    其中有效输入：
    $$\Delta x_{\text{eff}} = amountIn \cdot \frac{997}{1000}$$
  - 解得最小输入：
    $$amountIn = \left\lceil \frac{reserveIn \cdot amountOut \cdot 1000}{(reserveOut - amountOut) \cdot 997} \right\rceil$$
  - 实现通常采用整数除法，需注意向上取整以满足保守估计
- 变量说明:
  - `amountOut`: 目标输出数量
  - `amountIn`: 所需的最小输入数量（向上取整）
  - `reserveIn`/`reserveOut`: 当前交易对两侧储备
  - `Δx_eff`: 有效输入数量（扣费后）
  - `ceil(·)`: 数学上的向上取整运算，确保满足不变量与保守估计

### getAmountsOut

计算给定输入金额在指定路径下的后续输出金额。

```solidity
function getAmountsOut(uint amountIn, address[] memory path) public view returns (uint[] memory amounts);
```

- 参数:
  - `amountIn`: 路径首个代币的输入数量
  - `path`: 交换路径（代币地址数组）
- 返回:
  - `amounts`: 每一跳的输入/输出数量数组（最后一项为最终输出）
- 代码分析:
  - 直接代理库函数 `UniswapV2Library.getAmountsOut`
  - 路径长度需大于等于 2，否则抛出异常
  - 路径中每个地址必须是已部署的合约地址，否则抛出异常
  - 遍历路径，调用`UniswapV2Library.getAmountOut` 计算每一跳的输出金额，更新输入金额为下一跳的输入

### getAmountsIn

计算获得指定输出金额所需的路径输入金额。

```solidity
function getAmountsIn(uint amountOut, address[] memory path) public view returns (uint[] memory amounts);
```

- 参数:
  - `amountOut`: 路径末端期望的精确输出数量
  - `path`: 交换路径（代币地址数组）
- 返回:
  - `amounts`: 每一跳的输入/输出数量数组（首项为所需输入）
- 代码分析:
  - 直接代理库函数 `UniswapV2Library.getAmountsIn`
  - 路径长度需大于等于 2，否则抛出异常
  - 路径中每个地址必须是已部署的合约地址，否则抛出异常
  - 遍历路径，调用`UniswapV2Library.getAmountIn` 计算每一跳的输入金额，更新输出金额为下一跳的输出

---

## 内部方法 (Internal Helpers)

### \_addLiquidity

```solidity
function _addLiquidity(
    address tokenA,
    address tokenB,
    uint amountADesired,
    uint amountBDesired,
    uint amountAMin,
    uint amountBMin
) private returns (uint amountA, uint amountB);
```

- 参数:
  - `tokenA`/`tokenB`: 代币地址对
  - `amountADesired`/`amountBDesired`: 用户期望的两侧存入数量
  - `amountAMin`/`amountBMin`: 滑点保护的最小可接受数量
- 返回:
  - `amountA`/`amountB`: 按最优匹配与最小保护计算出的实际存入数量
- 代码分析:
  - 若配对不存在则在工厂中创建
  - 读取储备，根据等比计算最优存入数量
  - 使用 `quote` 依据当前储备比例计算 `amountBOptimal` 或 `amountAOptimal`
  - 根据最优值与期望值选择实际 `(A,B)`，并做最小值校验
  - 对最优值做最小数量保护检查
- 数学推导:
- 当前储备与价格比：
  $$R_A, R_B,\quad R_A:R_B$$
- 期望的最优配比为
  $$A^{*} = \frac{B_{\text{desired}} \cdot R_A}{R_B}$$
- 滑点保护要求：
  $$A \ge A_{\min},\ B \ge B_{\min}$$
- 变量说明:
  - `R_A`/`R_B`: 交易对中 A/B 代币当前储备
  - `A_desired`/`B_desired`: 用户期望存入的 A/B 数量
  - `A*`/`B*`: 按储备比例计算的最优匹配数量
  - `A_min`/`B_min`: 用户设置的最小可接受数量（滑点保护）
  - `(A,B)`: 实际采用的两侧存入数量（依据条件选择）

### \_swap

```solidity
function _swap(uint[] memory amounts, address[] memory path, address _to) private;
```

- 参数:
  - `amounts`: 每一跳的预计算输入/输出数量数组
  - `path`: 交换路径（代币地址数组）
  - `_to`: 最后一跳的实际接收地址
- 代码分析:
  - **核心逻辑**: 遍历路径，逐跳执行交换，将输出直接发送到下一跳配对地址。
  - **逐行解析**:
    1. 循环遍历路径 `i` 从 0 到 `path.length - 2`（即遍历每一对 Token）。
    2. 获取当前跳的输入 `input` 和输出 `output` 代币地址。
    3. `sortTokens`: 确定 `token0`，以便后续判断 `amount0Out` 还是 `amount1Out` 为非零值。
    4. `amountOut`: 从预计算数组 `amounts` 中取出下一跳应获得的输出数量。
    5. **输出分配**:
       - 若 `input == token0`，则 `(0, amountOut)`（输出为 `token1`）。
       - 若 `input == token1`，则 `(amountOut, 0)`（输出为 `token0`）。
    6. **接收地址判断**:
       - 若不是最后一跳（`i < path.length - 2`）：计算下一跳的配对地址 `pairFor(..., output, path[i+2])` 作为接收方。这避免了代币回到 Router 再转发，节省 Gas。
       - 若是最后一跳：使用传入的 `_to` 地址。
    7. **执行交换**:
       - 调用 `IUniswapV2Pair(pair).swap(...)`。
       - 最后一个参数 `new bytes(0)` 表示不执行闪电贷回调。
- **注意**: 该函数假设首个配对已经收到了输入代币（由调用者预先 `transfer` 或 `deposit`）。

---

## 其他

- 构造函数: 设置 `factory` 与 `WETH`
- `receive()`: 仅接受来自 WETH 合约的回退 ETH
