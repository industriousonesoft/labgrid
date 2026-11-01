# labgrid-mcp 与 labgrid-client 差异说明

本文对比 `labgrid-mcp`（MCP 服务器，源码位于 `../labgrid-mcp`）与
`labgrid-client`（labgrid 自带 CLI）在定位、架构、交互模型、功能覆盖和安全
策略上的差异。结论依据：

- labgrid 侧：`labgrid/remote/client.py`、`labgrid/remote/coordinator.py`、
  `doc/overview.rst`
- labgrid-mcp 侧：`src/labgrid_mcp/{server,coordinator,target,console,jobs,policy}.py`、
  `docs/DESIGN.md`、`README.md`（验证基线 labgrid 26.0）

本文件为内部资料，未纳入版本控制。

---

## 1. 一句话结论

**labgrid-client 是给人/脚本用的命令行前端；labgrid-mcp 是给 AI Agent 用的
MCP 服务器。两者都是 labgrid 的"客户端"，共享 coordinator、place 互斥和
driver 体系，但 labgrid-mcp 不是 labgrid-client 的包装器，而是自带
gRPC 连接与 Target/driver 栈的独立实现。**

---

## 2. 定位与形态

| 维度 | labgrid-client | labgrid-mcp |
|---|---|---|
| 目标用户 | 工程师、Shell 脚本、CI | LLM/Agent、MCP 客户端（Claude、Cursor、VS Code 等） |
| 形态 | 一次性 CLI 进程 + Python 库 + pytest 插件 | 长期驻留的 stdio MCP 服务器 |
| 调用方式 | 命令行参数 | MCP 工具调用（结构化 JSON 入参/出参） |
| 结果形式 | 文本输出 + 退出码 | 结构化 JSON（typed dict） |
| 会话状态 | 基本无状态（每次进程重建） | 有状态：gRPC 订阅、Target 缓存、console/job/forward 注册表、预约保活 |
| 分发方式 | `pip install labgrid` 自带 | `uvx labgrid-mcp` / `pip install labgrid-mcp` / Docker |
| 身份 | `LG_USERNAME`/`LG_HOSTNAME`，缺省用真实用户/主机名 | 相同（刻意与 client 一致） |

---

## 3. 架构差异

### 3.1 labgrid-client

```
CLI 参数 ──▶ ClientSession (argparse 驱动)
                ├── gRPC ──▶ coordinator（资源/place/预约）
                └── Target + RemotePlace + Driver
                        └── 数据面 ──▶ exporter（ser2net/SSH/HTTP）
```

- 动作方法不带参数，从 `self.args` 取输入并直接 `print()` 结果；
- pytest 插件复用同一套 `RemotePlace` + Target + driver 逻辑；
- 每次命令都是新进程，不保留连接与中间状态。

### 3.2 labgrid-mcp

```
MCP Client ──stdio/FastMCP──▶ labgrid-mcp
                                 ├── 自建 async gRPC 客户端 ──▶ coordinator
                                 │   （generated stubs + remote.common）
                                 └── Target + RemotePlace + Driver
                                         └── 数据面 ──▶ exporter（ser2net/SSH/HTTP）
```

DESIGN §3.1 明确为"按域拆分"的两条集成面：

1. **Coordinator gRPC**：直接使用 `labgrid.remote.generated` 的 stub 和
   `labgrid.remote.common` 数据类，覆盖发现、places、resources、reservations、
   acquire/release/allow、who（`coordinator.py`）。
2. **client 侧 Target + driver 栈**：为已 acquire 的 place 构造 `Target` +
   `RemotePlace`，实例化 `NetworkPowerDriver`、`SerialDriver`、`SSHDriver`、
   `DFUDriver`、`AndroidFastbootDriver`、`USBSDMuxDriver` 等，覆盖
   power/io/console/flash/mux（`target.py`）。

它**不导入也不实例化 `labgrid.remote.client.ClientSession`**，原因有两点
（`target.py` 模块注释）：

- `ClientSession` 的动作方法与 argparse 强耦合，无法程序化传参；
- 同一进程再建一条 gRPC 流会与已有连接共享 subchannel，导致 coordinator 中止会话。

