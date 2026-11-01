# labgrid 源码部署方案（Ubuntu 22.04 / Win10）

本文档给出从源码部署 labgrid 的具体执行方案，覆盖 Ubuntu 22.04 原生部署与
Windows 10 环境。本文件为本地执行手册，未纳入版本控制。

> 本文内容基于当前源码树验证：`pyproject.toml`（Python 版本与依赖）、
> `doc/getting_started.rst`（安装流程）、`doc/development.rst`（开发安装）、
> `contrib/systemd`（服务文件）。

## 平台支持结论

| 平台 | 结论 | 说明 |
|---|---|---|
| Ubuntu 22.04 | 原生源码部署，功能完整 | 需额外安装 Python 3.11+（系统自带 3.10） |
| Windows 10 | 通过 WSL2 中的 Ubuntu 22.04 部署 | 串口经 `usbipd-win` 透传为 `/dev/ttyUSB*` |
| Windows 10 | 通过 Docker Desktop 运行 client | 仅适合作为 client，容器内仍为 Linux 环境 |
| Windows 10 原生 Python | 不支持 | `labgrid.util.helper` 导入时依赖 `fcntl`/`pty`，且资源匹配依赖 `pyudev`/libudev |

两条硬约束：

1. **Python >= 3.11**：见 `pyproject.toml` 的 `requires-python`。
   Ubuntu 22.04 默认 Python 3.10.12 不满足要求。
2. **仅支持 Linux**：`labgrid/util/helper.py` 顶层 `import fcntl`、
   `import pty`；`labgrid/resource/udev.py` 依赖 `pyudev`。
   `pyproject.toml` 中 classifier 为 `Operating System :: POSIX :: Linux`。

---

## 方案 A：Ubuntu 22.04 原生部署

适用于 coordinator、exporter 与 client 同机或分机部署。

### A1. 安装系统依赖

```bash
sudo apt update
sudo apt install -y git build-essential pkg-config libudev-dev \
    ser2net microcom openssh-client rsync
```

说明：

- `ser2net`：exporter 导出串口所需。
- `microcom`/`telnet`：client 交互式 console 所需。
- `rsync`：ManagedFile 文件上传所需。

### A2. 安装 Python 3.11

方式一：deadsnakes PPA（推荐，系统级 Python 3.11）

```bash
sudo add-apt-repository -y ppa:deadsnakes/ppa
sudo apt update
sudo apt install -y python3.11 python3.11-venv python3.11-dev
```

方式二：uv（免 PPA，自带独立 CPython）

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.local/bin/env
uv python install 3.11
```

### A3. 获取源码并创建虚拟环境

```bash
cd ~
git clone https://github.com/labgrid-project/labgrid.git
cd labgrid
git checkout v26.0.1        # 可选：固定到发布 tag

python3.11 -m venv ~/labgrid-venv
# 使用 uv 时：uv venv --python 3.11 ~/labgrid-venv
source ~/labgrid-venv/bin/activate
pip install --upgrade pip
```

> **警告**：不要使用 `git clone --depth 1`。版本号由 `setuptools_scm`
> 从 git tag 推导，浅克隆没有 tag 会导致版本检测失败。若只能浅克隆，
> 安装时指定版本：
>
> ```bash
> SETUPTOOLS_SCM_PRETEND_VERSION=26.0.1 pip install .
> ```

### A4. 安装 labgrid

```bash
# 生产部署（普通安装）
pip install .

# 开发/调试（可编辑安装 + 全部开发依赖）
pip install -e ".[dev]"
```

按需追加可选功能：

```bash
pip install ".[snmp,modbus,modbusrtu,mqtt,onewire,pyvisa,vxi11,docker,graph]"
```

### A5. 安装后自检

```bash
python -c "import labgrid; print(labgrid.__version__)"
labgrid-client --help
labgrid-coordinator --help
labgrid-exporter --help
```

### A6. 权限与系统配置（exporter 机器）

```bash
# 串口权限（重新登录后生效）
sudo usermod -a -G dialout,plugdev "$USER"

# ManagedFile 缓存目录
sudo mkdir -p /var/cache/labgrid
sudo chgrp labgrid /var/cache/labgrid
sudo chmod 2775 /var/cache/labgrid

