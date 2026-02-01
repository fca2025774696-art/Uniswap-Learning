# 📚 Uniswap V2 学习总结报告

## 一、Swap（兑换模块）

### 1.1 核心概念
**兑换是用户最常用的功能**，用一定数量的代币A换取代币B。Uniswap V2 支持单跳和多跳兑换，自动处理最优路径。

### 1.2 两种兑换模式

#### **1.2.1 精确输入兑换** (`swapExactTokensForTokens`)
- **用户关注点**："我有100 DAI，最多能得多少ETH？"
- **特点**：用户指定输入数量，获得最大可能输出
- **安全机制**：设置 `amountOutMin` 防止滑点过大

```solidity
// 示例：用100 DAI兑换ETH（通过Router）
address[] memory path = [DAI, WETH];
uint amountIn = 100 * 1e18;  // 100 DAI
uint amountOutMin = 0.05 * 1e18;  // 最少得到0.05 ETH

router.swapExactTokensForTokens(
    amountIn,
    amountOutMin,
    path,
    msg.sender,
    block.timestamp + 300
);
```

#### **1.2.2 精确输出兑换** (`swapTokensForExactTokens`)
- **用户关注点**："我想要1 ETH，最少需要多少DAI？"
- **特点**：用户指定输出数量，系统计算最小输入
- **安全机制**：设置 `amountInMax` 防止成本过高

```solidity
// 示例：获得1 ETH，最多支付2500 DAI
address[] memory path = [DAI, WETH];
uint amountOut = 1 * 1e18;  // 1 ETH
uint amountInMax = 2500 * 1e18;  // 最多支付2500 DAI

router.swapTokensForExactTokens(
    amountOut,
    amountInMax,
    path,
    msg.sender,
    block.timestamp + 300
);
```

### 1.3 数学公式（核心）
```solidity
// 带手续费的兑换公式（0.3%手续费）
amountOut = (amountIn * 997 * reserveOut) / (reserveIn * 1000 + amountIn * 997)

// 反向计算公式
amountIn = (amountOut * reserveIn * 1000) / ((reserveOut - amountOut) * 997) + 1
```

### 1.4 多跳兑换示例
```solidity
// DAI → USDT → ETH（无直接交易对时）
address[] memory path = [DAI, USDT, WETH];
uint[] memory amounts = router.getAmountsOut(1000 * 1e18, path);
// amounts[0] = 1000 DAI（输入）
// amounts[1] = 999 USDT（中间）
// amounts[2] = 0.4 ETH（输出）
```

### 1.5 实际使用建议
- **滑点设置**：稳定币0.1-0.5%，主流币0.5-2%，山寨币2-5%
- **截止时间**：避免交易被延迟执行，通常设为当前时间+5分钟
- **前置查询**：先用 `getAmountsOut` 估算结果

---

## 二、Create Pool（创建交易对）

### 2.1 基本流程
创建新的代币交易对需要先通过 Factory 合约。

```solidity
// 创建DAI/ETH交易对
address pair = IUniswapV2Factory(factory).createPair(DAI, WETH);

// 检查是否已存在
if (pair == address(0)) {
    // 创建新交易对
    pair = factory.createPair(tokenA, tokenB);
}
```

### 2.2 创建条件
1. **代币必须不同**
2. **交易对不能已存在**
3. **不能是零地址**

### 2.3 交易对地址计算
```solidity
// 使用CREATE2确定性部署，相同代币对总是生成相同地址
function pairFor(address factory, address tokenA, address tokenB) internal pure returns (address pair) {
    (address token0, token1) = sortTokens(tokenA, tokenB);
    pair = address(uint(keccak256(abi.encodePacked(
        hex'ff',
        factory,
        keccak256(abi.encodePacked(token0, token1)),
        hex'96e8ac4277198ff8b6f785478aa9a39f403cb768dd02cbee326c3e7da348845f'
    ))));
}
```

