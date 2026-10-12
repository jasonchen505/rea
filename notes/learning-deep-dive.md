# REA 深入学习

上游 morluto/rea，MIT。fork 头为 7c9c1eb（上游 6.4.0，2026-10-11 发布）。2026-10-12 实测上游 136,222 stars / 30,603 forks，2026-04-14 创建，6 个月涨到 13 万星，增速罕见。npm 包名 rea-agents，本地单包结构（src/ 直出，1571 个 TS 文件，depth-1 clone 约 39MB）。

一句话：把逆向工程变成 agent 可调用的本地 MCP 工具集，从 App 行为一路下钻到 native binary，每一步结论都附证据和局限。

## 它解决什么问题

Agent 时代之前，逆向工程是人开着 Hopper/IDA/Ghidra 手工干的活。REA 的定位不是再造一个反汇编器，而是一个编排层：agent 说"看下这个 App 的搜索功能怎么实现的"，REA 负责选引擎、跑分析、把证据整理成 agent 能消费的结构化结果。关键洞察是逆向结论天然带不确定性，所以整个系统围绕"证据与未知"设计，而不是假装每个问题都有确定答案。

## 整体架构

入口只有一个分发器 scripts/rea.mjs，分两路：

- src/cli.ts：one-shot CLI（setup、诊断、分析、反编译）
- src/main.ts：MCP stdio server

两路共享 src/composition/，这是唯一的 production wiring 点。createBinarySession 把 Hopper/Ghidra/IDA 三个 deep provider 装进 AnalysisProviderRegistry，再挂一串 lazy 的 auxiliary provider。CLI 和 MCP 因此共享完全一致的 provider 选择、会话、Evidence 语义。分层依赖严格向内：

contracts（对外 schema）→ domain（纯函数：evidence、比较、应用图）→ provider adapters（各引擎协议与解析）→ application（共享工作流：会话、注册表、调查记录）→ adapters（server/、cli/）

AGENTS.md 把这条规则写死：引擎协议代码不许进 domain/application 层。

## 会话与 Provider 模型（核心设计）

这是全 repo 最值得学的部分。

- open_binary 打开目标后建立 BinarySession，一个 target 一次 immutable deep binding。之后的所有操作都路由到这次绑定，不再重新选型。
- AnalysisProviderRegistry 对候选做确定性排序选择，明确写着 no fallback。选不上就显式失败，不静默降级。
- 三个 deep provider（Hopper/Ghidra/IDA）之外是 disjoint 的 auxiliary families（JS、Android、firmware 等），由 SessionProviderRouter 做互斥路由，保持各 family 独立。
- binary_session 工具是 catalog 身份的权威来源。tools/list 永远返回完整工具清单，不可用的工具也列出来，只是标注不可用。开/关目标不改变 catalog，不发 list_changed 通知。
- 快照：BinarySessionRecords + AnalysisSnapshotCache 做二进制目标与 profile 的快照，支持兼容 facade 和 post-commit 通知。

## Evidence 与"未知"的认识论

- src/application/investigation/ 里有 EvidenceLedger 和 InvestigationRecords。工具返回完整 evidence inline，附带 provider 身份、源码位置、confidence 和 limitations。
- 大分析不走全量 payload：retained-evidence 引用（{"kind":"retained-evidence","evidence_id":"ev_..."}），配合 summaries/views 按需展开。这是给 agent 做 context 工程的务实设计。
- 缺失的覆盖率一律记为 unknown，不当作 absence。tool-design.md 反复强调：observed、derived、inferred 三种关系必须区分；relationship_coverage 的 complete 只表示"provider 观测到的直接关系集无已知截断返回"，不承诺穷尽所有运行时边。
- 错误用 tagged error algebra + Result 建模，user-visible-errors.csv 维护对外错误目录，error-contract.schema.json 做 schema 约束。

## Tool 设计哲学（docs/tool-design.md）

六种工具形状，按分析师的问题选型，不按前缀强制：

- inspect：单个目标的事实；search/list：候选发现；trace：证据间关系（typed edges）；compare：显式配对的工件；workflow：需要组合的常发结果；observe/capture：运行时行为（必须如实声明进程/文件/网络/UI 副作用和清理状态）

原则：优先 primitives，用 workflow 组合；public 命名保持 provider-neutral，各 provider 的真实覆盖率如实报告；limit 只允许来自真实的格式、协议、权限或实测资源约束，"agent 预算不够"不算加 limit 的理由。输入 schema 严格，广告出去的 schema 不得有递归引用（为模型 API 兼容还做了 union 拍平）。

## 各分析域覆盖矩阵

