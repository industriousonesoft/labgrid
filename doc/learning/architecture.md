# labgrid 分布式架构说明（团队版）

本文档面向使用和维护 labgrid 实验环境的团队成员，说明组件划分、核心概念、
典型工作流、网络与安全要求以及运维约定。本文依据当前源码与官方文档整理：
`doc/overview.rst`、`doc/usage.rst`、`doc/configuration.rst`、
`labgrid/remote/{coordinator,exporter,client,scheduler}.py`。
本文件为内部资料，未纳入版本控制。

---

## 1. 架构总览

labgrid 的分布式能力由三个进程组成：**coordinator**、**exporter**、**client**，
通过 gRPC（默认端口 `20408`）通信。架构上将"控制面"与"数据面"分离：

```
                 控制面（gRPC，经过 coordinator）
  ┌────────────┐   注册资源 / 上报状态    ┌─────────────┐
  │  Exporter  │ ───────────────────────▶ │ Coordinator │
  │  (接硬件)   │ ◀─────────────────────── │  (登记/预约)  │
  └─────┬──────┘   心跳 / 事件推送          └──────┬──────┘
        │                                          │
        │      数据面（不经过 coordinator）          │ 资源查询 / place 管理
        │  ser2net(RFC2217) / SSH / HTTP / VISA    │ acquire / release
        └──────────────────────────────────────────┴──────┐
                                                          ▼
                                                   ┌────────────┐
                                                   │   Client   │
                                                   │ CLI / pytest│
                                                   └────────────┘
```

关键原则：

1. **coordinator 不转发硬件数据**。client 访问串口、USB、视频等资源时直连 exporter，
   避免为每种资源类型在 coordinator 上增加转发接口。
2. **driver 与 strategy 只在 client 侧运行**。exporter 只负责"暴露资源"和"提供访问通道"。
3. **place（地点）是资源使用的逻辑单元**。资源按 exporter/group 组织，place 通过匹配规则
   聚合资源，并负责并发互斥。

---

## 2. 核心概念

| 概念 | 说明 | 所在侧 |
|---|---|---|
| Target | 一块被控板卡/设备的抽象，包含 Resource 与 Driver | client |
| Resource | 被动描述"如何访问某部分硬件"，如 `RawSerialPort`、`NetworkPowerPort` | client / exporter |
| ManagedResource | 可动态出现/消失的资源（udev 发现的 USB 设备等） | exporter 监视，client 使用 |
| Driver | 使用 Resource/底层 Driver 执行动作，如 `ShellDriver`、`NetworkPowerDriver` | client |
| Protocol | Driver 之间的能力契约，如 `ConsoleProtocol`、`CommandProtocol`、`PowerProtocol` | client |
| Strategy | 特殊 Driver，负责把板子从一个状态迁移到另一个状态（如 off → bootloader → shell） | client |
| Environment | YAML 描述的一组 Target，供 pytest/库加载 | client |
| Exporter Group | exporter 配置中的资源组，决定资源的组织与匹配粒度 | exporter |
| Location | 资源组的物理位置标签（如机架号） | exporter |
| Place | coordinator 上的"地点"：别名、注释、标签、资源匹配规则的集合 | coordinator |
| Match | `<exporter>/<group>/<class>/<name>` 形式的资源匹配式，`*` 为通配 | coordinator |
| RemotePlace | Environment 中的资源，用于把某个 place 的资源展开到本地 Target | client |
| Reservation | 基于标签的自动调度预约，用于 CI 等无人值守场景 | coordinator |

### 资源匹配示例

| 匹配式 | 含义 |
|---|---|
| `*/1001/*` | 所有 exporter 上 group 为 `1001` 的全部资源 |
| `*/1001/NetworkPowerPort` | 只要 group `1001` 里的电源资源 |
| `exporter1/hub1-port1/*` | 指定 exporter 上某个 USB hub 端口组的全部资源 |
| `exporter1/*/*/console-main` | 按资源名匹配（命名资源） |

