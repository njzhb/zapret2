# Zapret2 — Phase 2：Windows 网络链路与供应链静态审查

- **日期**：2026-11-23
- **项目**：[`njzhb/zapret2`](https://github.com/njzhb/zapret2)（上游 [`bol-van/zapret2`](https://github.com/bol-van/zapret2) 的 Fork）
- **固定源码基线**：`a18ce3f58cf7719b794596b0b01ebd442c2a92fc`
- **状态**：`PARTIAL / WINDOWS_STATIC_REVIEW / DRIVER_EXECUTION_HOLD`
- **证据边界**：连接 GitHub 读取官方源码和构建文档，结合 WinDivert 官方技术手册；没有完整安全审计、源码编译、驱动安装或真实线路测试。

**归档说明**：本报告中的 GitHub App 403 与未上传叙述是此前首次尝试的历史审计记录，不代表本次归档后的状态。仍未进行驱动、代理或网络测试。

## 1. 从截图宣传到实际能力

流传帖子说“本地开源神器”“无需 VPN/服务器”“绕过 SNI 阻断”“满速复活”。源码支持的技术事实是：Zapret2 通过捕获/处理本机或路由设备上的数据包、实现可编程的 DPI 规避策略；它不提供新的出口 IP、境外服务器或通用加密隧道。对于仅以 TLS/HTTP/QUIC 特征判定的 DPI 干扰，技术思路成立；对于目的 IP 封锁、DNS 污染、跨境拥堵或服务端故障，不存在万能方案。**不应把个别地区的成功经验外推为所有运营商与所有网站均可直连。**

## 2. Windows 实际运行链路

基于作者文档和源码，Windows 路径为：

```text
应用发起 TCP/UDP 通信
    ↓
Windows 内核网络栈 / WFP
    ↓
WinDivert 过滤器（取决于配置的拦截范围）
    ↓
winws2.exe：协议判读、选中匹配策略
    ↓
Lua 规则按条件处理报文
    ↓
WinDivert 重新注入/放行/丢弃
    ↓
继续向目标连接
```

注意事项：

- [`nfq2/darkmagic.c`](https://github.com/bol-van/zapret2/blob/a18ce3f58cf7719b794596b0b01ebd442c2a92fc/nfq2/darkmagic.c) 有 `WinDivertOpen` 和 `WinDivertClose` 代码路径，但单凭源码不能保证每一种异常退出都能让系统完全恢复。
- [`nfq2/win.c`](https://github.com/bol-van/zapret2/blob/a18ce3f58cf7719b794596b0b01ebd442c2a92fc/nfq2/win.c) 提供服务层 STOP/SHUTDOWN 处理，并通过 `bQuit` 请求退出；这不等于实际验证过 Windows 驱动卸载。
- [`nfq2/nfqws.c`](https://github.com/bol-van/zapret2/blob/a18ce3f58cf7719b794596b0b01ebd442c2a92fc/nfq2/nfqws.c) 显示 Windows 拦截过滤器构造逻辑、优先级、重复实例防护、loopback/LAN 等选项；这说明支持选择性过滤，**不说明任何默认策略天然安全**。
- 作者文档承认 WinDivert 无法像 Linux 的 connbytes/ct-packets 那样限制“仅捕获连接最初若干包”。如果全端口捕获整个数据流，CPU 开销和连接干扰风险会放大。

## 3. 驱动风险：不是双击普通工具那么简单

[WinDivert 2.2 官方文档](https://reqrypt.org/windivert-doc.html) 确认：

- `WinDivertOpen()` 可触发按需自动安装/加载驱动，调用进程须具备管理员权限。
- 它可捕获、过滤、修改、阻断、重新注入 IPv4/IPv6 网络包。
- 需要正确停止使用该驱动的应用；卸载通常还涉及服务或重启过程，不能在与其他可能共享 WinDivert 的程序同时运行时随意操作。
- 官方发布的预编译驱动有数字签名，但**必须检查下载到手的实际二进制**，本轮未核验签名。

在使用 v2rayN/本地代理的 Windows 电脑上，这类软件可能影响的是网络协议栈中的报文路径，而不仅是浏览器页面。它与现有代理**有可能共存，也有可能冲突**；必须通过隔离、定向过滤和回滚测试来证实。

## 4. Windows 发布供应链观察

上游 [GitHub Actions 构建脚本](https://github.com/bol-van/zapret2/blob/a18ce3f58cf7719b794596b0b01ebd442c2a92fc/.github/workflows/build.yml) 静态核查结果：

1. Windows 构建使用 GitHub Actions runner，设置 MSYS2/Cygwin。
2. Cygwin 安装指向历史 HTTP 镜像，参数包含 `check-sig: 'false'`。**这是软件供应链完整性验证薄弱点**，尤其值得在使用 release 二进制前独立核查。
3. 工作流编译 `winws2.exe`，打包 `cygwin1.dll`，并通过 WinDivert 上游固定版本 `v2.2.2` 下载 `WinDivert.dll` 与 `WinDivert64.sys`。
4. 官方 `v1.0.7` release 提供 `sha256sum.txt` 和完整压缩包，但**本轮没有下载、验签或逐项比对真实安装包**。校验清单本身并不能消除发布者或构建链被攻破的风险。
5. 截图流传的 `install.bat`、`start.bat`、v1-v10 策略选择，不是已核实的原作者仓库根目录标准入口，**不应未经审阅以管理员运行第三方整合包**。

## 5. 对既有网络的判断（条件推断）

| 故障假设 | 预计价值 | 风险/优先排除因素 |
|---|---|---|
| DPI 识别某类 TLS/HTTP/QUIC 报文并注入重置或阻断 | **值得研究** | 需记录失败类型；各运营商策略不同 |
| 大阪 VLESS+WS+TLS 某节点 SNI/证书失败 | **条件性** | 先区分证书/SNI 配置错误、IP 封锁与 DPI |
| Hysteria2 UDP443 不通 | **条件性且谨慎** | UDP 丢包、线路质量、服务端限制更常见；错误改写可能加重失败 |
| 大陆国际出口拥堵、跨境 TCP 重传/晚高峰衰减 | **基本不能直接解决** | 不更换实际路由路径或出口带宽 |
| 远端 VPS 的 TLS、WS、网络服务异常 | **不能解决** | 先修服务器端问题 |
| 想让浏览器不走代理，直接获得外国公网出口地址 | **不具备此能力** | Zapret2 并非 VPN 或海外出口 |

## 6. 建议的审计门槛

**Gate A — 只读（允许继续）**：固定源码 SHA；继续审查 `nfq2/darkmagic.c` 的 handle 生命周期与关闭异常、`nfq2/nfqws.c` 过滤器范围，检查 Windows 二进制的构建脚本与哈希文件；输出源文件/证据引用。

**Gate B — 隔离（待明确批准）**：只在隔离、可丢弃且不会影响当前 Windows 宿主机关键网络的环境测试驱动生命周期及异常退出回滚；独立验证 WinDivert 签名/哈希和二进制来源；不触碰 Catherine 现有代理。

**Gate C — 真实线路（暂缓）**：只在有证据支持 DPI 假设后，做单一非关键测试目标、单一协议的 A/B；测试期间隔离 CORAL，记录正常/异常访问率、握手结果、RTT、吞吐和系统资源；失败即时停止并验证回滚。

**禁止在未批准前执行**：以管理员运行任何 `install.bat` 或 `winws2.exe`、加载 WinDivert、全局捕获 TCP/UDP 443、改动 Windows 防火墙与网络栈、替换 v2rayN 现有主用链路、动 CORAL/Hashcat。

## 7. 本次已执行 / 未执行

**已执行：** GitHub 远程读取仓库元数据、父子 Fork 关系、源文件的选定区间及查找匹配、官方工作流、发布清单、WinDivert 官方文档；分析和形成研究记录。

**未执行：** 完整静态代码审计、编译、二进制下载与 SHA256 实物核验、病毒扫描、驱动安装、系统级运行、真实电信线路 A/B、修改仓库源码或电脑网络配置。

**仓库写入受阻：** 虽然已确认 `njzhb/zapret2` Fork 存在，GitHub 连接在提交 `RESEARCH/PHASE1_STATIC_REVIEW.md` 时报告 `403 Resource not accessible by integration`。进一步查询 GitHub App 安装范围发现，当前仅选择了 `njzhb/CORAL-Research-Cloud`，尚未把 `njzhb/zapret2` 加入安装授权仓库。因此不能把报告标记为已入库。

**NEXT：** 由仓库所有者使用 GitHub 网页上传本文件至 `RESEARCH/PHASE2_WINDOWS_SOURCE_REVIEW.md`，或在 [GitHub App 安装设置](https://github.com/settings/installations/181744205) 中将 `njzhb/zapret2` 加入“选定的仓库”后再尝试通过连接器写入。源码与网络部署继续 HOLD，保持独立研究。