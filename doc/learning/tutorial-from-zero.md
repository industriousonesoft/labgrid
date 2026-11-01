# labgrid 从零开始使用与配置教程

面向第一次接触 labgrid 的工程师，从安装、最小可运行示例开始，逐步覆盖串口、
电源、网络/SSH、USB 烧写与外设的配置，最后过渡到 pytest 和分布式环境。
所有 YAML/命令均与当前源码和仓库示例对应（`examples/`、`doc/configuration.rst`、
`doc/usage.rst`）。

> 适用环境：Linux（Ubuntu 22.04 已验证）。Windows 10 请先按
> `doc/deployment.md` 配置 WSL2，再按本文操作。

---

## 0. 10 分钟最小路径

如果板子已通过 USB 串口连接，只想马上跑起来：

```bash
# 1) 安装（详细步骤见第 2 节）
python3.11 -m venv ~/labgrid-venv
source ~/labgrid-venv/bin/activate
pip install -U pip
pip install .            # 在 labgrid 源码目录执行

# 2) 确认串口
ls -l /dev/ttyUSB0
sudo usermod -aG dialout "$USER"   # 重新登录生效

# 3) 写配置
mkdir -p ~/lab && cd ~/lab
cat > local.yaml <<'EOF'
targets:
  main:
    resources:
      RawSerialPort:
        port: "/dev/ttyUSB0"
        speed: 115200
    drivers:
      ManualPowerDriver: {}
      SerialDriver: {}
      ShellDriver:
        prompt: 'root@[\w-]+:[^ ]+ '
        login_prompt: ' login: '
        username: 'root'
EOF

# 4) 用 Python API 冒烟验证
python - <<'PY'
from labgrid import Environment
t = Environment("local.yaml").get_target("main")
shell = t.get_driver("ShellDriver")      # 自动激活
out, err, rc = shell.run("uname -a")
print(err, rc)
print("".join(out))
PY
```

能打印出内核版本即说明串口、登录、Shell 全链路正常。

---

## 1. 核心概念

| 概念 | 含义 | 例子 |
|---|---|---|
| Target | 一块被控设备的抽象 | `main` |
| Resource | 被动描述"如何访问硬件"，不含动作 | `RawSerialPort`、`NetworkPowerPort` |
| ManagedResource | 可热插拔、通过 udev 发现的资源 | `USBSerialPort`、`IMXUSBLoader` |
| Driver | 使用 Resource 执行动作 | `SerialDriver`、`ShellDriver` |
| Protocol | Driver 之间的能力契约 | `ConsoleProtocol`、`CommandProtocol` |
| Strategy | 把板子从状态 A 迁移到状态 B | `BareboxStrategy`、`ShellStrategy` |
| Environment | 描述 Target 的 YAML 文件 | `local.yaml` |
| Place | 分布式环境下的一组资源（coordinator 上） | `bench-12` |

依赖关系：Resource → Driver → 更高层 Driver/Strategy。例如
`RawSerialPort` → `SerialDriver` → `ShellDriver`。Target 在加载 YAML 时按
`bindings` 自动解析依赖；多个同类资源时可用名称显式绑定（见 4.6）。

---

## 2. 安装

### 2.1 系统依赖

```bash
sudo apt update
sudo apt install -y git build-essential pkg-config libudev-dev \
    python3.11 python3.11-venv python3.11-dev \
    microcom openssh-client rsync
```

- 串口调试：`microcom`（或 `picocom`）
- 分布式/文件传输：`openssh-client`、`rsync`
- 仅 exporter 需要：`ser2net`（见 `doc/deployment.md`）

Ubuntu 22.04 系统 Python 是 3.10，不满足 labgrid 的 `>=3.11` 要求，必须显式用
`python3.11` 建虚拟环境。Ubuntu 22.04 默认软件源不含 `python3.11`，若上面的
`apt install` 报找不到包，请按 `doc/deployment.md` 添加 deadsnakes PPA，或用
`uv`/`pyenv` 安装 3.11。

### 2.2 源码安装

```bash
git clone https://github.com/labgrid-project/labgrid.git
cd labgrid
python3.11 -m venv ~/labgrid-venv
source ~/labgrid-venv/bin/activate
pip install -U pip
pip install .                 # 生产使用
# 或 pip install -e ".[dev]" # 开发/调试
```

验证：

```bash
labgrid-client --help
labgrid-exporter --help
labgrid-coordinator --help
```

### 2.3 串口权限

```bash
sudo usermod -aG dialout,plugdev "$USER"
# 注销并重新登录，或 newgrp dialout
groups | grep dialout
```

---

## 3. 第一个最小实验（本地串口 + Shell）

### 3.1 确认硬件

```bash
lsusb
dmesg | tail -20                  # 插入后应出现 ttyUSB0
ls -l /dev/ttyUSB0
udevadm info -q property -n /dev/ttyUSB0 | sort
```

用 `microcom -s 115200 /dev/ttyUSB0` 手动确认能看到板子输出。按 `Ctrl-\` 进入菜单，
输入 `exit` 返回终端、`quit` 退出；若提示 "Device or resource busy"，先运行
`fuser -v /dev/ttyUSB0` 查占用。

注意：**microcom 能打开只说明 tty 存在，不等于数据通路正常**。按下面顺序自检：

1. 敲回车看是否有新输出：乱码多半是波特率不对；完全无反应则检查板子是否上电、
   TX/RX 是否交叉、GND 是否共地。
2. 回环测试（最可靠）：拔掉板子，短接 USB-UART 适配器自己的 TX 与 RX（或板上的
   回环跳线），在 microcom 里敲字符，应立即回显。能回显说明适配器、驱动、权限、
   波特率全部正常，问题在板子侧或线序。
3. 电平与线序：USB-UART 输出的是 3.3V/5V TTL，不能直连 RS-232；GND 必须相连，
   设备 TX 接适配器 RX、设备 RX 接适配器 TX。
4. 系统侧核对：`dmesg | tail` 无 USB 错误；`stty -F /dev/ttyUSB0 -a` 看实际参数；
   `fuser -v /dev/ttyUSB0` 确认无其他进程占用。
5. 需要留档时用 `microcom -l serial.log -d -s 115200 /dev/ttyUSB0`，退出后检查
   `serial.log` 里的原始字节。

如果手工串口正常、但 labgrid 的 `ShellDriver` 卡住，问题通常在提示符正则匹配，
而不是串口本身（见第 12 节）。

### 3.2 环境配置 `local.yaml`

```yaml
targets:
  main:
    resources:
      RawSerialPort:
        port: "/dev/ttyUSB0"
        speed: 115200
    drivers:
      ManualPowerDriver: {}
      SerialDriver: {}
      ShellDriver:
        prompt: 'root@[\w-]+:[^ ]+ '
        login_prompt: ' login: '
        username: 'root'