匹配规则**只在 acquire 时求值**，占用期间不可修改，release 后才能调整。

---

## 3. 三个组件的职责边界

### 3.1 labgrid-coordinator

- gRPC 服务端，默认监听 `[::]:20408`（`--listen` 可改）。
- 维护全部 exporter 上报的资源清单，并向 client 推送变更事件。
- 管理 place：创建/删除、别名、注释、标签、资源匹配规则。
- 并发控制：place 的 acquire/release、占用记录（谁在哪个主机上占用）、
  基于标签的 reservation 调度（`labgrid/remote/scheduler.py`）。
- 持久化：把资源快照与 place 定义写入**当前工作目录**的 `resources.yaml`、
  `places.yaml`（周期性保存）；acquire 状态不跨重启保留。
- 安全：支持 gRPC TLS（`--tls --cert --key`）。

不做：不转发数据、不运行 driver、不接触硬件。

### 3.2 labgrid-exporter

- 运行在物理连接硬件的机器上，默认以主机名作为 exporter 名称（`-n/--name` 可改）。
- 按 YAML 配置注册资源组；对 ManagedResource 使用 udev 监视热插拔，
  实时更新可用性与参数（如 `ID_SERIAL_SHORT`、设备节点）。
- 为资源提供访问通道：
  - 串口：启动 **ser2net**，以 RFC2217 方式提供网络串口；
  - USB/文件：依赖 **SSH**（含 `ssh -J` 跳板）与 `scp`/`rsync`；
  - 视频/音频：HTTP/RTSP 等；
  - 仪器：由 client 通过 VISA/网络协议直连。
- 发布可访问的 hostname/IP（`--hostname`、`--fqdn`）。
- `-i/--isolated`：强制 client 通过 SSH 隧道访问该 exporter 的所有资源。
- 维护 `/var/cache/labgrid` 作为 ManagedFile 上传目录。

不做：不运行 driver/strategy，不做 place 预约，不决定资源归属。

### 3.3 labgrid-client

- 面向用户的 CLI（`labgrid-client`）与 Python 库，也是 pytest 插件的底层。
- 查询：`resources`、`places`、`who`、`show`、`monitor`。
- place 配置：`create`、`delete`、`add-match`/`del-match`、`add-alias`、
  `set-comment`、`set-tags`。
- 占用：`acquire`/`release`、`allow`（授权其他用户）、reservation 相关
  （`reserve`、`wait`、`reservations`）。
- 直接操作：`power`、`io`、`console`、`dfu`、`fastboot`、`flashscript`、
  `bootstrap`、`sd-mux`、`usb-mux`、`video`、`audio`、`netns`、`telnet`。
- 远程文件/网络：`scp`、`sshfs`、`forward`。
- 生成测试配置：`labgrid-client -p <place> env` 输出 Environment YAML。
- 在 pytest 中通过 `RemotePlace` 把 place 资源展开成本地 Target，
  随后所有 driver 逻辑在 client 本地执行。

不做：不监视硬件（依赖 exporter）、不持久化 place 定义（在 coordinator）、
不常驻（每次调用为一次性进程）。

### 3.4 职责对照表

| 能力 | coordinator | exporter | client |
|---|---|---|---|
| 维护资源清单 | 汇总 | 数据来源 | 只读查询 |
| 维护 place 定义 | ✅ | ❌ | 发起修改 |
| 决定 place 归属 | ✅ | ❌ | 发起 acquire/release |
| 监视硬件热插拔 | ❌ | ✅ | ❌ |
| 提供串口/SSH/视频通道 | ❌ | ✅ | 消费通道 |
| 运行 driver/strategy | ❌ | ❌ | ✅ |
| 控制面流量 | 承载 | 参与 | 参与 |
| 数据面流量 | 不经过 | 服务端 | 发起端 |

---

## 4. 典型部署形态