**唯一会启动 labgrid CLI 的场景是 `labgrid-mcp demo`**：它拉起真实的
`labgrid-coordinator` 与 `labgrid-exporter` 子进程搭建假实验室
（`demo.py:271-272`）；正常连接真实 lab 时不会调用任何 labgrid 可执行文件。

---

## 4. 通信与身份

| 项目 | 两者共同点 | 差异 |
|---|---|---|
| coordinator 连接 | gRPC（默认 20408，可 TLS） | client 每次命令新建/断开；MCP 保持一条持久连接并自动重连 |
| 身份模型 | `LG_USERNAME`/`LG_HOSTNAME`，coordinator 按 OS 用户/主机记录占用 | 相同 |
| 并发仲裁 | place acquire 互斥 | 相同；MCP 每个实例一个身份，官方明确不建议多用户共享同一实例 |
| 数据面 | driver 走 ser2net/RFC2217、SSH、HTTP、VISA | 相同（复用 labgrid driver 实现） |
| 事件订阅 | client 有 `monitor` 实时事件流 | MCP 无实时流，用 `wait_for_change` 长轮询替代 |

---

## 5. 交互模型差异

| 场景 | labgrid-client | labgrid-mcp |
|---|---|---|
| 查询 | 文本表格/列表，人读 | JSON dict，Agent 直接消费 |
| 控制台 | 启动 `microcom`/`telnet` 进入**交互式 TTY** | 打开会话后通过 `console_read`/`console_send` **按句柄读写环形缓冲**，非 TTY |
| 烧写 | `dfu`/`fastboot`/`bootstrap` 等命令**阻塞直到完成** | 提交为**后台 job**，用 `flash_status`/`flash_logs` 轮询，避免 MCP 请求超时 |
| 预约 | `reserve --shell` + `wait`，靠后续命令推进 | 服务器进程内维持 **keepalive**，预约不会在两次工具调用之间静默过期 |
| 变更监视 | `monitor` 持续输出事件 | `wait_for_change` 游标长轮询（≤25 s，规避 MCP 客户端超时） |
| 错误 | stderr 文本 + 非零退出码 | 结构化 MCP tool error，保留 labgrid/gRPC 原因 |
| 生命周期 | 命令结束即退出 | 常驻；有 TTL 清理、release/退出时清理 console/job/forward |

---

## 6. 功能覆盖对照

### 6.1 查询、监控与占位

| 能力 | labgrid-client | labgrid-mcp |
|---|---|---|
| 资源/place/who | `resources`、`places`、`show`、`who` | `list_resources`、`list_places`、`show_place`、`who` |
| coordinator 信息 | `version` | `coordinator_info` |
| 事件监控 | `monitor`（流式） | `wait_for_change`（长轮询） |
| acquire/release/allow | `acquire`、`release`、`allow` | `acquire_place`（内置等待/自动预约）、`release_place`、`allow_place`、`release_from` |
| 预约 | `reserve`、`cancel-reservation`、`reservations`、`wait` | `reserve`、`cancel_reservation`、`reservation_wait`、`list_reservations` |

### 6.2 place 配置

| 能力 | labgrid-client | labgrid-mcp |
|---|---|---|
| 创建/删除 | `create`、`delete` | `add_place`、`delete_place`（默认关闭，需 allow） |
| 别名 | `add-alias`、`del-alias` | `add_place_alias`、`delete_place_alias` |
| 标签/注释 | `set-tags`、`set-comment` | `set_place_tags`、`set_place_comment` |
| 匹配 | `add-match`、`add-named-match`、`del-match` | `add_place_match`（`rename` 参数）、`delete_place_match`（默认关闭） |
| 生成环境文件 | `env` | ❌ 无对应工具 |

### 6.3 设备操作