### 2.4 初始流动性
**重要**：创建交易对后必须立即添加初始流动性，否则无法交易。

---

## 三、Add Liquidity（添加流动性）

### 3.1 基本概念
流动性提供者（LP）向交易对存入两种代币，获得LP Token作为凭证。

### 3.2 添加流程
```solidity
// 通过Router添加流动性（自动处理最优比例）
router.addLiquidity(
    tokenA,         // 代币A地址
    tokenB,         // 代币B地址
    amountADesired, // 期望提供代币A数量
    amountBDesired, // 期望提供代币B数量
    amountAMin,     // 最少提供代币A数量
    amountBMin,     // 最少提供代币B数量
    to,             // LP Token接收地址
    deadline        // 截止时间
);

// 返回三个值：
// 1. 实际使用的amountA
// 2. 实际使用的amountB
// 3. 获得的LP Token数量
```

### 3.3 比例要求
- 必须按当前池子比例添加，否则多余的代币会退回
- 首次添加时，可以任意设置比例（成为初始价格）

### 3.4 示例：为DAI/ETH池添加流动性
```solidity
// 假设当前价格：1 ETH = 2000 DAI
// 你想提供1 ETH + 2000 DAI

uint amountETH = 1 * 1e18;
uint amountDAI = 2000 * 1e18;

// 设置滑点容差（允许5%比例偏差）
uint amountETHMin = amountETH * 95 / 100;
uint amountDAIMin = amountDAI * 95 / 100;

router.addLiquidityETH{value: amountETH}(
    DAI,
    amountDAI,
    amountDAIMin,
    amountETHMin,
    msg.sender,
    block.timestamp + 300
);
```

---

## 四、Remove Liquidity（移除流动性）

### 4.1 基本概念
LP用LP Token换回对应比例的两种代币，并获得累积的手续费收益。

### 4.2 移除流程
```solidity
// 通过Router移除流动性
router.removeLiquidity(
    tokenA,
    tokenB,
    liquidity,      // 要销毁的LP Token数量
    amountAMin,     // 最少获得代币A数量
    amountBMin,     // 最少获得代币B数量
    to,             // 代币接收地址
    deadline
);

// 移除ETH流动性（特殊函数）
router.removeLiquidityETH(
    token,
    liquidity,
    amountTokenMin,
    amountETHMin,
    to,
    deadline
);
```

### 4.3 收益计算
LP的收益来自：
1. **手续费累积**：每笔交易的0.3%留在池中，按比例归属LP
2. **无常损失补偿**：通过手续费弥补价格波动带来的损失

### 4.4 示例：移除DAI/ETH流动性
```solidity
// 假设你有10个LP Token
uint lpAmount = 10 * 1e18;

// 查询能获得多少代币
(uint amountA, uint amountB) = router.quoteRemoveLiquidity(
    tokenA,
    tokenB,
    liquidity
);

// 设置最小接受量（滑点保护）
uint amountAMin = amountA * 98 / 100;  // 2%滑点容差
uint amountBMin = amountB * 98 / 100;

router.removeLiquidity(
    DAI,
    WETH,
    lpAmount,
    amountAMin,
    amountBMin,
    msg.sender,
    block.timestamp + 300
);
```

---

## 五、Flash Swap（闪电交换）

### 5.1 核心机制
**无需抵押**，先获得代币，再在回调中还款或支付等价物。

### 5.2 适用场景
- **套利**：在不同交易所间套利
- **清算**：偿还债务并清算抵押品
- **节省Gas**：无需预先准备资金

