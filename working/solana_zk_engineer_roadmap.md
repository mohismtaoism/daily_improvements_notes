# 从资深 C/C++ 工程师到 Solana / 零知识证明工程师：24 周实战路线图

> 版本：2026-10-28  
> 目标：以可验证的工程作品为核心，争取远程区块链研发岗位。  
> 学习强度：每周 12–15 小时；若每周只有 6–8 小时，按比例把 24 周拉长到 9–12 个月。  
> 适用对象：有多年 C/C++、Linux、调试、嵌入式/FPGA、机器人控制经验，但没有系统区块链研发经验的工程师。

---

## 0. 先给结论：不要把“零知识”当成唯一入口

你的 CV 显示，你不是初级程序员。你做过 Linux/C++ 上位机、TCP/IP、EtherCAT、机械臂控制、运动学算法移植、FPGA 模块、Android Framework/RIL，以及当前医疗设备软件与 STM32 预研。这些经历能迁移到区块链工程：复杂系统调试、协议与状态机、跨层定位问题、性能意识、测试和把算法交付为产品代码。CV 也显示你有 Python 基础，但 Rust、Solana、密码学工程项目尚未在简历中体现。不要把自己描述成“平庸软件工程师”；更准确的判断是：**有较强系统工程底子，需要补齐 Rust、链上运行模型、密码学基础和公开可审查的作品集。**

### 建议的岗位优先级

| 优先级 | 岗位方向 | 与现有技能的匹配 | 主要补课 | 建议 |
|---|---|---:|---|---|
| 1 | Solana / Rust 协议与基础设施工程师 | 高 | Rust、账户模型、交易/运行时、RPC、索引与测试 | 主线；先成为能交付 Solana 工程的 Rust 开发者 |
| 2 | ZK 系统工程 / prover 工程 / 密码学基础设施 | 中高 | 有限域、椭圆曲线、约束系统、证明系统、性能优化 | 第二阶段；可利用 C++/系统调试优势切入 |
| 3 | ZK 应用集成 / Solana 上的隐私或证明验证 | 中高 | 将证明系统和链上程序整合，理解验证成本与安全边界 | 适合做组合型作品集 |
| 4 | 纯密码学研究 / 新证明系统设计 | 当前较低 | 更深数学、论文研究、证明安全性、长期研究训练 | 暂不作为第一份远程工作的目标 |
| 5 | 交易机器人 / MEV / DeFi 自动化 | 技术上可学，风险高 | 市场微观结构、链上数据、风险控制、交易基础设施 | 可做研究工具，不建议把投机收益当职业计划 |

**主路线：Rust → Solana 程序与基础设施 → 密码学工程基础 → ZK 证明系统 → ZK + Solana 综合项目。**

24 周并不意味着“达到资深 ZK 研究员水平”。现实目标是：构建可展示的工程能力，具备申请初级至中级 Rust/Solana 工程岗位的证据，并判断自己是否适合进一步专攻 ZK。进入真正的 ZK 研发岗位可能需要 9–18 个月甚至更久，取决于投入、数学基础、项目质量和招聘市场。

---

## 1. 学习规则与每周节奏

### 每周时间分配（12–15 小时）

- 编程与实验：6–8 小时
- 阅读官方文档 / 论文：2–3 小时
- 测试、代码审查、写技术笔记：2 小时
- 英文技术表达 / README / 求职材料：1 小时

### 每周必须交付

1. 一个可运行的代码提交，避免只看教程。
2. 自动化测试与明确的运行步骤。
3. 一份短技术笔记：问题、设计、边界条件、测试、未解决问题。
4. 至少一次从干净环境重新构建。
5. 代码推送到公开 Git 仓库；涉及公司代码、医疗设备信息、密钥或专有协议的内容绝不上传。

### 工程质量标准

- Rust 项目：`cargo fmt --check`、`cargo clippy -- -D warnings`、`cargo test`
- Solana 项目：本地 validator / program-test 测试、成功与失败路径测试、账户权限测试
- ZK 项目：正例、负例、边界值、篡改 public inputs / proof 的测试
- README 必须写清楚：问题、架构、威胁模型、如何运行、测试覆盖什么、已知限制

### 建议的仓库结构

```text
rust-solana-zk-roadmap/
├── 00-rust-katas/
├── 01-chain-data-cli/
├── 02-solana-counter/
├── 03-solana-escrow/
├── 04-solana-indexer/
├── 05-finite-field/
├── 06-toy-r1cs/
├── 07-zk-proof-lab/
├── 08-solana-zk-demo/
└── notes/
```

