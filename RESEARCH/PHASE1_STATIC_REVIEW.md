# Zapret2：Phase 1 静态技术审查与网络适配预评估

**日期**：2026-11-23  
**研究负责人**：Nova（为 Beam 整理）  
**研究对象**：[`bol-van/zapret2`](https://github.com/bol-van/zapret2)  
**上游默认分支**：`master`  
**固定审查基线**：`a18ce3f58cf7719b794596b0b01ebd442c2a92fc`（连接器读取时间的 HEAD；后续可能前进）  
**目标仓库**：[`njzhb/zapret2`](https://github.com/njzhb/zapret2)（独立 Fork，已创建并核验）  
**STATUS**：`PARTIAL — FORK_VERIFIED / STATIC_REVIEW_ONLY / NO_RUNTIME_TEST`

**归档说明**：文内关于 GitHub App 403、未授权及未入库的叙述，属于初次上传的历史审计记录；不代表本报告存档后的权限状态。当前仍未进行驱动、代理或网络测试。

## 0. 结论

Zapret2 是真实、活跃的反 DPI 工程工具，**值得作为独立网络研究课题保存**；并非 VPN、国外服务器或通用网速加速器。它可能在某些基于 HTTP/TLS/QUIC 特征、SNI/协议签名的 DPI 阻断或整形场景改善连通性，但无法当然解决 DNS 污染、目的 IP 封禁、跨境路由拥堵、丢包、VPS 质量或运营商物理带宽问题。对于已经运行 v2rayN、VLESS+WS+TLS、Hysteria2 与 CORAL 的 Windows 主机，**当前不建议安装、提权执行或启用 WinDivert 全局抓包**。

## 1. 已核实的事实

1. 上游仓库：[`bol-van/zapret2`](https://github.com/bol-van/zapret2)。当前研究的是 Zapret2，不是旧版 `bol-van/zapret`，也不是社交媒体推广的第三方 Windows 包。旧版项目已明确把后续开发方向指向 Zapret2。相关文档：[`docs/readme.md`](https://github.com/bol-van/zapret2/blob/master/docs/readme.md)。
2. 技术定位：独立处理本机/路由设备流量，以阻碍部分 DPI 基于特征匹配或会话跟踪的封锁/限速；**无须外部服务器**，但也**不提供出口 IP 更换**。作者说明主要面向 OpenWrt，也兼容 Linux、FreeBSD、OpenBSD、Windows。
3. 核心结构：OS 内核捕获流量 → `nfqws2`（Windows 对应 `winws2`） → 协议/有效载荷解析 → Lua 可编程策略 → 按策略转发/修改/丢弃/注入。相关：[`docs/manual.en.md`](https://github.com/bol-van/zapret2/blob/master/docs/manual.en.md)、[`lua/`](https://github.com/bol-van/zapret2/tree/master/lua)、[`nfq2/`](https://github.com/bol-van/zapret2/tree/master/nfq2)。
4. Windows 使用 WinDivert 驱动。WinDivert 可以截获、修改、丢弃、注入数据包；首次打开时可能需要管理员权限并自动安装驱动。Windows 不能像 Linux `connbytes` 那样在内核中轻易限制“只截连接前几个包”，宽泛的全端口拦截可能影响 CPU/吞吐和其他过滤组件。官方手册：[`docs/readme.md`](https://github.com/bol-van/zapret2/blob/master/docs/readme.md)，WinDivert：[原作者文档](https://www.reqrypt.org/windivert-doc.html)。
5. 官方 Windows 构建：[`.github/workflows/build.yml`](https://github.com/bol-van/zapret2/blob/master/.github/workflows/build.yml) 产出 `winws2.exe` 与 Cygwin 运行时、WinDivert DLL/SYS；本地编译说明在 [`docs/compile/build_howto_windows.txt`](https://github.com/bol-van/zapret2/blob/master/docs/compile/build_howto_windows.txt)。社交媒体帖里的 `install.bat` + `start.bat` + v1~v10 菜单并非这次核对的上游仓库根目录标准安装入口，不能据此视为官方包。
6. 最近已核对正式发布版：[`v1.0.7`](https://github.com/bol-van/zapret2/releases/tag/v1.0.7)，2026-11-05 发布；它不是 11-22 的 master 源码。发布包中包含 SHA256 清单；未下载独立核验。
7. 许可证：[`docs/LICENSE.txt`](https://github.com/bol-van/zapret2/blob/master/docs/LICENSE.txt) 为 MIT；Fork 应保留原许可及版权声明。WinDivert 自身采用 LGPL-3.0，应另行保留第三方许可义务。

## 2. 对现有电信网络与代理体系的价值判断（推断，尚无实测）

| 问题 | Zapret2 可能的作用 | 判断 |
|---|---|---|
| 特定域名 TLS ClientHello/SNI、HTTP 首包或 QUIC 特征导致的 DPI 干扰 | 用协议变形、假包或切分使部分 DPI 判读失效 | **有研究意义；条件性有效** |
| VLESS+WS+TLS 到大阪节点的握手受到签名检查 | 仅在根因确实是 DPI 特征识别时可能间接改善 | **先证明阻断类型** |
| Hysteria2 UDP/443 | 在存在特定 UDP 特征封锁时可能有帮助；错误改写可能损害可靠性 | **高谨慎** |
| 对端 IP 被封、跨境路由拥堵、运营商高峰丢包或 TCP 重传 | 不会替换出口路径、提高国际带宽、修复远端节点 | **基本无效** |
| DNS 污染或解析异常 | 不等于可靠 DNS 修复工具 | **另行诊断** |
| CORAL 通过代理访问服务或交易端点 | 全局拦截可能修改/延迟网络报文；对于真实交易链路不可接受 | **禁止混用测试** |

## 3. 已识别的风险与边界

- **驱动/权限**：运行 `winws2` 涉及 WinDivert，不能当成普通“免安装 exe”对待。管理员权限、驱动加载、与其他过滤驱动冲突均需准备独立回滚。
- **供应链**：官方 Windows GitHub Actions 构建中，Cygwin 安装动作设置了 `check-sig: 'false'`，并使用旧的 HTTP 镜像；这属于可审计的构建链风险提示，**不等于发现恶意行为**。原作者构建同时下载固定版本 WinDivert。需进一步核验官方发布包 SHA256、来源、驱动签名与许可。
- **流量截获范围**：不应在宿主机直接截获全部 TCP/UDP 443，否则可能覆盖既有代理、实时程序及浏览器的关键连接。
- **第三方脚本**：截图所述 `install.bat/start.bat` 未在官方源码根目录核实。严禁直接运行社交媒体链接下载的“整合启动器”，先审核来源、哈希和操作内容。
- **策略正确性**：DPI 差异很大；某条对一个地区有效的 Lua 策略可能让其他站点中断。不存在可证明的“零成本、零延迟、全部直连满速”。
- **使用边界**：仅测试自己有权管理的网络和设备，遵守适用法律、运营商合同及相关规则。

## 4. 分阶段验证计划

### Gate A — 仅静态审查（当前）
- [x] 核对 GitHub 登录主体 `njzhb`，确认上游仓库及默认分支；读取近期提交 SHA。
- [x] 核对 MIT 许可、Windows 编译流程、WinDivert 依赖、Lua 策略架构、最新发布记录。
- [x] 确认第三方帖子脚本**不能直接等同官方入口**。
- [x] `njzhb/zapret2` Fork 已创建；GitHub API 核实 `fork=true`、`parent=bol-van/zapret2`，Fork `master` SHA 与基线一致。
- [x] 在 Fork 中归档本研究报告（`RESEARCH/PHASE1_STATIC_REVIEW.md`）；此前首次提交被 GitHub App 403 拒绝，后来重新执行归档。
- [ ] 逐文件复查 `nfq2/win.c`、`nfq2/nfqws.c`、Windows 构建脚本、`lua/` 默认策略及外部依赖来源；这轮并未宣称完整安全审计。

### Gate B — 隔离离线测试（另行授权）
- 在**不接触 Catherine 现有驱动/代理配置**的隔离环境中做构建或静态工具测试；先验证来源、哈希与签名。
- 不启动 WinDivert；不修改路由、防火墙、DNS、系统服务或启动项。
- 对不同 DPI 故障建立证据分类和可回滚测试矩阵；不以模拟数据替代真实线路结果。

### Gate C — 真实线路窄范围 A/B（尚未批准）
- 只在明确批准、已验证隔离和回滚方案后，选择低风险设备和单一测试目标进行测试。
- 记录基线：DNS、三次握手、TLS 握手、HTTP 成功率、延迟、丢包、网络重传；同时记录代理开关状态。
- 必须保证 CORAL/Hashcat 及现有代理不受影响；一旦连接中断、驱动冲突或系统网络异常即停止。

## 5. 本次执行与未执行

**实际执行**：使用已连接 GitHub 服务只读核对上游与 `njzhb` 账号、Fork 元数据和父子关系、远程查看目录、源码文档、Windows 代码、构建工作流、官方发布与近期提交；读取 WinDivert 官方技术文档。尝试向 Fork 新建研究文档，但 GitHub API 拒绝：`403 Resource not accessible by integration`，故未写入。

**没有执行**：没有成功写入 GitHub 研究报告；没有 clone/下载完整仓库；没有安装、编译或运行 Zapret2；没有安装/加载 WinDivert；没有访问或修改 Catherine、v2rayN、CORAL、Hashcat、防火墙、DNS、启动项；没有真实大陆电信线路 A/B 数据。

**NEXT**：保留 Fork 和固定 SHA；因为当前 GitHub App 未被授权访问新 Fork，可由仓库所有者在 [GitHub App 安装设置](https://github.com/settings/installations/181744205) 的选定仓库列表中加入 `njzhb/zapret2`，然后重试写入；也可通过 GitHub 网页手动上传本报告至 `RESEARCH/PHASE1_STATIC_REVIEW.md`。继续固定 SHA 的静态代码审查，不启动任何主机网络组件。

---

## 补充：Fork 建立后的核对与 Windows 静态发现（2026-11-23）

- API 核验 Fork：`njzhb/zapret2`；父仓库/源仓库：`bol-van/zapret2`；`master` SHA：`a18ce3f58cf7719b794596b0b01ebd442c2a92fc`。
- 源码 `nfq2/darkmagic.c` 能找到 `WinDivertOpen` 和 `WinDivertClose` 调用；`nfq2/win.c` 存在 Windows service STOP/SHUTDOWN 处理，但这**不等于在实际系统成功卸载驱动或恢复所有连接**。
- Windows 构建 workflow 会从官方 WinDivert 项目指定版本获取 DLL/SYS，并将它与 Cygwin 运行时及 `winws2.exe` 打包；Cygwin 工具链部分使用 HTTP 旧镜像且 `check-sig: false`，需要审计构建链和成品哈希。
- WinDivert 官方说明：首次 `WinDivertOpen` 可自动加载驱动，且调用进程需要管理员权限；退出进程与移除驱动的效果需要在隔离环境验证，不能在 Catherine 直接实验。
- 写入尝试：当前 GitHub 集成在创建 `RESEARCH/PHASE1_STATIC_REVIEW.md` 时收到 `403 Resource not accessible by integration`。进一步核对 GitHub App installation：该应用当前选定的仓库仅有 `njzhb/CORAL-Research-Cloud`，**不包括**新建的 `njzhb/zapret2`；这可以解释向新 Fork 写入时报 403。仓库本身存在且用户拥有管理权限，两个事实不冲突。

---

## 参考资料

- GitHub 上游：<https://github.com/bol-van/zapret2>
- 上游简介：<https://github.com/bol-van/zapret2/blob/master/docs/readme.md>
- 英文技术手册：<https://github.com/bol-van/zapret2/blob/master/docs/manual.en.md>
- Windows 构建说明：<https://github.com/bol-van/zapret2/blob/master/docs/compile/build_howto_windows.txt>
- CI 构建流水线：<https://github.com/bol-van/zapret2/blob/master/.github/workflows/build.yml>
- 官方发布：<https://github.com/bol-van/zapret2/releases/tag/v1.0.7>
- MIT 许可：<https://github.com/bol-van/zapret2/blob/master/docs/LICENSE.txt>
- WinDivert 官方技术文档：<https://www.reqrypt.org/windivert-doc.html>

**证据等级**：官方源码和文档证据为主；适用性分析属于逻辑推断，不等于在 Beam 网络上实测有效。