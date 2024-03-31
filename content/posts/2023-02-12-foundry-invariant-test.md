---
slug: foundry-invariant-test
title: Foundry Invariant Test 初探
date: 2023-02-12 10:00:00
authors: [Paco]
math: true
categories:
  - Solidity
---

Foundry 的文档终于更新了长期以来鲜为人知的 Invariant Test 功能。这个功能在过去几个月中一直在 Foundry 中是一个未被完全文档化的神秘功能，只在少数项目中得到了使用。但是，使用好 Invariant Test 可以显著提高合约的可靠性，可以预见到，在不久的将来，Invariant Test 将成为一名合约开发的必备技能之一。

我们来看看 Foundry 作者和其他开发者是怎么说的：

![gakonst tweet](/img/in-post/foundry-invariant-test/gakonst-tweet.png)

![prb tweet](/img/in-post/foundry-invariant-test/prb-tweet.png)

# 什么是 Invariant Test

Invariant Test 是指对系统中的某些不变性（Invariant）进行的测试。

以一个排序算法程序为例，对这个排序算法来说，它需要满足的不变性可以是：对任意数组进行排序后，在结果中任取两个元素 a, b，如果 a 的索引小于 b，则 a 一定小于等于 b。

上面所说的排序算法通常是一个无状态的程序。实际上，我们通常更需要的是对一个有状态系统来进行 Invariant Test，即**软件系统运行之后，在任意时刻，系统内部状态都一定要满足某种不变性约束**。

例如，一个电商系统（amazon, taobao）中，在任意时刻它一定要满足：

- 系统内收款总金额 >= 已售出商品的总金额
- 任意商品的卖出价格 >= 0
- ...

这些不变性约束通常是和软件系统的业务逻辑强相关的，因此在设计 Invariant Test 之前，我们首先需要深入理解业务模型，找到那些最需要进行测试的 Invariants。

在合约开发领域，我们可以有这样的 Invariant：

- Uniswap V2 Pool 必须满足： `x*y >= k`
- 对于一个普通 ERC20 合约，必须满足：`sum(balanceOf[user]) = totalSupply`

## 如何进行 Invariant Test

刚才我们说了，Invariant test 中的 Invariant 通常是对系统内部状态的约束。那么为了测试这些 Invariant，我们需要在测试中尽可能的制造出系统各种不同的内部状态，这样才能 cover 到更多的场景。在 Unit test 或者 Integration Test 中，系统中被测试的状态通常是我们人为制造出来的，这通常无法保证系统的 Invariant 能够在所有状态下也是成立，因为手动编写的 test 都很难有太高的 test coverage。

因此，在合约测试时，通常用如下方式来进行 Invariant Test：

- 在测试环境中部署出所有合约并初始化
- 随机的调用和所有合约的所有可调用函数，让系统进入随机的状态
- 每次调用结束，都检查 Invariant 是否满足

为了让系统状态更加随机，整个过程可能会随机调用几千或上万次系统内的函数，这样就会更容易发现系统中的 edge cases。

