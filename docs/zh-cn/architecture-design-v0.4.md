# Fiber Network  Architecture Design V0.4

- [Fiber Network  Architecture Design V0.4](#fiber-network--architecture-design-v04)
  - [Features](#features)
  - [概述](#概述)
  - [scripts](#scripts)
    - [payment channel typescript (PCT)](#payment-channel-typescript-pct)
    - [payment channel lockscript (PCL)](#payment-channel-lockscript-pcl)
  - [流程说明](#流程说明)
    - [创建 channel](#创建-channel)
    - [线下交易](#线下交易)
    - [协议关闭通道](#协议关闭通道)
    - [单向关闭通道](#单向关闭通道)
    - [发起挑战，更新 channel 状态](#发起挑战更新-channel-状态)
    - [清算 channel](#清算-channel)
    - [finalize channel](#finalize-channel)
    - [结算交易](#结算交易)
  - [用户视角看流程](#用户视角看流程)
    - [创建 channel](#创建-channel-1)
    - [channel 支付](#channel-支付)
    - [多跳支付](#多跳支付)
    - [关闭通道](#关闭通道)
  - [使用公用的 preimage cell 避免把 proof cell 永远留在链上](#使用公用的-preimage-cell-避免把-proof-cell-永远留在链上)
    - [proof cell 设计](#proof-cell-设计)
    - [preimage cell 设计](#preimage-cell-设计)
  - [扩展](#扩展)
    - [不关闭 channel 的情况下进行充值和取款的扩展](#不关闭-channel-的情况下进行充值和取款的扩展)
    - [存款场景](#存款场景)
    - [取款场景](#取款场景)
    - [让 HTLC 的空间占用为常数级别](#让-htlc-的空间占用为常数级别)

## Features

本方案使用 cell data 维护 channel 数据，表达 channel 状态转移。参考 sprite 方案进行设计，并使用了志春设计的 proof cell 方案。

实现的能力和存在的问题：

* 能力
    * 支持双人间支付
    * 支持多跳支付
    * 同一个 channel 支持并发多跳支付（concurrent conditional payments）
    * 支持不关闭 channel 进行充值和取款
* 限制
    * 在挑战期同一个人发起的 update 轮次有限制，该限制可以作为配置在创建 channel 的时候指定。这个限制是为了避免挑战期被无限拉长。

主要参考：

* [_Jan 写的 GPC 设计_](https://talk.nervos.org/t/a-generic-payment-channel-construction-and-its-composability/4697)
* [_sprite 构造_](https://github.com/amiller/sprites/blob/master/contractSprite.sol)

## 概述

![](/media/simple-payment-channel.jpg)

![image -1-](/media/htlc-process-zh-cn.png)

上面是典型的简单支付和 HTLC 场景。

channel 的构建有很多方案，有各自的 trade off。
使用 sprite 方法构造 channel 的方案比较简洁，但移植到 CKB 中有几个问题：

* 无法读取当前块高问题。通过 since 指定相对高度来实现挑战期的设计，规避需要绝对块高的场景。
* 更新 channel 时的共享状态问题。每个人只允许更新 N 次状态，会有标记进行记录。避免一个人不断更新 cell 导致另一方无法挑战，挑战过期而获利的攻击。在实践中，这个 N 值的设定会制约用户可以将自己的支付 delegate 给多少个 watchtower。且由于使用相对块高来实现挑战期，N 值设置的越大，挑战期可能越长。
* proof manager 合约。本质上是需要一个全局的状态查询能力，判断是否在时间 T 以前在链上提交了 preimage。以太坊方式的构造可以有全局状态，且调用查询接口可以在仲裁时自动完成，在 CKB 移植的时候，可以使用以下 workaround：
    * 是否在时间 T 之前提交了 preimage，改为是否在 T 之前构造了 proof cell。cell 中放 preimage，根据 cell 被包含的区块，判断是否在时间 T 之前。此处我们先假定这个 cell 将永远存在在链上。
    * 由于 CKB 上无法证明一个 cell 不存在，所以我们使用挑战的方式，给一个挑战期证明 proof cell 存在，过期未提交则认为不存在。

使用 cell data 记录 channel 数据，整个状态转移流程为：

* 创建 channel cell，状态为 OPEN
* 线下交换签名，协调最新状态
* 乐观情况
    * 双方协商关闭通道，确定好最终资产分配，并共同签名将 channel status 修改为 FINALIZED。
* 悲观情况
    * 由于作恶、超时等因素，未能协议完成关闭通道行为，一方发起通道结算保护自己的利益。单方面发起链上通道关闭交易，状态变更为 CLOSING，并启动挑战期倒计时。
    * 发起通道关闭交易后，可以使用线下交换的状态及签名作为 witness 更新 channel data 中的版本和余额信息
    * 超过挑战期后，可以发起交易，将状态更新为 CLEARING 状态，进入清算期。此状态下 channel 的数据版本无法再进行修改，仅进行 HTLC 和存取款结算。
    * 清算期结束后可以发起交易将状态置为 FINALIZED
* FINALIZED 状态的 cell 可以用来解锁 channel 中的 assets，解锁后 channel cell 被销毁

```
@startuml
hide empty description
[*] --> OPEN: 创建 channel cell
OPEN: 创建后可以链下交换签名更新 channel 状态

OPEN --> FINALIZED: 协商关闭通道

OPEN --> CLOSING : 发生争议，单方面关闭通道

CLOSING --> CLOSING : update
CLOSING: 双方可以使用更高 version 的 witness 更新状态

CLOSING --> CLEARING: 度过挑战期后
CLEARING: 双方可以使用对应的证明进行 HTLC 和存取款清算
CLEARING --> CLEARING : clear

CLEARING --> FINALIZED: 度过清算期后

FINALIZED --> [*] : 销毁 cell，解锁并结算 assets
@enduml
```

![image](/media/state-transfer-zh-cn.png)

## scripts

### payment channel typescript (PCT)

args

* bytes: payment channel id。用来唯一标识 channel。值为 tx first input outpoint + output index。

payment channel cell data

```
vector Bytes <byte>;
array Byte32 [byte; 32];
array Uint64 [byte; 8];
array Uint128 [byte; 16];
array Signature [byte; 65];
array Outpoint [byte; 40];
vector Outpoints <Outpoint>;
array PubkeyHashes [byte32; 2];
array Balances [Uint128; 2];
array Signatures [Signature; 2];

/*

- state
    - 0: OPEN
    - 1: CLOSING
    - 2: CLEARING
    - 3: FINALIZED
    - others: invalid
- version: latest channel version
- challenge_period: 挑战期长度，单位为 block 数（或者直接和 relative since 对应的数值相同，可读性差一些，但是合约校验简单些）
- clear_period: 清算期长度，单位为 block 数
- asset: union 类型，目前支持 CKB 和 SUDT，SUDT 的含义为 owner args
- pubkeyHash: 通道双方的公钥哈希
- balance: 通道双方的余额
- included_deposit 和 included_withdraw 用于不关闭通道进行存取款，可以先忽略，后文详述

*/
array CKB [byte; 1];
array SUDT [byte; 32];
union Asset {
   CKB,
   SUDT,
}

struct PaymentChannelCellBaseFixed {
    asset: Asset,
    challenge_period: Uint64,
    clear_period: Uint64,
    pubkey_hashes: PubkeyHashes,
    update_time_limit: byte,   // 双方链上更新 channel 状态的最大次数，u8
}

struct PaymentChannelCellBaseDynamic {
    state: byte,
    version: Uint64,
    balances: Balances,
}

/*

- lock: 随机数 R 的 hash
- preimage_timeout：链上仲裁时，如果 preimage_timeout 前有 preimage 被提交到链上，则 balance 归 to 所有；否则退回 from。
- to: 1 或 2
- preimage_shown: 非 0 表示原像按要求在链上进行了提交，钱应该归 to 所有，为 0 则表示未展示，里面的钱应该退回去

*/
struct HTLC {
    lock: Byte32,
    preimage_timeout: Uint64,
    balance: Uint128,
    to: byte,
    preimage_shown: byte,
}
vector HTLCS <HTLC>;

table PaymentChannelCell {
    fixed: PaymentChannelCellBaseFixed,
    dynamic: PaymentChannelCellBaseDynamic,
    htlcs: HTLCS,
    included_deposit: Outpoints,
    included_withdraw: Outpoints,
    update_times: [byte; 2],      // 双方发起更新的次数
}
```

驱动状态变更的 witness 数据结构

```
/*
线下协商 channel 最新状态用的数据结构

- channel_id 为 PCT 的 args
- signatures 中包含的签名，为对 data 的 raw bytes 进行 blake2b hash 后的 bytes32 进行的签名

*/
table UpdateChannelDataToSign {
    channel_id: Bytes,
    version: Uint64,
    balances: Balances,
    htlcs: HTLCS,
    included_deposit: Outpoints,
    included_withdraw: Outpoints,
}

table UnilateralCloseChannel {
    data: UpdateChannelDataToSign,
    signatures: Signatures,
    submitter_signature: Signature,
}

table UpdateChannel {
    data: UpdateChannelDataToSign,
    signatures: Signatures,
    submitter_signature: Signature,
}

/*

双边协商关闭通道

*/
table BilateralCloseChannelDataToSign {
    channel_id: Bytes,
    balances: Balances,
}

table BilateralCloseChannel {
    data: BilateralCloseChannelDataToSign,
    signatures: Signatures,
}

/*

清算 channel 用的数据结构

*/

table ClearHtlc {
   htlc_index: byte,  // 要清算的 htlc 的 index
   cell_dep_index: byte,  // proof 在 cell deps 中的顺序
   cell_dep_type: byte,   // 0 表示 proof cell，1 表示 preimage cell
}

table ClearChannel {
   htlcs: vector<CLearHtlc>,
}

array FinalizeChannel [byte; 1]; // 无含义，占位符

union Action {
    BilateralCloseChannelDataToSign
    UnilateralCloseChannel,
    UpdateChannel,
    ClearChannel,
    FinalizeChannel,
}

/*

- signature: 动作发起人的签名，签的是 action raw bytes hash

*/
table PaymentChannelWitness {
    action: Action,
}
```

逻辑

* 如果 input group len == 0 && output group len == 1：校验创建 channel 逻辑。
    * args 和 tx first input outpoint 相等
    * state 只能为 OPEN
* 如果 group len == 1 && output group len == 1：校验 channel 状态转移逻辑
* 如果 group len == 1 && output group len == 0：校验 channel 结算逻辑
* 其它情况均报错

channel 状态转移和结算逻辑，后文进行详细说明。

### payment channel lockscript (PCL)

args

* bytes: payment channel typescript hash。唯一匹配一个 payment channel typescript，当且仅当触发 channel 结算逻辑时解锁。

## 流程说明

### 创建 channel

交易示例：A 出资 100 CKB，B 出资 200 CKB，创建一个 channel

```
- inputs
    - capacity provide cell
        - capacity: 100 CKB
        - lockscript: A
    - capacity provide cell
        - capacity: 200 CKB
        - lockscript: B
    - other cells to provide channel cell capacity and tx fee
- outputs
    - channel cell
        - typescript: PCT
            - args: channel-id
        - lockscript: anyone-can-unlock
        - data
            - fixed
                - asset: CKB
                - challenge_period: 50
                - clear_period: 60
                - pubkey_hashes: [pubkey_hash of A, pubkey_hash of B]
            - dynamic
                - state: OPEN
                - version: 0
                - balances: [100 CKB, 200 CKB]
            - htlcs: []
            - update_times: [0, 0]
    - asset cell
        - capacity: 300 CKB
        - lockscript: PCL
```

PCT 逻辑

* 检验 channel 创建逻辑
    * channel-id 是第一个 input 的 outpoint + 自己的  output index
* 校验 data 的初始值
* 校验 balance 和 asset cells 锁定的金额匹配

其中 3 可以不由合约校验，如果状态转移失败或 lock 的金额不够，导致最终资产无法解锁，损失的是出资方。

### 线下交易

构造 UpdateChannelDataToSign 并交换签名。

### 协议关闭通道

双方协商一致，希望关闭通道。交换对 BilateralCloseChannelDataToSign 的签名。
由任意一方发起 BilateralCloseChannel 交易。

### 单向关闭通道

场景：

```
- inputs
    - old channel cell
        - data
            - dynamic
                - state: OPEN
                - version: 0
                - balances: [100 CKB, 200 CKB]
            - htlcs: []
- outputs
    - new channel cell
        - data
            - dynamic
                - state: CLOSING
                - version: 100
                - balances: [50 CKB, 120 CKB]
            - update_times: [0, 1]
            - htlcs:
                - htlc1
                    - lock
                    - balance: 130 CKB
                    - preimage_timeout: 160
                    - to: 1
                    - preimage_shown: 0
- witnesses
    - UnilateralCloseChannel
        - data
            - channel_id
            - version: 100
            - balances: [50 CKB, 120 CKB]
            - htlcs
                - htlc1
        - signatures
        - submitter_signature of B
```

逻辑

* state 由 OPEN 变更为 CLOSING
* 根据 submitter_signature 对应的人，增加 update_times 里对应人的次数

### 发起挑战，更新 channel 状态

如果另一方发现链上提交的是过期的状态，可以发起更新，示例：

```
- inputs
    - old channel cell
        - data
            - dynamic
                - state: CLOSING
                - version: 100
                - balances: [50 CKB, 120 CKB]
            - update_times: [0, 1]
            - htlcs:
                - htlc1
                    - lock
                    - balance: 130 CKB
                    - preimage_timeout: 160
                    - to: 1
- outputs
    - new channel cell
        - data
            - dynamic
                - state: CLOSING
                - version: 101
                - balances: [40 CKB, 130 CKB]
            - update_times: [1, 1]
            - htlcs:
                - htlc1
                    - lock
                    - balance: 130 CKB
                    - preimage_timeout: 160
                    - to: 1
- witnesses
```

逻辑

* 输出的 version 高于输入的 version，且新的 data 和 witness 中的 match
* 根据 submitter_signature 对应的人，增加 update_times 里对应人的次数，并限制 output 中的次数不高于 channel data 中的 update_time_limit

注意：这个步骤是 optional 的，仅当一方提交了旧的状态才会出现

### 清算 channel

度过挑战期后，commitment 版本固定，开始清算期。清算期间，可以提交 htlc、存取款的证据来修改 channel 状态

交易示例:

```
- header-deps
    - proof cell
        - data
            - hash
            - preimage
        - lock: proof-cell-lock
- inputs
    - old channel cell
        - since: 符合超时要求
        - data
            - dynamic
                - state: CLOSING
                - version: 101
                - balances: [40 CKB, 130 CKB]
            - htlcs:
                - htlc1
                    - lock
                    - balance: 130 CKB
                    - preimage_timeout: 160
                    - to: 1
                    - preimage_shown: 0
- outputs
    - new channel cell
        - data
            - dynamic
                - state: CLEARING
            - htlcs
                - htlc1
                   - preimage_shown: 1
- witnesses
    - ClearChannel
        - htlcs
            - htlc0
                - htlc_index: 0
                - header_dep_index: 0
                - header_dep_type: proof cell
```

校验逻辑

* input 中的 state 可以是 CLOSING 或 CLEARING，output 中的 state 为 CLEARING
    * 如果 input 为 CLOSING，需要校验 input since 符合超时要求
* 通过引用 proof cell  或 preimage cell 可以修改 htlc

    * htlc1 的 lock 和 proof cell 提供的原像匹配
    * preimage_shown 从 0 变为 1

### finalize channel

度过清算期后，可以发起 FianlizeChannel。

```
- inputs
    - old channel cell
        - since：清算期
        - data
            - dynamic
                - state: CLEARING
- outputs
    - new channel cell
        - data
            - dynamic
                - state: FINALIZED
- witnesses
    - FianlizeChannel
```

逻辑：

* since 用来保证度过了清算期

### 结算交易

结算交易示例：

```
- inputs
    - assets cell
        - lock: PCL
    - old channel cell
        - data
            - dynamic
                - state: FINALIZED
                - version: 101
                - balance1: [40 CKB, 130 CKB]
            - htlcs:
                - htlc1
                    - lock
                    - balance: 130 CKB
                    - preimage_timeout: 160
                    - to: 1
                    - preimage_shown: 1
- outputs
    - asset cell 1
        - capacity: 为 balance1 的 40 CKB
    - asset cell 2
        - capacity: 为 balance2 + htlc1 balance = 290 CKB
```

校验逻辑

* 如果没有 htlc，input channel cell height 高于 challenge timeout；如果有 htlc，需要高于 settle timeout
* 如果 preimage_shown 为 1，钱归 to 所有，否则为另一方所有
* 计算 outputs 里的 asset cell 金额与结算金额是否匹配

## 用户视角看流程

### 创建 channel

Alice 发起创建 channel 流程，构造好创建 channel 的交易，并签上自己的名发给对方。Bob 查看自己的 channel request，觉得满意就签上自己的名，发还给 Alice，并广播交易上链。

### channel 支付

每个节点在本地记录每个 version 的所有数据，及对应的status。

Alice 发起支付请求，先更新自己最新 channel version 记录，记录 status 为 pending，将 UpdateChannelDataToSign 和自己的签名发给对方。
Bob 检查数据，没问题则回复自己的签名，更新本地 channel version 记录，status 为 success。

Alice 收到 Bob 回复的签名后，更新本地 channel version 记录 status 为 success。

作恶场景
* Bob 收到了消息，但一直没有回复 Alice。此时 Alice 拥有 version 为 V 的数据，Bob 拥有 V + 1 的数据。Alice 可以设置一个响应时间，假如一直收不到 Bob 的响应，则去链上发起关闭通道。最坏的结果是 Bob 以 V+1 的数据进行清算。

### 多跳支付

假设场景为通过路径 A -> B -> C -> D -> E 支付 100 CKB，每个节点收取 1 CKB 手续费。

```
@startuml
participant A order 10
participant B order 20
participant C order 30
participant D order 40
participant E order 50

E -> A: 生成随机数 R，给 A 发送 Hash(R)
A -> B: HTLC(103 CKB, T)
B -> C: HTLC(102 CKB, T)
C -> D: HTLC(101 CKB, T)
D -> E: HTLC(100 CKB, T)
E -> D: R + New-Commitment-D-E
D -> C: R + New-Commitment-C-D
C -> B: R + New-Commitment-B-C
B -> A: R + New-Commitment-A-B
@enduml
```

![image -1-](/media/htlc-process-zh-cn.png)


* HTLC(103 CKB, T) 代表包含了 HTLC 的 commitment，金额为 103 CKB，原像提交过期时间为 T（单位为块高）。A 给 B 发送 new proposal 的时候，应估算 T 为当前块高 + 容忍期 + 足够将原像发上链的时间。
* R + New-Commitment-D-E 表示将原像发给上游，并签署新的结算了 HTLC 的 commitment

异常情况处理

* 仅有一种作恶情况需要处理，即和下游签署了新的 commitment，在将 R + New-Commitment 发给上游时，上游无响应。在等待一段时间的容忍期后，如果还没有响应，需要构造 proof cell 发到链上，保证之后结算的时候自己不会有损失。
* 由于中间环节的每个人都是先收到条件支付的钱，再用相同的条件支付给下游钱，而条件的仲裁在整个流程中是一致的（依赖链上数据）且不会过期，因此在其它未响应状态下资产均不会有损失，也无需去链上发起通道结算。可以在重新获取响应后重新协商 commitment。

### 关闭通道

1. 发起交易，用手上最新的状态发起关闭通道
2. 如果对方用老的状态恶意发起关闭通道，我们可以监听到状态并用更新的状态发起更新
3. 过了挑战期后发起 FINALIZE（对方先发了正常的，就不用发了，对方先发了恶意的，我们重新发起进行覆盖）
4. 如果 FINALIZE 后没有要清算的 HTLC，跳到发起 SETTLE 交易步骤，否则开始清算 HTLC 的流程
5. 如果有自己是收款方且在 preimage-timeout 前提交了 proof cell 的 HTLC，需要发起交易，将对应的 HTLC 中的 preimage-shown 字段置为 1，保证自己拿到这笔钱，否则不用操作。
6. 发起 SETTLE 交易
7. 发起 channel 结算交易

## 使用公用的 preimage cell 避免把 proof cell 永远留在链上

前文我们假设了 proof cell 会永远留在链上，这样可以简化各种过期时间的设计，但是会永远占用用户的 capacity。
一个方案是，设计 1 个公用的 preimage cell，用来收集链上的 proof cell，并将数据记录到 preimage cell data 中。

* preimage cell data 为一个 Sparse Merkle Tree 的 root，可以以老的 preimage cell + proof cell 为 input，将 proof cell 的 lock: (preimage + submit-time) 这个数据记录到 SMT 中，生成新的 preimage cell。
* 用 since 保证至少 N 个 block 后才能被更新。此处 N 的确定取决于我们要给用户留多少引用 preimage cell 的时间窗口，避免 preimage cell 一直被作为 input 消耗掉，用户难以用来作为 cell dep。例如我们设置为每 5 个块，preimage cell 可以被更新。这样用户如果应用 preimag cell 遇到 dead outpoint 错误，只要重试即可。
* proof cell 只有这一个场景下会被销毁。因此一个被提交到链上的 proof cell 要么存在于链上，要么销毁后，数据转移到了 preimage cell 中。
* 理论上可以生成多个 preimage cell 交互使用来减少 proof cell 被回收的等待时间。但不一定解决问题，因为去中心化场景下，任何人可以更新，最坏情况下和单个 preimage cell 效果一样。
* 使用 preimage cell 代替 proof cell 作为 HTLC 仲裁时，需要在 witness 里提供 SMT 里 lock: (preimage + submit-time) 这个 kv 对的存在性证明。

### proof cell 设计

* data
    * lock-hash
    * preimage
* typescript: null
* lockscript: proof-lockscript
    * args: preimage-id
    * code logics
        * 仅当 input 里有 preimage-id 对应的 typescript 时才能解锁

### preimage cell 设计

* data
    * proof-root
        * k: lock-hash
        * v: (preimage, submit-time)
* typescript
    * args: preimage-id，typeid 思路保证 preimage cell 唯一性
    * code logics
        * input since 约束为 5 个 block 后可以上链
        * 仅当 inputs 里有 proof cell 时，可以解析其内容，加入 SMT 的 kv 对
* lockscript: anyone-can-unlock

## 扩展

### 不关闭 channel 的情况下进行充值和取款的扩展

channel 的资产都通过 PCL 守护。增加或减少其中的资产和 channel cell 互不影响。因此可以设计方案，在不关闭 channel 的情况下进行充值和取款。此处的重点是，要保证 充值/取款 操作和签发新的 commitment 两个行为的原子性。

### 存款场景

* 将 PCL 改造成 ACP，可以接收任何人的存款
* 设计 channel-deposit-typescript，在往 PCL 存款的同时，生成一个 deposit-proof-cell，cell data 为存款的证明
* A 完成存款后，找 B 签署新的 commitment。channel data 中增加一个字段 included_deposit，是一个 deposit-proof-cell outpoint 的集合。
    * 如果 B 配合签署，在 included_deposit 中加入 A 的存款证明，并修改 A 的余额，生成新的 commitment 进行签署
        * B 签署了包含存款证明的 commitment 后，即使退出时引用该 proof cell，也会被 PCT 忽略。保证了 A 无法使用两次这笔存款。
        * 后续 A 销毁了 deposit-proof-cell，在后续签发的 commitment 中可以放心的从 included_deposit 中移除该 cell
    * 如果 B 不配合签署，A 可以认为 B 作恶，发起关闭 channel。在 channel 的 update 过程中，A 可以引用 deposit-proof-cell 驱动 channel cell 状态转移，增加结算时自己的余额。channel 的校验逻辑为，如果 included_deposit 中不包含引用的 deposit-proof-cell，则增加对应 pubkey 的余额，否则忽略。


存款交易示例


```
- inputs
    - asset cell
        - capacity: 100 CKB
        - lockscript: PCL
- outputs
    - asset cell
        - capacity: 120 CKB
        - lockscript: PCL
    - deposit proof cell
        - data
          - deposit: 20 CKB
          - asset: 0x0
          - recipient_pubkey: pubkey1
          - channel_id

```

### 取款场景

* A 找 B 签发一个取款凭证，用来解锁 PCL 取出里面的部分资产，同时生成一个取款凭证 withdraw-proof-cell，用 since 保证该 cell 在 N 个块内无法被销毁。
* B 找 A 签发新的 commitment，channel data 中新增字段 included_withdraw。balance 为计算了取款的新 balance，included_withdraw 中包含了 withdraw-proof-cell。
    * 如果 A 拒绝签发新 commitment，B 关闭通道，并应用 withdraw-proof-cell 扣减 A 的余额
    * A 签发了新 commitment，则 channel 内总余额已减少。后续每次签发新交易均保留 included_withdraw 直到 withdraw-proof-cell 被销毁，被销毁后再签 commitment 可以不用包含该 withdraw-proof-cell。

### 让 HTLC 的空间占用为常数级别

目前的设计里，如果一个 channel 的 HTLC 数量较多，用户 close 的时候需要更多的 capacity 进行操作。尽管用户需要付出的成本是这部分 capacity 占用从 CLOSING 开始到 channel cell 销毁的机会成本，但是肯定导致需要用户在一开始拿出更多的 CKB，用户体验上不太好。考虑让 HTLC 占用的空间为常数级别。

具体方案

* htlcs 字段改为 htlc_root，存放 htlc 数据的 merkle root。
* 在 ClearChannel 的时候直接结算掉通道的余额部分，channel cell 保留。之前设计里每一次 ClearChannel 都是修改 channel cell 的状态，可以改成每次都直接清算对应的资产。
* 每引用一个 htlc proof 就提供一个数据，解锁对应的钱。超时后引用剩下未处理的 htlc，解锁对应的钱。

参考资料
- https://github.com/jjyr/sparse-merkle-tree