不要在第一个月就搭建复杂微服务、Kubernetes、全套监控。先证明自己能写对、测全、解释清楚一个小系统。

---

## 2. 先安装开发环境

建议在 Ubuntu Linux 上学习，和你已有的 Linux 开发习惯一致。把工具版本记入 `notes/environment.md`，不要无意识追随最新 nightly。

- Git、GCC/Clang、CMake、Python 3、`jq`
- Rust stable、Cargo、rustfmt、Clippy
- Solana/Agave CLI 与 Anchor：按官方安装文档选择相互兼容的版本
- Node.js 包管理器：仅在所选 Anchor / 测试栈需要时安装
- Docker：后期做索引器或服务部署时再用
- 编辑器：VS Code、CLion 或你熟悉的 Vim 均可

官方入口：
- Rust Book：https://doc.rust-lang.org/book/
- Rust By Example：https://doc.rust-lang.org/rust-by-example/
- Rustlings：https://github.com/rust-lang/rustlings
- Solana 开发者文档：https://solana.com/docs
- Anchor 文档：https://www.anchor-lang.com/docs
- Solana Cookbook：https://solanacookbook.com/

版本提醒：Solana 生态工具更新较快。安装时优先看项目当前官方安装说明与版本兼容性；不要把几年前教程中的命令当成永远有效的标准。

---

# 第一阶段：Rust 工程能力（第 1–4 周）

目标：把 C++ 工程经验转化为符合 Rust 习惯的可维护代码。暂时不写链上程序。

## 第 1 周：所有权、借用、错误处理

**阅读**
- The Rust Programming Language：章节 1–10，重点所有权、结构体、枚举、模式匹配、错误处理、泛型与 trait。
- Rust By Example：相关章节。
- Rustlings：变量、函数、if、move、references、structs、enums、error handling。

**编程任务**
1. 用 Rust 实现一个二进制协议解析器：输入字节流，解析 `magic + version + length + payload + checksum`。
2. 输入可能被截断、长度超限、校验失败；不得 panic，返回有意义的错误类型。
3. 编写至少 20 个单元测试，覆盖空包、短包、最大长度、错误 magic、错误 checksum、附加垃圾字节。
4. 实现一个流式解析接口：一次输入可能只有半个包，也可能连续输入多个包。

**与旧经验的连接**
这相当于把你熟悉的 MCU/FPGA 通信协议解析移植到 Rust。重点不是协议本身，而是把内存安全、边界检查和错误处理变成语言习惯。

**交付物**
`00-rust-katas/protocol-parser`，包含设计说明、测试矩阵和 fuzz 测试计划。

## 第 2 周：trait、泛型、迭代器、模块设计

**阅读**
- Rust Book：traits、generics、iterators、collections。
- Rust API Guidelines：https://rust-lang.github.io/api-guidelines/

**编程任务**
1. 为协议解析器定义 `FrameDecoder` trait。
2. 用 trait 抽象校验算法，分别实现 CRC-16（自选明确多项式与参数）和简单测试校验器。
3. 实现一个日志分析 CLI：读取模拟设备日志，按错误码计数、按时间排序、输出 JSON/CSV。
4. 将解析层、业务层、命令行层拆成独立模块。

**验收**
- 业务逻辑不依赖命令行参数解析库。
- 错误通过 `Result` 传递。
- 对大文件逐行处理，不一次性读入全部内容。
- 至少 1 个集成测试。

## 第 3 周：并发、网络、异步概念

**阅读**
- Rust Book：threads、channels、shared-state concurrency。
- Tokio tutorial：https://tokio.rs/tokio/tutorial
- Rust 标准库 `std::sync` 与 `std::thread` 文档。

**编程任务**
1. 写一个 TCP 日志接收器，接受多个客户端连接。
2. 使用 channel 把网络读取与解析/存储分开。
3. 处理断线、慢客户端、错误输入、服务关闭。
4. 增加一个 mock server 与端到端测试。

**验收**
- 不能因一个客户端发错数据而杀死整个服务。
- 写出背压设计说明。
- 能解释线程、async task、channel 各自适合什么场景。

## 第 4 周：Rust 小型工程结项

**编程任务**
- 将前三周内容整理成 `chain-data-cli`：读取本地 JSONL / 日志数据，校验、统计、过滤并输出报告。
- 添加 CI（GitHub Actions 或等效工具）：fmt、clippy、test。
- 写一篇 800–1200 字技术说明，主题是“如何把一个 C/C++ 协议解析器重构为 Rust”。