| 能力 | labgrid-client | labgrid-mcp |
|---|---|---|
| 电源 | `power`（on/off/cycle/status） | `get_power_state`、`set_power` |
| 数字 IO | `io` | `get_io`、`set_io` |
| SD/USB mux | `sd-mux`、`usb-mux` | `get_sd_mux`、`set_sd_mux`、`set_usb_mux` |
| 串口 console | `console`（microcom/telnet） | `console_open`/`console_read`/`console_send`/`console_close` |
| SSH 命令 | `ssh`、`cmd` | `ssh_run` |
| 文件传输 | `scp`、`rsync`、`sshfs` | `put_file`、`get_file`（scp 语义）；❌ sshfs/rsync |
| 端口转发 | `forward` | `forward_open`（-L）、`forward_remote_open`（-R）、`forward_close`、`forward_list` |
| DFU/fastboot | `dfu`、`fastboot` | `flash_dfu`、`flash_fastboot`（后台 job） |
| 引导/bootloader | `bootstrap` | `bootstrap`（loader: imx/mxs/rk/uuu/bdimx） |
| 脚本烧写 | `flashscript` | `flash_script` |
| 写镜像 | `write-image` | `write_image`（dd/bmaptool） |
| 视频/音频 | `video`、`audio`、`screen`、`scrcpy` | ❌ 不支持（README 明确限制） |
| 网络命名空间 | `netns` | ❌ 不支持 |
| ADB/TMC | `adb`、`tmc`、`tmc-channel`/`query` 等 | ❌ 不支持 |
| telnet 跳转 | `telnet` | ❌ 不支持 |
| 命令补全/帮助 | `help`、`complete` | MCP 协议自身提供工具发现，无 CLI 补全 |

### 6.4 pytest 与库集成

| 能力 | labgrid-client | labgrid-mcp |
|---|---|---|
| pytest 插件（`--lg-env`、fixtures） | ✅ 同一包内提供 | ❌ |
| Environment YAML 生成/加载 | ✅（`env`、`-c`） | ❌（内部按 place 动态构建 Target） |
| 自定义 driver/strategy | ✅（库方式） | ✅（复用同一 driver 栈，但不负责测试编排） |

---

## 7. labgrid-mcp 相对 client 的增强

1. **结构化接口**：47 个工具全部返回 JSON，Agent 无需解析文本。
2. **有状态会话**：持久 gRPC 订阅、Target 缓存、console 环形缓冲、后台 job。
3. **长任务支持**：烧写类操作异步化，配套 `flash_status`/`flash_logs`。
4. **预约保活**：进程内 keepalive，避免 Agent 两次调用之间 reservation 过期。
5. **安全门控**：
   - `LABGRID_MCP_READONLY=1` 只注册只读工具；
   - `LABGRID_MCP_ALLOW` 按类别白名单；
   - `flash` 与 `place_delete` **默认关闭**，必须显式开启；
   - 每个工具带 `readOnly`/`destructive`/`idempotent` 注解。
6. **MCP Resources**：`labgrid://places`、`labgrid://resources`、
   `labgrid://reservations`、`labgrid://sessions` 等可直接浏览。

## 8. labgrid-client 相对 mcp 的独有能力

1. **交互式终端体验**：`console` 通过 microcom/telnet 提供真正的 TTY；
2. **pytest 工作流**：插件、fixtures、`--lg-env`、step 日志；
3. **多媒体与调试工具**：`video`、`audio`、`screen`、`scrcpy`、`netns`、`telnet`；
4. **ADB/TMC**：`adb`、`tmc` 系列；
5. **sshfs/rsync**、`env` 生成、`help`/`complete`/`version`；
6. **无额外进程**：不需要常驻 MCP 服务器，脚本/CI 直接调用。

---

## 9. 安全模型差异

| 项目 | labgrid-client | labgrid-mcp |
|---|---|---|
| 认证 | 无内置账号密码，依赖网络隔离/TLS/SSH | 相同（README：Security is delegated to the network） |
| 授权粒度 | 仅 place acquire 互斥 + `allow` | 额外提供 readonly/allowlist/opt-in 策略 |
| 危险操作 | 命令可用即可执行（如 `delete`、烧写） | 默认禁用 `flash`、`place_delete`，需显式允许 |
| 身份隔离 | 每个 OS 用户/主机一个身份 | 每个 MCP 实例一个身份；不建议多人共享实例 |

