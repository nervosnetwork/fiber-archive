# Roadmap

- 06.18（2w）：完成合约 MVP
- 07.09（3w）：完成 offchain 部分 MVP
- TBD：P2P、路由功能开发，支持多跳支付

## MVP 版本合约开发规划

* preimage cell 使用不会过期的版本
    * 降低用户心智负担，由于 HTLC Proof cell 不会过期，大部分情况下都不需要触发结算通道，链下协商解决。在存在过期的情况下，即使只是用户掉线，也可能为了避免 proof cell 过期，需要关闭通道保证资产安全
    * 支持支付手续费，如果手续费过低，市场上没人愿意打包 proof cell 到 preimage cell，提交 proof cell 的人会损失 proof cell 占用的 capacity 的机会成本。用户可以自行打包，自行支付交易手续费，释放 proof cell 占用的 capacity。
* HTLC 使用常数级别占用方案
    * channel 中使用 HTLC ROOT，计算方法为朴素的 merkle tree 算法，HTLC 内容序列化后 hash，然后按照 vec<HTLC Hash> 顺序构造 HTLC ROOT。
    * channel cell data 中增加 HTLC ROOT（bytes32），HTLC length（U64），HTLC-clear-bitmap（bytes，表示已经结算的 HTLC 的 bitmap）
    * ClearChannel 数据结构中增加 HTLC、proof
* HTLC 和 channel 分开结算，允许直接从 asset cell 里拿钱
    * 简化设计，大部分的逻辑都放到 PCT 中
    * 一个正常的 HTLC 流程应当很快（ 约等于（程序处理速度 + 网络延迟） * 2 * 路由的路径长度 N，一般可以在 1 分钟内），与关闭通道允许提交状态的挑战期相比（对于 Optimistic Rollup，大家容许的挑战期为一周）几乎可以忽略不计。
    * 分开结算，且都直接从 asset cell 拿钱，可以复用取款逻辑，且 channel 余额和 HTLC 余额都可以在第一之间支取，降低流动性成本。

交付列表
- 包含上述提到的完整功能的合约代码
- 走完文档中提到的所有的流程的 integration test

## offchain 部分 MVP 版本开发规划

交付列表
- 根据配置起节点
- 有用户交互框架（RPC 接口）
    - 支持接口：请求创建通道、查看请求列表、同意创建通道、发起支付
- 有节点之间交互框架（可以先实现为 HTTP RPC，后续替换为 P2P）
    - 支持接口：创建通道，发起支付等
- 有 demo 展示两人间创建通道、支付、关闭通道等流程

## 后续 features list

- 合约
    - [ ] 不关闭通道进行充值和提取
- offchain
    - [ ] 搭建基本框架，实现直接支付
    - [ ] 实现路由方法，实现多跳支付