| 形态 | 组成 | 适用场景 |
|---|---|---|
| 单机开发 | 本机 coordinator + exporter + client | 个人开发、功能验证 |
| 集中式 lab | 一台 coordinator（服务器）+ 多台 exporter（接板子）+ 多人 client | 团队共享板卡 |
| 跨网段 | exporter 位于隔离网段，client 通过 `LG_PROXY` 或 exporter `-i` 走 SSH 隧道 | 安全隔离的测试网 |
| CI 集成 | CI runner 作为 client，配合 `reserve`/`wait` 与 `LG_RESERVATION` | 自动化回归 |
| 容器化 | 使用 `dockerfiles/` 构建 coordinator/exporter/client 镜像 | 快速部署、环境一致性 |

建议：coordinator 放在稳定、可达的服务器上；exporter 靠近硬件；client 随用户/CI。

---

## 5. 典型工作流

### 5.1 启动 coordinator

```bash
mkdir -p /srv/labgrid/coordinator
cd /srv/labgrid/coordinator
labgrid-coordinator            # 默认 [::]:20408
# 生产建议：直接使用 contrib/systemd 下的 unit + DynamicUser/StateDirectory
```

> `resources.yaml`、`places.yaml` 写入当前工作目录，**工作目录必须固定并纳入备份**。

### 5.2 配置并启动 exporter

```yaml
# /etc/labgrid/exporter.yaml
bench-12:
  location: rack12
  USBSerialPort:
    match:
      '@ID_PATH': 'pci-0000:05:00.0-usb-3-1.3'
  NetworkPowerPort:
    model: netio
    host: netio12
    index: 3
```

```bash
labgrid-exporter -c coordinator.example.com:20408 /etc/labgrid/exporter.yaml
```

> exporter 配置只有资源（无 `targets`/`drivers`），资源条目语法与 Environment 的
> `resources:` 相同；driver 始终由 client 侧的 Environment 定义。

同一 group 内需要多个同类资源时，用资源名 + `cls` 区分：

```yaml
bench-12:
  console-main:
    cls: USBSerialPort
    match: {'@ID_PATH': 'pci-0000:05:00.0-usb-3-1.3'}
  console-secondary:
    cls: USBSerialPort
    match: {'@ID_PATH': 'pci-0000:05:00.0-usb-3-1.4'}
```

### 5.3 创建并配置 place

```bash
export LG_COORDINATOR=coordinator.example.com:20408

labgrid-client -p bench-12 create
labgrid-client -p bench-12 set-tags board=imx6ull soc=nxp
labgrid-client -p bench-12 set-comment "i.MX6ULL EVK, 串口在 exporter1"
labgrid-client -p bench-12 add-alias evk-01
labgrid-client -p bench-12 add-match 'exporter1/bench-12/*'
```

### 5.4 占用、运行测试、释放

```bash
labgrid-client -p bench-12 acquire
labgrid-client -p bench-12 env          # 仅打印展开后的资源片段（不含 drivers）
pytest --lg-env remote.yaml test_shell.py   # remote.yaml 见 5.5
labgrid-client -p bench-12 release
```

CI 无人值守场景：

```bash
eval $(labgrid-client reserve --shell board=imx6ull)
labgrid-client wait
labgrid-client -p +$LG_RESERVATION env   # 仅资源片段；pytest 用 5.5 的 RemotePlace 配置
pytest --lg-env remote.yaml
labgrid-client -p +$LG_RESERVATION release
```

### 5.5 测试侧 Environment（RemotePlace）

```yaml
targets:
  main:
    resources:
      RemotePlace:
        name: bench-12       # place 名；不支持 bench-* 这类通配
    drivers:
      SerialDriver: {}
      ShellDriver:
        prompt: 'root@[\w-]+:[^ ]+ '
        login_prompt: ' login: '
        username: root
```

`RemotePlace` 会连接 coordinator，等待资源可用，并把该 place 的资源作为本地资源
暴露给 Target；driver 不需要知道资源来自远端。