**阶段门槛**
你应能不照抄教程，独立写出包含模块、错误处理、测试和 CLI 的 Rust 项目。若做不到，延长本阶段 1–2 周，不要急着学零知识。

**免费课程**
- Rustlings（免费开源）：https://github.com/rust-lang/rustlings
- 官方 Rust Book（免费）：https://doc.rust-lang.org/book/
- Google 的 Rust 教学材料可作为补充，但以官方语言文档为主。

---

# 第二阶段：区块链与 Solana 心智模型（第 5–6 周）

目标：能解释区块链状态、签名、交易、账户、费用、共识和 Solana 的账户模型。不要先把钱包连接 UI 当成区块链开发。

## 第 5 周：密码学与区块链基础

**学习内容**
- 哈希函数、数字签名、密钥对、Merkle tree。
- UTXO 与账户模型的差别。
- 交易签名与验证、重放攻击、nonce / blockhash。
- 区块、共识、最终性、分叉、费用市场。
- 公链节点、RPC、索引器和链上程序之间的职责边界。

**免费阅读**
- Bitcoin 白皮书：https://bitcoin.org/bitcoin.pdf
- Ethereum Developer Docs（用于对比账户模型与智能合约）：https://ethereum.org/en/developers/docs/
- Solana 官方文档：https://solana.com/docs

**编程训练**
1. 使用成熟密码学库，不自己发明密码算法，实现一个文件哈希与签名验证 CLI。
2. 用 Python 编写一个简化 Merkle tree：构造叶子、计算根、生成 inclusion proof、验证 proof。
3. 故意篡改叶子、路径节点和根，测试验证必须失败。
4. 写一页“哈希、签名、加密分别解决什么问题”的说明。

**安全要求**
学习用的 toy 实现不应用于资产、密钥托管或生产安全系统。密码算法应使用经过审查的库。

## 第 6 周：Solana 交易与账户

**阅读与动手**
- Solana 官方文档中的 Accounts、Transactions、Programs、Fees、Clusters。
- Solana Cookbook 中交易、钱包、RPC 相关示例。

**编程任务**
1. 创建本地开发钱包，只用于本地测试。
2. 启动本地 validator，获取 SOL 测试余额，发送一笔转账。
3. 用 Rust 客户端读取账户与交易状态。
4. 编写 CLI：输入交易签名，输出状态、slot、错误摘要（使用官方 RPC 接口）。
5. 记录一次失败交易，并解释失败发生在客户端构造、签名、提交、执行还是确认阶段。

**必须理解的 Solana 概念**
- Account 是状态存储单元；program 是执行逻辑。
- 指令声明所需账户与权限。
- 交易由一个或多个指令组成。
- 账户 owner 与签名者是不同概念。
- PDA 是由程序 ID 和 seeds 推导出的地址；它不是普通私钥钱包。
- 交易成功提交与达到所需确认级别不是同一概念。

**阶段门槛**
能画出一笔交易从客户端到 RPC、验证、执行、状态更新与确认的流程图，并能从日志定位一笔失败交易。

---

# 第三阶段：Solana 链上程序（第 7–11 周）

目标：做出安全、可测试的链上程序。这是求职的第一条可行主线。

## 第 7 周：第一个 Solana 程序

**学习**
- Solana 官方“Developing Programs”。
- Anchor 官方文档的入门与测试章节。

**任务**
实现一个 Counter 程序：
- 初始化计数账户。
- 仅授权管理员可增加或重置计数。
- 记录账户版本或初始化状态。
- 明确账户空间与序列化布局。
- 编写成功、未授权、错误账户、重复初始化等测试。

**重点**
理解 account constraints、signer、owner、PDA seeds、账户空间、错误码。不要只追求“本地跑通”。

## 第 8 周：账户安全与常见漏洞

**阅读**
- Solana 官方安全指南（从官方文档导航进入）。
- Anchor 的账户约束、PDA、CPI、测试相关文档。

**任务**
给 Counter 编写攻击测试：
1. 未授权用户尝试修改状态。
2. 传入错误 owner 的账户。
3. 传入错误 PDA 或 seeds。
4. 尝试重复初始化。
5. 传入错误账户关系。
6. 检查整数溢出与边界值。
7. 测试不同账户顺序与缺失账户。

**交付物**
`security-review.md`：每个测试描述攻击前提、预期失败原因、实际结果。

