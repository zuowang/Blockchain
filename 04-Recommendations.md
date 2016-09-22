
## Recommendations for Smart Contract Security in Solidity
对Solidity编写智能合约的安全建议

### External Calls 外部调用

#### Avoid external calls when possible尽量避免外部调用
<a name="avoid-external-calls"></a>

Calls to untrusted contracts can introduce several unexpected risks or errors. External calls may execute malicious code in that contract _or_ any other contract that it depends upon. As such, every external call should be treated as a potential security risk, and removed if possible. When it is not possible to remove external calls, use the recommendations in the rest of this section to minimize the danger.
对不受信任合同的调用可以带来几个意外的风险或错误。外部调用可能在该合同里以及依赖于该合同的其他任何合约执行恶意代码。因此，每一个外部调用都应视为潜在的安全风险，如果可能删除每个外部调用。当外部调用不能删除，请使用在本节的其余建议尽量减少危险。

<a name="avoid-call-value"></a>

#### Use `send()`, avoid `call.value()`使用”send”避免使用“call.value”

When sending Ether, use `someAddress.send()` and avoid `someAddress.call.value()()`.
当发送以太币时，使用someAddress.send() 避免使用someAddress.call.value.
External calls such as `someAddress.call.value()()` can trigger malicious code. While `send()` also triggers code, it is safe because it only has access to gas stipend of 2,300 gas. Currently, this is only enough to log an event, not enough to launch an attack.
如 'someAddress.call.value()()' 调用外部可以触发恶意代码。虽然 send （）' 也触发代码，它是安全的，因为它只能够消耗的 2,300个 油气津贴。目前，这只是足够记录一个事件，还不足以发动攻击。

```
// bad 
if(!someAddress.call.value(100)()) { // forwards remaining gas
    // Some failure code
}

// good
if(!someAddress.send(100)) { // gas is limited
    // Some failure code
}
```

<a name="handle-external-errors"></a>

#### Handle errors in external calls在外部调用中处理错误
 
Solidity offers low-level call methods that work on raw addresses: `address.call()`, `address.callcode()`, `address.delegatecall()`, and `address.send`. These low-level methods never throw an exception, but will return `false` if the call encounters an exception. On the other hand, *contract calls* (e.g., `ExternalContract.doSomething()`) will automatically propagate a throw (for example, `ExternalContract.doSomething()` will also `throw` if `doSomething()` throws).
Solidity编程语言提供工作在裸地址的低级调用方法: 'address.call()'、 'address.callcode()'、 'address.delegatecall()' 和 'address.send'。这些低级的方法永远不会引发异常，但如果调用遇到异常将返回 false。另一方面，* 合约调用 * (例如，' ExternalContract.doSomething()') 将自动传播一个异常抛出 （例如，'ExternalContract.doSomething()' 将异常抛出  如果'doSomething()'有异常抛出）。