place 查找走的是**子串匹配**（name 或 alias 包含该字符串），不是 glob：
`name: bench-12` 可以精确命中；`name: bench-*` 会按字面量查找而不命中；
`name: bench` 若同时匹配多个 place 会报错。因此 place 名应保持唯一、可读。

---

## 6. 访问路径与依赖

| 资源类型 | 访问方式 | 前提 |
|---|---|---|
| 串口（`USBSerialPort` 等） | exporter 启动 ser2net，client 通过 RFC2217 | 两端口互通；exporter 有 `ser2net` |
| USB 引导/烧写（`IMXUSBLoader`、`AndroidUSBFastboot` 等） | client 经 SSH 到 exporter 执行工具/隧道 | client→exporter 免密 SSH |
| 文件传输（ManagedFile） | `scp`/`rsync`，或共享文件系统（NFS/SMB）跳过传输 | 路径在两端一致（共享 FS 场景） |
| 电源/IO | client 直连电源设备管理地址 | client 网络可达设备 |
| 视频/音频 | HTTP/RTSP 流 | client 可达流地址 |
| 仪器（PyVISADevice） | client 直连 VISA 资源 | VISA 后端可达仪器 |

> 结论：**client 必须能免密 SSH 登录所有相关 exporter**，这是 labgrid 的硬性前提；
> 使用 `ControlMaster`/`ControlPersist` 可显著提升体验。

---

## 7. 并发、调度与预约

- **acquire 互斥**：同一时刻一个 place 只被一个用户占用；coordinator 记录
  用户名/主机名。`who` 可查看当前占用情况。
- **匹配冻结**：资源匹配只在 acquire 时求值，占用期间修改需先 release。
- **别名**：一个 place 可有多个 alias；板子在不同机架间移动时，测试可固定使用 alias。
- **标签调度**：place 打 tags（如 `board=imx6ull`、`soc=nxp`），`reserve` 按过滤条件
  自动分配，支持并发排队；分配结果通过 `LG_RESERVATION` 传递。
- **授权**：`labgrid-client -p <place> allow <user>` 允许其他用户临时访问。

---

## 8. 安全与网络

| 项目 | 说明 |
|---|---|
| 端口 | coordinator gRPC 默认 `20408`；TLS 时需开放并配置证书 |
| TLS | `labgrid-coordinator --tls --cert ... --key ...`；client/exporter 加 `--tls`，必要时 `--cacert` |
| 身份 | 无内置账号密码，身份取自操作系统用户与主机名；**必须依赖网络隔离与 TLS** |
| SSH | client↔exporter 免密 SSH；可配合 `ssh -J`、ProxyCommand |
| 隧道 | client 侧 `LG_PROXY=<ssh-host>` 将 coordinator/exporter 流量全部走 SSH 隧道；exporter 侧 `-i` 强制隔离模式 |
| 设备权限 | exporter 用户需在 `dialout`/`plugdev` 组；udev 属性用于匹配 |
| 文件缓存 | `/var/cache/labgrid`，建议使用 `contrib/systemd/tmpfiles.d` + `sysusers.d` |
| 防火墙 | 只需开放 coordinator 端口与 exporter 的 SSH（及必要的 ser2net/视频端口，通常绑定内网） |

---

## 9. 运维与可靠性

- **持久化**：`resources.yaml`（可重建）与 `places.yaml`（不可重建，务必备份）
  位于 coordinator 工作目录。
- **单点**：coordinator 是单进程单点，无内置集群/HA；故障时 client/exporter 无法登记与预约。
  重启后需重新 acquire。
- **重连**：exporter 与 coordinator 之间有心跳/重连机制（keepalive 参数见
  `labgrid/remote/exporter.py`）；coordinator 重启后 exporter 会重新注册资源。
- **版本一致性**：coordinator、exporter、client 建议使用同一 labgrid 版本；
  升级顺序建议 coordinator → exporter → client，升级前备份 YAML 状态文件。
- **日志/排障**：各进程支持 `-d/--debug`；client 有 `monitor` 事件流；
  可启用 `pystuck` 进行远程调试；coordinator 可配 statsd 上报（见 `contrib/`）。