```

说明：

- `ManualPowerDriver` 不控制硬件，只在需要上电/重启时提示人工操作，
  提示中显示的是 Target 名称（`main`）；
- `SerialDriver` 绑定 `ConsoleProtocol`；
- `ShellDriver` 绑定 `ConsoleProtocol`，实现命令执行与文件传输协议；
- `prompt`/`login_prompt` 是**正则表达式**，必须匹配板子实际输出；
- 若目标板登录需要密码，在 `ShellDriver` 中加 `password: 'yourpassword'`；
  缺少时会报 `Exception: Password entry needed but no password set`。
  空密码写 `password: ''`；如果串口已经停在登录后的 shell 提示符，
  则不会走登录流程，无需 password。

### 3.3 用 Python API 执行命令

3.3/3.4 及第 10 节的示例都使用同一个目录（按第 0 节创建即可）：

```text
~/lab/
├── local.yaml       # 3.2
├── run.py           # 3.3 的脚本（也可用 python - <<'PY' 内联，不落盘）
├── conftest.py      # 3.4
├── test_shell.py    # 3.4
└── logs/            # 可选，--lg-log 生成
```

脚本保存为 `run.py`：

```python
from labgrid import Environment

env = Environment("local.yaml")
target = env.get_target("main")
shell = target.get_driver("ShellDriver")     # 自动激活依赖链

stdout, stderr, rc = shell.run("cat /proc/version")
print(rc, stdout, stderr)

out = shell.run_check("uname -a")            # 非零退出码会抛异常
print(out[0])
```

```bash
cd ~/lab
source ~/labgrid-venv/bin/activate
python run.py
```

普通 Python 脚本默认不显示 labgrid 的步骤日志，排错时在 `run.py` 开头加：

```python
import logging
logging.basicConfig(level=logging.DEBUG, format="%(asctime)s %(levelname)s %(name)s: %(message)s")
```

并用 `timeout 150 python -u run.py` 运行，避免无限等待且输出不被缓冲。
`ShellDriver` 的默认静默期可能长达 60s（`login_timeout`）加 30s（prompt 检查）。

`Environment("local.yaml")` 按**当前工作目录**解析相对路径；如果从其他目录启动
脚本，改用绝对路径，例如：

```python
import pathlib
env = Environment(str(pathlib.Path(__file__).with_name("local.yaml")))
```

### 3.4 用 pytest 执行

`conftest.py` 和 `test_shell.py` 放在 `local.yaml` 同一目录。pytest 会自动向上
查找 `conftest.py`，所以 `~/lab` 下的 fixture 对子目录中的测试同样生效。

pytest 插件随 labgrid 一起安装。先加一个自定义 fixture：

```python
# conftest.py
import pytest

@pytest.fixture(scope="session")
def command(target):
    shell = target.get_driver("CommandProtocol")
    target.activate(shell)
    return shell
```

```python
# test_shell.py
def test_shell(command):
    stdout, stderr, returncode = command.run("cat /proc/version")
    assert returncode == 0
    assert stdout
    assert not stderr
    assert "Linux" in stdout[0]
```

```bash
cd ~/lab
pytest --lg-env local.yaml --capture=no -v
```

`--lg-env` 也可用环境变量 `LG_ENV` 指定；`--lg-env` 的相对路径同样按当前工作目录
解析。若在别处启动 pytest，可写 `pytest --lg-env ~/lab/local.yaml ~/lab/test_shell.py`。

---

## 4. 串口配置详解

### 4.1 三种串口资源

| 资源 | 适用场景 | 关键参数 |
|---|---|---|
| `RawSerialPort` | 本机固定设备名 | `port`、`speed` |
| `USBSerialPort` | 本机 USB 串口，热插拔/重启后设备名变化 | `match`（udev）、`speed` |
| `NetworkSerialPort` | 串口服务器或 ser2net 导出的串口 | `host`、`port`、`speed`、`protocol` |

`RawSerialPort` 简单但脆弱：重新插拔后可能从 `ttyUSB0` 变成 `ttyUSB1`。长期使用
建议用 `USBSerialPort` + udev 匹配。

### 4.2 udev 匹配（推荐）

```yaml
USBSerialPort:
  match:
    ID_SERIAL_SHORT: 'P-00-00682'
    ID_USB_INTERFACE_NUM: '00'
  speed: 115200
```

或按物理路径匹配（更稳定，不受枚举顺序影响）：

```yaml
USBSerialPort:
  match:
    '@ID_PATH': 'pci-0000:05:00.0-usb-0:1.2.5.5'
  speed: 115200