## 第 9 周：PDA、状态机和事件

**任务**
实现一个设备/任务注册表（用虚构数据，不要使用真实医疗设备信息）：
- 用户注册一个记录。
- 使用 PDA 按用户和记录 ID 派生地址。
- 状态从 `Created → Active → Closed` 迁移。
- 非法状态迁移必须失败。
- 通过事件记录状态变更。
- 设计关闭与回收账户的策略，并说明租金处理。

**为什么适合你**
你做过设备状态机、手柄 attach 状态、机器人模块管理。把这些工程经验转化成链上状态机是自然的切入点。但链上状态与硬件状态不同：交易可能失败、账户权限由运行时强制执行，所有输入都必须视为不可信。

## 第 10 周：Escrow / Vault 小项目

**任务**
实现一个最小托管交换程序（仅 devnet/localnet 测试资产）：
- maker 创建托管单。
- taker 接受并完成交换。
- maker 在允许条件下取消。
- 使用 PDA 管理状态。
- 明确 token program 与账户关系。
- 覆盖重复执行、错误 mint、错误 token account、未授权调用、余额不足等失败路径。

**安全边界**
这只是学习项目，不可直接托管真实资产。资金类程序需要独立安全审计、形式化威胁分析、充分测试和明确的升级/权限策略。

## 第 11 周：集成测试与部署流程

**任务**
- 建立从干净环境启动 local validator、部署程序、运行测试的脚本。
- 把程序部署到 devnet，记录程序 ID、部署步骤和验证方法。
- 写一份“本地测试与 devnet 行为差异”说明。
- 用 CI 跑 Rust 测试与程序测试。
- 对每个指令写账户权限表：账户名、signer、writable、owner、校验规则。

**阶段门槛**
你必须能解释并测试程序的安全边界，而不只是演示 happy path。若账户安全测试不足，先补测试，不进入 ZK。

---

# 第四阶段：Solana 基础设施与数据工程（第 12–14 周）

目标：把链上程序之外的系统能力也变成求职筹码。很多团队需要 RPC 集成、事件索引、数据处理、可靠性和性能工程。

## 第 12 周：RPC 客户端与可靠性

**任务**
写一个 Rust 服务，定时读取指定 program 的账户/交易数据：
- 请求超时、重试与退避。
- 区分暂时性错误与永久性错误。
- 指标：请求数、错误率、延迟。
- 缓存最近处理的 slot / signature。
- 避免重复处理。

## 第 13 周：事件索引器

**任务**
为第 9 周注册表程序构建最小索引器：
- 订阅或轮询 program 相关数据。
- 解码事件/账户。
- 存入 SQLite 或 PostgreSQL。
- 支持按用户、状态、slot 查询。
- 处理重启、重复事件、断线重连和数据回补。

**测试**
- 同一事件重复送达，不产生重复业务记录。
- 服务重启后从 checkpoint 继续。
- 断线后能回补缺失范围。
- 数据库事务失败时不推进 checkpoint。

## 第 14 周：性能、可观测性与项目发布

**任务**
- 给索引器增加结构化日志、健康检查和基本指标。
- 生成测试数据，做一次吞吐量与延迟基准。
- 写清楚瓶颈、测试环境和可复现步骤。
- 发布 `solana-program-indexer` v0.1.0。

**求职价值**
这条路径可以申请 Rust backend、Solana infrastructure、indexer、RPC integration 等岗位；它比直接从零申请纯 ZK 研究岗位更现实。

---

# 第五阶段：数学与密码学工程基础（第 15–17 周）

目标：为 ZK 做准备。你需要重新激活离散数学、模运算与有限域，而不是先冲进复杂论文。

## 第 15 周：离散数学与有限域

**学习**
- 模运算、最大公因数、扩展欧几里得算法。
- 群、环、域的直观定义。
- 有限域 GF(p) 的加法、减法、乘法、逆元。
- 多项式与插值的基础。
- 复杂度与概率基础。

**免费课程**
- MIT OpenCourseWare Mathematics for Computer Science：https://ocw.mit.edu/courses/6-1200j-mathematics-for-computer-science-fall-2024/
- MIT OCW Linear Algebra（可按需要补）：https://ocw.mit.edu/
- Khan Academy 的 Modular Arithmetic / Abstract Algebra 相关材料：https://www.khanacademy.org/

