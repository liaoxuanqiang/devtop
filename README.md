# DevTop

通过 **GitHub Actions** 在云端按需启动一台 **Windows 远程开发服务器**，借助 **Tailscale** 组网，实现随时随地通过 **RDP** 安全访问自己的云端桌面环境。

---

## ✨ 功能特性

- 🖥️ **一键启动 Windows 云桌面**：使用 GitHub 官方的 `windows-latest` Runner，无需自备服务器
- 🔑 **基于 GitHub Secrets 的安全认证**：RDP 密码从 Secrets 读取且不出现在日志中，用户名从 runner 进程读取
- 🌐 **Tailscale 私有组网**：通过 WireGuard 协议加密组网，无需公网 IP，也无需在路由器上做端口映射
- 🔒 **RDP 远程访问**：支持 `mstsc`（Windows） / Microsoft Remote Desktop（macOS / iOS / Android）等标准 RDP 客户端
- ⚡ **自动清理旧设备**：每次启动都会通过 Tailscale API 删除历史遗留的 `devtop` 设备，避免设备堆积
- 🧹 **自动卸载不必要的软件**：自动移除 Chrome、Firefox、Unity Hub、R 等占用资源的预装软件
- 🕐 **手动 / 定时双触发**：支持在 GitHub 页面手动触发，也可按计划自动运行

---

## 🛠️ 工作原理

整个流程由 `.github/workflows/devtop.yml` 定义，包含以下 5 个阶段：

| 步骤 | 说明 |
|------|------|
| ① 配置核心 RDP 设置 | 启用远程桌面服务和 NLA，保持 Windows 防火墙开启，仅允许 Tailscale IPv4 地址访问 RDP |
| ② 配置现有 RDP 用户 | 从 runner 进程读取当前用户名，为现有本地账户设置 `RDP_PASSWORD`；不创建账户或修改用户组 |
| ③ 卸载不必要的软件 | 通过注册表遍历卸载列表，清理 Chrome / Firefox / Unity Hub / R / Rtools 等 |
| ④ 安装并配置 Tailscale | 安装 Tailscale，完成设备认证、清理旧设备并固定主机名，广播子网路由并启用 Exit Node |
| ⑤ 保持连接 | 循环输出日志保活 Runner，确保云桌面在任务超时（默认 360 分钟）前持续在线 |

---

## 📋 使用前提

### 1. 配置 GitHub Secrets

进入仓库 **Settings → Secrets and variables → Actions → New repository secret**，添加以下 Secrets：

| Secret 名称 | 是否必填 | 说明 |
|-------------|----------|------|
| `RDP_PASSWORD` | ✅ 必填 | 当前 runner 账户的远程登录密码，建议使用强密码（大小写字母 + 数字 + 符号） |
| `TAILSCALE_AUTH_KEY` | ✅ 必填 | Tailscale 认证密钥，用于让新设备加入你的 tailnet 网络 |
| `TAILSCALE_API_TOKEN` | ⚠️ 可选 | Tailscale API Token，配置后可自动清理旧设备并管理设备名称 |

> 💡 获取 Tailscale 密钥：
> - **Auth Key**：登录 [Tailscale 管理后台](https://login.tailscale.com/admin/settings/authkeys) → *Settings → Keys* → *Generate auth key*（建议勾选 *Reusable*，便于定时任务反复使用）
> - **API Token**：*Settings → Keys* → *Generate API access token*（需要 `Devices` 读写权限）

### 2. 允许 Actions 运行

在仓库 **Settings → Actions → General** 中允许 GitHub Actions 运行。本工作流将 `GITHUB_TOKEN` 权限设为 `{}`，无需授予仓库读写权限；Tailscale 凭据通过 Actions Secrets 单独提供。

---

## 🚀 使用方法

### 方式一：手动触发（推荐，首次使用）

1. 打开仓库页面 → 点击 **Actions** 标签页
2. 在左侧选择 **devtop** 工作流
3. 点击右侧 **Run workflow** 下拉按钮 → 点击绿色的 **Run workflow**
4. 等待工作流执行完毕（首次约需 8–15 分钟，包含环境初始化与软件安装）

### 方式二：定时自动运行

工作流已配置定时触发：

```yaml
schedule:
  - cron: '0 */6 * * *'   # 每 6 小时自动运行一次
```

> ⚠️ 注意：GitHub 对公共仓库的 `schedule` 触发存在一定的延迟（可能推迟 15 分钟至数小时），且长期不活跃的仓库可能不会执行定时任务。建议优先使用手动触发。

---

## 🔗 连接云端桌面

工作流执行完成后，进入 **Actions → devtop → 最新一次的运行记录**，展开 *Maintain Connection* 步骤，日志末尾会输出连接信息：

```
=== RDP 服务已就绪 ===
RDP Address (Tailscale IP): 100.x.x.x
Username: <runner account>
Password: ********
===================================
```

### 🖥️ 通过 RDP 连接（图形界面）

| 平台 | 客户端 | 连接地址 |
|------|--------|----------|
| Windows | 系统自带 `mstsc`（远程桌面连接） | Tailscale IP |
| macOS / iOS | Microsoft Remote Desktop | 同上 |
| Android | RD Client | 同上 |

首次连接需在 **Tailscale 处于运行状态的设备**上执行（保证能路由到云桌面的虚拟网段）。

---

## ⚙️ 工作流详情

### 触发方式

```yaml
on:
  workflow_dispatch:      # 手动触发
  schedule:               # 定时触发
    - cron: '0 */6 * * *' # 每 6 小时
```

### 运行环境

- **Runner**：`windows-latest`
- **超时时间**：`360 分钟`（6 小时），与定时计划周期对齐，超时后云桌面会自动关机释放

### Tailscale 配置说明

工作流会自动完成以下 Tailscale 相关操作：

1. 通过 `winget` 安装 Tailscale 客户端（已安装则自动跳过，保证幂等）
2. 使用 Auth Key 完成设备认证，主机名固定为 `devtop`
3. （可选，需 API Token）自动清理同名的旧设备，防止重复设备堆积
4. 广播检测到的 RFC1918 私有 IPv4 子网路由，并启用 Exit Node（可在 Tailscale 管理后台审批使用）；即使未检测到子网，也会单独启用 Exit Node

---

## ⚠️ 注意事项

- **计费提醒**：GitHub Actions 免费额度包含 2000 分钟/月（公共仓库免费额度更多），长时间运行会消耗额度，请按需使用
- **安全说明**：工作流保持 Windows 防火墙和 NLA 开启，并将 RDP 入站规则限制为 Tailscale IPv4 地址范围；客户端需支持 NLA
- **密码安全**：`RDP_PASSWORD` 会作为环境变量传入工作流，虽然不会明文打印，但请避免使用个人常用密码
- **权限提示**：Tailscale 的子网路由与 Exit Node 配置需要登录管理后台进行人工审批后才能生效
- **局域网访问**：使用 Exit Node 时访问客户端自己的局域网，需在该客户端的 Tailscale 设置中单独开启 Allow LAN access
- **连接前提**：RDP 连接前，请确保本机 Tailscale 已登录并处于同一 tailnet 网络
- **并发保护**：工作流已启用并发控制（`concurrency`），定时任务与手动触发不会同时运行，后触发的运行会自动取消旧的运行，避免多台机器互相冲突

---

## 📜 免责声明

本项目仅用于个人学习与开发测试。请遵守 GitHub Acceptable Use Policy 以及当地法律法规，**禁止**将该环境用于任何恶意用途（如挖矿、攻击、滥用等）。因使用本配置而产生的任何后果由使用者自行承担。