```

规则要点（`doc/configuration.rst` udev Matching 一节）：

1. 所有匹配项必须同时成立；
2. 以 `@` 开头的键匹配**父设备**。`USBSerialPort` 需要 `@ID_PATH`/`@ID_SERIAL_SHORT`，
   因为实际 tty 设备在 USB 串口转换器之下；
3. 内置匹配已要求 `SUBSYSTEM=tty` 与父设备 `SUBSYSTEM=usb`；
4. udev 属性值区分大小写，十六进制要完全一致；
5. `ID_PATH` 可能缺失，可加 udev 规则补齐：
   `SUBSYSTEMS=="usb", IMPORT{builtin}="path_id"`；
6. 不确定匹配项时，运行 `labgrid-suggest`，然后**插拔设备**，它会打印可直接粘贴的
   YAML 片段。

`ID_PATH` 与 `@ID_PATH` 的具体区别（以本机 `ttyACM0` 为例）：

| 匹配键 | 检查对象 | 本机实际值 |
|---|---|---|
| `ID_PATH` | 被匹配的 tty 设备自身 | `pci-0000:00:14.0-usb-0:3.2:1.0` |
| `@ID_PATH` | 父设备链（不含自身），任一父设备匹配即可 | USB 设备：`pci-0000:00:14.0-usb-0:3.2`；接口：`...-0:3.2:1.0` |

所以 `USBSerialPort` 常用 `@ID_PATH` 指向 USB 设备父节点（不带 `:1.0`）；
也可以写接口路径（带 `:1.0`），两者都能命中，但前者更贴近"物理 USB 口"。
直接匹配 USB 设备本身的资源（如 `AndroidUSBFastboot`、`IMXUSBLoader`）才用
不带 `@` 的 `ID_PATH`。多个 `@` 条件之间仍是 AND，每个条件由某个父设备满足即可。

调试 udev 属性：

```bash
udevadm info -q property -n /dev/ttyUSB0 | sort
udevadm info -a -n /dev/ttyUSB0 | head -60
```

从已可用的 `RawSerialPort` 迁移时：

```bash
DEV=/dev/ttyUSB0    # 也可能是 /dev/ttyACM0
udevadm info -q property -n "$DEV" |
    grep -E 'ID_PATH|ID_SERIAL_SHORT|ID_USB_INTERFACE_NUM|SUBSYSTEM'
```

```yaml
USBSerialPort:
  match:
    '@ID_PATH': 'pci-0000:00:14.0-usb-0:3.2'   # 换成上面的实际值
    ID_USB_INTERFACE_NUM: '00'                  # 多接口/多串口时保留
  speed: 115200
```

同时**删除原来的 `RawSerialPort`**：两者都继承 `SerialPort`，未显式绑定时
`SerialDriver` 会报 `multiple resources matching SerialPort found`。

不登录、只验证资源是否匹配：

```python
from labgrid import Environment

t = Environment("local.yaml").get_target("main")
r = t.get_resource("USBSerialPort")
print(r.avail, r.port, r.match)
```

应输出 `True /dev/ttyUSB0 ...`（或 `ttyACM0`）。`ttyACM*` 同样使用
`USBSerialPort`，不要被名字误导；它同样有 `SUBSYSTEM=tty`、父设备 `usb`。

为什么 `USBSerialPort` 看起来比 `RawSerialPort` 慢：

- `RawSerialPort` 是静态资源，创建时直接标记为可用，**不检查设备是否存在**；
  设备不在时，要到打开串口才报错。
- `USBSerialPort` 是 `ManagedResource`：`UdevManager` 启动时枚举现有 udev 设备，
  用每个资源逐条匹配（初始开销约为资源数 × 设备数），之后靠 udev 事件更新。
- 设备插入后还要等内核枚举、驱动绑定（如 ftdi_sio/cdc_acm）和 udev 处理完，
  这段时间通常为几百毫秒到数秒。
- `Target.await_resources()` 的检查间隔是 0.5s，`USBResource` 默认等待超时 5s，
  所以首次等待可能额外多出半秒。
- 本机实测单个串口：`RawSerialPort` 约 0 ms，`USBSerialPort` 约 30 ms。如果体感
  是数秒，通常是设备刚插入/尚未就绪，或 `ShellDriver` 在等登录提示符，而不是
  匹配算法本身慢。

想兼顾稳定与速度：先插好设备再启动流程；固定机架也可以直接用
`/dev/serial/by-id/...` 或 `/dev/serial/by-path/...` 符号链接配 `RawSerialPort`，
但这样不会跟随热插拔更新可用状态。

### 4.3 SerialDriver 参数

| 参数 | 默认 | 作用 |
|---|---|---|
| `txdelay` | `0.0` | 每发送一个 chunk 前的延迟（秒） |
| `txchunk` | `1` | 每次发送的字节数，`txdelay` 按其计 |
| `timeout` | `3.0` | 网络串口连接超时 |
| `linesep` | `\n` | 发送整行时的行分隔符 |

板子串口丢字符时，常用 `txdelay: 0.01`、`txchunk: 1`（仓库示例即如此配置）。

### 4.4 ShellDriver 参数

```yaml
ShellDriver:
  prompt: 'root@[\w-]+:[^ ]+ '
  login_prompt: ' login: '
  username: 'root'
  password: 'root'          # 可选
  login_timeout: 60         # 登录提示等待时间
  await_login_timeout: 2    # 静默多久后补发回车
  post_login_settle_time: 0 # 登录后等待日志安静的秒数
  console_ready: '...'      # 可选，内核提示可按回车激活控制台
  keyfile: '/path/id_rsa'   # 可选，登录后上传，供 SSHDriver 使用