**编程任务**
用 Rust 实现 `FieldElement`：
- 以一个小素数 p 为模。
- 加、减、乘、求逆、除法。
- 逆元不存在时返回错误。
- 测试恒等式、分配律、逆元性质。
- 与 Python 小脚本交叉验证随机测试数据。

**数学验收**
你能解释为什么在有限域里非零元素存在乘法逆元，以及为什么模数应选素数（本课程先聚焦素数域）。

## 第 16 周：椭圆曲线与承诺概念

**学习**
- 椭圆曲线群运算的概念。
- 标量乘法、离散对数问题的角色。
- Pedersen commitment 的直觉：隐藏性与绑定性。
- Fiat–Shamir heuristic 的用途与风险边界。
- 哈希承诺与代数承诺的差别。

**阅读**
- *Proofs, Arguments, and Zero-Knowledge*（Justin Thaler，公开讲义）：https://people.cs.georgetown.edu/jthaler/ProofsArgsAndZK.html
- ZKProof Community Reference：https://zkproof.org/
- Vitalik 的 ZK-SNARK 介绍（直观入门）：https://vitalik.eth.limo/general/2021/01/26/snarks.html

**编程任务**
- 在小素数域上实现一个教学用 Pedersen-style commitment 的简化模型，或者使用成熟曲线库完成小规模承诺演示。
- 明确标注：玩具实现没有经过审计，不可用于实际隐私保护或资产系统。
- 测试打开承诺、错误随机数、篡改消息的行为。

## 第 17 周：从电路到约束系统

**学习**
- 算术电路：加法门、乘法门。
- R1CS：约束形如 `(A·z) * (B·z) = (C·z)`。
- witness 与 public inputs。
- 完备性、可靠性、零知识性这三个性质分别是什么。
- proving 与 verification 的成本差异。

**编程任务**
1. 写一个玩具 R1CS 表示层。
2. 编码约束 `x * x = y` 与 `a + b = c`。
3. 实现 witness 检查器，拒绝无效 witness。
4. 给每个约束写失败测试。
5. 输出约束数量与变量数量统计。

**阶段门槛**
能用自己的话解释：证明者证明的是某个 witness 满足公开约束，而验证者不必看到 witness 本身；但“有一个约束检查器”不等于“已经有零知识证明系统”。

---

# 第六阶段：真正的 ZK 工具与证明系统（第 18–21 周）

目标：不自己发明密码学，而是使用成熟生态理解证明生成、验证、公开输入与性能权衡。

## 第 18 周：选择一条证明系统路线

不要同时学习 Groth16、PLONK、STARK、Halo2、Nova、所有 zkVM。先选一个工具链做完整闭环。

建议顺序：
1. 用 Circom + snarkjs 做一次电路、证明、验证闭环，便于观察完整流程。
2. 再根据岗位方向选择一个 Rust 生态项目深入（例如 arkworks 或 Halo2；具体项目与版本以其官方仓库当前维护状态为准）。
3. 若更偏系统工程，可以研究 zkVM 的程序执行、证明生成和性能剖析。

参考：
- Circom：https://docs.circom.io/
- snarkjs：https://github.com/iden3/snarkjs
- arkworks：https://arkworks.rs/
- Halo2（Zcash）：https://github.com/zcash/halo2
- RISC Zero 文档：https://dev.risczero.com/
- SP1 文档：https://docs.succinct.xyz/

**任务**
完成一个“知道某个秘密输入满足指定关系”的教学电路：
- witness 为私有输入。
- public input 为公开声明。
- 正确 witness 生成 proof 并通过验证。
- 错误 witness 无法生成有效 proof，或验证失败。
- 修改 public input 后验证必须失败。
- 记录 proving time、verification time、proof size 和约束规模。

## 第 19 周：阅读论文，不追求一次读懂

**必读**
- *The Knowledge Complexity of Interactive Proof Systems*（Goldwasser, Micali, Rackoff, 1985）：先读摘要、定义与核心思想。
- *On the Size of Pairing-based Non-interactive Arguments*（Groth, 2016，Groth16）：理解 succinctness 与 pairing-based proof 的基本思路。
- *Scalable, Transparent, and Post-Quantum Secure Computational Integrity*（STARK 相关原始论文）：理解透明设置、哈希与可扩展性的大方向。
- Justin Thaler 公开讲义相关章节。

**阅读模板**
每篇论文只需先回答：
1. 它要解决什么问题？
2. 证明者与验证者分别做什么？
3. 公共输入与私有 witness 是什么？
4. 安全性依赖哪些假设？
5. 证明大小、证明时间、验证时间有什么取舍？
6. 它没有解决什么问题？