### 5.3 代码实现
```solidity
// 1. 调用Pair合约的swap函数
IUniswapV2Pair(pair).swap(
    amount0Out,  // 要借出的token0数量
    amount1Out,  // 要借出的token1数量
    address(this), // 接收借出代币的合约地址
    data         // 回调数据
);

// 2. 实现回调接口
contract MyContract {
    function uniswapV2Call(
        address sender,
        uint amount0,
        uint amount1,
        bytes calldata data
    ) external {
        // 验证调用者必须是Pair合约
        require(msg.sender == pair, "Unauthorized");
        
        // 执行套利逻辑...
        // 在其他交易所卖出借来的代币
        // 买回更多该代币
        
        // 计算需要归还的金额（本金 + 0.3%手续费）
        uint amountOwed = amount0 + amount1;
        uint fee = amountOwed * 3 / 1000;  // 0.3%手续费
        uint totalOwed = amountOwed + fee;
        
        // 归还代币给Pair合约
        IERC20(token).transfer(pair, totalOwed);
        
        // 剩余利润归自己
    }
}
```

### 5.4 安全注意事项
- 必须在**同一交易**中归还代币
- 计算好手续费和利润
- 验证调用者身份防止恶意回调

---

## 六、TWAP（时间加权平均价格）

### 6.1 基本概念
TWAP是Uniswap V2的内置预言机，提供抗操纵的时间加权价格。

### 6.2 工作原理
- 每次价格更新时记录**累计价格**
- 通过时间差计算平均价格
- 需要至少两个时间点的观测值

### 6.3 代码示例
```solidity
// 查询TWAP价格
function getTWAP(address pair, uint32 interval) external view returns (uint256 price) {
    // 定义观测时间点
    uint32[] memory secondsAgos = new uint32[](2);
    secondsAgos[0] = interval;  // 过去的时间点
    secondsAgos[1] = 0;         // 当前时间点
    
    // 获取累计价格
    (int56[] memory tickCumulatives, ) = IUniswapV2Pair(pair).observe(secondsAgos);
    
    // 计算时间加权平均tick
    int56 tickCumulativeDelta = tickCumulatives[1] - tickCumulatives[0];
    int24 timeWeightedAverageTick = int24(tickCumulativeDelta / int56(uint56(interval)));
    
    // 将tick转换为价格
    price = getPriceAtTick(timeWeightedAverageTick);
}

// 简单价格查询（无TWAP）
function getCurrentPrice(address pair) public view returns (uint256) {
    (uint112 reserve0, uint112 reserve1, ) = IUniswapV2Pair(pair).getReserves();
    // 假设reserve0是token0，reserve1是token1
    return reserve1 * 1e18 / reserve0;
}
```

### 6.4 实际应用
- **DeFi协议清算**：避免价格操纵
- **衍生品定价**：获取公允价格
- **自动策略**：基于历史价格决策

---

## 七、App（应用层面）

### 7.1 前端集成

#### **7.1.1 基础兑换功能**
```javascript
// 使用ethers.js + Uniswap SDK
import { ethers } from 'ethers';
import { swapTokens, getBestTrade } from '@uniswap/sdk';

async function swapTokens() {
    // 1. 连接钱包
    const provider = new ethers.providers.Web3Provider(window.ethereum);
    await provider.send("eth_requestAccounts", []);
    const signer = provider.getSigner();
    
    // 2. 查询最佳路径和价格
    const trade = await getBestTrade(
        inputAmount,
        inputToken,
        outputToken
    );
    
    // 3. 执行兑换
    const tx = await swapTokens(signer, trade, {
        slippageTolerance: 0.5,  // 0.5%滑点
        deadline: Math.floor(Date.now() / 1000) + 300
    });
    
    await tx.wait();
}
```

#### **7.1.2 添加流动性界面**
```javascript
// 简化版添加流动性逻辑
async function addLiquidity() {
    const router = new ethers.Contract(ROUTER_ADDRESS, ROUTER_ABI, signer);
    
    // 检查授权
    const allowanceA = await tokenA.allowance(user, ROUTER_ADDRESS);
    if (allowanceA.lt(amountA)) {
        await tokenA.approve(ROUTER_ADDRESS, ethers.constants.MaxUint256);
    }
    
    // 添加流动性
    const tx = await router.addLiquidity(
        tokenA.address,
        tokenB.address,
        amountA,
        amountB,
        amountAMin,
        amountBMin,
        user,
        Math.floor(Date.now() / 1000) + 300
    );
}
```

