# luci-app-frpc-multi

用于 OpenWrt / iStoreOS 的 LuCI 多连接 FRP 客户端管理插件。

每个连接运行独立的 `frpc` 进程，可连接不同的 `frps` 主端。在同一页面管理连接、代理规则、运行状态和日志，无需为每台主端复制服务脚本。

## 特性

- 一键添加连接，在编辑弹窗中填写名称；内部 ID 自动生成。
- 连接名称支持中文，连接表格、状态卡片和代理所属连接统一显示名称。
- 支持 TCP、UDP、HTTP、HTTPS、TCPMUX、STCP、XTCP、SUDP。
- 集成令牌、TLS、传输协议、多路复用、连接池、心跳、DNS、出口代理、带宽限制、加密和压缩等选项。
- 支持 HTTP 域名/认证、访问端 visitor、HTTP/SOCKS5/static_file/unix_domain_socket 插件。
- 每连接一个 procd 实例，可独立重启、启用、停用。
- 默认登录失败不退出；procd 无限重试；可选每分钟看门狗。
- 显示代理状态与错误原因，按连接查看日志。
- 保存配置后重新生成 TOML，只重启配置发生变化的实例。

**容量说明：**没有固定的连接数量输入限制，但并非物理上无限。受设备资源和管理 API 端口数量限制；当前自动分配范围为 `27400–27999`（600 个端口，其他程序占用会减少可用数量）。

## 环境要求

- 使用 **apk** 包管理器的 OpenWrt / iStoreOS。
- LuCI、rpcd ucode 支持，以及 `ucode` 的 uci/ubus/fs/socket 模块。
- `frpc` 支持 TOML 和 `webServer` 状态 API；已在 **frpc 0.66.0、iStoreOS 25.12.5 x86_64** 上测试。
- 插件本身为 `noarch`，不包含 FRP 二进制。其他平台需自行提供兼容的 `frpc`。
- **不支持直接安装到 opkg/ipk 系统**，目前没有 ipk 安装包。

## 安装

从本仓库的 **Releases** 下载 `.apk` 和 `SHA256SUMS`，核对校验值后上传路由器：

```sh
apk add --allow-untrusted /tmp/luci-app-frpc-multi-1.0.0-r7.apk
```

当前发布包没有签名，因此需要 `--allow-untrusted`。请只安装可信来源的包。

安装后打开 **服务 → frp 多客户端**。

## 使用

1. 点击「添加连接…」，填写名称、主端地址和端口、令牌等。
2. 点击「添加代理…」，选择所属连接，设置本地目标和远程端口。
3. 保存并应用，在状态卡片中确认代理在线。

名称是页面显示名，内部 ID 用于 procd 实例、日志标识和生成文件名。已有配置的内部 ID 不会被自动改名。

默认 TLS 开启；是否关闭取决于你的主端配置。`登录失败即退出` 默认关闭，若启用该选项，连不上主端时进程会退出，仍由 procd/看门狗按配置重试。

「停用」会写入配置，看门狗不会拉起已停用连接；仅用命令 `stop` 停止进程，则可能被看门狗再次启动。

**不会自动迁移或停用其他 FRPC 插件。**迁移时应停用旧实例，避免多个客户端重复注册相同代理名或远程端口。插件也不会自动创建公网防火墙放行规则。

## 文件与命令

| 路径 | 用途 |
|---|---|
| `/etc/config/frpc_multi` | UCI 配置，含连接令牌，权限 600 |
| `/etc/init.d/frpc-multi` | 多实例服务 |
| `/var/etc/frpc-multi/<id>.toml` | 生成的运行配置，权限 600 |
| `/usr/sbin/frpc-multi-watchdog` | 每分钟守护 |
| `/usr/share/rpcd/ucode/luci.frpc-multi` | 状态、日志、操作 RPC |

```sh
/etc/init.d/frpc-multi checkconfig
/etc/init.d/frpc-multi restart <内部ID>
/etc/init.d/frpc-multi reload
apk del luci-app-frpc-multi
```

升级保留修改后的配置；卸载停止实例并移除本插件的开机项和定时任务，修改后的 UCI 配置可能保留。删除含令牌配置前请自行备份。

## 安全注意事项

- 每个连接的管理 API 仅监听 `127.0.0.1`，使用自动随机密码。
- LuCI RPC 由登录权限控制；不要将路由器管理页面直接暴露到公网。
- 配置、运行 TOML 和日志可能包含敏感信息，不要上传到公开 Issue。
- 附加 TOML 仅适用于可信管理员，不是安全沙箱。不要覆盖自动生成的管理接口认证或重复声明已有字段。
- 当前服务以 root 运行；安装脚本会更新 cron 并重载 rpcd。操作前建议备份设备。

## 构建

在安装 Docker、Python 3 的 Linux 主机上：

```sh
VER=1.0.0-r7 sh build-apk.sh "$PWD" "$PWD/dist"
```

构建使用 `alpine:edge` 容器中的 `apk mkpkg`。标签会随上游变化，二次构建不保证与现有发布包逐字节相同。

当前通过上述脚本手动构建。首个公开版本未配置 GitHub Actions，以避免为发布额外授予 `workflow` 权限。

## 测试与版本

详见 [CHANGELOG.md](CHANGELOG.md)。曾测试配置生成、frpc verify、单连接操作、看门狗、权限、临时本机 frps 端到端转发，以及安装/升级/卸载/重装。**未进行真实断电启动验收，也未测试所有主端版本和每一种代理协议。**

本仓库不包含设备的真实 UCI 配置、令牌或用户日志。

## 许可证
该项目基于luci-app-frpc修改
新增插件实现采用 Apache-2.0。服务脚本中的 OpenWrt FRPC/procd 模式包含上游来源，详见 [NOTICE](NOTICE) 和 `LICENSES/`。FRP 为独立依赖，不打包其二进制。
