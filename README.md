# DevTop

通过 **GitHub Actions** 在云端按需启动一台 **Windows 远程开发机**，借助 **Tailscale** 组网，实现随时随地通过 **RDP** 或 **SSH** 安全访问自己的云端桌面环境。

---

## ✨ 功能特性

- 🖥️ **一键启动 Windows 云桌面**：使用 GitHub 官方的 `windows-latest` Runner，无需自备服务器
- 🔑 **基于 GitHub Secrets 的安全认证**：用户名与密码全部从 Secrets 读取，不硬编码、不出现在日志中
- 🌐 **Tailscale 私有组网**：通过 WireGuard 协议加密组网，无需公网 IP，也无需在路由器上做端口映射
- 🔒 **RDP + SSH 双通道远程访问**：
  - RDP：`mstsc`（Windows） / Microsoft Remote Desktop（macOS / iOS / Android）
  - SSH：任何支持 OpenSSH 的终端（`ssh` 命令直连）
- ⚡ **自动清理旧设备**：每次启动都会通过 Tailscale API 删除历史遗留的 `devtop` 设备，避免设备堆积
- 🧹 **自动卸载不必要的软件**：自动移除 Chrome、Firefox、Unity Hub、R 等占用资源的预装软件
- 🕐 **手动 / 定时双触发**：支持在 GitHub 页面手动触发，也可按计划自动运行

---

## 🛠️ 工作原理

整个流程由 `.github/workflows/devtop.yml` 定义，包含以下 5 个阶段：

| 步骤 | 说明 |
|------|------|
| ① 配置核心 RDP 设置 | 启用远程桌面服务、关闭网络级身份认证（NLA）、禁用 Windows 防火墙 |
| ② 创建 RDP 用户 | 从 Secrets 读取 `RDP_USERNAME` / `RDP_PASSWORD` 创建本地管理员账户，并加入 "Remote Desktop Users" 组 |
| ③ 卸载不必要的软件 | 通过注册表遍历卸载列表，清理 Chrome / Firefox / Unity Hub / R / Rtools 等 |
| ④ 安装并配置 Tailscale | 安装 Tailscale 与 OpenSSH Server，完成设备认证、命名、MagicDNS 配置，并开放 22 端口 |
| ⑤ 保持连接 | 循环输出日志保活 Runner，确保云桌面在任务超时（默认 3600 分钟）前持续在线 |

---

## 📋 使用前提

### 1. 配置 GitHub Secrets

进入仓库 **Settings → Secrets and variables → Actions → New repository secret**，添加以下 4 个变量：

| Secret 名称 | 是否必填 | 说明 |
|-------------|----------|------|
| `RDP_USERNAME` | ✅ 必填 | 远程登录用户名（如 `admin`），缺失时工作流会直接报错退出 |
| `RDP_PASSWORD` | ✅ 必填 | 远程登录密码，建议使用强密码（大小写字母 + 数字 + 符号） |
| `TAILSCALE_AUTH_KEY` | ✅ 必填 | Tailscale 认证密钥，用于让新设备加入你的 tailnet 网络 |
| `TAILSCALE_API_TOKEN` | ⚠️ 可选 | Tailscale API Token，配置后可自动清理旧设备、设置 MagicDNS 与设备名称 |

> 💡 获取 Tailscale 密钥：
> - **Auth Key**：登录 [Tailscale 管理后台](https://login.tailscale.com/admin/settings/authkeys) → *Settings → Keys* → *Generate auth key*（建议勾选 *Reusable*，便于定时任务反复使用）
> - **API Token**：*Settings → Keys* → *Generate API access token*（需要 `Devices` 读写权限）

### 2. 允许 Actions 运行

仓库 **Settings → Actions → General** 中，确保 *Workflow permissions* 设置为 **Read and write permissions** 或至少允许运行工作流。

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
RDP Address (MagicDNS):    devtop.xxx.ts.net   # 需配置 API Token 才会显示
Username: admin
Password: ********
===================================
```

### 🖥️ 通过 RDP 连接（图形界面）

| 平台 | 客户端 | 连接地址 |
|------|--------|----------|
| Windows | 系统自带 `mstsc`（远程桌面连接） | Tailscale IP 或 MagicDNS 域名 |
| macOS / iOS | Microsoft Remote Desktop | 同上 |
| Android | RD Client | 同上 |

首次连接需在 **Tailscale 处于运行状态的设备**上执行（保证能路由到云桌面的虚拟网段）。

### 💻 通过 SSH 连接（命令行）

```bash
ssh admin@100.x.x.x
# 或使用 MagicDNS 域名（需配置 API Token）
ssh admin@devtop.xxx.ts.net
```

密码为 Secrets 中配置的 `RDP_PASSWORD`。

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
- **超时时间**：`3600 分钟`（60 小时），超时后云桌面会自动关机释放

### Tailscale 配置说明

工作流会自动完成以下 Tailscale 相关操作：

1. 通过 `winget` 安装 Tailscale 客户端
2. 安装并启动 OpenSSH Server，开放防火墙 22 端口
3. 使用 Auth Key 完成设备认证，主机名固定为 `devtop`
4. （可选，需 API Token）自动清理同名的旧设备，防止重复设备堆积
5. （可选，需 API Token）自动开启 MagicDNS，便于使用域名访问
6. 自动广播本地子网路由，并启用 Exit Node 功能（可在 Tailscale 管理后台审批使用）

---

## ⚠️ 注意事项

- **计费提醒**：GitHub Actions 免费额度包含 2000 分钟/月（公共仓库免费额度更多），长时间运行会消耗额度，请按需使用
- **安全警告**：该工作流会**关闭 Windows 防火墙**并要求**关闭网络级身份认证（NLA）**，请务必为 RDP 用户设置强密码，并通过 Tailscale 私有网络访问，避免暴露到公网
- **密码安全**：`RDP_PASSWORD` 会作为环境变量传入工作流，虽然不会明文打印，但请避免使用个人常用密码
- **权限提示**：Tailscale 的子网路由与 Exit Node 配置需要登录管理后台进行人工审批后才能生效
- **连接前提**：RDP / SSH 连接前，请确保本机 Tailscale 已登录并处于同一 tailnet 网络

---

## 📜 免责声明

本项目仅用于个人学习与开发测试。请遵守 GitHub Acceptable Use Policy 以及当地法律法规，**禁止**将该环境用于任何恶意用途（如挖矿、攻击、滥用等）。因使用本配置而产生的任何后果由使用者自行承担。