### 7.2 安全最佳实践

#### **7.2.1 输入验证**
```solidity
// 合约层面验证
modifier validate(address[] calldata path) {
    require(path.length >= 2, "Invalid path");
    require(path[0] != path[1], "Identical addresses");
    require(deadline >= block.timestamp, "Expired");
    _;
}
```

#### **7.2.2 错误处理**
```javascript
// 前端错误处理
async function safeSwap() {
    try {
        // 先估算Gas和结果
        const estimatedGas = await contract.estimateGas.swap(...);
        const estimatedOut = await contract.callStatic.swap(...);
        
        // 执行交易
        const tx = await contract.swap(...);
        const receipt = await tx.wait();
        
        if (receipt.status === 1) {
            console.log("交易成功");
        }
    } catch (error) {
        console.error("交易失败:", error.message);
        // 显示用户友好错误信息
        if (error.message.includes("INSUFFICIENT_OUTPUT_AMOUNT")) {
            alert("滑点过大，请调整设置");
        } else if (error.message.includes("EXPIRED")) {
            alert("交易已过期，请重新尝试");
        }
    }
}
```

### 7.3 性能优化

#### **7.3.1 批量交易**
```solidity
// 批量执行多个兑换（节省Gas）
function batchSwap(
    address[][] calldata paths,
    uint256[] calldata amountsIn,
    uint256[] calldata amountsOutMin
) external {
    for (uint i = 0; i < paths.length; i++) {
        router.swapExactTokensForTokens(
            amountsIn[i],
            amountsOutMin[i],
            paths[i],
            msg.sender,
            block.timestamp + 300
        );
    }
}
```

#### **7.3.2 Gas优化**
```solidity
// 使用内存变量减少存储读取
function optimizedSwap() internal {
    // 缓存存储变量到内存
    uint112 _reserve0 = reserve0;
    uint112 _reserve1 = reserve1;
    
    // 使用内存变量进行计算
    // ...
    
    // 最后更新存储
    reserve0 = uint112(balance0);
    reserve1 = uint112(balance1);
}
```

---

## 八、总结与建议

### 8.1 学习路径回顾
1. **先理解数学模型** → 恒定乘积、价格公式、滑点计算
2. **再学合约架构** → Factory/Pair/Router 职责分离
3. **掌握核心功能** → 兑换、流动性、闪电交换
4. **最后应用实践** → 前端集成、安全优化

### 8.2 核心要点
- **价格由储备比例决定**：`P = Y/X`
- **手续费永远存在**：0.3%从输入扣除
- **安全第一**：滑点保护、截止时间、输入验证
- **Gas很重要**：多跳比多次单跳更省Gas

### 8.3 开发检查清单
- [ ] 设置了合理的滑点容差
- [ ] 添加了截止时间检查
- [ ] 验证了代币授权状态
- [ ] 处理了可能的回滚情况
- [ ] 考虑了前端用户体验

### 8.4 资源推荐
- **官方文档**：https://docs.uniswap.org/
- **GitHub仓库**：v2-core / v2-periphery
- **测试工具**：Foundry + 主网分叉
- **监控工具**：Etherscan、Tenderly

---

这份总结涵盖了Uniswap V2的主要模块，每个部分都包含了核心概念、代码示例和实际建议。建议在实际开发中：
1. **先用测试网练习**，避免主网损失
2. **从小额交易开始**，逐步验证逻辑
3. **监控Gas费用**，优化用户体验
4. **持续学习更新**，DeFi领域变化快

Uniswap V2是DeFi的基础设施，深入理解其原理对后续学习V3、V4和其他AMM协议都有帮助。