不要把“看过论文”当成掌握。请写一页自己的解释，并用小例子检验理解。

## 第 20 周：Rust ZK 库实践

根据第 18 周的体验选择一个库，做最小的可复现项目。

**任务**
- 定义一个小算术关系。
- 生成证明并验证。
- 对非法输入、错误公开输入、损坏证明、错误参数做负向测试。
- 用 profiler 或计时器测量 proving 与 verification。
- 记录依赖版本、运行环境、内存占用和限制。

**安全要求**
不要在生产中使用自行拼装的密码协议或未经审计的参数。真实系统要遵循所选库的安全指南与可信设置要求，并请有经验的密码学工程师审查。

## 第 21 周：ZK 性能工程

这周发挥你在 C/C++、调试、嵌入式和硬件边界方面的经验。

**任务**
- 选择一个证明生成热点。
- 用 profiler 找到瓶颈，不要凭猜测优化。
- 比较不同输入规模或约束规模。
- 记录 CPU、内存、耗时、证明大小。
- 解释优化为何有效，以及是否影响正确性。
- 如果库支持，比较不同曲线/证明系统或不同参数，但不要把不可比的配置混成结论。

**交付物**
`zk-benchmark-report.md` + 可复现 benchmark 脚本。

---

# 第七阶段：Solana + ZK 综合项目（第 22–24 周）

目标：做一个能向招聘方展示端到端理解的项目，而不是宣称发明新证明系统。

## 项目：私有资格证明 + 链上验证演示

**示例场景**
用户证明自己持有一个符合规则的凭证，例如“年龄超过阈值”或“属于允许列表”，但不公开完整秘密数据。使用虚构数据与教学参数。

**架构**
1. 本地工具生成私有 witness。
2. ZK 工具生成 proof 与必要的 public inputs。
3. Solana 程序接收 public inputs 与 proof。
4. 链上验证或通过明确的外部验证架构完成验证。
5. 程序记录验证结果，防止同一凭证被重复使用（若业务场景需要）。

**重要现实约束**
不要假设任意证明系统都能低成本直接在 Solana 上验证。验证成本、曲线与预编译支持、proof 编码、交易大小、计算单元、可信设置和升级策略都会限制设计。若选用的证明系统不适合直接链上验证，先做“链下验证 + 链上 attestation”的明确演示，并把信任假设写出来，不要把它描述成等价的 trustless 验证。

**验收清单**
- 公开架构图。
- 说明 threat model、信任边界与隐私边界。
- 正例与负例测试。
- 篡改 public inputs、proof、账户权限后应失败。
- 写清楚是否链上验证、是否依赖可信设置、是否存在中心化验证者。
- 给出本地一键运行命令。
- 录制 3–5 分钟英文演示，介绍设计取舍。

**可选增强**
- 为链上程序加 Anchor 测试。
- 把链上事件写入索引器。
- 加入 CI 与 benchmark。
- 写一个简短安全评审报告。

---

# 3. 每周课程总表

| 周 | 主题 | 主要编码产出 | 阅读/理论产出 |
|---:|---|---|---|
| 1 | Rust 所有权、错误处理 | 二进制协议解析器 | 所有权与边界检查笔记 |
| 2 | trait、泛型、模块 | 日志分析 CLI | 模块化设计说明 |
| 3 | 并发、TCP、async | TCP 日志接收器 | 背压与并发模型说明 |
| 4 | 工程化 | CI、集成测试、发布 | C++ 到 Rust 迁移报告 |
| 5 | 密码学/区块链基础 | Merkle tree + 签名 CLI | 哈希、签名、承诺区别 |
| 6 | Solana 账户/交易 | RPC 交易诊断 CLI | 交易生命周期图 |
| 7 | Anchor 入门 | Counter 程序 | 账户模型说明 |
| 8 | 安全测试 | 攻击测试套件 | 安全评审 |
| 9 | PDA 与状态机 | 注册表程序 | 状态迁移表 |
| 10 | Escrow | 托管交换 demo | 资金流与威胁模型 |
| 11 | 部署与 CI | localnet/devnet 流程 | 权限矩阵 |
| 12 | RPC 可靠性 | Rust RPC 客户端 | 重试与幂等设计 |
| 13 | 索引器 | SQLite/Postgres indexer | checkpoint 与回补说明 |
| 14 | 可观测性 | 指标、benchmark、发布 | 性能报告 |
| 15 | 有限域 | FieldElement | 数学性质笔记 |
| 16 | 曲线与承诺 | 教学承诺演示 | 承诺与 Fiat–Shamir 笔记 |
| 17 | 算术电路/R1CS | 玩具约束系统 | witness/public input 解释 |
| 18 | ZK 闭环 | 第一份 proof demo | proving/verification 记录 |
| 19 | 论文阅读 | 论文复现笔记 | 三篇论文阅读卡片 |
| 20 | Rust ZK 库 | proof 生成/验证项目 | 负向测试与依赖审查 |
| 21 | 性能工程 | benchmark | 性能报告 |
| 22 | 综合项目设计 | 架构与 threat model | 设计评审 |
| 23 | 集成与安全测试 | Solana + ZK demo | 信任边界说明 |
| 24 | 发布与求职 | v1.0、视频、README | 简历项目描述与复盘 |