目前有 Contract Invariant test 功能的工具有：[echidna](https://github.com/crytic/echidna), [dapptools](https://github.com/dapphub/dapptools) 和 [foundry](https://github.com/foundry-rs/foundry)。

## Invariant Test 可以发现什么类型的 bug

可以参考这篇文章：[Using automatic analysis tools with MakerDAO contracts](https://forum.openzeppelin.com/t/using-automatic-analysis-tools-with-makerdao-contracts/1021)，作者使用了多种工具来测试是否能使用 Invaraint Test 来找出一个在 makerDAO 使用合约中的已知 Bug。

## 与其他测试类型的区别

- Unit test 的测试对象通常是单个函数，而 Invariant Test 的测试对象是整个系统。
- Integration test 更加关注的是系统内部各个组件是否能正常协同工作，而 Invariant Test 则关注系统内状态的 Invariant 约束。
- Fuzz test/Property test 通常也是针对某一个函数或功能，而 Invariant Test 可以理解成是对整个系统进行的 Fuzz Test。
- Monkey test/chaos engineering 这种测试通常是测试系统的 robustness，而非 Invariant。

# Tests in Foundry

如果你还不知道什么是 Foundry 或者如何在 Foundry 写测试，那么请先阅读 [Writing Test](https://book.getfoundry.sh/forge/writing-tests)。

## Fuzz Test

在进入 Invariant Test 话题之前，我们需要先了解一些 Fuzz Testing 的概念。如果一个 foundry 的测试函数中带有参数，那么它就是 Fuzz test，这些参数都会由 foundry 随机生成并运行。

我们以 Foundry 默认项目中的代码为例，运行 `forge init counter`，Foundry 会生成一个项目，并包含如下合约：

```Solidity
pragma solidity ^0.8.13;

contract Counter {
    uint256 public number;

    function setNumber(uint256 newNumber) public {
        number = newNumber;
    }

    function increment() public {
        number++;
    }
}
```

其中的测试代码为：

```Solidity
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../src/Counter.sol";

contract CounterTest is Test {
    Counter public counter;

    function setUp() public {
        counter = new Counter();
        counter.setNumber(0);
    }

    function testIncrement() public {
        counter.increment();
        assertEq(counter.number(), 1);
    }

    function testSetNumber(uint256 x) public {
        counter.setNumber(x);
        assertEq(counter.number(), x);
    }
}
```

这里的 `testIncrement()` 是一个普通单元测试，而 `testSetNumber(uint256 x)` 则是一个 Fuzz Test，foundry 在测试时会自动生成参数 `x` 的值，并代入此测试函数中运行。

我们运行测试：

```bash
❯ forge test
...

Running 2 tests for test/Counter.t.sol:CounterTest
[PASS] testIncrement() (gas: 28356)
[PASS] testSetNumber(uint256) (runs: 256, μ: 27797, ~: 28342)
Test result: ok. 2 passed; 0 failed; finished in 6.33ms
```

测试 `testSetNumber(uint256)` 时，输出中的 `(runs: 256, μ: 27797, ~: 28342)` 分别表示此测试运行的次数为 256 次，gas 开销的平均值和中位数。

在这 256 次测试中，Foundry 每次会生成一个随机的 `x` 值，并传给 `testSetNumber(uint256)` 运行。

## Fuzz 策略

默认情况下，Foundry 会进行 256 次 fuzz test，每次都会使用随机的参数来进行测试。在选用随机数时，Foundry 会结合下面 3 种策略：

- 根据参数类型，选择一些边界值，例如参数类型时 `uint32`，那么则使用 `[0, 1, 2, 3]` 和 `[2^32-3, 2^32-3, 2^32-2, 2^32-1]`
- 选择字典（Dictionary）中的值
- 随机生成

这里的字典指的是 Foundry 在测试环境中收集到的一些值，例如 `PUSH` 操作对象，合约 bytecode 中的常数，合约 storage 中的值等等。通过使用字典中的值测试，更容易提高 Fuzz test 的测试覆盖率。

Fuzz Test 的这些策略都可以通过配置来调整，具体可以参考 [Config Reference - Testing](https://book.getfoundry.sh/reference/config/testing#fuzz)。

在 Fuzz Test 中可以使用 `vm.assume()` 或者 `forge-std` 中的 `bound()` 来过滤一些不想要的参数。

如果是对单个值的过滤推荐使用 `vm.assume()`，而如果想进行范围过滤，则更推荐使用 `bound()`，因为在使用 `vm.assume()` 时如果范围太小，可能导致 fuzz 时 reject 过多而失败（默认配置最多允许 65536 次 reject）。

## Invariant Test

测试合约中，所有以 `invariant` 开头的函数都会被当作是 Invariant test 来运行。Foundry 会通过这样的策略来进行 Invariant test：

1. 在 `setUp()` 中 deploy 的合约都会被当作目标合约
2. Foundry 调用 `setUp()` 开始新一轮的测试，此时所有合约的状态为 `setUp()` 后的初始状态
3. Foundry 随机调用某个合约的某个函数（根据合约的 ABI 自动生成调用的参数，类似 Fuzz Test）
4. 如果函数调用的失败，测试并不会中断
5. 函数调用结束后，测试预定义的 Invariant
6. 重复第 3 步，直至调用 `depth` 次
7. 重复第 2 步，直至运行 `run` 轮

这里的 `run` 和 `depth` 都是可以配置的参数，默认配置下 `run = 256, depth = 15`。

### 简单示例

我们对下面这个合约来进行 Invariant Test：

```Solidity
pragma solidity ^0.8.13;

contract ExampleContract1 {
    uint256 public a;
    uint256 public b;
    uint256 public total;

    function addToA(uint256 amount) external {
        a += amount;
        total += amount;
    }

    function addToB(uint256 amount) external {
        b += amount;
        total += amount;
    }
}
```

这里，我们编写两个 Invariant：

- `a + b == total`
- `a + b >= a`

测试合约如下：

```Solidity
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import {ExampleContract1} from "../src/Example.sol";

contract InvariantExample1 is Test {
    ExampleContract1 foo;

    function setUp() external {
        foo = new ExampleContract1();
    }

    function invariant_A() external {
        assertEq(foo.a() + foo.b(), foo.total());
    }

    function invariant_B() external {
        assertGe(foo.a() + foo.b(), foo.a());
    }
}
```

运行测试：

```bash
❯ forge test
...

Running 2 tests for test/Example.sol:InvariantExample1
[PASS] invariant_A() (runs: 256, calls: 3840, reverts: 1270)
[PASS] invariant_B() (runs: 256, calls: 3840, reverts: 1270)
Test result: ok. 2 passed; 0 failed; finished in 199.86ms
```

可以看到，每个测试运行了 `3840=256*15` 次函数调用，revert 次数为 1270（参数过大时可能 overflow 而 revert）。

这里虽然 revert 了 1270 次，但是这些 revert 并不会中断测试，因为 Invariant Test 关注的是 Invariant 约束是否满足。

如果设置 `fail_on_revert = true`，则测试中的 revert 会导致测试中断。

### 测试目标分布和过滤

上面的例子只包含一个合约，因此所有调用都会针对此合约进行，如果测试中包含多个合约，则调用会平均分配到这些合约上。

目前 foundry 只实现了最基本的平均分配策略，每一个合约被调用的概率是相同的，在内部，每一个函数被调用的概率也是相同的。

例如，系统中有两个合约，分别包含 2 个和 4 个函数，则测试时随机调用的分布概率是：

```
targetContract1: 50%
├─ function1: 50% (25%)
└─ function2: 50% (25%)

targetContract2: 50%
├─ function1: 25% (12.5%)
├─ function2: 25% (12.5%)
├─ function3: 25% (12.5%)
└─ function4: 25% (12.5%)
```

`forge-std/Invariant` 中提供了一系列的函数，可以用来过滤需要调用的函数或合约以及 `sender`，这些函数包括 `excludeContract/excludeSender/excludeArtifact` 和 `targetArtifact/targetArtifactSelector/targetContract/targetSelector/targetSender`，具体用法这里不再赘述。

### 最佳实践

在一个大型系统中，通常包含非常多的合约，以及复杂的业务逻辑，那么完全由 Foundry 随机去调用众多合约中的函数，效率会非常低。目前社区比较推崇使用 Handler-Based Testing 的方式来进行复杂的 Invariant Test。

Handler 合约可以认为是介于系统合约和 Foundry 之间的中间层，用来将系统中合约的函数进行包装和组合，在测试时，我们可以让 Foundry 只对 Handler 合约中的函数来进行随机调用，这样就避免了一些无效的调用。

下面示例对一个 ERC-4626 合约来进行测试（摘自 Foundry book）：

```Solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.17;

interface IERC20Like {

    function balanceOf(address owner_) external view returns (uint256 balance_);

    function transferFrom(
        address owner_,
        address recipient_,
        uint256 amount_
    ) external returns (bool success_);

}

contract Basic4626Deposit {

    /**********************************************************************************************/
    /*** Storage                                                                                ***/
    /**********************************************************************************************/

    address public immutable asset;

    string public name;
    string public symbol;

    uint8 public immutable decimals;

    uint256 public totalSupply;

    mapping(address => uint256) public balanceOf;

    /**********************************************************************************************/
    /*** Constructor                                                                            ***/
    /**********************************************************************************************/

    constructor(address asset_, string memory name_, string memory symbol_, uint8 decimals_) {
        asset    = asset_;
        name     = name_;
        symbol   = symbol_;
        decimals = decimals_;
    }

    /**********************************************************************************************/
    /*** External Functions                                                                     ***/
    /**********************************************************************************************/

    function deposit(uint256 assets_, address receiver_) external returns (uint256 shares_) {
        shares_ = convertToShares(assets_);

        require(receiver_ != address(0), "ZERO_RECEIVER");
        require(shares_   != uint256(0), "ZERO_SHARES");
        require(assets_   != uint256(0), "ZERO_ASSETS");

        totalSupply += shares_;

        // Cannot overflow because totalSupply would first overflow in the statement above.
        unchecked { balanceOf[receiver_] += shares_; }

        require(
            IERC20Like(asset).transferFrom(msg.sender, address(this), assets_),
            "TRANSFER_FROM"
        );
    }

    function transfer(address recipient_, uint256 amount_) external returns (bool success_) {
        balanceOf[msg.sender] -= amount_;

        // Cannot overflow because minting prevents overflow of totalSupply,
        // and sum of user balances == totalSupply.
        unchecked { balanceOf[recipient_] += amount_; }

        return true;
    }

    /**********************************************************************************************/
    /*** Public View Functions                                                                  ***/
    /**********************************************************************************************/

    function convertToShares(uint256 assets_) public view returns (uint256 shares_) {
        uint256 supply_ = totalSupply;  // Cache to stack.

        shares_ = supply_ == 0 ? assets_ : (assets_ * supply_) / totalAssets();
    }

    function totalAssets() public view returns (uint256 assets_) {
        assets_ = IERC20Like(asset).balanceOf(address(this));
    }

}
```

现在假设我们想对 `deposit()` 函数进行测试，我们可以把 `deposit` 相关的操作包装到 Handler 合约中：

```Solidity
function deposit(uint256 assets) public virtual {
    asset.mint(address(this), assets);

    asset.approve(address(token), assets);

    uint256 shares = token.deposit(assets, address(this));
}
```

这个函数包含了相关 token 的 `mint`, `approve` 以及 `deposit`，然后我们在 `setUp()` 中使用 `targetContract()` 让 Foundry 在测试时只调用 Handler 合约，这样就能减少 revert 的次数，提高效率。使用 Handler 之后，Invariant Test 过程如下图：

![handler](https://user-images.githubusercontent.com/44272939/216420091-8a5c2bcc-d586-458f-be1e-a9ea0ef5961f.svg)

### Ghost Variable

在 Handler 合约中，还可以增加一些状态变量，用来记录一些中间变量，这些状态变量可能不会被放到实际项目的合约中，但是可以作为 Invariant 约束来使用，这些变量通常称作 "ghost variables"。

例如，在上面的 `deposit()` 函数中，我们对总 deposit 数量进行追踪：

```Solidity
function deposit(uint256 assets) public virtual {
    asset.mint(address(this), assets);

    asset.approve(address(token), assets);

    uint256 shares = token.deposit(assets, address(this));

    sumBalanceOf += shares;
}
```

这里的 `sumBalanceOf` 就是 Handler 中的一个 ghost variable，它可以和合约中的 `totalSupply` 组成一个 Invariant：`sumBalanceOf == totalSupply`。

使用 Handler 合约可以帮助提高 Invariant Test 的效率，但是如果 Handler 合约太过复杂，那么有可能在 Handler 合约中藏有 bug，所以最好是对 Handler 合约也编写一些简单的测试来保证 Handler 的正确性。

可以在 Handler 的函数中加入一些 assert 来进行测试，例如给 `deposit()` 加上 `assertEq()` 测试：

```Solidity
function deposit(uint256 assets) public virtual {
    asset.mint(address(this), assets);

    asset.approve(address(token), assets);

    uint256 beforeBalance = asset.balanceOf(address(this));

    uint256 shares = token.deposit(assets, address(this));

    assertEq(asset.balanceOf(address(this)), beforeBalance + assets);

    sumBalanceOf += shares;
}
```

还可以给函数参数使用 `bound` 来保证 handler 函数不会失败，这样就可以在测试时开启 `fail_on_revert = true`，测试时每一次 revert 都会中断整个测试。但是这样做的潜在坏处是，缩小 input 参数范围之后，也可能同时缩小了 test coverage。

### Actor

上面的测试中，函数的 sender 都是 `address(this)`，我们可以使用 actor 模式来给测试生产随机的调用者，可以对待测试函数使用如下 `modifier`：

```Solidity
address[] public actors;

address internal currentActor;

modifier useActor(uint256 actorIndexSeed) {
    currentActor = actors[bound(actorIndexSeed, 0, actors.length - 1)];
    vm.startPrank(currentActor);
    _;
    vm.stopPrank();
}
```

更多关于 Invariant Test 使用的最佳实践演示，可以参考这个仓库：[lucas-manuel/invariant-examples](https://github.com/lucas-manuel/invariant-examples)。

### Call Summary

Foundry 目前不支持显示在 Invariant Test 中每个函数调用的次数，如果我们想记录和展示这些数据，可以在 Handler 中使用状态变量和 modifier 将每次函数调用都记录下来，最后使用 `console` 打印统计的结果。


例如在 `Handler` 中统计调用情况：

```Solidity
contract Handler {
    mapping(bytes32 => uint256) public numCalls;

    modifier countCall(bytes32 key) {
        numCalls[key]++;
        _;
    }

    ...

    function callSummary() external view {
        console.log("Call summary:");
        console.log("-------------------");
        console.log("deposit", calls["deposit"]);
        console.log("withdraw", calls["withdraw"]);
    }
}
```

在测试中加入一个专门的 Invariant Test case 来输入 call summary：

```Solidity
    function invariant_callSummary() public view {
        handler.callSummary();
    }
```

接下就可以运行这个 test case 来检查统计结果：

```
❯ forge test -vv -m invariant_callSummary
```

注意，结果的输出中只会展示**最后一次** 测试所产生的输出。因此它只能代表最后一轮 run 中的结果。

### Regression Test & Mutation Test

如果使用 Invariant Test 发现了系统中的 bug，我们最好将出现 bug 时系统中的状态记录下来，这样可以在修复 bug 之后还原此状态进行回归测试。

### Test your tests

当我们编写了很多 Invariant Test case 之后，即使这些 test case 都能通过，我们其实也无法能保证测试 case 都是正确的。例如，我们定义了一个永远都正确的 Invariant，那么我们的测试也永远不会出错。

为了验证我们的测试 case 能够发现真正的代码 bug，我们可以这样做：

过程如下：

- 手动修改代码， 故意让其出现 bug
- 运行 Invariant Test，此时一些 test case 应该出错
- 把出现 bug 时的代码和修复后的代码使用 `git diff src/XXX.sol > bug1.patch` 保存为 patch 文件，方便之后再次重现

这种测试方式叫做 [mutation testing](https://www.wikiwand.com/en/Mutation_testing) ，有一些工具能够帮助自动基于 solidity 来生成 mutation，如：[gambit](https://github.com/Certora/gambit)。

Foundry 未来也可能集成自动化的 mutation testing，相关讨论在 [https://github.com/foundry-rs/foundry/issues/478](https://github.com/foundry-rs/foundry/issues/478)。

## 更多资料

- [lucas-manuel/invariant-examples](https://github.com/lucas-manuel/invariant-examples)，Invariant Test 最佳实践的完整示例参考。
- [horsefacts/weth-invariant-testing](https://github.com/horsefacts/weth-invariant-testing)，[eth_call](https://twitter.com/eth_call) 写的 Invariant Test 教程，使用 WETH9 合约为范例，一步步教你如何编写 Invariant Test，涵盖了 Handler, Ghost Variable, Call Summary, Mutation Test 等内容，非常值得学习。
- [How to Foundry 2.0: Brock Elmore](https://www.youtube.com/watch?v=EHrvD5c93JU)，这个视频介绍了 Foundry 最新功能的用法，当然也包含 Invariant Test，其中还透露了 [Brock Elmore](https://twitter.com/brockjelmore) 正在开发的 symbolic execution 工具（将更有利于提高测试覆盖率），整个视频都值得一看。
- [Learn how to fuzz like a pro](https://www.youtube.com/watch?v=QofNQxW_K08&list=PLciHOL_J7Iwqdja9UH4ZzE8dP1IxtsBXI)，Trail of bits 团队对如何使用 echidna 进行 fuzz/invariant test 的详细教学视频。

想要学习其他项目在生产环境中使用 Invariant Test 的案例，可以参考这些仓库：

- [maple-labs/maple-core-v2](https://github.com/maple-labs/maple-core-v2/tree/main/tests/invariants)，maple finance 大量使用了 handler 模式的 Invariant Test，并且对在测试中发现 bug 使用[回归测试](https://github.com/maple-labs/maple-core-v2/blob/00f01ae7175885f8d49ac201a1c72465e320b2f6/tests/e2e/Regression.t.sol#L53)的方式来确保修复
- [optimism](https://github.com/ethereum-optimism/optimism/tree/develop/packages/contracts-bedrock/contracts/test/invariants)，Optimism 中的 Invariant Test，使用了 Handler + Actor 结合的模式

## Open Issues

Foundry 的 Invariant Test 还在持续的开发和改进中，如果你想对项目进行贡献，不妨看看以下相关 issue：

- [https://github.com/foundry-rs/foundry/issues/3005](https://github.com/foundry-rs/foundry/issues/3005)
- [https://github.com/foundry-rs/foundry/issues/2985](https://github.com/foundry-rs/foundry/issues/2985)
- [https://github.com/foundry-rs/foundry/issues/2962](https://github.com/foundry-rs/foundry/issues/2962)
- [https://github.com/foundry-rs/foundry/issues/2986](https://github.com/foundry-rs/foundry/issues/2986)
- [https://github.com/foundry-rs/foundry/issues/585](https://github.com/foundry-rs/foundry/issues/585)


如果你是一个 Foundry 爱好者，下一个项目请记得一定使用 Invariant Test!


