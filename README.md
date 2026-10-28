# Aether.Coin.Desktop

**虚拟币线的 GUI 仓**——钱包图形界面，一个核三个头：`Wallet.App`（共享 View/ViewModel/服务层，零平台 API）+ 桌面 / Android / iOS 三个 head。与 ⑮ `aether-ai-app` 同一套「共享 `net10.0` 库 + 平台 head」形态（方案A）。

**定位 = ① `aether-ledger` 的第二个客户端**，与钱包 CLI（⑯ `aether-coin-hosts` 的 `Aether.Ledger.Wallet`）完全平级：走同一条节点 RPC、持自己的种子、自己签名，节点不知道它存在。本仓不实现任何链逻辑——引擎在 ①，这里只有 UI。

2026-10-28 自 ⑯ 拆出（同日 ⑯ 由 `aether-coin` 改名 `aether-coin-hosts`），对齐 CTP 线 `aether-ctp-hosts` / `aether-ctp-desktop` 的「宿主 / GUI」分法；项目名 / 命名空间零改动（`Aether.Ledger.Wallet.App` 一族保持不变）。

## 组成

| 目录 | 形态 | 是什么 | 引 |
|---|---|---|---|
| `Aether.Ledger.Wallet.App` | UI 核心 | View / ViewModel / 服务层，三端共享 | ① |
| `Aether.Ledger.Wallet.Desktop` | 桌面 head | Avalonia 桌面窗口（WinExe）；含 `--selftest` 无头自检 | App |
| `Aether.Ledger.Wallet.Android` | 手机 head | `net10.0-android`；TFM/接线就位，种子安全存储待第一批 | App |
| `Aether.Ledger.Wallet.iOS` | 手机 head | `net10.0-ios`；同上 | App |

种子能力全部经 `ISeedStore` 由 head 注入：桌面 = `FileSeedStore`（明文 + `0600`，运维级托管），移动 head 目前**故意注入 `MissingSeedStore`**——界面上直接显示「本平台还没有接种子安全存储」，而不是偷偷写明文。**Keychain / Keystore 落地前，手机端不参与真实资金。**

## 移动端上线前的三关（按序）

1. **种子落盘**：手机必须进 Keychain / Android Keystore——种子 = 全部财富；明文占位方案会被真的用来存钱，宁可不接。
2. **节点可达性**：`:17805` 默认只绑回环；手机要连须开监听 / 防火墙 / 隧道 / 反代，且节点不该直接暴露公网（iOS 连局域网还会弹本地网络权限）。⚠ 2026-10-28 现状：生产节点是 `0.0.0.0:17805` 公网可达（与这条建议相反，功能优先，安全性待办）。
3. **传输安全**：节点 RPC 是明文 TCP JSON-RPC、无鉴权——公网可达前必须加 TLS（或前置网关鉴权）。

## 构建与测试

```bash
# 共享层 + 桌面 head（CI 就是这条线）
dotnet build Aether.Ledger.Wallet.Desktop/Aether.Ledger.Wallet.Desktop.csproj -c Release

# 钱包 UI 无头自检：进程内起真节点 → 从日志读 treasury 种子 → VM 命令建/导钱包 → 转账 → 查余额（6 项断言，退出码 0 = 通过）
dotnet run --project Aether.Ledger.Wallet.Desktop -c Release -- --selftest

# 跑桌面界面（需要图形环境）
dotnet run --project Aether.Ledger.Wallet.Desktop -c Release
```

移动 head 只在 macOS/Linux 本机构建，不进 CI（Linux runner 没有 android/ios workload）：

```bash
dotnet build Aether.Ledger.Wallet.Android/Aether.Ledger.Wallet.Android.csproj -c Release
dotnet build Aether.Ledger.Wallet.iOS/Aether.Ledger.Wallet.iOS.csproj -c Release -p:RuntimeIdentifier=iossimulator-arm64
```

实测（2026-10-28，macOS，拆仓前在 ⑯ 侧验证）：Android 0 警告 0 错误；iOS 0 错误、41 个警告全部是 IL2xxx trim 警告（Avalonia / BouncyCastle 的反射裁剪提示），Release 全量 AOT 约 18 分钟——首次构建别以为是卡死。

`slnx` 里放全 4 个项目只为 IDE 可见；**CI 不 build 整个 slnx**（会把 android/ios TFM 一起拽进来）。`--selftest` 与 ⑮ 的 `--smoke` 同一个理由：没有显示器的环境也要能验「XAML 绑定的属性名/命令真的通」。

## 边界

- **只引 ①**（`Aether.Ledger`：LedgerClient / LedgerWallet / 交易签名，与 CLI 引同一个项目）。不引 ⑩ `aether-workbench`（单页竖流布局够用，要统一外观时再引）、暂不引 ④（提现 RPC 属按需后续）、不认识任何 CTP 类型。
- 节点在 ⑯ `aether-coin-hosts`（`Aether.Ledger.Node`，默认 `127.0.0.1:17805`）——本仓是它的客户端，`--node` 可指远程（SSH 隧道形态）。
- 服务器部署与本仓无关：`deploy-ledger` 只管 ⑯ 三件套，从不构建 UI。

## 来历

2026-10-28 钱包 UI 骨架先落在 ⑯（当时口径「一个新客户端，不是新引擎，也不是新线」，不值得开仓）；同日拆出独立仓 + ⑯ 改名 `aether-coin-hosts`。拆分理由：GUI 与宿主的部署域（App 分发 vs `deploy-ledger`）、演进驱动（UX/种子存储 vs 引擎接线）不同，且 ⑯ 保持「纯宿主仓」与 ⑧ 同构。拆分是纯搬家：`Wallet.App` 只引 ①、三头只引 App、零消费者切换。