# 或使用仓库自带配置
sudo cp contrib/systemd/tmpfiles.d/* /usr/lib/tmpfiles.d/
sudo cp contrib/systemd/sysusers.d/* /usr/lib/sysusers.d/
sudo systemd-sysusers
sudo systemd-tmpfiles --create

# coordinator 端口（如启用 ufw）
sudo ufw allow 20408/tcp
```

> **注意**：client 到 exporter 的免密 SSH 只用于需要「在 exporter 上执行命令」
> 的资源，例如 USB Loader / 烧录、文件传输、SSH 登录目标板。`NetworkSerialPort`
> 由 exporter 启动 `ser2net`，client 直接走 TCP（RFC2217/raw），**不依赖 SSH**。

### A7. 启动分布式环境

```bash
# 1) coordinator（状态库写入当前工作目录，建议固定目录）
mkdir -p ~/labgrid-state
cd ~/labgrid-state
labgrid-coordinator &

# 2) exporter
cat > ~/exporter.yaml <<'EOF'
example-group:
  location: mylab
  USBSerialPort:
    match:
      ID_SERIAL_SHORT: ID23421JLK
    speed: 115200
EOF
labgrid-exporter ~/exporter.yaml &

# 3) client
export LG_COORDINATOR=localhost:20408
labgrid-client resources
labgrid-client -p example-place create
labgrid-client -p example-place add-match */example-group/*
labgrid-client -p example-place acquire
labgrid-client -p example-place console
```

### A7.1 三者同机（单机）部署

coordinator、exporter、client 跑在同一台 Ubuntu 上时，地址统一用
`127.0.0.1`，并让 exporter 以本机用户身份运行。

目录建议：

```text
~/labgrid/
├── state/          # coordinator 工作目录（resources.yaml/places.yaml）
├── exporter.yaml
└── remote.yaml
```

`exporter.yaml`（只写资源，路径换成实际值）：

```yaml
bench-local:
  location: local
  USBSerialPort:
    match:
      '@ID_PATH': 'pci-0000:00:14.0-usb-0:3.2'
    speed: 115200
  RKUSBLoader:
    match:
      ID_PATH: 'pci-0000:00:14.0-usb-0:3.4'
```

`remote.yaml`（client 侧，含 drivers）：

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

不需要烧录时删掉 `RKUSBDriver` 和 `images:`。

**启动**：

```bash
# 终端 1：coordinator，只监听本机
mkdir -p ~/labgrid/state
cd ~/labgrid/state
labgrid-coordinator -l 127.0.0.1:20408

# 终端 2：exporter
labgrid-exporter -c 127.0.0.1:20408 -n local \
  --hostname 127.0.0.1 ~/labgrid/exporter.yaml
```

`--hostname 127.0.0.1` 很关键：它决定展开后的 `NetworkSerialPort.host`，
否则 client 可能拿到主机名而去连外部地址。

**初始化 place（只需一次）**：

```bash
export LG_COORDINATOR=127.0.0.1:20408
labgrid-client resources
labgrid-client -p bench-local create
labgrid-client -p bench-local add-match 'local/bench-local/*/*'
labgrid-client -p bench-local show
```

**日常使用**：

```bash
labgrid-client -c remote.yaml acquire
labgrid-client -c remote.yaml console
pytest --lg-env remote.yaml test_shell.py
labgrid-client -c remote.yaml release
```

**本机 SSH 前提**（只有 USB/烧录/文件传输需要）：

```bash
ssh -o BatchMode=yes 127.0.0.1 true
```

若失败，为运行 exporter/client 的同一个用户配置本机免密：

```bash
ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
ssh -o BatchMode=yes 127.0.0.1 true
```

如果 exporter 以 root 或 systemd 服务运行，需要配置**该用户**的
`authorized_keys`；否则 `RKUSBDriver` 等远端命令会报 SSH 认证失败。

**没有本机 SSH 时的替代方案**：exporter 只发布串口，USB 资源留在
`remote.yaml` 本地：

```yaml
targets:
  main:
    resources:
      RemotePlace:
        name: bench-local
      RKUSBLoader:
        match:
          ID_PATH: 'pci-0000:00:14.0-usb-0:3.4'
    drivers:
      SerialDriver: {}
      ShellDriver: {}
      RKUSBDriver:
        usb_loader: rk_loader
        image: rk_image
```

同时从 `exporter.yaml` 中删除 `RKUSBLoader`，避免 place 展开后出现重复资源。
这样串口走 ser2net、USB 走本地，都不需要 SSH；代价是 USB 资源不受 place
加锁保护，只适合单用户。

**systemd 托管**：沿用 A8 的 unit， coordinator 的 `ExecStart` 加
`-l 127.0.0.1:20408`，exporter 的 `ExecStart` 加
`--hostname 127.0.0.1`，并在 exporter 的 `Environment` 中设置：

```ini
[Service]
Environment="LG_COORDINATOR=127.0.0.1:20408"
```

### A8. systemd 托管（生产）

```bash
sudo cp contrib/systemd/labgrid-coordinator.service \
        contrib/systemd/labgrid-exporter.service /etc/systemd/system/