- Native deep：Hopper（Unix socket + bridge/hopper_bridge.py 跑在 Hopper 进程内）、Ghidra（headless，ReaGhidraBridge.java 打包脚本，ephemeral DB，不修改可执行字节，不接管 GUI）、IDA（适配上游 ida-pro-mcp，live GUI 或自建 headless DB，只读，绝不保存/关闭已挂载的 GUI 数据库）、JEB（调 caller 自起的本地 JEB MCP server）、pwntools/pwndbg（BYO 离线 ELF）。
- JS/Electron：自研管线。parse → 静态分析 → 语义分析/IR，全在独立 worker 进程里跑（可配 heap 与超时预算，资源耗尽时保留已完成的事实和显式 unknown）。读 ASAR 或解包目录，做 AST-only bundle 投影，source map 用 pinned codec 解码，识别 Electron 主/渲染进程边界，V8 inspector 被动发现，runtime observation 再与静态结果 reconciliation。
- Managed：.NET PE/CLI 元数据 + IL，NativeAOT 只读解析（x64 PE32+/.NET 8），Go 二进制，EVM（EVMole，有界 worker），WASM（WABT 由 caller 提供）。
- Android：APK 静态（pinned headless JADX）+ ADB 真机（device inspect、split-APK 获取、logcat、截图、文件传输、app lifecycle），Apktool 解 manifest 与本地化字符串。
- Firmware：binwalk/unblob 适配，自有进程监管，报告归一化，解压到私有目录。
- Browser：CDP/Playwright 捕获（coverage、listener、source attribution），HAR（含 BOM 前缀容忍），WebSocket，scenario（storage 分区播种、loopback）。
- Process：src/process 统一监管，owned process groups、private roots、deadlines、有界文件读；Windows 有专用 Node-API 实现（Job Object、DACL）。

## Setup 与 Skill 的产品思考

- npx rea-agents setup：展示计划变更、要求批准、自动备份；支持 Claude Code、Codex、Cursor、Gemini CLI、Grok Build、Pi、Hermes、Qwen Code 等一长串 agent；明确承诺不安装/升级任何无关软件（Node、Homebrew、Java、Ghidra、IDA 全是 BYO）。
- rea doctor：检查各客户端的 JSON/TOML 注册，把 command vector 标为 aligned/stale/missing/invalid。
- .agents/skills/reverse-engineer-anything（已 v39）+ skills.sh 上架。skill 指导语很克制：按 target 路由首个工具；工具可用就别跑 setup；普通源码仓库分析不要用 REA。
- 针对真实客户端上限的务实处理：REA_MCP_INPUT_SCHEMA_PROFILE=compact，把单个工具 input schema 压到 13312 字节以内，因为 Moonshot/Kimi 类客户端超过约 13.7KB 会直接拒掉整个请求（issue #1484 的实测结论）。

## 工程实践

- 配置全走 zod 校验 env；进程、文件、网络副作用在工具契约里如实声明。
- 测试：vitest，tests/{acceptance,boundary,conformance,evaluation,fixtures}，fixtures 里放 fake launcher/bridge/CDP 缝。测试哲学写在 docs/testing.md：真实 CLI/MCP workflow 优先，模拟 provider 不能证明真实 provider 的行为；删测试时必须保留 malformed 输入、取消、清理、权限、容量、目标身份的覆盖。
- 评估：src/evaluation 做 agent release evaluation，对 session transcript 打分（含 known-answer 评估）。用 agent 自己的行为 transcript 做回归，这是少见但合理的路子。
- 发布：release-please + Conventional Commits，CHANGELOG 逐条致谢外部贡献者（6.4.0 一次致谢了几十人）。docs/public/product-catalog.json 等是构建产物，不入库。
- 合规姿态明确：DISCLAIMER 划定 lawful use（安全研究、恶意软件分析、CTF、互操作、软件保存、调试自有软件），DRM/作弊/未授权测试/复制专有代码等请求在 issue 里直接关闭不回应。

## 局限与诚实评价

- Deep native 能力完全依赖 BYO 引擎，REA 自身不做反汇编，是编排加证据层。没有 Hopper/Ghidra/IDA 的用户只能用到 JS/Android/firmware/browser 这些自研管线。
- Windows 支持是 P0 级别，部分路径只读；NativeAOT 的 Windows-host Ghidra workflow 自己都标注未在真机验证。
- 复杂度是 provider-neutral 的代价：61 个 server 注册模块，大量 contracts，新人学习曲线陡。
- 与单引擎 MCP（如 ida-pro-mcp）比，差异化不在反汇编质量，在会话模型、evidence-first 和跨域一站式（一个 MCP 覆盖 native/JS/Android/firmware/browser）。
- 6 个月 13 万星但项目年轻，大版本迁移文档（migration-6.2.md）说明 API 还在快速变化。

## 对我的启发

- "缺失覆盖是 unknown，不是 absence"，以及"不可能的状态让它 loud 地炸"（registry 无 fallback）：给 agent 设计工具时，显式表达不确定性比假装确定更有价值。这和之前从 labs-molt 维护者那里学到的是一脉相承的。
- retained-evidence 引用加 summary/view 的渐进展开，是处理大分析 context 爆炸的干净模式。
- compact schema profile 说明：做 MCP server 要实测目标客户端的真实上限，而不是假设 schema 越全越好。
- 一个 composition 点同时服务 CLI 和 MCP，保证两路语义一致，避免了双实现漂移。
