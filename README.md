# luci-app-frpc-multi

用于 OpenWrt / iStoreOS 的 LuCI 多连接 FRP 客户端管理插件。每个连接都是一个独立的 `frpc` 进程，可以分别连接不同的主端（frps）。

## 全新系统一键安装

在全新 OpenWrt / iStoreOS 上以 root 身份复制并运行下面**这一条命令**。脚本会自动检测 apk/opkg 与 arm64/x86_64，更新系统软件源并选择对应安装包。包依赖会由系统包管理器解析，同时安装软件源中的匹配架构 `frpc` 内核；无需先手动安装 frpc。

```sh
wget -O /tmp/install-frpc-multi.sh https://raw.githubusercontent.com/myc2002/luci-app-frpc-multi/main/install.sh && sh /tmp/install-frpc-multi.sh
```

支持：apk + arm64/aarch64、apk + x86_64、opkg + arm64/aarch64、opkg + x86_64。其他架构会明确提示不支持。

安装后打开：**LuCI → 服务 → frp 多客户端**。

脚本需要全新系统已接通网络且配置了对应版本的软件源，以解析并安装 `frpc` 依赖。如果系统软件源没有兼容版本的 `frpc`，包管理器会报出依赖错误，不会偷偷从别处下载或安装其他来源的二进制。

安装包未签名；请只使用本项目 HTTPS Release，下载后按随附 `SHA256SUMS` 校验。APK/IPK 本身是通用 LuCI 脚本，不包含 CPU 专用二进制；APK 文件按架构分别标记，IPK 内部架构为 `all`。FRPC 内核由系统的 apk/opkg 软件源按本机架构自动解析安装。

## 功能

- 每个「连接」运行一个独立 frpc 进程，可管理多个 frps 主端。
- 一键添加连接；填写显示名称，内部 ID 自动生成。
- 连接名支持中文，连接表格、状态卡片、代理所属连接统一显示名称。
- 支持 tcp、udp、http、https、tcpmux、stcp、xtcp、sudp。
- 集成令牌、TLS、协议、多路复用、连接池、心跳、DNS、出口代理、带宽限制、加密/压缩。
- HTTP 域名/认证、STCP/XTCP/SUDP visitor，以及 HTTP/SOCKS5/static_file/unix_domain_socket 插件。
- 每条代理的在线状态、错误和连接日志；连接可独立启用、停用、重启。
- 登录失败默认持续重试；procd 无限重启；可选每分钟看门狗。

**容量说明：**不限制可创建的 UCI 连接段数量，但不代表设备物理资源无限。当前状态 API 自动使用本机 `127.0.0.1:27400–27999`；600 个端口是状态 API 的上限，已被其他程序占用时可用数量会减少。设备 CPU、内存和网络也会限制实际连接数。

## 兼容要求

- APK：使用 apk 包管理器的 OpenWrt/iStoreOS。
- IPK：使用 opkg 的 OpenWrt。
- LuCI、rpcd ucode 和 `ucode` 的 uci/ubus/fs/socket 模块。
- `frpc` 0.66.0 已验证；需具备 TOML 配置与 loopback `webServer` 状态 API 支持的版本。
- 已在 iStoreOS 25.12.5 x86_64 上安装及运行验证。其他设备请先备份并确认软件源能提供 `frpc`。

## 使用

1. 点击「添加连接…」，填写名称、主端地址和端口、令牌等。
2. 点击「添加代理…」，选择所属连接，设置本地目标与远程端口。
3. 保存并应用，在状态卡片确认连接和代理在线。

名称是人看的显示名；内部 ID（如 `c1`）仅供系统使用，在编辑弹窗中只读显示。已有连接 ID 不会自动改变。

默认 TLS 开启（依照 frp 默认值）；是否关闭取决于主端配置。`登录失败即退出` 默认关闭；停用连接后，看门狗不会拉起它。不要把新连接的远端端口与其他 FRP 实例重复。

**不会自动迁移或停用其他 FRPC 插件。** 如果你把现有实例手动迁入，请先规划并停用旧实例，避免重复注册同名代理/远端端口。插件不会自动创建公网防火墙放行规则。

## 文件与命令

- `/etc/config/frpc_multi`：UCI 配置（权限 600，含连接认证信息）。
- `/etc/init.d/frpc-multi`：生成 `/var/etc/frpc-multi/<ID>.toml`（权限 600），每条连接一个 procd 实例。
- `/usr/sbin/frpc-multi-watchdog`：每分钟守护；可在「全局」关闭。

常用命令：

```sh
/etc/init.d/frpc-multi checkconfig [内部ID]
/etc/init.d/frpc-multi restart <内部ID>
/etc/init.d/frpc-multi reload
```

升级保留 UCI 配置；卸载会停用/停止连接并移除本插件的 cron 行，但保留配置文件。卸载前请自行备份。

## 校验

下载相应包及同目录 `SHA256SUMS` 后执行 `sha256sum -c SHA256SUMS`（也可只校验某个 APK/IPK）。正式 Release 还附构建与变更说明。

## 构建

- 完整 Release 包矩阵（两 APK + 两 IPK）：`sh build-release-matrix.sh`；需要 Docker。
- IPK：使用 `scripts/make-ipk.py` 生成标准 opkg IPK（`Architecture: all`，FRPC 由 `Depends` 从 opkg 源按设备架构安装）。示例和矩阵脚本见 `build-release-matrix.sh`。
- 公开仓库未配置 GitHub Actions，避免申请额外 `workflow` 权限；手动构建不会自动覆盖正式 Release。

## 测试范围

曾测试配置生成和 `frpc verify`、单实例管理/无限重试、看门狗、loopback API 认证、临时本机 frps 端到端转发、LuCI 页面、APK 安装/升级/卸载/重装。真实设备冷启动行为未实测；并非所有 FRP 版本和协议组合均完成验证。

## 安全

- frpc 管理 API 只监听 `127.0.0.1`，凭据随机生成。
- rpcd API 受 LuCI ACL 保护；不要把路由器管理界面暴露到公网。
- 附加 TOML 是管理员输入，不是安全沙箱。配置、运行 TOML 和日志可能包含敏感信息；请勿在公开 Issue 上传真实配置/令牌。
- 服务以 root 运行；安装/升级脚本更新 cron 并重载 rpcd，请只从可信 HTTPS Release 安装。

## 许可证

新增代码 Apache-2.0；服务脚本中的 OpenWrt FRPC/procd 模式参考 GPL-2.0 上游，见 `NOTICE` 和 `LICENSES/GPL-2.0.txt`。FRP 二进制是独立依赖，不包含在插件包中。