sudo cp contrib/systemd/sysusers.d/* /usr/lib/sysusers.d/
sudo cp contrib/systemd/tmpfiles.d/* /usr/lib/tmpfiles.d/
```

随后：

1. 修改 service 中的 `ExecStart=`，指向虚拟环境中的可执行文件，例如
   `/home/<user>/labgrid-venv/bin/labgrid-coordinator` 与
   `/home/<user>/labgrid-venv/bin/labgrid-exporter /etc/labgrid/exporter.yaml`。
2. 在 exporter service 的 `SupplementaryGroups` 中加入 `dialout`
   （视发行版可能为 `plugdev` 或 `tty`）。
3. 设置协调器地址：

   ```bash
   sudo systemctl edit labgrid-exporter.service
   ```

   ```ini
   [Service]
   Environment="LG_COORDINATOR=<coordinator-host>:20408"
   ```

4. 启用服务：

   ```bash
   sudo systemd-sysusers
   sudo systemctl daemon-reload
   sudo systemctl enable --now labgrid-coordinator labgrid-exporter
   ```

### A9. 验收清单

- `systemctl status labgrid-coordinator labgrid-exporter` 均为 active。
- `labgrid-client resources` 能看到 exporter 上报的资源。
- `labgrid-client -p <place> console` 能进入串口。
- 使用 `examples/shell` 跑通一次 pytest：

  ```bash
  pytest --lg-env local.yaml --capture=no test_shell.py
  ```

---

## 方案 B：Windows 10

### B1. WSL2 + Ubuntu 22.04（推荐）

Windows 侧（管理员 PowerShell）：

```powershell
wsl --install -d Ubuntu-22.04
wsl --update
```

要求：Win10 2004 及以上（建议 22H2），BIOS 已开启虚拟化。

WSL 内部：复用方案 A 的全部步骤，并注意：

- 源码必须 clone 在 WSL 自己的文件系统（如 `~/labgrid`），不要放在
  `/mnt/c/...`（性能差且受 CRLF 影响）。
- WSL2 中不存在 COM 端口，只有 `/dev/ttyUSB*`，配置中必须使用 Linux
  设备名。

USB 串口透传（usbipd-win），Windows 管理员 PowerShell：

```powershell
winget install --interactive --exact dorssel.usbipd-win
usbipd list
usbipd bind --busid 2-1
usbipd attach --wsl --busid 2-1
```

WSL 内部：

```bash
sudo apt install -y linux-tools-generic hwdata
sudo update-alternatives --install /usr/local/bin/usbip usbip \
    $(ls /usr/lib/linux-tools/*/usbip | tail -n1) 20
ls -l /dev/ttyUSB*
```

> **警告**：WSL2 为 NAT 网络。client 访问局域网 coordinator 没有问题；
> 但不要把 coordinator/exporter 部署在 Win10 的 WSL2 中供其他机器访问——
> Win10 不支持 mirrored networking，需要额外 `netsh portproxy` 且 WSL IP
> 会变化。正式环境应将 coordinator/exporter 部署在 Linux 主机上，
> Win10 仅作为 client。

### B2. Docker Desktop（client 场景）

Docker Desktop 启用 WSL2 后端后：

```bash
git clone https://github.com/labgrid-project/labgrid.git
cd labgrid

docker build --target labgrid-client -t labgrid/client:src \
    -f dockerfiles/Dockerfile .

docker run --rm -it \
    -e LG_COORDINATOR=<coordinator-host>:20408 \
    -v "$PWD:/work" -w /work \
    labgrid/client:src labgrid-client places
```

说明：

- 镜像内部为 Linux 环境，规避全部 Windows 兼容问题。
- 源码树缺少 git tag 时，构建时指定版本：`--build-arg VERSION=26.0.1`。
- 容器内运行 pytest 同样可行；访问本地串口仍需 usbipd 或 Linux 主机的
  `--device`。

### B3. 原生 Windows 不可行的原因

- `labgrid/util/helper.py` 顶层导入 `fcntl`、`pty`，Windows 无此模块，
  `import labgrid` 直接失败。
- `pyudev` 依赖 libudev，USB 串口资源依赖 udev 属性匹配，Windows 无对应机制。
- 项目 classifier 明确为 `Operating System :: POSIX :: Linux`，CI 仅覆盖 Linux。

---

## 常见问题

| 现象 | 原因与处理 |
|---|---|
| `requires a different Python: 3.10.x not in '>=3.11'` | 用 `python3.11 -m venv` 重建虚拟环境，确认 `pip -V` 指向 3.11 |
| `setuptools_scm` 报 `unable to detect version` | 完整 clone（含 tag），或设置 `SETUPTOOLS_SCM_PRETEND_VERSION` |
| 编译 grpcio 很慢或失败 | 升级 pip 以命中 manylinux wheel：`pip install -U pip` |
| exporter 资源列表为空 | 检查 `dialout` 组、`ser2net` 安装情况，以及 `ID_SERIAL_SHORT` 是否与 `udevadm info` 输出一致 |
| `labgrid-client console` 找不到程序 | client 机器安装 `microcom` 或 `telnet` |
| RemotePlace 相关操作失败 | client 到 exporter 的免密 SSH 未配置 |
| WSL 中 `/dev/ttyUSB0` 不存在 | 执行 `usbipd attach --wsl --busid <x-y>`；重新插拔后需重新 attach |
| WSL 中安装后命令找不到 | 确认已在同一 shell 会话中激活虚拟环境 |