```

常见问题：`bash >= 5.1` 会启用 bracketed paste，在提示符前输出
`\e[?2004h`。若提示符匹配失败，可在 `prompt` 开头加 `\x1b\[\?2004h`。

另一个常见提示符风格是 `[root@M3576 ~]# `（方括号包裹、没有冒号）。必须按实际
输出写，例如：

```yaml
prompt: '\[root@\S+ [^\]]*\]# '
```

`[` 在正则中需转义；YAML 里用单引号，避免 `#` 被当作注释。可通过
`logging.basicConfig(level=logging.DEBUG)` 查看串口实际读到的提示符。

### 4.5 直接用 NetworkSerialPort（不经过 labgrid exporter）

```yaml
resources:
  NetworkSerialPort:
    host: 'serial-server.example.com'
    port: 4001
    speed: 115200
    protocol: 'rfc2217'       # 或 'raw'
drivers:
  SerialDriver: {}
  ShellDriver:
    prompt: 'root@[\w-]+:[^ ]+ '
    login_prompt: ' login: '
    username: 'root'
```

### 4.6 多串口/多资源时的显式绑定

同一 Target 有两个串口时，需要给资源命名并让驱动按名字绑定。注意 YAML 映射
不允许重复键，同类资源的多个实例必须写成**列表**：

```yaml
targets:
  main:
    resources:
    - RawSerialPort:
        name: console
        port: /dev/ttyUSB0
    - RawSerialPort:
        name: secondary
        port: /dev/ttyUSB1
    drivers:
    - SerialDriver:
        name: console_driver
        bindings:
          port: console
    - ShellDriver:
        prompt: 'root@[\w-]+:[^ ]+ '
        login_prompt: ' login: '
        username: 'root'
        bindings:
          console: console_driver
```

`bindings` 的键是驱动内部声明的绑定名（`SerialDriver` 的 `port`，
`ShellDriver` 的 `console`）；值可以是资源名或驱动名。`bindings` 只对同一
条目中的驱动生效。

理解要点：**Target 是"一块被测设备/一套被测系统"的抽象，不是一条串口连接**。
同一块板子上的调试串口、MCU 串口、蓝牙 HCI 串口都属于同一个 Target，各自是
独立的 Resource，可以独立激活、并行使用；只有两块相互独立测试的板子才应拆成
两个 Target（如 `main`、`secondary`）。因此多串口时：

- 每个串口资源必须有唯一 `name`；
- 每个使用它的驱动也要有 `name`，并通过 `bindings` 指向对应资源/驱动；
- Python 中按名字取：`target.get_driver("ShellDriver", name="console_shell")`，
  或按资源过滤：`target.get_driver("SerialDriver", resource=mcu_port)`；
- 若多个同类资源都没有显式绑定，labgrid 会报
  `multiple resources matching ... found`，这是防止连错端口的保护。

---

## 5. 电源配置

### 5.1 手动电源

```yaml
ManualPowerDriver: {}
```

它没有硬件参数，提示语中的名称取自 Target 名。配合 pytest 时需
`-s`/`--capture=no` 才能看到提示。

### 5.2 外部命令电源

```yaml
ExternalPowerDriver:
  cmd_on: 'my-powerctl on'
  cmd_off: 'my-powerctl off'
  cmd_cycle: 'my-powerctl cycle'   # 可选
  delay: 2.0                       # 未提供 cmd_cycle 时，off→on 间隔
```

### 5.3 网络电源（PDU）

```yaml
resources:
  NetworkPowerPort:
    model: 'gude'
    host: 'powerswitch.example.com'
    index: 0
drivers:
  NetworkPowerDriver:
    delay: 5.0
```

`model` 支持多种 PDU/实现，常用的有：

| model | 设备 |
|---|---|
| `gude` / `gude24` / `gude8031` / `gude8225` / `gude8316` | Gude 系列 PDU（HTTP API） |
| `netio` | NETIO 系列 |
| `apc` | APC PDU（SNMP） |
| `eaton` | Eaton ePDU（SNMP） |
| `digitalloggers_http` / `digitalloggers_restapi` | Digital Loggers |
| `eth008` | Robot-Electronics ETH008 |
| `eg_pms2_network` | EG PM2 |

完整列表（含 `digipower`、`mfi_mpower`、`netio_kshell`、`raritan`、`shelly_gen1/2`、
`tplink`、`tapo` 等）见 `labgrid/driver/power/` 目录和 `doc/configuration.rst`
的 NetworkPowerPort 一节。

Tasmota 插座不使用 `NetworkPowerPort`，而是独立的 `TasmotaPowerPort`（MQTT）：

```yaml
resources:
  TasmotaPowerPort:
    host: 'mqtt.example.com'
    avail_topic: 'tasmota/plug1/tele/LWT'
    power_topic: 'tasmota/plug1/cmnd/POWER'
    status_topic: 'tasmota/plug1/stat/POWER'
drivers:
  TasmotaPowerDriver: {}
```

需要安装 `mqtt` extra：`pip install ".[mqtt]"`。

### 5.4 USB 电源（uhubctl）

```yaml
resources:
  USBPowerPort:
    match:
      ID_PATH: 'pci-0000:00:14.0-usb-0:2:1.0'
    index: 1
drivers:
  USBPowerDriver: {}
```

需要系统安装 `uhubctl`，且 udev `ID_PATH` 需包含接口（通常以 `:1.0` 结尾）。

---

## 6. 网络与 SSH

### 6.1 SSH 登录目标板

```yaml
resources:
  NetworkService:
    address: '192.168.1.20'
    username: 'root'
    password: 'root'        # 可选；推荐用 keyfile
    port: 22
drivers:
  SSHDriver:
    keyfile: '/home/user/.ssh/id_rsa'
    connection_timeout: 30.0
```

```python
target.get_driver("SSHDriver").run("uname -a")
```

`SSHDriver` 实现 `CommandProtocol` 和 `FileTransferProtocol`，可直接替换
`ShellDriver` 供 pytest 使用。

### 6.2 网络接口

```yaml
resources:
  NetworkInterface:
    ifname: 'eth0'
  USBNetworkInterface:
    match:
      ID_PATH: 'pci-0000:06:00.0-usb-0:1.3.2:1.0'
```

配合 `NetworkInterfaceDriver`/`RawNetworkInterfaceDriver` 使用（DHCP、配置 IP、
抓包等）。

> 远程 exporter 场景下，`NetworkService.address` 可带设备作用域后缀（如
> `192.168.1.20%eth1`），由 exporter 通过 `socat` 代理 SSH，需要管理员预先配置
> `labgrid-bound-connect` sudo 助手。

---

## 7. 其他常用外设

### 7.1 USB Loader / Fastboot / DFU

```yaml
resources:
  IMXUSBLoader:
    match:
      ID_PATH: 'pci-0000:06:00.0-usb-0:1.3.2:1.0'
  AndroidUSBFastboot:
    match:
      ID_PATH: 'pci-0000:06:00.0-usb-0:1.3.2:1.0'
  DFUDevice:
    match:
      ID_PATH: 'pci-0000:06:00.0-usb-0:1.3.2:1.0'
drivers:
  IMXUSBDriver: {}
  AndroidFastbootDriver: {}
  DFUDriver: {}
```

- `IMXUSBLoader` 也用于 `UUUDriver`、`BDIMXUSBDriver`；
- Rockchip 用 `RKUSBLoader` + `RKUSBDriver`；
- 网络 fastboot 用 `AndroidNetFastboot`（地址 + 端口 5554）。

#### 7.1.1 Rockchip 烧录线配置

先分清设备当前处于哪种 USB 模式，配置完全不同：

| 模式 | 典型 USB ID | 用途 | labgrid 资源 |
|---|---|---|---|
| 系统已启动（ADB） | `2207:0006`（产品名 `rk3xxx`，配置名 `adb`） | 执行命令、传文件、重启 | `USBADBDevice` + `ADBDriver` |
| MaskROM / Loader | `2207:xxxx`，PID 随 SoC 变化 | 烧写 bootloader/镜像 | `RKUSBLoader` + `RKUSBDriver` |

确认当前模式：

```bash
lsusb -d 2207:
udevadm info -q property -n /dev/bus/usb/001/115 | grep -E 'ID_(VENDOR|MODEL)_ID|ID_PATH'
```

`RKUSBLoader.filter_match()` 只接受一组硬编码的 Rockchip PID，例如：

| SoC | PID | SoC | PID |
|---|---|---|---|
| RK3588 | `350b` | RK3568 | `350a` |
| RK3576 | `350e` | RK3562 | `350d` |
| RK3528 | `350c` | RK3399 | `330c` |
| RK3328 | `320c` | RK3288 | `320a` |
| RK3308 | `330e` | PX30 | `330d` |
| RK3128 | `310c` | RK3036 | `301a` |
| RV1126 | `110b` | RV1106 | `110c` |

**`2207:0006` 不在列表中**，所以 ADB 模式下无论怎么配 `RKUSBLoader` 都不会
`avail=True`。要烧录，需按住板子的 MaskROM/Recovery 键再上电或重新插线，
用 `lsusb -d 2207:` 确认 PID 已变成上表中的值。同一物理口重新枚举后
`ID_PATH` 不变，因此用 `ID_PATH` 匹配最稳。

> `adb reboot loader` / `reboot edl` 是否进入 loader 模式取决于板子的
> bootloader；不确定时用按键进 MaskROM。

**烧录配置**（环境文件，`ID_PATH` 直接匹配 USB 设备本身，不加 `@`）：

```yaml
tools:
  rk-usb-loader: /usr/local/bin/rk-usb-loader

images:
  rk_loader: /home/me/images/rk3588_spl_loader.bin   # 一阶段 loader
  rk_image: /home/me/images/rootfs.img               # 要写入的镜像

targets:
  main:
    resources:
      RKUSBLoader:
        match:
          ID_PATH: 'pci-0000:00:14.0-usb-0:3.4'
    drivers:
      RKUSBDriver:
        usb_loader: rk_loader   # 先用 rk-usb-loader db 下载到 DRAM
        image: rk_image         # 再用 rk-usb-loader wl 0x40 写入存储
```

要点：

1. `ID_PATH` 是 USB 设备自身的路径（到 `...-0:3.4` 为止），不要加 `:1.0`
   接口后缀；也不要用 `@ID_PATH`，`@` 是查父设备，`RKUSBLoader` 匹配的是
   USB 设备本身；
2. `usb_loader` 与 `image` 都是 `images:` 里的键名，不是直接路径；`load()`
   会依次执行 `rk-usb-loader db <loader>` 和 `rk-usb-loader wl 0x40 <image>`；
3. 也可用 `ID_SERIAL_SHORT` 匹配，但 ADB 模式与 MaskROM 模式的序列号可能
   不同，优先 `ID_PATH`；
4. 未安装工具时 `RKUSBDriver` 会报找不到 `rk-usb-loader`（本地当前即如此），
   需要自行编译：

```bash
sudo apt install build-essential pkg-config libusb-1.0-0-dev
git clone --depth=1 https://git.pengutronix.de/git/barebox
gcc -O2 -o rk-usb-loader barebox/scripts/rk-usb-loader.c $(pkg-config --cflags --libs libusb-1.0)
sudo install -m 0755 rk-usb-loader /usr/local/bin/
```

5. 普通用户访问 USB 设备需要 udev 权限：

```bash
# /etc/udev/rules.d/99-rockchip.rules
SUBSYSTEM=="usb", ATTR{idVendor}=="2207", MODE="0666"
```

```bash
sudo udevadm control --reload-rules && sudo udevadm trigger
```

**只想在 ADB 模式下操作**（不烧录）时，省去 udev 匹配，直接用序列号：

```yaml
targets:
  main:
    resources:
      USBADBDevice:
        serialno: '2d5a8e882e5784e7'
    drivers:
      ADBDriver: {}
```

```bash
sudo apt install adb
adb devices   # 确认设备在线
```

`ADBDriver` 支持 `run()`、`put()`/`get()` 和 `reset()`，但**不能烧写
bootloader**；烧写仍要走上面的 `RKUSBLoader` 路径。

**分布式场景**：在 exporter 配置中写同样的 `RKUSBLoader` 资源，客户端用
`RemotePlace` 获取即可；exporter 会自动把它导出为 `NetworkRKUSBLoader`，
客户端无需手动配置该类。

### 7.2 USB 大容量存储 / SD mux

```yaml
resources:
  USBSDMuxDevice:
    match:
      '@ID_PATH': 'pci-0000:00:14.0-usb-0:1.2'
  LXAUSBMux:
    match:
      '@ID_PATH': 'pci-0000:00:14.0-usb-0:1.2'
  USBMassStorage:
    match:
      ID_PATH: 'pci-0000:06:00.0-usb-0:1.3.2:1.0-scsi-0:0:0:3'
drivers:
  USBSDMuxDriver: {}
  LXAUSBMuxDriver: {}
  USBStorageDriver: {}
```

写镜像需要 `dd` 或 `bmaptool`；挂载文件系统写入还需要 `python3-gi` 与
`gir1.2-udisks-2.0`（可选）。

### 7.3 数字 IO

```yaml
resources:
  HttpDigitalOutput:
    url: 'http://relay.example/set'
    body_asserted: 'On'
    body_deasserted: 'Off'
  ModbusTCPCoil:
    host: '192.168.23.42:502'
    coil: 1
drivers:
  HttpDigitalOutputDriver: {}
  ModbusCoilDriver: {}
```

### 7.4 仪器（PyVISA）

```yaml
resources:
  PyVISADevice:
    type: 'TCPIP'
    url: '192.168.1.50'
drivers:
  PyVISADriver: {}
```

`target.get_driver("PyVISADriver").get_session()` 返回原生 pyvisa resource，
可 `write`/`query`，具体命令见仪器手册。

> 以上资源/驱动多数需要安装对应 extras，例如
> `pip install ".[pyvisa,modbus,snmp,onewire]"`。

---

## 8. Environment 配置进阶

### 8.1 文件结构

```yaml
targets:
  main:
    resources: {...}
    drivers: {...}
  secondary:
    resources: {...}
    drivers: {...}

imports:
  - my_strategy.py          # 自定义 Strategy/Driver 的 Python 文件

options:
  some_option: value
```

### 8.2 模板替换

```yaml
targets:
  main:
    resources:
      RemotePlace:
        name: !template '$LG_PLACE'
    drivers:
      SerialDriver: {}
```

- `$LG_*` 替换为对应环境变量；
- `$BASE` 替换为 YAML 文件所在目录。

### 8.3 激活与资源等待

`target.get_driver("ShellDriver")` 会自动激活依赖链，并等待 ManagedResource
出现（按资源超时）。手动控制：

```python
target.activate(shell)
target.deactivate(shell)
```

同类驱动互斥时（如 Bootloader 与 Shell），activation 会处理依赖的
activate/deactivate。

---

## 9. 使用 Strategy 管理启动流程

`ShellStrategy`（off → shell）：

```yaml
targets:
  main:
    resources:
      RawSerialPort:
        port: "/dev/ttyUSB0"
    drivers:
      ManualPowerDriver: {}
      SerialDriver: {}
      ShellDriver:
        prompt: 'root@[\w-]+:[^ ]+ '
        login_prompt: ' login: '
        username: 'root'
      ShellStrategy: {}
```

```python
strategy = target.get_driver("ShellStrategy")
strategy.transition("shell")
```

`BareboxStrategy`（off → barebox → shell）与 `UBootStrategy` 需要额外配置
`BareboxDriver`/`UBootDriver`，示例见 `examples/strategy/local.yaml`。

pytest 中可用 `strategy` fixture；板子已在已知状态时用
`--lg-initial-state=shell` 跳过启动流程（要求 Strategy 实现 `force()`）。

---

## 10. pytest 使用

### 10.1 常用选项

| 选项 | 作用 |
|---|---|
| `--lg-env=FILE` | 指定 Environment YAML（等同 `LG_ENV`） |
| `--lg-coordinator=HOST[:PORT]` | 指定 coordinator（默认 `127.0.0.1:20408`） |
| `--lg-log=DIR` | 保存串口日志，`DIR/console_main` 可 `tail -F` 实时查看 |
| `--lg-initial-state=STATE` | 设定 Strategy 初始状态 |
| `-vv` / `-vvv` | 步骤报告 / 调试日志 |
| `--capture=no` / `-s` | ManualPowerDriver 提示可见 |

### 10.2 内置 fixtures

| fixture | 内容 |
|---|---|
| `env` | `Environment` 对象（session 级） |
| `target` | 默认 `main` Target |
| `strategy` | Target 中的 Strategy（若配置） |

`command` 不是内置 fixture，需要像 3.4 那样在 `conftest.py` 中自定义。

### 10.3 示例测试

```python
def test_kernel(command):
    out = command.run_check("cat /proc/version")   # 返回 stdout 行列表，失败抛异常
    assert "Linux" in out[0]

def test_failure(command):
    out, err, rc = command.run("false")
    assert rc != 0
```

```bash
pytest --lg-env local.yaml --lg-log=logs --capture=no -vv
tail -F logs/console_main      # 另开终端观察串口
```

---

## 11. 从本地到分布式（简要）

本地 Environment 直接访问硬件；多机共享时增加 exporter/coordinator/place。
完整架构见 `doc/architecture.md`，部署见 `doc/deployment.md`。

先分清三类 YAML，它们的 schema 和读取者完全不同：

| 文件 | 顶层结构 | 读取者 | 作用 |
|---|---|---|---|
| `local.yaml` / `remote.yaml` | `targets` → `resources` + `drivers` | `Environment`、pytest、库 | 描述"一块板子用什么驱动" |
| `exporter.yaml` | 资源组名 → 资源列表 + `location` | `labgrid-exporter` | 描述"这台机器向 coordinator 发布哪些硬件" |
| coordinator 的 `resources.yaml` / `places.yaml` | 内部状态 | `labgrid-coordinator` | 自动持久化，不要手工编辑 |

所以 `local.yaml` 里的 `SerialDriver` 不会出现在 `exporter.yaml` 中：exporter 只发布
资源，driver 留在 client 侧（`remote.yaml` 的 `RemotePlace` 会把 exporter 的资源
展开成本地资源，再绑定同样的 driver）。**资源条目本身完全通用**：类名和参数由
同一套 Resource 类定义，`labgrid-suggest` 输出的片段既可粘到 Environment 的
`resources:` 下，也可直接作为 exporter 的资源条目。同一个 udev 匹配的串口，两种
文件里的资源写法一致，区别只在容器：

```yaml
# exporter.yaml：顶层是资源组名，条目直接就是资源
bench-12:
  USBSerialPort:
    match:
      '@ID_PATH': 'pci-0000:05:00.0-usb-0:1.2.5.5'
```

```yaml
# local.yaml：顶层是 target，资源放在 resources 下，并额外有 drivers
targets:
  main:
    resources:
      USBSerialPort:
        match:
          '@ID_PATH': 'pci-0000:05:00.0-usb-0:1.2.5.5'
    drivers:
      SerialDriver: {}
```

教程前面 `local.yaml` 用 `RawSerialPort` 只是为了让第一个例子最短，并不是两种文件
的语法差异；`USBSerialPort` + udev 匹配在本地 Environment 中同样可用。

最小流程：

```bash
# exporter 机器（连接硬件，-n 指定对外名称）
labgrid-exporter -c coordinator.example.com:20408 -n exporter1 /etc/labgrid/exporter.yaml

# 任意 client 机器
export LG_COORDINATOR=coordinator.example.com:20408
labgrid-client resources
labgrid-client -p bench-12 create
labgrid-client -p bench-12 add-match 'exporter1/bench-12/*'
labgrid-client -p bench-12 acquire
labgrid-client -p bench-12 show      # 确认 place 与匹配到的资源
labgrid-client -p bench-12 env       # 查看展开后的资源（仅 resources，无 drivers）
pytest --lg-env remote.yaml test_shell.py
labgrid-client -p bench-12 release
```

推荐手写 `remote.yaml`，用 `RemotePlace` 让资源随 place 动态展开：

```yaml
targets:
  main:
    resources:
      RemotePlace:
        name: bench-12
    drivers:
      SerialDriver: {}
      ShellDriver:
        prompt: 'root@[\w-]+:[^ ]+ '
        login_prompt: ' login: '
        username: 'root'
```

`RemotePlace` 通过 `LG_COORDINATOR`（或 `--lg-coordinator`）连接 coordinator；
也可用 `name: !template '$LG_PLACE'` 从环境变量取 place 名。另一种写法是把
`labgrid-client env` 打印的资源条目直接粘贴到 `resources:` 下（纯资源、不含
drivers），但不会跟随 place 后续变更。

Exporter 配置示例：

```yaml
bench-12:
  location: rack12
  USBSerialPort:
    match:
      '@ID_PATH': 'pci-0000:05:00.0-usb-0:1.2.5.5'
  NetworkPowerPort:
    model: netio
    host: netio12
    index: 3
```

前提：client 能免密 SSH 到 exporter（USB/文件操作必需）。

### 11.1 用 exporter/client 替代 `python run.py`

`labgrid-client` 不直接读取 `local.yaml`，它只与 coordinator 通信；硬件必须由
exporter 发布，driver 仍在 client 侧的 Environment 中。三者的分工：

| 组件 | 读取的配置 | 职责 |
|---|---|---|
| `labgrid-coordinator` | 无（运行时生成 `resources.yaml`/`places.yaml`） | 保存资源、place、锁 |
| `labgrid-exporter` | `exporter.yaml` | 连接硬件并向 coordinator 发布资源 |
| `labgrid-client` / pytest | `remote.yaml`（`RemotePlace` + drivers） | 申请 place、展开资源、运行 driver |

**1）exporter 配置**（只写资源，不写 drivers）：

```yaml
bench-local:
  location: desk
  USBSerialPort:
    match:
      '@ID_PATH': 'pci-0000:00:14.0-usb-0:3.2'
    speed: 115200
  RKUSBLoader:
    match:
      ID_PATH: 'pci-0000:00:14.0-usb-0:3.4'
```

**2）client 环境配置** `remote.yaml`（`RemotePlace` 会展开 place 中的资源）：

```yaml
targets:
  main:
    resources:
      RemotePlace:
        name: bench-local
    drivers:
      SerialDriver: {}
      ShellDriver:
        prompt: '\[root@\S+ [^\]]*\]# '
        login_prompt: ' login: '
        username: root
        password: ''
      RKUSBDriver:
        usb_loader: rk_loader
        image: rk_image

images:
  rk_loader: /abs/path/rk_loader.bin
  rk_image: /abs/path/rootfs.img
```

`images:` 的路径由 client 读取，再通过 exporter 同步到硬件所在机器；
`tools:` 中的 `rk-usb-loader` 路径必须是 **exporter 上**的路径。

**3）启动 coordinator**（在固定目录中运行，状态文件写在当前目录）：

```bash
mkdir -p ~/lg-coordinator && cd ~/lg-coordinator
labgrid-coordinator -l 0.0.0.0:20408
```

**4）启动 exporter**（连接硬件，`-n` 是导出名前缀，`--hostname` 是 client
访问该机器的地址）：

```bash
labgrid-exporter -c 127.0.0.1:20408 -n bench-pc --hostname 192.168.1.10 exporter.yaml
```

**5）client 创建 place 并匹配资源**：

```bash
export LG_COORDINATOR=127.0.0.1:20408
labgrid-client resources
labgrid-client -p bench-local create
labgrid-client -p bench-local add-match 'bench-pc/bench-local/*/*'
labgrid-client -p bench-local show
```

`show` 中每个资源都有 `avail` 字段；`avail: False` 说明硬件未就绪（例如
Rockchip 仍在 ADB 模式而不是 MaskROM）。

**6）使用**：

```bash
# 只锁 place + 直接连串口
labgrid-client -p bench-local acquire
labgrid-client -p bench-local console          # -l 断线重连，--logfile 记录
labgrid-client -p bench-local release

# 带 driver 的交互 / 测试
labgrid-client -c remote.yaml acquire
labgrid-client -c remote.yaml console
pytest --lg-env remote.yaml test_shell.py
labgrid-client -c remote.yaml release
```

`labgrid-client env` 只打印展开后的资源；driver 来自 `remote.yaml`，不会出现在
`env` 输出中。常用参数：

| 参数 | 含义 |
|---|---|
| `-x HOST:PORT` | coordinator 地址（也可用 `LG_COORDINATOR`） |
| `-p PLACE` | place 名（也可用 `LG_PLACE`） |
| `-c remote.yaml` | client 侧 Environment（含 drivers） |
| `resources` / `places` / `who` | 查看资源、place、占用者 |
| `acquire` / `release` | 加锁 / 解锁 place |

> 单机使用也仍需要 coordinator + exporter。如果不需要多用户共享、预约或远程
> 访问，直接 `python run.py` / `pytest --lg-env local.yaml` 更简单；工具链的价值
> 在于 place 互斥、资源集中管理和跨机器访问。

### 11.2 三者同机（单机）部署

coordinator、exporter、client 在同一台机器时，地址统一用 `127.0.0.1`：

```bash
mkdir -p ~/labgrid/state
cd ~/labgrid/state
labgrid-coordinator -l 127.0.0.1:20408
```

```bash
# 另一个终端
labgrid-exporter -c 127.0.0.1:20408 -n local \
  --hostname 127.0.0.1 ~/labgrid/exporter.yaml
```

`--hostname 127.0.0.1` 决定展开后的 `NetworkSerialPort.host`，漏掉它 client
可能拿到主机名或外部网卡地址。

exporter 配置沿用 11.1 的写法，只需把资源换成实际硬件；client 配置仍用
`RemotePlace`：

```yaml
targets:
  main:
    resources:
      RemotePlace:
        name: bench-local
    drivers:
      SerialDriver: {}
      ShellDriver:
        prompt: '\[root@\S+ [^\]]*\]# '
        login_prompt: ' login: '
        username: root
        password: ''
```

初始化 place（只需一次）：

```bash
export LG_COORDINATOR=127.0.0.1:20408
labgrid-client resources
labgrid-client -p bench-local create
labgrid-client -p bench-local add-match 'local/bench-local/*/*'
labgrid-client -p bench-local show
```

日常使用：

```bash
labgrid-client -c remote.yaml acquire
labgrid-client -c remote.yaml console
pytest --lg-env remote.yaml test_shell.py
labgrid-client -c remote.yaml release
```

单机最容易忽略的是 **USB/烧录资源仍需要本机免密 SSH**（串口走 ser2net
的 TCP，不需要 SSH）：

```bash
ssh -o BatchMode=yes 127.0.0.1 true
```

失败时用运行 client 的同一用户配置：

```bash
ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

SSH 命令以 client 当前用户登录本机，因此该用户还必须在 `dialout`/`plugdev`
组中。若不想配置 SSH，可让 exporter 只发布串口，把 `RKUSBLoader` 留在
`remote.yaml` 的 `resources:` 里本地使用；代价是 USB 不受 place 加锁保护。

systemd 托管、混合模式细节和完整配置见 `doc/deployment.md` 的 A7.1 节。

---

## 12. 调试与故障排查

| 现象 | 排查 |
|---|---|
| `NoDriverFoundError` | YAML 中资源/驱动名拼错，或缺少依赖资源；检查 `bindings` |
| 串口打不开 / Permission denied | 用户在 `dialout` 组并重新登录；`fuser /dev/ttyUSB0` 查占用 |
| 串口乱码 | 波特率不匹配；先用 `microcom -s <baud>` 验证 |
| ShellDriver 卡在登录 | `prompt`/`login_prompt` 正则不匹配；用 `--lg-log` + `tail -F` 看实际输出；检查 `bash` bracketed paste 前缀 |
| `Password entry needed but no password set` | 板子出现了 `Password:` 提示但 `ShellDriver` 没配 `password`；加上 `password: '...'`（空密码用 `''`），或先用 microcom 确认用户名/密码 |
| `python run.py` 长时间无输出 | 默认可能静默 60s（`login_timeout`）+ 30s（prompt 检查）；用 `timeout 150 python -u run.py`，加 `logging.basicConfig(level=DEBUG)`；关闭仍占用串口的 microcom（`fuser -v /dev/ttyUSB0`）；按登录后的实际提示符修正 `prompt`，注意 bash 5.1+ 的 bracketed paste 前缀 |
| udev 资源一直不可用 | `udevadm info` 核对属性；确认 `@` 前缀用于父设备；`labgrid-suggest` 生成配置 |
| 板子未上电 | 使用 Strategy 时必须有 PowerProtocol 驱动；ManualPowerDriver 需 `-s` |
| 网络串口超时 | `NetworkSerialPort.host/port` 是否可达；`protocol` 选 `raw` 还是 `rfc2217` |
| 远程 place 无资源 | match 表达式与 exporter/group 名不一致；`labgrid-client show` 查看 |
| 测试日志缺失 | 加 `--lg-log=DIR`；`-vvv` 打开调试 |
| 反复出现无法进入的状态 | Strategy 失败后会标记 broken；修复后重启 pytest 进程 |

常用排查命令：

```bash
udevadm info -q property -n /dev/ttyUSB0
labgrid-suggest                 # 然后插拔设备
pytest --lg-env local.yaml -vvv --capture=no
tail -F logs/console_main
```

---

## 13. 验收清单

- [ ] `labgrid-client --help` 正常
- [ ] `microcom` 能直接看到板子串口输出
- [ ] `local.yaml` 能被 `Environment` 加载，`ShellDriver` 激活成功
- [ ] Python API / pytest 能执行 `uname -a`
- [ ] 掉电重启后（使用 udev 匹配时）资源能自动重新出现
- [ ] Strategy 能从 `off` 迁移到 `shell`
- [ ] 分布式模式下 `acquire`/`console`/`release` 正常（可选）

---

## 14. 参考

- 概念：`doc/overview.rst`
- 配置参考：`doc/configuration.rst`（资源/驱动/udev 匹配/Strategy）
- 使用与 pytest：`doc/usage.rst`
- 示例：`examples/shell`、`examples/strategy`、`examples/remote`、`examples/usb`
- 分布式架构：`doc/architecture.md`
- 部署：`doc/deployment.md`
- 开发自定义驱动：`doc/development.rst`