---

# 4. 推荐免费 MOOC 与长期学习资料

## Rust / 系统工程
- The Rust Programming Language：https://doc.rust-lang.org/book/
- Rust By Example：https://doc.rust-lang.org/rust-by-example/
- Rustlings：https://github.com/rust-lang/rustlings
- Tokio tutorial：https://tokio.rs/tokio/tutorial
- MIT Missing Semester（命令行、Git、调试、工具链）：https://missing.csail.mit.edu/

## 区块链与 Solana
- Solana 官方文档：https://solana.com/docs
- Anchor 官方文档：https://www.anchor-lang.com/docs
- Solana Cookbook：https://solanacookbook.com/
- Solana Program Examples：https://github.com/solana-developers/program-examples
- Ethereum Developer Docs（比较智能合约模型）：https://ethereum.org/en/developers/docs/
- Bitcoin 白皮书：https://bitcoin.org/bitcoin.pdf

## 数学与密码学
- MIT Mathematics for Computer Science：https://ocw.mit.edu/courses/6-1200j-mathematics-for-computer-science-fall-2024/
- MIT OCW（线性代数、概率等课程入口）：https://ocw.mit.edu/
- ZKProof 社区资源：https://zkproof.org/
- Justin Thaler 公开讲义：https://people.cs.georgetown.edu/jthaler/ProofsArgsAndZK.html
- Vitalik：ZK-SNARKs 入门文章：https://vitalik.eth.limo/general/2021/01/26/snarks.html

## ZK 工具与论文
- Circom 文档：https://docs.circom.io/
- snarkjs：https://github.com/iden3/snarkjs
- arkworks：https://arkworks.rs/
- Halo2：https://github.com/zcash/halo2
- RISC Zero：https://dev.risczero.com/
- SP1：https://docs.succinct.xyz/

### 如何选择资源
- 课程与工具文档优先用官方材料。
- 博客用于直觉理解，不替代安全规范和论文。
- 学习资料中的版本、命令和项目活跃状态可能变化，开始每个阶段时先检查官方仓库的最新说明。
- 不需要把每个工具都学一遍。每阶段选择一个主工具，完成项目，再决定是否扩展。

---

# 5. 求职策略：用工程证据代替“我学过 ZK”

## 作品集至少包含这四个项目

1. **Rust protocol parser**：证明你能用 Rust 写可靠、可测试的底层代码。
2. **Solana state machine / escrow**：证明你理解账户权限、PDA、状态迁移与安全测试。
3. **Solana indexer**：证明你能做真实服务、RPC、数据库、重试、幂等和运维。
4. **ZK proof demo + benchmark**：证明你理解 witness、public inputs、proof/verify 流程与性能取舍。

## 每个项目的 README 必须回答
- 为什么要做？
- 架构是什么？
- 如何在干净环境运行？
- 哪些情况会失败？
- 如何测试安全边界？
- 有哪些性能数据？
- 有哪些已知限制？
- 如果用于生产，还缺什么？

## 岗位搜索关键词
- Rust Engineer
- Solana Engineer / Solana Protocol Engineer
- Blockchain Infrastructure Engineer
- Solana Indexer / RPC Engineer
- ZK Engineer / Zero-Knowledge Engineer
- Prover Engineer / zkVM Engineer
- Cryptography Engineer（通常数学与密码学要求更高）

