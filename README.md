# C3 硬件键鼠：Agent 首次接入

把本页链接交给 Agent，并说：**“按这份文档接入我的 C3 设备，加载设备中的 Skill，并检查蓝牙连接。”**

本页解决首次接入：电脑先按这里安装串口读取依赖，再从设备读取安装与使用 Skill。设备固件是后续操作方法的来源。本页不要求事先拥有 `hidctl` 或项目源码。

当前流程面向 **macOS + 原生 USB Serial/JTAG 的 ESP32-C3**，设备名 **C3 Serial HID**。Windows/Linux、其他 USB 转串口芯片未按此流程验证，不直接套用。设备应已刷入支持 `SKILL` 命令的固件。

## 给 Agent 的执行顺序

用户让你按本页接入时，执行依赖检查、Skill 读取和蓝牙状态检查。不要只复述步骤。此授权不包含刷机、删除配对记录或向应用发送测试键鼠输入。

### 1. 准备电脑端读取环境

先确认 macOS 和 Python 3 可用：

```sh
uname -s
python3 --version
```

若 Python 不可用，先通过本机现有软件包管理器或 [Python 官方安装包](https://www.python.org/downloads/macos/)安装 Python 3；遇到系统授权界面交由用户完成，不假定安装成功。

原生 C3 USB 在 macOS 上应能枚举为串口。先用数据线接入设备；以下 pySerial 是用户态通信库，不是 USB 系统驱动。不要为原生 C3 安装 CH340/CP210x 驱动。若始终没有设备，检查线缆、接口及 USB 枚举，不能通过尚不可用的串口安装驱动。[C3 USB 官方说明](https://docs.espressif.com/projects/esp-idf/en/stable/esp32c3/api-guides/usb-serial-jtag-console.html)

创建或复用专用环境，并验证依赖：

```sh
python3 -m venv "$HOME/.local/share/c3-hid/bootstrap-venv"
"$HOME/.local/share/c3-hid/bootstrap-venv/bin/python" -m pip install pyserial==3.5
"$HOME/.local/share/c3-hid/bootstrap-venv/bin/python" -m serial.tools.list_ports -v
```

依赖说明：[pySerial 官方安装文档](https://pyserial.readthedocs.io/en/latest/pyserial.html#installation)。已安装相同版本时可直接复用。

### 2. 从设备读取 Skill

在本次工作的临时目录中把以下代码保存为 `load_c3_skill.py`。脚本只发送 `PING`、`SKILL`、`STATUS`，不会移动、点击、打字或刷机。

```python
import argparse
import hashlib
import json
import secrets
import time
from pathlib import Path

import serial
from serial.tools import list_ports


def load(port=None):
    candidates = [p for p in list_ports.comports()
                  if (p.vid, p.pid) == (0x303A, 0x1001)]
    if port:
        candidates = [p for p in candidates if p.device == port]
    if len(candidates) != 1:
        raise RuntimeError("需要唯一的原生 C3 串口；检查连接或用 --port 明确指定目标")
    device = candidates[0]
    # macOS 上先设 DTR=true / RTS=false，再打开原生 USB，避免复位。
    s = serial.Serial(port=None, baudrate=115200, timeout=0.1,
                      write_timeout=1, exclusive=True)
    s.dtr, s.rts, s.port = True, False, device.device

    def ask(command):
        seq = str(secrets.randbelow(2147483646) + 1)
        s.write((seq + " " + command + "\n").encode("ascii"))
        deadline, pending = time.monotonic() + 3, bytearray()
        while time.monotonic() < deadline:
            pending.extend(s.read(s.in_waiting or 1))
            if len(pending) > 8192:
                raise RuntimeError("串口响应过长")
            while b"\n" in pending:
                line, _, rest = pending.partition(b"\n")
                pending = bytearray(rest)
                fields = line.decode("ascii", errors="replace").strip().split(" ", 2)
                if len(fields) < 2 or fields[0] != seq:
                    continue
                if fields[1] != "DONE":
                    raise RuntimeError("设备拒绝请求：" + line.decode("ascii", errors="replace"))
                return fields[2] if len(fields) == 3 else ""
        raise TimeoutError("串口超时；检查端口占用和设备固件")

    s.open()
    try:
        for attempt in range(3):
            try:
                pong = ask("PING")
                break
            except TimeoutError:
                if attempt == 2:
                    raise
        if pong != "PONG SERIALHID/1":
            raise RuntimeError("不是兼容的 C3 Serial HID 固件")
        data, identity = bytearray(), None
        while True:
            fields = ask("SKILL " + str(len(data))).split()
            if len(fields) != 5 or fields[0] != "SKILL":
                raise RuntimeError("固件不支持预期的 Skill 分块格式")
            total, offset, digest = int(fields[1]), int(fields[2]), fields[3]
            chunk = bytes.fromhex(fields[4])
            if not (0 < total <= 16384 and offset == len(data) and 0 < len(chunk) <= 192):
                raise RuntimeError("Skill 长度或偏移无效")
            if len(digest) != 64 or any(c not in "0123456789abcdef" for c in digest):
                raise RuntimeError("Skill 校验字段无效")
            if identity is None:
                identity = (total, digest)
            if identity != (total, digest):
                raise RuntimeError("读取中途文档发生变化")
            data.extend(chunk)
            if len(data) > total:
                raise RuntimeError("Skill 数据超出声明长度")
            if len(data) == total:
                break
        if hashlib.sha256(data).hexdigest() != digest:
            raise RuntimeError("Skill SHA256 校验失败")
        text = data.decode("utf-8")
        if not text.startswith("---\n") or "\nname: c3-hid\n" not in text.split("\n---", 1)[0]:
            raise RuntimeError("设备返回的不是预期的 c3-hid Skill")
        state = ask("STATUS").split()
        if not state or state[0] != "STATUS":
            raise RuntimeError("设备状态格式无效")
        status = dict(item.split("=", 1) for item in state[1:])
        return bytes(data), {"port": device.device, "bytes": len(data),
                             "sha256": digest, "status": status}
    finally:
        s.close()


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--port")
    parser.add_argument("--output", type=Path, required=True)
    args = parser.parse_args()
    data, result = load(args.port)
    destination = args.output.expanduser().absolute()
    destination.parent.mkdir(parents=True, exist_ok=True)
    # 新建文件；如果已有文件，保留它并让调用方选择新的输出位置。
    with destination.open("xb") as output:
        output.write(data)
    print(json.dumps({"ok": True, **result, "path": str(destination)}, ensure_ascii=False))


if __name__ == "__main__":
    main()
```

使用专用 Python 执行，输出路径选本次临时目录中的新文件：

```sh
"$HOME/.local/share/c3-hid/bootstrap-venv/bin/python" load_c3_skill.py \
  --output ./device-skill/c3-hid/SKILL.md
```

存在多块 C3 时先确认目标，再增加 `--port /dev/cu.usbmodem实际端口`，不要猜测。串口被其他程序占用时先提示用户关闭对应串口监视器，不随意终止进程。

如果设备对 `SKILL` 返回 `BAD_ARGUMENTS`，说明可能是旧固件；停止本流程并说明需要支持 Skill 读取的固件，不自行刷写。校验失败、错误设备或超时均不能当作接入成功。

### 3. 加载 Skill，检查蓝牙

完整阅读刚导出的 `SKILL.md`，按其中安装与使用流程继续。Skill 含可独立使用的 Python 串口客户端，因此没有项目源码也能继续。需要持久安装时按当前 Agent 的技能目录规则保存；已有同名文件先比较，不覆盖用户修改。

USB 只负责供电/命令通信，实际键鼠输入必须通过蓝牙：

1. 目标电脑开启蓝牙，在系统设置中配对并连接 **C3 Serial HID**。首次连接需要人工操作系统界面时，明确告诉用户设备名及步骤；不可假定自动配对成功。
2. 检查 `STATUS`：必须 `connected=1`、`encrypted=1`，键盘操作还需 `keyboard=1`，鼠标操作还需 `mouse=1`。USB 所接电脑和蓝牙目标电脑可能不同，应确认实际目标。
3. 未就绪时停止输入，按 Skill 处理。读取 Skill 本身不需要蓝牙连接。
4. 接入完成后报告串口、Skill 校验结果及蓝牙就绪状态，等待或继续用户原先明确要求的操作。不要向未知窗口发送“测试文字”或点击。

## 维护约定

本页是 GitHub 上的首次接入入口；设备中的 Skill 是当前固件对应的操作说明。本页只发布读取和安装流程，不要求下载整套工程或编译工具链。SHA256 用于检查传输完整性，不是设备身份认证；本流程用于用户自己的可信设备。
