# Aether.Coin.Desktop — 设计说明

## 1. 这个仓是什么、不是什么

| 是 | 不是 |
|---|---|
| 虚拟币线的 **GUI**：钱包 App 的共享 View/VM（`Wallet.App`）+ 桌面/Android/iOS 三个 head | 账本实现——引擎在 ① `aether-ledger`，本仓一行链逻辑都没有 |
| ① 的**第二个客户端**：与 ⑯ `aether-coin-hosts` 的钱包 CLI 完全平级（同一条 RPC、各自的种子、自己签名） | 宿主仓——节点/CLI/验证器/部署在 ⑯；`deploy-ledger` 从不构建本仓 |
| UI 消费者（与 ⑪⑮ 同族的 GUI 仓形态） | 布局引擎消费者——暂不引 ⑩，单页竖流够用 |

## 2. 为什么从 ⑯ 拆出来（2026-10-28）

钱包 UI 骨架当日先落在 ⑯（当时判断「一个新客户端，不是新引擎，也不是新线」，不值得开仓）；同日拆出，理由：

- **对齐 CTP 线的分法**：⑧ `aether-ctp-hosts`（宿主）/ ⑪ `aether-ctp-desktop`（GUI）——币线同构：⑯ 改名 `aether-coin-hosts`，GUI 进本仓；
- **部署域分道**：宿主三件套走 `deploy-ledger`（服务器 Windows 服务），GUI 是本地 / 移动 App 分发——两类发布通道没有共享构建上下文的理由；
- **演进驱动不同**：UI 跟着 UX / 种子存储走，宿主跟着 ① 引擎接线走；
- 拆分是纯搬家：`Wallet.App` 只引 ①、三头只引 App、零消费者切换（与 ⑧→⑯ 那次拆分同款且更简单）。

## 3. 一个核三个头，种子能力是头决定的

```
        Wallet.App（View + VM + WalletService，零平台 API）
        ├─ Wallet.Desktop   Avalonia 桌面窗口（WinExe）+ --selftest
        ├─ Wallet.Android   net10.0-android，TFM/接线就位
        └─ Wallet.iOS       net10.0-ios，TFM/接线就位
```

三条约束，都是「宁可功能少，也不做错」：

- **VM 不引用任何平台 API**：`Aether.Ledger.Wallet.App` 只引 ①，view 层是同一份 XAML。手机上不用第二套界面——布局是「一条竖流」（`ScrollViewer` + `MaxWidth`），窄屏不挤、宽屏不散。
- **种子不是 VM 的事**：全部经 `ISeedStore`（`HasSeed` / `Describe` / `ReadAsync` / `WriteAsync`）。桌面 head 注入 `FileSeedStore`（明文 + `0600`），移动 head 注入 `MissingSeedStore`——`WriteAsync` 抛 `NotSupportedException`，VM 统一捕获并把界面切成「种子存储未接入」。**手机端在 Keychain / Keystore 落地前不碰真钱**，这比「先写个明文文件占位、以后再换」安全得多：占位方案会真的被用来存钱。
- **读种子必须显式触发**：VM 构造不读种子（只查 `HasSeed` 显示提示），点「刷新/转账」才惰性读入内存；导入种子后立刻把输入框清空。

为什么敢在手机上只留接线：`net10.0` + 纯托管 Ed25519（① 的选型原因之一就是 libsodium 在 iOS 链接不过）已经证明可移植；缺的只是**存种子的地方**，那是平台 API，恰好是头应该负责的部分。

验证方式：`--selftest` 不开窗口，进程内起真节点 → 从日志读 treasury 种子 → 走 VM 的 `ImportWalletCommand` / `TransferCommand` → 查收款余额。它同时证明三件事：链路通、VM 逻辑对、**XAML 绑定的属性名没写错**（VM 改了名而 XAML 没改，编译不报错，只会静默不更新——这类 bug 只有跑起来才抓得到）。

## 4. 运行与验证

- 构建：`dotnet build Aether.Ledger.Wallet.Desktop/…csproj -c Release`（CI 只构建 App + Desktop；android/ios 在 Linux runner 没有 workload，进 CI 只会红，由 macOS 本机构建）。
- L2（自动，无 GUI）：`dotnet run --project Aether.Ledger.Wallet.Desktop -c Release -- --selftest` → 6 项断言全绿。
- L2（手工）：起 ⑯ 节点 → 桌面 head 连 `127.0.0.1:17805` → 建/导钱包 → 转账 → 余额刷新。
- 移动 head：Android `dotnet build …Android.csproj`；iOS `dotnet build …iOS.csproj -c Release -p:RuntimeIdentifier=iossimulator-arm64`（AOT 约 18 分钟）。

## 5. 已知约束与待办

| # | 级别 | 事项 | 现状 |
|---|---|---|---|
| 1 | P1 | 移动端种子安全存储 | 移动 head 注入 `MissingSeedStore`——能编译、能连节点、能查余额，但**不能建/导钱包**（界面禁用 + 一行说明）。第一批做 Keychain / Android Keystore |
| 2 | P1 | 节点可达性 | `:17805` 默认只绑回环；手机要连须开监听 / 隧道 / 反代，且节点不该直接暴露公网（iOS 连局域网弹本地网络权限） |
| 3 | P1 | RPC 明文无鉴权 | 真要远程 / 手机连公网节点，必须先上 TLS + 鉴权 |
| 4 | P2 | 无测试项目 | UI 验证目前靠 `--selftest`；VM 逻辑积累后补 xunit（对齐 ⑮ 的 VM 级集成测试做法） |
| 5 | P3 | 反射绑定 | `AvaloniaUseCompiledBindingsByDefault=false`（与 ⑪⑮ 同款）；改 compiled bindings 是一次单独的重构 |

## 6. 与文档的关系

- 宿主与部署：⑯ `aether-coin-hosts`（README / `doc/funding-runbook.md`）
- 迁移决策：工作区根 `MIGRATION.md` §3 / §16.1
- 工作区级架构：工作区根 `README.md` §一 B
