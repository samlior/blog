# Uniswap

[Uniswap](https://app.uniswap.org/#/swap) 可以说是现在(2020-11)最火爆的AMM(Automated Market Maker 自动做市商)

## 仓库地址

[Core](https://github.com/Uniswap/uniswap-v2-core)

Core仓库存放了三份合约, 分别是UniswapV2ERC20.sol, UniswapV2Factory.sol, UniswapV2Pair.sol.
+ UniswapV2ERC20.sol是Uniswap实现的一份ERC20合约, 和普通ERC20合约基本没有区别, 增加了permit方法, 可以通过密码学操作approve.
+ UniswapV2Factory.sol是一个交易对工程合约, 有两个功能, 第一个是创建新的交易对并把新合约地址和交易对的信息储存在合约里, 第二个是设置流动性手续费的受益人.
+ UniswapV2Pair.sol继承于UniswapV2ERC20.sol, 同时实现了交易对合约.

[Periphery](https://github.com/Uniswap/uniswap-v2-periphery)

Periphery只存放了一份合约, UniswapV2Router02.sol(其实还有UniswapV2Router01.sol, 只不过02是01的升级版, 在01的基础上增加了PermitSupporting, 可以使用密码学操作流动性代币, 不影响主要逻辑, 这里不讨论), 还有另一份以库的形式存在的合约UniswapV2Library.sol.

+ UniswapV2Router02.sol是Uniswap核心路由合约, 用户只会直接和router合约交互, 并且具体兑换的金额也是在router合约中完成.

+ UniswapV2Library.sol实现了所有金额的具体计算逻辑, 比如指定输入计算输出, 或指定输出计算输入等.

## 合约地址

[Factory](https://etherscan.io/address/0x5c69bee701ef814a2b6a3edd4b1652cb9cc5aa6f)

[Router02](https://etherscan.io/address/0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D)

## 逻辑流程

![avatar](../img/Uniswap.png)

## 主要代码分析

### Pair
UniswapV2Pair.sol继承于UniswapV2ERC20.sol, 也就是说每一个单独的交易对都是一个ERC20, 这个ERC20就是这个交易对的流动性代币. 向这个交易对质押代币就可以获得这个交易对的ERC20流动性代币, 销毁这个交易对的ERC20流动性代币就可以赎回质押的代币. (这两次操作的时间差之间如果有其他用户在这个交易对上进行了交易, 就会产生手续费. 这部分手续费会留在池中, 用户赎回的时候就会多赎回一点, 由此产生收益)

```js
// 工厂的地址.
address public factory;
// 第一种代币的地址.
address public token0;
// 第二种代币的地址.
address public token1;

// 此交易对的第一种代币的余额.
uint112 private reserve0;
// 此交易对的第二种代币的余额.
uint112 private reserve1;
```

```js
function _update(uint balance0, uint balance1, uint112 _reserve0, uint112 _reserve1) private;
```
_update方法主要用于更新当前交易对对两种代币对余额.

```js
function _mintFee(uint112 _reserve0, uint112 _reserve1) private returns (bool feeOn);
```
_mintFee主要用于增发手续费.

此处的手续费是指, 每当用户质押代币获取流动性代币的时候, 会额外增发一定比例的流动性代币给factory合约中设置的手续费收益人.

目前(2020-11)Uniswap没有开启这部分手续费.

```js
function mint(
    address to // 接受流动性代币的地址.
) external lock returns (
    uint liquidity // 增发量.
);
```
mint方法内部会调用ERC20的_mint方法, 给目标地址增发流动性代币.

具体增发的流动性代币的数量取决于交易对地址的两种代币的余额.

(router调用这个接口之前, 会把用户质押的金额转给交易对, 由于此时还有没用调用_update, 因此可以通过balance当前余额和reserve上一次余额之差计算出此次用户质押了多少金额)

如果当前交易池中两种资产的余额都为0, 那么新增发的流动性为Math.sqrt(amount0.mul(amount1)).sub(MINIMUM_LIQUIDITY), 其中MINIMUM_LIQUIDITY为用于留在池中的最小流动性代币数量, 会转给黑洞账户.

如果两种资产的余额不为0, 那么新增发的流动性为Math.min(amount0.mul(_totalSupply) / _reserve0, amount1.mul(_totalSupply) / _reserve1)

```js
function burn(
    address to // 接受赎回代币的地址.
) external lock returns (
    uint amount0, // 赎回的第一种代币的金额.
    uint amount1 // 赎回的第二种代币的金额.
);
```
burn用于销毁所有属于此交易对(address(this))的流动性代币, 并把可以赎回的代币转给目标地址.

(router调用这个接口之前会把要销毁的流动性代币转给这个交易对的地址)

具体的计算公式为 amount = liquidity * balance / totalSupply.

其中liquidity为销毁的流动性代币数量, balance为某种代币的总量, totalSupply流动性代币的总量.

```js
function swap(
    uint amount0Out, // 第一种代币的输出金额.
    uint amount1Out, // 第二种代币的输出金额.
    address to, // 接受输出金额的地址.
    bytes calldata data // 调用参数, 目前没有用到.
) external lock;
```
swap用于扣除交易对的某种代币的余额, 并转给指定的用户, 一般amount0Out和amount1Out中有一个为0.

注意swap并不包含计算逻辑, 具体应该转多少金额由router计算, pair中的swap只负责转账.

---

### Router

router是Uniswap的核心逻辑, 无论是增加流动性, 去除流动性, 还是交换代币, 都直接与这份合约交互. 以下只着重分析几个函数, 其他函数逻辑类似.

```js
// 工厂地址.
address public immutable override factory;
// WETH地址, ETH和其他代币交换的时候会先转化为WETH, 确保逻辑一致.
address public immutable override WETH;
```

```js
function _addLiquidity(
    address tokenA, // 第一种代币的合约地址.
    address tokenB, // 第二种代币的合约地址.
    uint amountADesired, // 用户希望的增加的第一种代币的金额.
    uint amountBDesired, // 用户希望的增加的第二种代币的金额.
    uint amountAMin, // 用户设置的增加的第一种代币的最小金额.
    uint amountBMin // 用户设置的增加的第二种代币的最小金额.
) internal virtual returns (
    uint amountA, // 具体可以质押的第一种代币的金额.
    uint amountB // 具体可以质押的第二种代币的金额.
);
```
_addLiquidity用于计算对给定交易对及用户的金额设置, 具体能质押多少金额.

(质押流动性时, 新加入的资产必须和池中已有的资产比例保持一致, 否则会扰乱价格)

如果交易池中目前两种代币的余额都为0, 那么具体质押的金额就是用户希望的金额, 也就是amountADesired和amountBDesired.

如果代币余额不为0, 那么就对amountADesired和amountBDesired分别调用一次library.quote, 同时检查计算结果是否满足用户设置的最小值.

最后计算出具体可以质押多少金额并返回.

```js
function addLiquidity(
    address tokenA, // 第一种代币的合约地址.
    address tokenB, // 第二种代币的合约地址.
    uint amountADesired, // 用户希望的增加的第一种代币的金额.
    uint amountBDesired, // 用户希望的增加的第二种代币的金额.
    uint amountAMin, // 用户设置的增加的第一种代币的最小金额.
    uint amountBMin, // 用户设置的增加的第二种代币的最小金额.
    address to, // 接受流动性代币的地址.
    uint deadline // 超时时间.
) external virtual override ensure(deadline) returns (
    uint amountA, // 具体质押的第一种代币的金额.
    uint amountB, // 具体质押的第二种代币的金额.
    uint liquidity // 增发的流动性代币数量.
);
```
addLiquidity内部会调用_addLiquidity计算出具体的质押金额.

然后把质押金额的两种代币转给pair合约.

最后调用pair合约的mint函数.

(mint函数内部可以根据新增加的余额的数量计算出此次用户的质押金额)

```js
function removeLiquidity(
    address tokenA, // 第一种代币的合约地址.
    address tokenB, // 第二种代币的合约地址.
    uint liquidity, // 用户希望销毁的流动性代币的数量.
    uint amountAMin, // 用户设置的第一种代币的最小的赎回金额.
    uint amountBMin, // 用户设置的第二种代币的最小的赎回金额.
    address to, // 接受赎回代币的地址.
    uint deadline // 超时时间.
) public virtual override ensure(deadline) returns (
    uint amountA, // 具体赎回的第一种代币的金额.
    uint amountB // 具体赎回的第二种代币的金额.
);
```
removeLiquidity用于销毁流动性代币, 赎回质押的代币.

先把要销毁的流动性代币从用户手中转到交易对地址中.

之后调用交易对的burn方法, 在burn方法中, 交易对合约会获取自己的流动性代币的当前余额, 这个就是用户要销毁的流动性代币的数量

```js
function _swap(
    uint[] memory amounts, // 每个交易对对应的金额.
    address[] memory path, // 交易路径(由不同代币的地址组成, 两两相邻的地址就代币一个交易对).
    address _to // 最后输出的代币的收款地址.
) internal virtual;
```

_swap用于在若干个交易对中连续交互资产.

比如现在path为[ETH, USDT, BTC], amounts为[1, 2 ,3],

那么其含义为将1个ETH放入ETH-USDT交易对中, 换出2个USDT, 在把2个USDT放入USDT-BTC交易对中, 换出3个BTC.

amounts和path都是上层函数计算好的结果, 因此, _swap中直接调用pair的swap方法, 转移交易对中的资产到接受账户.

```js
function swapExactTokensForTokens(
    uint amountIn, // 用户输入的具体金额.
    uint amountOutMin, // 用户设置的最小输出金额.
    address[] calldata path, // 用户指定的兑换路径.
    address to, // 接受输出金额的帐户.
    uint deadline // 超时时间.
) external virtual override ensure(deadline) returns (
    uint[] memory amounts // 每个交易对对应的金额.
)
```

swapExactTokensForTokens用于指定输入金额, 计算输出金额, 并兑换.

其内部会调用library的getAmountsOut方法计算最终的输出金额.

如果输出金额满足最小输出金额则调用_swap完成所有交易对的金额变动及最后的转账.

```js
function swapTokensForExactTokens(
    uint amountOut, // 用户指定的具体的输出金额.
    uint amountInMax, // 用户设置的最大输入金额.
    address[] calldata path, // 用户指定的兑换路径.
    address to, // 接受输出金额的帐户.
    uint deadline // 超时时间.
) external virtual override ensure(deadline) returns (
    uint[] memory amounts // 每个交易对对应的金额.
);
```
swapTokensForExactTokens用于指定输出金额, 计算输入金额, 并兑换.

其内部会调用library的getAmountsIn方法计算最终的输入金额.

如果输入金额满足最大输入金额则调用_swap完成所有交易对的金额变动及最后的转账.

---

### Library

```js
function quote(
    uint amountA, // 第一种代币的金额.
    uint reserveA, // 交易对中第一种代币的总余额.
    uint reserveB // 交易对中第二种代币的总余额.
) internal pure returns (
    uint amountB // 可以质押的第二种代币的金额.
);
```

quote用于计算当一种代币的金额固定时, 在当前两种代币余额的同比例下, 可以质押多少第二种代币.

公式很简单 amountB = amountA.mul(reserveB) / reserveA.

```js
function getAmountOut(
    uint amountIn, // 输入的代币的金额.
    uint reserveIn, // 输入的代币的总余额.
    uint reserveOut // 输出的代币的总余额.
) internal pure returns (
    uint amountOut // 输出的代币的金额.
);
```

getAmountOut用于根据交易池中当前两种代币的余额和输入资产的数量, 计算可以输出多少资产.

核心的公式是 x * y = k, 在这里就是 (reserveIn + amountIn) * (reserveOut - amountOut) = reserveIn * reserveOut.

如果在带上千分之三的手续费就是 (reserveIn + amountIn * 997 / 1000) * (reserveOut - amountOut) = reserveIn * reserveOut.

最后可以变形为 amountOut = amountIn * 997 * reserveOut / (reserveIn * 1000 + amountIn * 997).

```js
function getAmountIn(
    uint amountOut, // 输出的代币的金额.
    uint reserveIn, // 输入的代币的总余额.
    uint reserveOut // 输出的代币的总余额.
) internal pure returns (
    uint amountIn // 输入的代币的金额.
)
```

getAmountIn和getAmountOut原理一样, 过程相反.

核心的公式是 x * y = k, 在这里就是 (reserveIn + amountIn) * (reserveOut - amountOut) = reserveIn * reserveOut.

如果在带上千分之三的手续费就是 (reserveIn + amountIn * 997 / 1000) * (reserveOut - amountOut) = reserveIn * reserveOut.

最后可以变形为 amountIn = 1000 * reserveIn * amountOut / 997 * (reserveOut - amountOut).

```js
function getAmountsOut(
    address factory, // factory合约地址.
    uint amountIn, // 输入的代币的金额.
    address[] memory path // 兑换路径.
) internal view returns (
    uint[] memory amounts // 每个交易对对应的金额.
)
```

getAmountsOut内部会遍历path, 获取每个交易对当前的两种代币的余额, 再调用getAmountOut计算输出的金额, 保存在一个列表中返回.

```js
function getAmountsIn(
    address factory, // factory合约地址.
    uint amountOut,// 输出的代币的金额.
    address[] memory path // 兑换路径.
) internal view returns (
    uint[] memory amounts // 每个交易对对应的金额.
)
```

getAmountsIn和getAmountsOut原理一样, 只是倒序遍历path, 并调用getAmountIn.