- **systemd**：使用 `contrib/systemd/labgrid-{coordinator,exporter}.service`，
  `ExecStart` 指向虚拟环境；exporter 需配置 `SupplementaryGroups=dialout` 与
  `LG_COORDINATOR` 环境变量。

---

## 10. 故障排查

| 现象 | 可能原因 | 处理 |
|---|---|---|
| `labgrid-client resources` 为空 | exporter 未连上 coordinator、配置为空、端口/防火墙 | 看 exporter 日志；确认 `-c` 地址；`monitor` 观察事件 |
| place 里没有资源 | 匹配式错误、资源不可用 | `show` 查看匹配；`resources -v` 核对 `<exporter>/<group>/<class>/<name>` |
| acquire 失败/被占用 | 他人持有、reservation 未分配 | `who`、`reservations`；联系持有人或 `allow` |
| console 打不开 | ser2net 未启动、设备被占用、client 缺 `microcom`/`telnet` | exporter 日志查 ser2net；`fuser /dev/ttyUSB*`；安装工具 |
| USB 烧写/引导失败 | SSH 不通、exporter udev 未识别、权限不足 | 手工 `ssh -J exporter user@device` 验证；`udevadm info`；检查用户组 |
| 资源频繁抖动 | USB 供电/线缆、udev 匹配过宽 | 用 `@ID_PATH` 等稳定属性；检查硬件 |
| TLS 连接失败 | 证书/CA 不匹配、未加 `--tls` | 统一 `--cert/--key/--cacert`；设置 `LG_COORDINATOR_TLS` |
| coordinator 重启后 place 占用丢失 | acquire 状态不持久化 | 重新 acquire；重要操作前确认占用 |
| place 定义丢失 | 工作目录变化导致未读到 `places.yaml` | 固定工作目录并备份 |

---

## 11. 团队约定与最佳实践

1. **命名**：exporter 名用主机名；group 以测试台/机架命名（如 `bench-12`）；
   place 与 group 对齐，别名指向板卡型号。
2. **group 粒度**：一个 group 对应"一块板子的所有接口"，便于 `add-match` 一次匹配；
   需要排除某类资源时用按 class 的匹配式。
3. **稳定匹配**：USB 资源优先使用 `@ID_PATH`、`ID_SERIAL_SHORT`，避免 `@SUBSYSTEM` 过宽匹配。
4. **标签**：统一标签体系（`board=`、`soc=`、`vendor=`），CI 用 `reserve` 而不是固定 place。
5. **配置即代码**：exporter/place 配置纳入内部 Git 仓库（本文件与部署文档除外），
   变更走评审；coordinator 状态定期备份。
6. **不共享账号**：acquire 记录的是操作系统用户，多人共用账号会导致占用无法区分。
7. **测试可重复**：strategy + `--lg-initial-state` 用于开发期跳过已知状态；
   CI 中始终从 off 状态启动。
8. **超时意识**：ManagedResource 等待、串口/SSH 驱动均有超时；网络抖动时优先排查连接路径而非加大超时。

---

## 12. 参考

- 架构与概念：`doc/overview.rst`
- 远程使用：`doc/usage.rst`（remote usage、reservation、environment variables）
- 配置参考：`doc/configuration.rst`（Exporter Configuration、RemotePlace、各 Resource/Driver）
- 开发指南：`doc/development.rst`（自定义 Driver/Strategy）
- 本仓库实现：
  - `labgrid/remote/coordinator.py`（gRPC 服务、place、调度）
  - `labgrid/remote/exporter.py`（资源注册、ser2net、代理）
  - `labgrid/remote/client.py`（CLI、RemotePlace 客户端）
  - `labgrid/remote/scheduler.py`（标签调度）
  - `labgrid/remote/proto/labgrid-coordinator.proto`（gRPC 接口定义）
- 部署方案：`doc/deployment.md`（Ubuntu 22.04 / Win10）
