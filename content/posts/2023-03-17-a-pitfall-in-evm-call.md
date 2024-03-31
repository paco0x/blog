---
slug: a-pitfall-in-evm-call
title: 眼见不一定为实 —— EVM CALL OPcode 中的 gas 陷阱
date: 2023-02-17 11:00:00
authors: [Paco]
math: true
categories:
  - Solidity
---

故事要从 Optimism Bedrock 说起，这些天在研究 Optimism Bedrock 的代码，在无意中发现了一个之前一直没注意到问题：在一些 edge case 下，**EVM `CALL`（也包含 `STATICCALL`, `DELEGATECALL`） Opcode 使用的 gasLimit 可能和预期不一致**。

一般情况下，这个设计不会对应用产生太大的安全风险，但是在 Optimism Bedrock 的设计中，这个问题可能被恶意利用，从而导致用户的资产被永久锁定，产生严重的后果，甚至 Optimism 团队在最开始也没有意识到这个问题。因此我觉得有必要写一篇文章来揭露这个很容易被忽视的问题。

## Optimism Bedrock 的跨链交易

Optimism 中为跨链交易设计了两种叫做 Deposits 和 Withdrawals 的交易类型，分别表示 L1 -> L2 和 L2 -> L1 的交易类型。通过这两种类型的交易， 用户可以从 L1 调用 L2 地址，也可以从 L2 调用 L1 地址，同时还可以在交易中设定 `msg.value` 从而实现 ETH 的跨链转移。

在 Optimism 中，这两种跨链交易都是**不可以重试的（not retryable）**，这意味着用户在发起交易后，无论交易成功与否，用户都无法再重试已经执行过的交易。这样设计的结果是，如果用户的交易中附带了 ETH（`msg.value`），一旦交易失败，这些 ETH 就会被永久锁定在合约中，用户无法再次发起交易来取回这些 ETH。

因此，用户在使用跨链交易转移 ETH 时，需要非常小心，确保交易不会失败。在 Optimism 中，交易失败的原因有很多，其中一种是，此交易消耗的 gas 超过了用户指定的 gasLimit：Optimism 在处理跨链交易时，会按照用户指定的 gasLimit 来执行交易，如果交易执行过程中消耗的 gas 超过了 gasLimit，那么交易就会失败。

我们来看一下一个 Withdrawal 跨链交易（ L2 -> L1 ）的过程：

1. 用户在 L2 的 `L2ToL1MessagePasser` 合约发起一个跨链交易，目标链为 L1，需要指定交易的 `target`, `value`, `gasLimit`, `data`
2. Optimism 会对这个交易参数计算 hash，并将 hash 作为 key，保存在合约的一个 mapping 中
3. Optimism 的 proposer 会将 `L2ToL1MessagePasser` 合约的 storage root 保存到 L1 的 `L2OutputOracle` 合约中
4. 用户在 L1 调用 `OptimismPortal.proveWithdrawalTransaction()`，使用 storage root 和 proofs 来证明交易的存在性
5. 等待 7 天的挑战期
6. 任何人可以调用 `OptimismPortal.finalizeWithdrawalTransaction()` 来将用户的跨链交易执行
7. 在 `OptimismPortal.finalizeWithdrawalTransaction()` 中会转发执行用户的交易，具体会使用 `call(gas, target, value, ...)` 来完成，其中 `gas` 需要大于或等于用户所指定的 `gasLimit`
8. 如果交易中 `value` 不为 0，则会从 `OptimismPortal` 合约转出指定数量的 ETH 到 `target` 地址
9. 交易一旦被执行，无论成功与否，都不可以再重新执行了

我们忽略前面在 L2 发起交易和在 L1 证明的步骤，只关注 6，7，8 步中交易执行部分。`OptimismPortal.finalizeWithdrawalTransaction()` 函数执行用户交易相关的代码片段：