## 简历叙述建议
不要把“完成教程”写成工作经验，也不要声称自己是密码学专家。可以写成：
- Built a Rust-based binary protocol parser with streaming decode, explicit error handling, and boundary-focused tests.
- Implemented and tested Solana programs with PDA-based state, authorization constraints, and adversarial test cases.
- Built an idempotent Solana event indexer with checkpoint recovery and RPC retry handling.
- Implemented a proof-generation and verification demo, documenting public inputs, witness handling, security assumptions, and performance measurements.

等这些项目真正完成后再写进简历，并附仓库链接与可复现步骤。

## 远程求职行动
- 第 4 周开始：在 GitHub 每周发布一个有意义的里程碑。
- 第 8 周开始：关注 Rust/Solana 工程岗位，阅读职位描述并把缺口加入训练清单。
- 第 12 周开始：对开源项目做小型、可审查的贡献，例如修复文档、补测试或最小 bug。
- 第 16 周开始：针对岗位做定制化简历，不要群发同一版。
- 第 24 周：准备 Rust ownership、Solana account security、RPC/indexing、基础密码学与项目设计面试。

---

# 6. 每月复盘与转向规则

## 第 4 周复盘
如果 Rust 还需要大量照抄：
- 延长 Rust 训练 2 周；
- 暂缓 Anchor；
- 多写解析器、CLI、并发测试。

## 第 11 周复盘
如果能写程序但安全测试薄弱：
- 先补账户权限、PDA 校验、重复调用、状态迁移测试；
- 不要用“能部署”代替“能安全解释”。

## 第 14 周复盘
如果你喜欢服务端、数据管线、故障定位：
- 优先投 Rust/Solana infrastructure、indexer、RPC integration；
- ZK 作为差异化技能继续学，不必强迫自己成为理论研究员。

## 第 21 周复盘
如果你喜欢数学、论文、证明系统性能优化：
- 继续 ZK 系统工程、zkVM、prover 性能方向；
- 加强有限域、椭圆曲线、概率论、复杂度和证明系统。

如果你更喜欢链上业务、安全与协议：
- 继续 Solana program / protocol / security engineering；
- 把 ZK 当成应用能力，不必追求设计新的证明系统。

---

# 7. 安全与职业风险提醒

- 不要把学习项目部署到主网管理真实资产。
- 不要上传助记词、私钥、公司源代码、医疗设备日志、客户数据或专有协议。
- 不要自行设计生产密码算法或承诺方案。
- 资金类合约必须把重入/重复执行、权限、账户关系、升级权限、经济攻击与异常路径纳入威胁模型。
- ZK 证明“数学上验证通过”不代表整个应用安全；输入来源、证明电路正确性、可信设置、合约逻辑和业务规则都可能出错。
- 求职期间不要把投机交易收益当作可预测收入；先用工程作品获得面试机会。

---

# 8. 第一周现在就开始：Day 1–7

每天 1.5–2 小时即可。

### Day 1
- 安装 Rust stable，运行 `rustc --version`、`cargo --version`。
- 阅读 Rust Book 的 Getting Started 与变量章节。
- 创建 Git 仓库 `rust-solana-zk-roadmap`。

### Day 2
- 学所有权与借用。
- 写函数：输入 `&[u8]`，检查 magic、version 和 length。
- 为短输入、错误 magic 写测试。

### Day 3
- 学 `Result`、自定义错误类型、`?` 运算符。
- 为长度超限、checksum 错误定义独立错误。
- 禁止用 `unwrap()` 处理外部输入。

### Day 4
- 完成完整帧解码器。
- 定义协议格式并写进 README。
- 写 10 个边界测试。

### Day 5
- 实现多帧解析。
- 输入中间截断的包时，区分“需要更多数据”和“数据非法”。
- 写集成测试。

### Day 6
- 加入 fuzz testing（可选使用 cargo-fuzz；先读其官方说明）。
- 用随机输入确认解析器不会 panic。
- 记录发现的 bug 并修复。

### Day 7
- 从干净目录重新构建并运行测试。
- 写 `notes/week01.md`：设计、错误处理、测试覆盖、未解决问题。
- 推送代码。

**第一周完成定义：** 不只是“编译成功”，而是你能向另一位工程师解释解析器的输入契约、错误语义、内存安全策略和测试边界。

---

## 最后一句

你的最佳策略不是把过去二十多年的经验清零、从“区块链小白”重新开始，而是把已有的系统工程能力迁移到 Rust 与 Solana，再用数学和密码学训练逐步进入 ZK。**先建立可雇佣的 Rust/Solana 工程能力，再把 ZK 做成差异化专长**，通常比直接冲刺纯理论 ZK 岗位更稳妥。