### 9.1 对 MCP 层安全策略的满足度

| 策略 | 满足度 | 实际实现与缺口 |
|---|---|---|
| place 独占，防止多 Agent 抢板 | **基本满足** | 复用 coordinator 的 acquire 互斥；`acquire_place` 串行完成 reserve→wait→acquire 并带 keepalive；所有硬件工具执行前重复校验 `place.acquired == 本实例身份`（`target.py:_check_owned`、`server.py:_check_place_owned`）。缺口：身份是 `host/user` 自声明，coordinator 无认证；多个 Agent 共享一个 MCP 实例会共享同一身份。 |
| 工具白名单与参数校验 | **部分满足** | `LABGRID_MCP_READONLY` 只注册只读工具；`LABGRID_MCP_ALLOW` 按类别放行；`flash`、`place_delete` 默认关闭，必须显式开启；FastMCP/Pydantic 做类型校验，另有 match 段数、身份格式、power action、write_image mode、本地文件存在性等检查。缺口：只有**类别级**白名单，没有按工具/place/路径的白名单；`flash_script`、`ssh_run` 可执行任意本地脚本/远端命令；没有镜像目录或路径限制。 |
| 烧录/断电前人工确认或审批 | **不满足** | 没有 `confirm` 步骤、审批服务、延时或双人确认。只有 MCP 的 `destructiveHint` 注解，是否弹窗取决于 MCP 客户端；`flash` 默认关闭，但 `power`/`io`/`console`/`ssh` 默认开启，Agent 可直接断电/重启。 |
| 操作审计与镜像版本回滚 | **不满足** | 串口日志是 64 KiB 内存环形缓冲、空闲 600 s 清理；烧录日志是 256 KiB 内存缓冲、完成 1 h 后清理，进程重启即丢失。没有结构化的“谁在何时对哪个 place 调用了什么工具”审计流，也没有镜像版本库、校验和、烧录前备份或回滚机制。 |

补充风险：coordinator 本身无认证、无 TLS 默认；metadata RPC 没有所有权检查，MCP 只能在客户端侧补
`force=True` 门禁。需要合规级策略时，应在 MCP 外层增加网关/代理：工具调用审计、审批令牌、
镜像白名单与回滚，以及每 Agent 独立身份和网络隔离。

---

## 10. 部署与版本要求

| 项目 | labgrid-client | labgrid-mcp |
|---|---|---|
| 安装 | `pip install .`（labgrid 包） | `uvx labgrid-mcp` / `pip install labgrid-mcp` / Docker |
| 依赖 | labgrid 自身依赖 | 以未修改的 labgrid 作为库依赖；需要 **gRPC 时代 coordinator（labgrid ≥ 24，验证至 26.x）** |
| 运行位置 | 开发者/CI 主机 | 能访问 coordinator 的主机；推荐每个用户/Agent 一个实例 |
| 平台 | Linux（POSIX 依赖） | 同样需要 Linux 环境（依赖 labgrid） |

---

## 11. 选型建议

| 场景 | 推荐 |
|---|---|
| 人工调试、看串口、临时操作 | `labgrid-client` |
| 自动化测试、CI 回归 | `labgrid-client` + pytest |
| AI Agent 驱动板卡、自然语言操作 | `labgrid-mcp` |
| 需要视频/音频/netns/ADB/TMC | `labgrid-client` |
| 需要给 Agent 提供只读或受限控制面 | `labgrid-mcp`（readonly/allowlist） |
| 需要把烧写做成非阻塞长任务 | `labgrid-mcp`（job 机制） |

> 两者可以共存：MCP 服务器与 `labgrid-client` 用户共享同一 coordinator、
> 同一 place 互斥规则，互不干扰。

---

## 12. 参考

- labgrid：`doc/overview.rst`、`doc/usage.rst`、`labgrid/remote/client.py`
- labgrid-mcp：`README.md`、`docs/DESIGN.md`（§3 架构、§11 实现细节）、
  `src/labgrid_mcp/{coordinator,target,server,policy,jobs,console}.py`
- 相关内部文档：`doc/architecture.md`、`doc/deployment.md`