> 引用的代码片段均来自于 Optimism Bedrock 在 Sherlock contest 的代码，版本为 [3f4b3c3281](https://github.com/ethereum-optimism/optimism/tree/3f4b3c328153a8aa03611158b6984d624b17c1d9)

```Solidity
function finalizeWithdrawalTransaction(Types.WithdrawalTransaction memory _tx) external {
    ...

    bytes32 withdrawalHash = Hashing.hashWithdrawal(_tx);

    // Check that this withdrawal has not already been finalized, this is replay protection.
    require(
        finalizedWithdrawals[withdrawalHash] == false,
        "OptimismPortal: withdrawal has already been finalized"
    );

    // Mark the withdrawal as finalized so it can't be replayed.
    finalizedWithdrawals[withdrawalHash] = true;

    // We want to maintain the property that the amount of gas supplied to the call to the
    // target contract is at least the gas limit specified by the user. We can do this by
    // enforcing that, at this point in time, we still have gaslimit + buffer gas available.
    require(
        gasleft() >= _tx.gasLimit + FINALIZE_GAS_BUFFER,
        "OptimismPortal: insufficient gas to finalize withdrawal"
    );

    // Set the l2Sender so contracts know who triggered this withdrawal on L2.
    l2Sender = _tx.sender;
    bool success = SafeCall.call(
        _tx.target,
        gasleft() - FINALIZE_GAS_BUFFER,
        _tx.value,
        _tx.data
    );

    ...
}
```

上面的代码片段中，首先会通过交易 hash 检查交易是否已经被执行过了，接着会确认 `gasleft() > _tx.gasLimit + 20000`(`FINALIZE_GAS_BUFFER = 20000`)，即剩余的 gas 需要大于用户指定的 gasLimit，然后使用 `SafeCall.call` 来执行用户的交易。

再来看看 `SafeCall.call` 的实现：

```Solidity
library SafeCall {
    function call(
        address _target,
        uint256 _gas,
        uint256 _value,
        bytes memory _calldata
    ) internal returns (bool) {
        bool _success;
        assembly {
            _success := call(
                _gas, // gas
                _target, // recipient
                _value, // ether value
                add(_calldata, 0x20), // inloc
                mload(_calldata), // inlen
                0, // outloc
                0 // outlen
            )
        }
        return _success;
    }
}
```

可以看到最终是调用 `CALL` opcode 来完成用户交易的转发，并且用户的交易成功与否都不会影响此函数的运行。这意味着，当用户在跨链转移 ETH 时，即使用户的交易失败了，`OptimismPortal.finalizeWithdrawalTransaction()` 也会返回成功，并且用户也无法再次发起交易来取回 ETH。

在 Sherlock audit contest 中，针对这段代码，一共有 2 个 high 级别的 bug，其中一个就是利用 gasLimit，让用户的交易 gas 不足而进行的攻击，具体细节在：[Malicious user can finalize other’s withdrawal with less than specified gas limit, leading to loss of funds](https://github.com/sherlock-audit/2023-01-optimism-judging/issues/109)。

这个 bug 的原因是 `OptimismPortal.finalizeWithdrawalTransaction()` 在检查确认剩余 gas，到实际执行用户交易之间，进行了 `SSTORE` 等其他操作，这些操作会消耗 gas，导致实际执行交易时，在  `CALL` 中指定的 gas 会小于用户指定的 gasLimit，这会导致在极端情况下本来可以成功的交易因为 gas 不足而失败。

要修复这个 bug，可以通过这样的方式来完善这个检查：

```Solidity
require(gasleft() >= _tx.gasLimit + FINALIZE_GAS_BUFFER + 5_122); // Add more on gas buffer
```

这里的 `5_122` 就是在检查完成之后，`CALL` 之前，此函数所需要消耗的额外 gas。这样就可以保证在执行交易之前，传递给 `CALL` 的 gas 大于用户所指定的 gasLimit。

但是问题真的解决了吗？

## EIP-150/EIP-114 and the 63/64 gas rule

在无意中翻越 Solidity Docs 时，我留意到了这样一种攻击模式([Call Stack Depth](https://docs.soliditylang.org/en/v0.8.19/security-considerations.html#call-stack-depth))：

![call-stack-depth](/img/in-post/a-pitfall-in-evm-call/call-stack-depth.png)

在以太坊实现中，EVM Call Stack 不能超过 1024，否则交易失败。在早期，这个规则给了攻击中故意制造 Call Stack 爆炸从而让交易失败的可能。为了解决这个问题，在 [EIP-150](https://eips.ethereum.org/EIPS/eip-150) 中，通过引入 63/64 规则的方式，使得这种攻击不再成为可能。

具体做法是，EIP-150 之后，当 EVM 在执行 Call/StaticCall/DelegateCall 时，gasLimit 最多不可以超过当前剩余的 gas 的 63/64，这样一来，如果 Call Stack 过多，可用 gasLimit 会不断减少，创造 1024 个 Call Stack 已经成为了不可能：`1 * (63/64)^1024 = 0.0000000976`。

为了更深入的理解这个 63/64 规则，我翻看了一下 go-ethereum 的代码。我们先回顾一下 `CALL` opcode，它需要 7 个参数：`gas`, `address`, `value`, `argsOffset`, `argsSize`, `retOffset`, `retSize`（参考 [CALL opcode](https://www.evm.codes/#f1?fork=merge)）。这些参数都是在 EVM 的栈中保存的，而 `gas` 参数是在栈顶的第一个参数。

在 go-ethereum 中，`CALL` 的执行函数是这样的，[core/vm/instructions.go > opCall](https://github.com/ethereum/go-ethereum/blob/58d0f6440ba82a0cbf327ca77b7cd795c2d687c6/core/vm/instructions.go#L664-L703)：

```go
func opCall(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
    stack := scope.Stack
    // Pop gas. The actual gas in interpreter.evm.callGasTemp.
    // We can use this as a temporary value
    temp := stack.pop()
    gas := interpreter.evm.callGasTemp
    // Pop other call parameters.
    addr, value, inOffset, inSize, retOffset, retSize := stack.pop(), stack.pop(), stack.pop(), stack.pop(), stack.pop(), stack.pop()
    toAddr := common.Address(addr.Bytes20())

    ...

    ret, returnGas, err := interpreter.evm.Call(scope.Contract, toAddr, args, gas, bigVal)

    ...

    return ret, nil
}
```

我们可以看到在函数中直接忽略了栈顶的第一个参数 `gas`，而是使用了 `interpreter.evm.callGasTemp` 来作为 gasLimit 的值。

我们再来看看 `interpreter.evm.callGasTemp` 是如何赋值的，首先是在 [core/vm/gas_table.go > gasCall](https://github.com/ethereum/go-ethereum/blob/58d0f6440ba82a0cbf327ca77b7cd795c2d687c6/core/vm/gas_table.go#L389)：

```go
func gasCall(evm *EVM, contract *Contract, stack *Stack, mem *Memory, memorySize uint64) (uint64, error) {
    ...
    evm.callGasTemp, err = callGas(evm.chainRules.IsEIP150, contract.Gas, gas, stack.Back(0))
    ...
    return gas, nil
}
```

这里调用了 `callGas()` 来给 `evm.callGasTemp` 赋值，其中最后一个参数是 `stack.Back(0)`，也就是栈顶的第一个参数 `gas`。

接着在 [core/vm/gas.go > callGas](https://github.com/ethereum/go-ethereum/blob/58d0f6440ba82a0cbf327ca77b7cd795c2d687c6/core/vm/gas.go#L37-L53) 中：

```go
// callGas returns the actual gas cost of the call.
//
// The cost of gas was changed during the homestead price change HF.
// As part of EIP 150 (TangerineWhistle), the returned gas is gas - base * 63 / 64.
func callGas(isEip150 bool, availableGas, base uint64, callCost *uint256.Int) (uint64, error) {
    if isEip150 {
        availableGas = availableGas - base
        gas := availableGas - availableGas/64
        // If the bit length exceeds 64 bit we know that the newly calculated "gas" for EIP150
        // is smaller than the requested amount. Therefore we return the new gas instead
        // of returning an error.
        if !callCost.IsUint64() || gas < callCost.Uint64() {
            return gas, nil
        }
    }
    if !callCost.IsUint64() {
        return 0, ErrGasUintOverflow
    }

    return callCost.Uint64(), nil
}
```

这段代码实现了 63/64 的规则：

- 当 `gas` 参数为 0 时， gasLimit 的值就是 `availableGas - availableGas/64`
- 如果 `gas` 参数不为 0，那么 gasLimit 的值就是 `min(gas, availableGas - availableGas/64)`

也就是说，**在 `CALL` opcode 的参数中，gas 参数仅表示 gasLimit 的最大值，同时 gasLimit 的值也不可以超过 `availableGas - availableGas/64`**。而这个细节几乎没有在任何关于 EVM opcode 的文档中被提及。

为了方便理解，我们举一个例子：

假设当前 EVM 执行上下文中 `availableGas` 为 1000，我们进行一次 `CALL` 操作： `call(990, ...)`，指定 gasLimit 为 990，但是实际上，gasLimit 会被 EVM 设定为 `availableGas - availableGas/64` 的值就是 985，也就是说在实际调用时 gasLimit 为 985。

上面的例子中，可能会引入一个问题：如果这个 `CALL` 成功执行至少需要消耗 990 gas，那么这个 `CALL` 执行会失败，因为执行时的 gasLimit 为 985。

## Exploit Optimism

在仔细研究了 63/64 规则之后，我开始思考这个规则是否会导致任何安全问题。而 Optimism Bedrock 中的 Withdrawal 交易（L2 -> L1）就是一个很好的例子，因为它不可以重试，执行失败可能导致用户 ETH 被锁定在合约中。

我们来回顾一下 Optimism 的 withdrawal 交易的执行过程：

```Solidity
function finalizeWithdrawalTransaction(Types.WithdrawalTransaction memory _tx) external {
    ...
    require(
        gasleft() >= _tx.gasLimit + FINALIZE_GAS_BUFFER,
        "OptimismPortal: insufficient gas to finalize withdrawal"
    );

    ...

    bool success = SafeCall.call(
        _tx.target,
        gasleft() - FINALIZE_GAS_BUFFER,
        _tx.value,
        _tx.data
    );
    ...
}
```

在执行用户的 withdrawal 交易时，OP 先确认当前剩余的 gas 大于 `_tx.gasLimit + FINALIZE_GAS_BUFFER`，然后再调用 `SafeCall.call()` 来执行用户的 withdrawal 交易。

我们假设 Optimism 修复了上面提到的 Sherlock 中的[问题](https://github.com/sherlock-audit/2023-01-optimism-judging/issues/109)，将第一步检查中的 `FINALIZE_GAS_BUFFER` 由 `20_000` 增大为 `25_122`。这样修改后，就能保证在 `SafeCall.call` 调用用户交易时，gasLimit 一定大于用户设定的 gasLimit 吗？

因为 63/64 规则的存在，答案是否定的。如果在调用 `SafeCall.call()` 时 `gasleft() - FINALIZE_GAS_BUFFER` 参数的值大于 `gasleft() * 63 / 64`，EVM 就会使用 `gasleft() * 63 / 64` 作为实际的 gasLimit，而不是代码中指定的 gasLimit，这样就有可能导致本来可以成功的用户交易执行失败。

假设第一步检查中 `FINALIZE_GAS_BUFFER = 25_122`，为了构造出满足上面条件的 case，我们需要：

1. `gasleft()  >= _tx.gasLimit + 20_000 + 5_122`
2. `(gasleft() - 20_000) > gasleft() * 63 / 64`

得出，当 `_tx.gasLimit > 1,254,878` 时，就有可能会因为 gasLimit 小于用户的设定而导致用户交易失败。

## 测试用例

我们参考 [issue-109](https://github.com/sherlock-audit/2023-01-optimism-judging/issues/109) 中的测试用例，可以验证这个攻击 case 是否有效。

简化后的 `Portal` 合约和用户需要调用的合约：

```Solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {console} from "forge-std/console.sol";

library SafeCall {
    /**
     * @notice Perform a low level call without copying any returndata
     *
     * @param _target   Address to call
     * @param _gas      Amount of gas to pass to the call
     * @param _value    Amount of value to pass to the call
     * @param _calldata Calldata to pass to the call
     */
    function call(
        address _target,
        uint256 _gas,
        uint256 _value,
        bytes memory _calldata
    ) internal returns (bool) {
        console.log("gas provided to call: ", _gas, ", gasLeft: ", gasleft());
        bool _success;
        assembly {
            _success := call(
                _gas, // gas
                _target, // recipient
                _value, // ether value
                add(_calldata, 0x20), // inloc
                mload(_calldata), // inlen
                0, // outloc
                0 // outlen
            )
        }
        return _success;
    }
}

contract GasUser {
    uint[] public s;

    function store(uint i) public {
        for (uint j = 0; j < i; j++) {
            s.push(1);
        }
    }
}

contract Portal {
    address l2Sender;

    struct Transaction {
        uint gasLimit;
        address sender;
        address target;
        uint value;
        bytes data;
    }

    constructor(address _l2Sender) {
        l2Sender = _l2Sender;
    }

    function execute(Transaction memory _tx) public {
        console.log("gas left in the beginning: ", gasleft());
        require(
            gasleft() >= _tx.gasLimit + 20_000 + 5_122,
            "OptimismPortal: insufficient gas to finalize withdrawal"
        );

        // Set the l2Sender so contracts know who triggered this withdrawal on L2.
        l2Sender = _tx.sender;

        // Trigger the call to the target contract. We use SafeCall because we don't
        // care about the returndata and we don't want target contracts to be able to force this
        // call to run out of gas via a returndata bomb.
        bool success = SafeCall.call(
            _tx.target,
            gasleft() - 20_000,
            _tx.value,
            _tx.data
        );
    }
}
```

这里我使用 `console.log` 加了一些方便调试的 log。

测试用例：

```Solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import {GasUser, Portal} from "../src/Portal.sol";

contract GasUserTest is Test {
    Portal public c;
    GasUser public gu;

    function setUp() public {
        c = new Portal(0x000000000000000000000000000000000000dEaD);
        gu = new GasUser();
    }

    function testGasLimit() public {
        gu.store{gas: 1_368_975}(60);
        assert(gu.s(59) == 1);
    }

    function _executePortalWithGivenGas(uint gas) public {
        c.execute{gas: gas}(
            Portal.Transaction({
                gasLimit: 1_368_975,
                sender: address(69),
                target: address(gu),
                value: 0,
                data: abi.encodeWithSignature("store(uint256)", 60)
            })
        );
    }

    function testPortalCatchesGasTooSmall() public {
        vm.expectRevert(
            bytes("OptimismPortal: insufficient gas to finalize withdrawal")
        );
        _executePortalWithGivenGas(1_368_975 + 20_000);
    }

    function testPortalBugWithEnoughGas() public {
        _executePortalWithGivenGas(1_368_975 + 30_000);

        // It now reverts because the array has a length of 0.
        vm.expectRevert();
        gu.s(0);
    }

    function testPortalSucceedsWithEnoughGas() public {
        _executePortalWithGivenGas(1_368_975 + 50_000); // add more gas to make it succeed
        assert(gu.s(59) == 1);
    }
}
```

测试中会请求 `Portal` 来调用 `GasUser.store(60)`，此函数正常执行需要的 gas 为 `1_368_975`。我们来看 `testPortalBugWithEnoughGas` 这个 case：

```bash
❯ forge test -vvv --match-test testPortalBugWithEnoughGas
...

Running 1 test for test/Portal.t.sol:PortalTest
[PASS] testPortalBugWithEnoughGas() (gas: 1391120)
Logs:
  gas left in the beginning:  1397920
  gas provided to call: 1369583 , gasLeft: 1389336

Test result: ok. 1 passed; 0 failed; finished in 517.92µs
```

可以看到，log 中输出的 `gas provided to call: 1369583, gasLeft: 1389336` 表明 call 中指定的 gasLimit 已经大于 `1_368_975`，但是由于 63/64 规则，实际执行时的 gasLimit 并没有这么多（gasLeft 不够多），最后导致执行失败。这就给了攻击者故意选定一个 gasLimit 来让用户的交易失败的机会，结果可能导致用户的 ETH 被锁定在合约中。

## Bug 修复

正当我以为自己发现了一个新 bug，想报告给 Optimism 团队时，打开 GitHub 检查了一下他们的最新代码，发现他们已经意识到了这个问题，并且在一周前就完成了修复，可以看他们最新版 [SafeCall.callWithMinGas()](https://github.com/ethereum-optimism/optimism/blob/2351ed462355cd516ddb8176d1ffdae607d1b7ed/packages/contracts-bedrock/contracts/libraries/SafeCall.sol#L56-L63) 函数中的修复方式。

好吧，看来与 Bug Bounty 无缘了，不过这也算是一个值得写篇文章记录的「坑」了。