If you choose to use the low-level call methods, make sure to handle the possibility that the call will fail, by checking the return value. Note that the [Call Depth Attack](https://github.com/ConsenSys/smart-contract-best-practices/#call-depth-attack) can cause *any* call to fail, even if the external contract's code is working and non-malicious.
如果您选择使用低级的调用方法，通过检查返回值确保合理处理这些调用可能的失败，。请注意，[调用深度攻击] (https://github.com/ConsenSys/smart-contract-best-practices/#call-depth-attack) 可以导致 * 任何 * 调用失败，即使外部合同代码是正常和非恶意。

```
// bad
someAddress.send(55); // doesn't check return value for failure
someAddress.call.value(55)(); // this is doubly dangerous, as it will forward all remaining gas and doesn't check for result
someAddress.call.value(100)(bytes4(sha3("deposit()"))); // if deposit throws an exception, the raw call() will only return false and transaction will NOT be reverted
如果存款引发异常，原始 call （） 将只返回 false 和交易将无法恢复

// good
if(!someAddress.send(55)) { // checks return value for failure
    // Some failure code
}

ExternalContract(someAddress).deposit.value(100); // throws exception on failure
```

<a name="expect-control-flow-loss"></a>

#### Don't make control flow assumptions after external calls
在外部调用后不要想当然的执行控制流程
 
Whether using *raw calls* or *contract calls*, assume that malicious code will execute if `ExternalContract` is untrusted. Even if `ExternalContract` is not malicious, malicious code can be executed by any contracts *it* calls. One particular danger is malicious code may hijack the control flow, leading to race conditions. (See [Race Conditions](https://github.com/ConsenSys/smart-contract-best-practices/#race-conditions) for a fuller discussion of this problem).
如果 '外部合约' 是不受信任无论使用 * 原始调用 * 或 * 合同调用 *，都有可能会执行恶意代码。即使 '外部合约' 不是恶意的，恶意代码也可以由任何合同自己调用来执行。一个特别危险的是恶意代码可能劫持的控制流，导致争用场景。
(See[Race Conditions](https://github.com/ConsenSys/smart-contract-best-practices/#race-conditions) for a fuller discussion of this problem).关于这个问题更加全面的讨论请看（https://github.com/ConsenSys/smart-contract-best-practices/#race-conditions)

<a name="favor-pull-over-push-payments"></a>

#### Favor *pull* over *push* for external calls
对于外部调用倾向于 * 拉 * 而不是 * 推 * 
As we've seen, external calls can fail for a number of reasons, including external errors and malicious [Call Depth Attacks](https://github.com/ConsenSys/smart-contract-best-practices/#call-depth-attack). To minimize the damage caused by such failures, it is often better to isolate each external call into its own transaction that can be initiated by the recipient of the call. This is especially relevant for payments, where it is better to let users withdraw funds rather than push funds to them automatically. (This also reduces the chance of [problems with the gas limit](https://github.com/ConsenSys/smart-contract-best-practices/#dos-with-block-gas-limit).)
正如我们已经看到，外部调用会有很多失败的原因，包括外部错误和恶意代码 [调用深度攻击] (https://github.com/ConsenSys/smart-contract-best-practices/#call-depth-attack) 的数目。为了尽量减少这种失败所造成的损害，它最好经常地将每个外部调用隔离成它自己的事务，这些事务可以通过接受方的调用所初始化。特别是对于付款相关事务，最好地让用户撤回资金，而不是自动把资金返回用户。

```
// bad
contract auction {
    address highestBidder;
    uint highestBid;

    function bid() {
        if (msg.value < highestBid) throw;

        if (highestBidder != 0) {
            if (!highestBidder.send(highestBid)) { // if this call consistently fails, no one else can bid
                throw;
            }
        }

       highestBidder = msg.sender;
       highestBid = msg.value;
    }
}

// good
contract auction {
    address highestBidder;
    uint highestBid;
    mapping(address => uint) refunds;

    function bid() external {
        if (msg.value < highestBid) throw;

        if (highestBidder != 0) {
            refunds[highestBidder] += highestBid; // record the refund that this user can claim
        }

        highestBidder = msg.sender;
        highestBid = msg.value;
    }

    function withdrawRefund() external {
        uint refund = refunds[msg.sender];
        refunds[msg.sender] = 0;
        if (!msg.sender.send(refund)) {
            refunds[msg.sender] = refund; // reverting state because send failed
        }
    }
}
```

<a name="mark-untrusted-contracts"></a>

#### Mark untrusted contracts
表示不被信任的合约
When interacting with external contracts, name your variables, methods, and contract interfaces in a way that makes it clear that interacting with them is potentially unsafe. This applies to your own functions that call external contracts.
与外部合同进行交互时，用一种清晰的方式来命名您的变量、 方法和合同接口，标明与他们交互时存在潜在的不安全。这适用于您自己调用外部合约的函数。

```
// bad
Bank.withdraw(100); // Unclear whether trusted or untrusted
这种命名对于是否被信任不清楚
function makeWithdrawal(uint amount) { // Isn't clear that this function is potentially unsafe 这种命名对于该函数是否潜在安全不清楚
    UntrustedBank.withdraw(amount);
}

// good
UntrustedBank.withdraw(100); // untrusted external call 标明非信任的外部调用
TrustedBank.withdraw(100); // external but trusted bank contract maintained by XYZ Corp 标明这是被XYZ公司维护的外部可信任银行合约

function makeUntrustedWithdrawal(uint amount) {
    UntrustedBank.withdraw(amount);
}
```

<a name="beware-rounding-with-integer-division"></a>

### Beware rounding with integer division
当心与整数除法的舍入
 
All integer divison rounds down to the nearest integer. If you need more precision, consider using a multiplier, or store both the numerator and denominator.

(In the future, Solidity will have a fixed-point type, which will make this easier.)

```所有的整数除法都会四舍五入到最接近的整数。如果你需要更高的精度，请考虑使用乘数，或存储的分子和分母。

（在未来，Solidity编程语言会有定点的类型，这将使这更容易）。

// bad
uint x = 5 / 2; // Result is 2, all integer divison rounds DOWN to the nearest integer

// good
uint multiplier = 10;
uint x = (5 * multiplier) / 2;

uint numerator = 5;
uint denominator = 2;
```

### Remember that Ether can be forcibly sent to an account
记住，以太币可以强制送往某个帐户

Beware of coding an invariant that strictly checks the balance of a contract.

An attacker can forcibly send wei to any account and this cannot be prevented (not even with a fallback function that does a `throw`).

The attacker can do this by creating a contract, funding it with 1 wei, and invoking
`selfdestruct(victimAddress)`.  No code is invoked in `victimAddress`, so it
cannot be prevented.
提防不变量编码用于严格检查合同余额的的。

攻击者可以强行向任何帐户发送wei，这无法被阻止 （甚至不能做 '抛出功能' 的回退函数）。

攻击者可以通过创建一个合约、资助该合约  1 魏，并调用
' selfdestruct(victimAddress)'。 没有代码调用在 'victimAddress'，所以它
无法阻挡。


### Remember that on-chain data is public
牢记链上数据是公开的
Many applications require submitted data to be private up until some point in time in order to work. Games (eg. on-chain rock-paper-scissors) and auction mechanisms (eg. sealed-bid second-price auctions) are two major categories of examples. If you are building an application where privacy is an issue, take care to avoid requiring users to publish information too early.
许多应用程序要求提交的数据是私有的直到某些时间点才能工作。游戏 (如。 链上剪刀石头) 和拍卖机制 (如.密封投标第二价格拍卖) 两大类的实例。如果你正在构建的应用程序隐私部分是个问题的话，小心避免要求用户太早发布信息。

Examples:

* In rock paper scissors, require both players to submit a hash of their intended move first, then require both players to submit their move; if the submitted move does not match the hash throw it out.
* 在石头纸剪刀游戏中，要求这两名玩家，首先提交他们预期举动的哈希值，然后需要两个玩家提交他们的举动;如果提交的移动不匹配哈希把它扔掉。

* In an auction, require players to submit a hash of their bid value in an initial phase (along with a deposit greater than their bid value), and then submit their action bid value in the second phase.
* 在拍卖中，需要玩家在初始阶段 （包括存款大于其 投标 的值），提交他们的出投标价的哈希，然后在第二阶段提交他们投标价 的值。

* When developing an application that depends on a random number generator, the order should always be (1) players submit moves, (2) random number generated, (3) players paid out. The method by which random numbers are generated is itself an area of active research; current best-in-class solutions include Bitcoin block headers (verified through http://btcrelay.org), hash-commit-reveal schemes (ie. one party generates a number, publishes its hash to "commit" to the value, and then reveals the value later) and [RANDAO](http://github.com/randao/randao).
* 当开发的应用程序依赖于一个随机数发生器，顺序永远应该是球员 （1） 玩家提交的招式、 （2） 生成的随机数 （3）玩家付款。所生成的随机编号的方法本身就是研究的一个活跃领域;当前的同类最佳解决方案包括比特币块标题 （通过 http://btcrelay.org）验证，哈希提交揭示计划 (ie。 一方产生一个数、 发布其哈希值来"承诺"该随机数，然后揭示的该随机数) 和 [RANDAO] (http://github.com/randao/randao)。

* If you are implementing a frequent batch auction, a hash-commit scheme is also desirable.
如果你要频繁实现批量拍卖，哈希提交计划也是可取的。

### In 2-party or N-party contracts, beware of the possibility that some participants may "drop offline" and not return
2 方或 N 方在合同中，要小心一些与会者可能"脱机"，并不返回的可能性
 
Do not make refund or claim processes dependent on a specific party performing a particular action with no other way of getting the funds out. For example, in a rock-paper-scissors game, one common mistake is to not make a payout until both players submit their moves; however, a malicious player can "grief" the other by simply never submitting their move - in fact, if a player sees the other player's revealed move and determiners that they lost, they have no reason to submit their own move at all. This issue may also arise in the context of state channel settlement. When such situations are an issue, (1) provide a way of circumventing non-participating participants, perhaps through a time limit, and (2) consider adding an additional economic incentive for participants to submit information in all of the situations in which they are supposed to do so.
不要陷入退款或索赔过程依赖于具体一方执行特定的操作，而且没有其他办法提取资金。例如，在剪刀石头游戏中，一个常见的错误是不进行判决，直到这两名球员提交他们的动作;然而，恶意玩家可以对其他玩家"耍无赖"，只需简单的不提交他的表决 — — 事实上，如果玩家可以看到其他玩家的表决和判断，他们便没有理由提交他们自己的表决了。这个问题也可能发生在国家间方案。当这种情况下是一个问题时，（1） 提供一种规避消极参与者，或许可以通过有时间限制的方法，和 （2） 考虑添加额外的经济诱因，鼓励与会者在所有情况下提供他们应该资料。

<a name="keep-fallback-functions-simple"></a>

### Keep fallback functions simple
保持反馈函数的简洁
[Fallback functions](http://solidity.readthedocs.io/en/latest/contracts.html#fallback-function) are called when a contract is sent a message with no arguments (or when no function matches), and only has access to 2,300 gas when called from a `.send()`. If you wish to be able to receive Ether from a `.send()`, the most you can do in a fallback function is log an event. Use a proper function if a computation or more gas is required.
[反馈函数](http://solidity.readthedocs.io/en/latest/contracts.html#fallback-function) 当一个合约不带参数的发送一个信息是会被调用 （或当没有匹配函数） 的消息时，当 调用'.send()'时只消耗2,300 气体。如果你想要能够从 '.send()'接收以太币，你在反馈函数中可以做最多的是记录一个事件。如果需要一次运算或更多的气体请使用适当的函数。

```
// bad
function() { balances[msg.sender] += msg.value; }

// good
function() { throw; }
function deposit() external { balances[msg.sender] += msg.value; }

function() { LogDepositReceived(msg.sender); }
```

<a name="mark-visibility"></a>

### Explicitly mark visibility in functions and state variables
对于函数和状态变量应该明确醒目标记
 
Explicitly label the visibility of functions and state variables. Functions can be specified as being `external`, `public`, `internal` or `private`. For state variables, `external` is not possible. Labeling the visibility explicitly will make it easier to catch incorrect assumptions about who can call the function or access the variable.
对于函数和状态变量应该明确醒目标记。函数可以指定作为 '外部'、 '公开'、 '内部' 或 '私人'。对于状态变量，定义为'外部' 是不可能的。明确醒目的标记会容易抓到哪些想当然谁可以调用函数或访问变量的不正确判断。
```
// bad
uint x; // the default is private for state variables, but it should be made explicit
        // 对于状态变量默认是私有的，但是也应该被明确标识
function transfer() { // the default is public 默认是共有的。
    // public code
}

// good
uint private y;
function transfer() public {
    // public code
}

function internalAction() internal {
    // internal code
}
```


<a name="beware-division-by-zero"></a>

### Beware division by zero
当心零除
Currently, Solidity [returns zero](https://github.com/ethereum/solidity/issues/670) and does not `throw` an exception when a number is divided by zero. You therefore need to check for division by zero manually.
目前Solidity编程语言中当一个数除以零时不会抛出意外。因此你必须手动检查零除
```
// bad
function divide(uint x, uint y) returns(uint) {
    return x / y;
}

// good
function divide(uint x, uint y) returns(uint) {
   if (y == 0) { throw; }

   return x / y;
}
```

<a name="differentiate-functions-events"></a>

### Differentiate functions and events
函数与事件的区别
Favor capitalization and a prefix in front of events (we suggest *Log*), to prevent the risk of confusion between functions and events. For functions, always start with a lowercase letter, except for the constructor.
倾向于在事件前面大写前缀 (我们建议 * Log  *)，以防止函数和事件之间混淆的风险。对于函数，总是以一个小写字母开头，除了构造函数开头。

```
// bad
event Transfer() {}
function transfer() {}

// good
event LogTransfer() {}
function transfer() external {}
```

