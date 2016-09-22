
## Recommendations for Smart Contract Security in Solidity
��Solidity��д���ܺ�Լ�İ�ȫ����

### External Calls �ⲿ����

#### Avoid external calls when possible���������ⲿ����
<a name="avoid-external-calls"></a>

Calls to untrusted contracts can introduce several unexpected risks or errors. External calls may execute malicious code in that contract _or_ any other contract that it depends upon. As such, every external call should be treated as a potential security risk, and removed if possible. When it is not possible to remove external calls, use the recommendations in the rest of this section to minimize the danger.
�Բ������κ�ͬ�ĵ��ÿ��Դ�����������ķ��ջ�����ⲿ���ÿ����ڸú�ͬ���Լ������ڸú�ͬ�������κκ�Լִ�ж�����롣��ˣ�ÿһ���ⲿ���ö�Ӧ��ΪǱ�ڵİ�ȫ���գ��������ɾ��ÿ���ⲿ���á����ⲿ���ò���ɾ������ʹ���ڱ��ڵ����ཨ�龡������Σ�ա�

<a name="avoid-call-value"></a>

#### Use `send()`, avoid `call.value()`ʹ�á�send������ʹ�á�call.value��

When sending Ether, use `someAddress.send()` and avoid `someAddress.call.value()()`.
��������̫��ʱ��ʹ��someAddress.send() ����ʹ��someAddress.call.value.
External calls such as `someAddress.call.value()()` can trigger malicious code. While `send()` also triggers code, it is safe because it only has access to gas stipend of 2,300 gas. Currently, this is only enough to log an event, not enough to launch an attack.
�� 'someAddress.call.value()()' �����ⲿ���Դ���������롣��Ȼ send ����' Ҳ�������룬���ǰ�ȫ�ģ���Ϊ��ֻ�ܹ����ĵ� 2,300�� ����������Ŀǰ����ֻ���㹻��¼һ���¼����������Է���������

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

#### Handle errors in external calls���ⲿ�����д������
 
Solidity offers low-level call methods that work on raw addresses: `address.call()`, `address.callcode()`, `address.delegatecall()`, and `address.send`. These low-level methods never throw an exception, but will return `false` if the call encounters an exception. On the other hand, *contract calls* (e.g., `ExternalContract.doSomething()`) will automatically propagate a throw (for example, `ExternalContract.doSomething()` will also `throw` if `doSomething()` throws).
Solidity��������ṩ���������ַ�ĵͼ����÷���: 'address.call()'�� 'address.callcode()'�� 'address.delegatecall()' �� 'address.send'����Щ�ͼ��ķ�����Զ���������쳣����������������쳣������ false����һ���棬* ��Լ���� * (���磬' ExternalContract.doSomething()') ���Զ�����һ���쳣�׳� �����磬'ExternalContract.doSomething()' ���쳣�׳�  ���'doSomething()'���쳣�׳�����

If you choose to use the low-level call methods, make sure to handle the possibility that the call will fail, by checking the return value. Note that the [Call Depth Attack](https://github.com/ConsenSys/smart-contract-best-practices/#call-depth-attack) can cause *any* call to fail, even if the external contract's code is working and non-malicious.
�����ѡ��ʹ�õͼ��ĵ��÷�����ͨ����鷵��ֵȷ����������Щ���ÿ��ܵ�ʧ�ܣ�����ע�⣬[������ȹ���] (https://github.com/ConsenSys/smart-contract-best-practices/#call-depth-attack) ���Ե��� * �κ� * ����ʧ�ܣ���ʹ�ⲿ��ͬ�����������ͷǶ��⡣

```
// bad
someAddress.send(55); // doesn't check return value for failure
someAddress.call.value(55)(); // this is doubly dangerous, as it will forward all remaining gas and doesn't check for result
someAddress.call.value(100)(bytes4(sha3("deposit()"))); // if deposit throws an exception, the raw call() will only return false and transaction will NOT be reverted
�����������쳣��ԭʼ call ���� ��ֻ���� false �ͽ��׽��޷��ָ�

// good
if(!someAddress.send(55)) { // checks return value for failure
    // Some failure code
}

ExternalContract(someAddress).deposit.value(100); // throws exception on failure
```

<a name="expect-control-flow-loss"></a>

#### Don't make control flow assumptions after external calls
���ⲿ���ú�Ҫ�뵱Ȼ��ִ�п�������
 
Whether using *raw calls* or *contract calls*, assume that malicious code will execute if `ExternalContract` is untrusted. Even if `ExternalContract` is not malicious, malicious code can be executed by any contracts *it* calls. One particular danger is malicious code may hijack the control flow, leading to race conditions. (See [Race Conditions](https://github.com/ConsenSys/smart-contract-best-practices/#race-conditions) for a fuller discussion of this problem).
��� '�ⲿ��Լ' �ǲ�����������ʹ�� * ԭʼ���� * �� * ��ͬ���� *�����п��ܻ�ִ�ж�����롣��ʹ '�ⲿ��Լ' ���Ƕ���ģ��������Ҳ�������κκ�ͬ�Լ�������ִ�С�һ���ر�Σ�յ��Ƕ��������ܽٳֵĿ��������������ó�����
(See[Race Conditions](https://github.com/ConsenSys/smart-contract-best-practices/#race-conditions) for a fuller discussion of this problem).��������������ȫ��������뿴��https://github.com/ConsenSys/smart-contract-best-practices/#race-conditions)

<a name="favor-pull-over-push-payments"></a>

#### Favor *pull* over *push* for external calls
�����ⲿ���������� * �� * ������ * �� * 
As we've seen, external calls can fail for a number of reasons, including external errors and malicious [Call Depth Attacks](https://github.com/ConsenSys/smart-contract-best-practices/#call-depth-attack). To minimize the damage caused by such failures, it is often better to isolate each external call into its own transaction that can be initiated by the recipient of the call. This is especially relevant for payments, where it is better to let users withdraw funds rather than push funds to them automatically. (This also reduces the chance of [problems with the gas limit](https://github.com/ConsenSys/smart-contract-best-practices/#dos-with-block-gas-limit).)
���������Ѿ��������ⲿ���û��кܶ�ʧ�ܵ�ԭ�򣬰����ⲿ����Ͷ������ [������ȹ���] (https://github.com/ConsenSys/smart-contract-best-practices/#call-depth-attack) ����Ŀ��Ϊ�˾�����������ʧ������ɵ��𺦣�����þ����ؽ�ÿ���ⲿ���ø�������Լ���������Щ�������ͨ�����ܷ��ĵ�������ʼ�����ر��Ƕ��ڸ������������õ����û������ʽ𣬶������Զ����ʽ𷵻��û���

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
��ʾ�������εĺ�Լ
When interacting with external contracts, name your variables, methods, and contract interfaces in a way that makes it clear that interacting with them is potentially unsafe. This applies to your own functions that call external contracts.
���ⲿ��ͬ���н���ʱ����һ�������ķ�ʽ���������ı����� �����ͺ�ͬ�ӿڣ����������ǽ���ʱ����Ǳ�ڵĲ���ȫ�������������Լ������ⲿ��Լ�ĺ�����

```
// bad
Bank.withdraw(100); // Unclear whether trusted or untrusted
�������������Ƿ����β����
function makeWithdrawal(uint amount) { // Isn't clear that this function is potentially unsafe �����������ڸú����Ƿ�Ǳ�ڰ�ȫ�����
    UntrustedBank.withdraw(amount);
}

// good
UntrustedBank.withdraw(100); // untrusted external call ���������ε��ⲿ����
TrustedBank.withdraw(100); // external but trusted bank contract maintained by XYZ Corp �������Ǳ�XYZ��˾ά�����ⲿ���������к�Լ

function makeUntrustedWithdrawal(uint amount) {
    UntrustedBank.withdraw(amount);
}
```

<a name="beware-rounding-with-integer-division"></a>

### Beware rounding with integer division
��������������������
 
All integer divison rounds down to the nearest integer. If you need more precision, consider using a multiplier, or store both the numerator and denominator.

(In the future, Solidity will have a fixed-point type, which will make this easier.)

```���е��������������������뵽��ӽ����������������Ҫ���ߵľ��ȣ��뿼��ʹ�ó�������洢�ķ��Ӻͷ�ĸ��

����δ����Solidity������Ի��ж�������ͣ��⽫ʹ������ף���

// bad
uint x = 5 / 2; // Result is 2, all integer divison rounds DOWN to the nearest integer

// good
uint multiplier = 10;
uint x = (5 * multiplier) / 2;

uint numerator = 5;
uint denominator = 2;
```

### Remember that Ether can be forcibly sent to an account
��ס����̫�ҿ���ǿ������ĳ���ʻ�

Beware of coding an invariant that strictly checks the balance of a contract.

An attacker can forcibly send wei to any account and this cannot be prevented (not even with a fallback function that does a `throw`).

The attacker can do this by creating a contract, funding it with 1 wei, and invoking
`selfdestruct(victimAddress)`.  No code is invoked in `victimAddress`, so it
cannot be prevented.
������������������ϸ����ͬ���ĵġ�

�����߿���ǿ�����κ��ʻ�����wei�����޷�����ֹ ������������ '�׳�����' �Ļ��˺�������

�����߿���ͨ������һ����Լ�������ú�Լ  1 κ��������
' selfdestruct(victimAddress)'�� û�д�������� 'victimAddress'��������
�޷��赲��


### Remember that on-chain data is public
�μ����������ǹ�����
Many applications require submitted data to be private up until some point in time in order to work. Games (eg. on-chain rock-paper-scissors) and auction mechanisms (eg. sealed-bid second-price auctions) are two major categories of examples. If you are building an application where privacy is an issue, take care to avoid requiring users to publish information too early.
���Ӧ�ó���Ҫ���ύ��������˽�е�ֱ��ĳЩʱ�����ܹ�������Ϸ (�硣 ���ϼ���ʯͷ) ���������� (��.�ܷ�Ͷ��ڶ��۸�����) �������ʵ������������ڹ�����Ӧ�ó�����˽�����Ǹ�����Ļ���С�ı���Ҫ���û�̫�緢����Ϣ��

Examples:

* In rock paper scissors, require both players to submit a hash of their intended move first, then require both players to submit their move; if the submitted move does not match the hash throw it out.
* ��ʯͷֽ������Ϸ�У�Ҫ����������ң������ύ����Ԥ�ھٶ��Ĺ�ϣֵ��Ȼ����Ҫ��������ύ���ǵľٶ�;����ύ���ƶ���ƥ���ϣ�����ӵ���

* In an auction, require players to submit a hash of their bid value in an initial phase (along with a deposit greater than their bid value), and then submit their action bid value in the second phase.
* �������У���Ҫ����ڳ�ʼ�׶� �������������� Ͷ�� ��ֵ�����ύ���ǵĳ�Ͷ��۵Ĺ�ϣ��Ȼ���ڵڶ��׶��ύ����Ͷ��� ��ֵ��

* When developing an application that depends on a random number generator, the order should always be (1) players submit moves, (2) random number generated, (3) players paid out. The method by which random numbers are generated is itself an area of active research; current best-in-class solutions include Bitcoin block headers (verified through http://btcrelay.org), hash-commit-reveal schemes (ie. one party generates a number, publishes its hash to "commit" to the value, and then reveals the value later) and [RANDAO](http://github.com/randao/randao).
* ��������Ӧ�ó���������һ���������������˳����ԶӦ������Ա ��1�� ����ύ����ʽ�� ��2�� ���ɵ������ ��3����Ҹ�������ɵ������ŵķ�����������о���һ����Ծ����;��ǰ��ͬ����ѽ�������������رҿ���� ��ͨ�� http://btcrelay.org����֤����ϣ�ύ��ʾ�ƻ� (ie�� һ������һ������ �������ϣֵ��"��ŵ"���������Ȼ���ʾ�ĸ������) �� [RANDAO] (http://github.com/randao/randao)��

* If you are implementing a frequent batch auction, a hash-commit scheme is also desirable.
�����ҪƵ��ʵ��������������ϣ�ύ�ƻ�Ҳ�ǿ�ȡ�ġ�

### In 2-party or N-party contracts, beware of the possibility that some participants may "drop offline" and not return
2 ���� N ���ں�ͬ�У�ҪС��һЩ����߿���"�ѻ�"���������صĿ�����
 
Do not make refund or claim processes dependent on a specific party performing a particular action with no other way of getting the funds out. For example, in a rock-paper-scissors game, one common mistake is to not make a payout until both players submit their moves; however, a malicious player can "grief" the other by simply never submitting their move - in fact, if a player sees the other player's revealed move and determiners that they lost, they have no reason to submit their own move at all. This issue may also arise in the context of state channel settlement. When such situations are an issue, (1) provide a way of circumventing non-participating participants, perhaps through a time limit, and (2) consider adding an additional economic incentive for participants to submit information in all of the situations in which they are supposed to do so.
��Ҫ�����˿��������������ھ���һ��ִ���ض��Ĳ���������û�������취��ȡ�ʽ����磬�ڼ���ʯͷ��Ϸ�У�һ�������Ĵ����ǲ������о���ֱ����������Ա�ύ���ǵĶ���;Ȼ����������ҿ��Զ��������"ˣ����"��ֻ��򵥵Ĳ��ύ���ı�� �� �� ��ʵ�ϣ������ҿ��Կ���������ҵı�����жϣ����Ǳ�û�������ύ�����Լ��ı���ˡ��������Ҳ���ܷ����ڹ��Ҽ䷽�����������������һ������ʱ����1�� �ṩһ�ֹ�����������ߣ��������ͨ����ʱ�����Ƶķ������� ��2�� ������Ӷ���ľ������򣬹��������������������ṩ����Ӧ�����ϡ�

<a name="keep-fallback-functions-simple"></a>

### Keep fallback functions simple
���ַ��������ļ��
[Fallback functions](http://solidity.readthedocs.io/en/latest/contracts.html#fallback-function) are called when a contract is sent a message with no arguments (or when no function matches), and only has access to 2,300 gas when called from a `.send()`. If you wish to be able to receive Ether from a `.send()`, the most you can do in a fallback function is log an event. Use a proper function if a computation or more gas is required.
[��������](http://solidity.readthedocs.io/en/latest/contracts.html#fallback-function) ��һ����Լ���������ķ���һ����Ϣ�ǻᱻ���� ����û��ƥ�亯���� ����Ϣʱ���� ����'.send()'ʱֻ����2,300 ���塣�������Ҫ�ܹ��� '.send()'������̫�ң����ڷ��������п����������Ǽ�¼һ���¼��������Ҫһ�����������������ʹ���ʵ��ĺ�����

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
���ں�����״̬����Ӧ����ȷ��Ŀ���
 
Explicitly label the visibility of functions and state variables. Functions can be specified as being `external`, `public`, `internal` or `private`. For state variables, `external` is not possible. Labeling the visibility explicitly will make it easier to catch incorrect assumptions about who can call the function or access the variable.
���ں�����״̬����Ӧ����ȷ��Ŀ��ǡ���������ָ����Ϊ '�ⲿ'�� '����'�� '�ڲ�' �� '˽��'������״̬����������Ϊ'�ⲿ' �ǲ����ܵġ���ȷ��Ŀ�ı�ǻ�����ץ����Щ�뵱Ȼ˭���Ե��ú�������ʱ����Ĳ���ȷ�жϡ�
```
// bad
uint x; // the default is private for state variables, but it should be made explicit
        // ����״̬����Ĭ����˽�еģ�����ҲӦ�ñ���ȷ��ʶ
function transfer() { // the default is public Ĭ���ǹ��еġ�
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
�������
Currently, Solidity [returns zero](https://github.com/ethereum/solidity/issues/670) and does not `throw` an exception when a number is divided by zero. You therefore need to check for division by zero manually.
ĿǰSolidity��������е�һ����������ʱ�����׳����⡣���������ֶ�������
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
�������¼�������
Favor capitalization and a prefix in front of events (we suggest *Log*), to prevent the risk of confusion between functions and events. For functions, always start with a lowercase letter, except for the constructor.
���������¼�ǰ���дǰ׺ (���ǽ��� * Log  *)���Է�ֹ�������¼�֮������ķ��ա����ں�����������һ��Сд��ĸ��ͷ�����˹��캯����ͷ��

```
// bad
event Transfer() {}
function transfer() {}

// good
event LogTransfer() {}
function transfer() external {}
```

