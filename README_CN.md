<p align="center">
  中文 | <a href="README.md">English</a>
</p>

# ESP_ATMod+ (ESP8266 增强型 AT 固件)

[![License: LGPL-2.1](https://img.shields.io/badge/License-LGPL--2.1-blue.svg)](LICENSE)
[![PlatformIO](https://badges.registry.platformio.org/packages/espressif8266/board/esp01_1m.svg)](https://platformio.org/lib/show/11992/ESP_ATMod)
[![Arduino Version](https://img.shields.io/badge/Arduino-3.0%2B-green.svg)](https://github.com/esp8266/Arduino)
[![MQTT Support](https://img.shields.io/badge/MQTT-3.1.1-orange.svg)](https://mqtt.org/)

**支持现代 TLS 1.2 + MQTT 3.1.1 的增强型 ESP8266 AT 固件**

本固件基于 [Arduino ESP8266](https://github.com/esp8266/Arduino#arduino-on-esp8266) 框架开发。

**版本：** 0.6.0 (含 MQTT 支持)

---

## 目录

- [项目目的](#项目目的)
- [功能特性](#功能特性)
- [安装说明](#安装说明)
- [AT 指令列表](#at-指令列表)
- [MQTT 指令](#mqtt-at-commands) ⭐ 新增
- [使用示例](#完整示例)
- [注意事项](#注意事项)

---

## 项目目的

乐鑫官方提供的 AT 固件仅包含基本的 TLS 密码套件。特别是缺少基于 GCM 的密码套件，导致 SSL 功能在越来越多的网站上无法使用。本固件解决了这个问题。

**目标是在 ESP8266 Arduino 项目中的 BearSSL 库中启用所有现代密码套件，包括服务器身份验证（服务器证书检查）。**

**此外，此增强版本还新增了用于 IoT 消息传递的 MQTT 3.1.1 协议支持，这是原始 ESP AT 固件所没有的。**

本固件可适配 1024 KB Flash，甚至可以在 ESP-01 模块（8 Mbit Flash）上运行。

## 功能特性

### TLS 安全特性（原有）
- ✅ 现代 TLS 1.2 密码套件（包括 GCM）
- ✅ 证书指纹验证
- ✅ 证书链验证
- ✅ TLS MFLN 扩展支持（RFC 3546）

### MQTT 协议（新增 ⭐）
- ✅ 通过 [PubSubClient](https://github.com/knolleary/PubSubClient) 实现 MQTT 3.1.1 协议
- ✅ 用户名/密码认证
- ✅ 支持 QoS 0、1、2
- ✅ 最多 8 个同时订阅的主题
- ✅ 单条消息最大发布长度：2048 字节
- ✅ 重连后自动重新订阅

### 硬件兼容性
- **目标设备**：ESP8266（特别是 1MB Flash 的 ESP-01 模块）
- **连接数**：最多 5 个同时 TCP/TLS 连接（多路复用模式）
- **内存优化**：针对低内存设备进行了优化

## 项目状态

本固件仍在开发中。已在实际设备上测试运行，但可能存在与预期行为的偏差。

测试环境使用 [WifiEsp 库](https://github.com/bportaluri/WiFiEsp) 和更新的 [WiFiEspAT 库](https://github.com/jandrassy/WiFiEspAT)。

## 安装说明

有两种编译和烧录此固件的方案：

### 方案一：使用 Arduino IDE

首先需要安装 [Arduino IDE](https://www.arduino.cc/en/software) 和 [ESP8266 核心](https://github.com/esp8266/Arduino#installing-with-boards-manager)。然后将本仓库的所有源文件下载到名为 **ESP_ATMod** 的文件夹中，编译并上传到您的 ESP 模块。

烧录完成后，模块将在 RX 和 TX 引脚上以 115200 波特率、8 数据位、无校验位打开串口连接。您可以使用任何串口终端工具与模块通信。

**重要提示：** 从固件版本 0.4.0 开始，需要使用 ESP8266 Arduino Core 3.0+ 版本。

### 方案二：使用 PlatformIO

除了 Arduino IDE，还可以使用 PlatformIO 进行编译和烧录：

1. 安装 [PlatformIO](https://platformio.org/)
2. 确保设备处于烧录模式
3. 在终端中进入本仓库根目录，执行以下命令编译并上传：
   ```
   platformio run --target upload
   ```

已针对 ESP-01 Black 模块进行配置和测试。

## 添加 TLS 证书

证书通过 LittleFS 存储在 ESP 的文件系统中。添加证书的步骤如下：

**重要：证书必须为 .pem 格式。**

1. 将要添加的证书复制到 ESP_ATMod/data 目录
2. 安装 [LittleFS 文件系统上传插件](https://github.com/earlephilhower/arduino-esp8266littlefs-plugin#installation)
3. 选择菜单 工具 > ESP8266 LittleFS Data Upload。这将开始将文件上传到 ESP8266 Flash 文件系统。完成后，IDE 状态栏会显示 LittleFS Image Uploaded 消息。大文件系统可能需要几分钟。
4. 现在将 ESP_ATMod 程序上传到 ESP。
5. 上传的证书现已加载并可使用（可以使用 [AT+CIPSSLCERT](#atcipsslcert---加载查询或删除-tls-ca-证书) 命令查看）。
6. （可选）可以删除 data 目录中的 .gitkeep 文件。它仅用于在 git 中推送和拉取 data 目录。不删除也不会有问题。

## AT 指令列表

下表列出了支持的 AT 指令。注释中仅给出了本实现与乐鑫原始 AT 固件实现的差异。指令按照乐鑫文档实现，包括指令顺序。更多信息请参考[乐鑫官方文档](https://www.espressif.com/sites/default/files/documentation/4a-esp8266_at_instruction_set_en.pdf)。

带 _DEF 和 _CUR 后缀的 AT 指令（与标准 AT 固件一样）有一个不带 _DEF/CUR 的未文档化版本，用于向后兼容（以及向前兼容，因为 AT 2 不使用 _DEF/CUR）。不带 _DEF/CUR 的命令在查询时表现为 _CUR，在设置时表现为 _DEF（将参数存储到 Flash）。

| 指令 | 描述 |
| - | - |
| [**基础 AT 指令**](https://docs.espressif.com/projects/esp-at/en/latest/AT_Command_Set/Basic_AT_Commands.html#basic-at-commands) | |
| AT | 测试 AT 启动 |
| AT+RST | 重启模块 |
| AT+GMR | 查看版本信息 |
| ATE | 配置 AT 指令回显 |
| AT+RESTORE | 恢复出厂设置 |
| AT+UART_CUR | 当前 UART 配置，不保存到 Flash |
| AT+UART_DEF | 默认 UART 配置，保存到 Flash |
| AT+SYSRAM | 查询当前剩余堆大小和最小堆大小 |
| [**Wi-Fi AT 指令**](https://docs.espressif.com/projects/esp-at/en/latest/AT_Command_Set/Wi-Fi_AT_Commands.html#wi-fi-at-commandss) | |
| AT+CWMODE | 设置 Wi-Fi 模式（Station/SoftAP/Station+SoftAP）|
| AT+CWJAP_CUR | 连接 AP，未实现 &lt;pci_en&gt; 参数 |
| AT+CWJAP_DEF | 连接 AP，保存到 Flash。未实现 &lt;pci_en&gt; 参数 |
| AT+CWLAPOPT | 设置 AT+CWLAP 指令的配置 |
| AT+CWLAP | 列出可用的 AP |
| AT+CWQAP | 断开 AP 连接 |
| AT+CWSAP_CUR | 开启 SoftAP，&lt;ecn&gt; 参数未使用。如果 &lt;pwd&gt; 不为空则使用 WPA_WPA2_PSK |
| AT+CWSAP_DEF | 连接 AP，保存到 Flash。&lt;ecn&gt; 参数未使用。如果 &lt;pwd&gt; 不为空则使用 WPA_WPA2_PSK |
| AT+CWDHCP_CUR | 启用/禁用 DHCP - SoftAP DHCP 服务器启用未实现 |
| AT+CWDHCP_DEF | 启用/禁用 DHCP 并保存到 Flash - SoftAP DHCP 服务器启用未实现 |
| AT+CWAUTOCONN | 上电时自动连接 AP |
| AT+CIPSTAMAC_CUR | 设置或打印 ESP8266 Station 的 MAC 地址。仅实现了查询功能 |
| AT+CIPSTAMAC_DEF | 设置或打印存储在 Flash 中的 ESP8266 Station MAC 地址。仅实现了查询功能 |
| AT+CIPAPMAC_CUR | 设置或打印 ESP8266 SoftAP 的 MAC 地址。仅实现了查询功能 |
| AT+CIPAPMAC_DEF | 设置或打印存储在 Flash 中的 ESP8266 SoftAP MAC 地址。仅实现了查询功能 |
| AT+CIPSTA_CUR | 查询/设置 ESP Station 的 IP 地址 |
| AT+CIPSTA_DEF | 设置并/或打印当前 IP 地址、网关和网络掩码，保存到 Flash |
| AT+CIPAP_CUR | 查询/设置 SoftAP 的当前 IP 地址 |
| AT+CIPAP_DEF | 设置并/或打印 SoftAP IP 地址、网关和网络掩码，保存到 Flash |
| AT+CWHOSTNAME | 查询/设置 ESP Station 的主机名 |
| [**TCP/IP AT 指令**](https://docs.espressif.com/projects/esp-at/en/latest/AT_Command_Set/TCP-IP_AT_Commands.html) | |
| AT+CIPSTATUS | 获取 TCP/UDP/SSL 连接状态和信息 |
| AT+CIPDOMAIN | 解析域名 |
| AT+CIPSTART | 建立 TCP 连接或 SSL 连接。同一时间只能有一个 TLS 连接 |
| [AT+CIPSSLSIZE](#atcipsslsize---设置-tls-接收缓冲区大小) | 更改接收缓冲区大小（512、1024、2048 或 4096 字节）|
| AT+CIPSEND | 在普通传输模式或 Wi-Fi 透传模式下发送数据 |
| AT+CIPCLOSEMODE | 设置 TCP 连接关闭模式 |
| AT+CIPCLOSE | 关闭 TCP/SSL 连接 |
| AT+CIFSR | 获取本地 IP 地址和 MAC 地址 |
| AT+CIPMUX | 启用/禁用多连接模式。最多 5 个连接，其中只有 1 个可以是 TLS |
| AT+CIPSNTPCFG | 查询/设置时区和 SNTP 服务器 |
| AT+CIPSNTPTIME | 查询 SNTP 时间 |
| AT+CIPDINFO | 设置 +IPD 消息模式 |
| [AT+CIPRECVMODE](#atciprecvmode-atciprecvdata-atciprecvlen-ssl-模式下) | 查询/设置 Socket 接收模式 |
| [AT+CIPRECVDATA](#atciprecvmode-atciprecvdata-atciprecvlen-ssl-模式下) | 在被动接收模式下获取 Socket 数据 |
| [AT+CIPRECVLEN](#atciprecvmode-atciprecvdata-atciprecvlen-ssl-模式下) | 在被动接收模式下获取 Socket 数据长度 |
| AT+CIPDNS_CUR | 查询/设置 DNS 服务器信息 |
| AT+CIPDNS_DEF | 默认 DNS 设置，保存到 Flash |
| [AT+CIPSERVER](#atcipserver-atcipservermaxconn-and-atcipsto) | 创建/删除 TCP 服务器 |
| AT+CIPSERVERMAXCONN | 设置服务器允许的最大连接数 |
| AT+CIPSTO | 设置 TCP 服务器超时时间 |
| **新增指令** | |
| [AT+SYSCPUFREQ](#atsyscpufreq---设置或查询当前-cpu-频率) | 设置或查询当前 CPU 频率 |
| [AT+RFMODE](#atrfmode---获取和更改物理-wifi-模式) | 设置物理 WiFi 模式 |
| [AT+CIPSSLAUTH](#atcipsslauth---设置和查询-tls-认证模式) | 设置和查询 TLS 认证模式 |
| [AT+CIPSSLFP](#atcipsslfp---加载或打印-tls-服务器证书-sha-1-指纹) | 加载或打印 TLS 服务器证书指纹 |
| [AT+CIPSSLCERTMAX](#atcipsslcertmax---查询或设置最大证书加载数量) | 查询或设置可加载的最大证书数量 |
| [AT+CIPSSLCERT](#atcipsslcert---加载查询或删除-tls-ca-证书) | 加载、查询或删除 TLS CA 证书 |
| [AT+CIPSSLMFLN](#atcipsslmfln---检查给定站点是否支持-mfln-tls-扩展) | 检查站点是否支持最大片段长度协商（MFLN）|
| [AT+CIPSSLSTA](#atcipsslsta---检查-mfln-协商的状态) | 打印连接的 MFLN 状态 |
| [AT+SNTPTIME](#atsystime---返回当前-utc-时间) | 获取 SNTP 时间 |
| [**MQTT AT 指令**](#mqtt-at-commands) ⭐ | |
| [AT+MQTTUSERCFG](#at-mqttusercfg--配置-mqtt-连接) | 配置 MQTT 连接参数（客户端 ID、用户名、密码、端口）|
| [AT+MQTTCONN](#at-mqttconn--连接或断开-mqtt-代理) | 连接或断开 MQTT 代理 |
| [AT+MQTTPUB](#at-mqttpub--发布消息) | 发布消息到主题 |
| [AT+MQTTSUB](#at-mqttsub--订阅主题) | 订阅主题 |
| [AT+MQTTUNSUB](#at-mqttunsub--取消订阅主题) | 取消订阅主题 |
| [**以太网 AT 指令**](https://docs.espressif.com/projects/esp-at/en/latest/esp32/AT_Command_Set/Ethernet_AT_Commands.html) | |
| AT+CIPETHMAC_CUR | 设置或打印以太网接口的 MAC 地址 |
| AT+CIPETHMAC_DEF | 设置或打印存储在 Flash 中的以太网接口 MAC 地址。保存到 Flash 未实现 |
| AT+CIPETH_CUR | 查询/设置以太网接口的 IP 地址 |
| AT+CIPETH_DEF | 设置并/或打印当前 IP 地址、网关和网络掩码，保存到 Flash |
| AT+CEHOSTNAME | 查询/设置以太网接口的主机名 |

---

## 修改过的指令

### **AT+CIPSSLSIZE - 设置 TLS 接收缓冲区大小**

设置 TLS 接收缓冲区大小。根据 [RFC3546](https://tools.ietf.org/html/rfc3546)，大小可以是 512、1024、2048、4096 或 16384（默认）字节。该值用于所有后续 TLS 连接，已打开的连接不受影响。

**命令：**
```
AT+CIPSSLSIZE=512
```

**响应：**

```
OK
```

### **AT+CIPRECVMODE, AT+CIPRECVDATA, AT+CIPRECVLEN 在 SSL 模式下**

以下命令：
- AT+CIPRECVMODE（设置 TCP 或 SSL 接收模式）
- AT+CIPRECVDATA（在被动接收模式下获取 TCP 或 SSL 数据）
- AT+CIPRECVLEN（在被动接收模式下获取 TCP 或 SSL 数据长度）

在 SSL 模式下的工作方式与 TCP 模式相同。

### **AT+CIPSERVER, AT+CIPSERVERMAXCONN 和 AT+CIPSTO**

标准 AT 固件只支持一个服务器。本固件支持使用相同的 AT+CIPCIPSERVER 命令创建多达 5 个服务器。

在标准 AT 固件 1.7 中，即使端口不同，再次执行 `AT+CIPSERVER=1,<port>` 会打印 no change 和 OK。这里它会启动一个新服务器。只有达到最大服务器数量时才返回 "no change"。

在标准 AT 固件 1.7 中，执行 AT+CIPSERVER=0 会停止唯一的服务器。这里它会停止第一个服务器。执行 `AT+CIPSERVER=0,<port>` 会停止监听 `<port>` 的服务器。

CIPSERVERMAXCONN 和 CIPSTO 是全局设置，适用于所有服务器。

### **AT+CWDHCP**

在标准 AT 固件中，AT_CWDHCP 为 STA 启用/禁用 DHCP 客户端（模式 0），并为 SoftAP 启动/停止 DHCP 服务器（模式 1）。在 ESP_ATMod 中，SoftAP DHCP 服务器始终启用。SoftAP 未实现 AT+CWDHCP 命令。

对于以太网支持，增强了 AT+CWDHCP 命令。以太网 DHCP 客户端通过模式 3 启用/禁用。对于 AT+CWDHAP?，第三位返回以太网 DHCP 客户端的状态。

---

## 新增指令

### **AT+SYSCPUFREQ - 设置或查询当前 CPU 频率**

设置和查询 CPU 频率。唯一有效的值是 80 和 160 MHz。

**查询：**

**命令：**
```
AT+SYSCPUFREQ?
```

**响应：**
```
+SYSCPUFREQ:80

OK
```

**设置：**

**命令：**
```
AT+SYSCPUFREQ=<freq>
```

**响应：**
```
OK
```

freq 的值可以是 80 或 160。

### **AT+RFMODE - 获取和更改物理 WiFi 模式**

设置和查询物理 WiFi 模式。

**查询：**

**命令：**
```
AT+RFMODE?
```

**响应：**
```
+RFMODE:1

OK
```

**设置：**

**命令：**
```
AT+RFMODE=<mode>
```

**响应：**
```
OK
```

&lt;mode&gt; 的允许值：

| 模式 | 描述 |
| - | - |
| 1 | IEEE 802.11b |
| 2 | IEEE 802.11g |
| 3 | IEEE 802.11n |

### **AT+CIPSSLAUTH - 设置和查询 TLS 认证模式**

设置或查询所选的 TLS 认证模式。默认为无认证。尽量避免这种情况，因为它不安全且容易受到中间人攻击。

**查询：**

**命令：**
```
AT+CIPSSLAUTH?
```

**响应：**
```
+CIPSSLAUTH=0

OK
```

**设置：**

**命令：**
```
AT+CIPSSLAUTH=<mode>
```

**响应：**
```
OK
```

&lt;mode&gt; 的允许值：

| 模式 | 描述 |
| - | - |
| 0 | 无认证。默认。不安全 |
| 1 | 服务器证书指纹验证 |
| 2 | 证书链验证 |

切换到模式 1 仅在预加载了证书 SHA-1 指纹时成功（参见 AT+CIPSSLFP）。

切换到模式 2 仅在预加载了 CA 证书时成功（参见 AT+CIPSSLCERT）。

### **AT+CIPSSLFP - 加载或打印 TLS 服务器证书 SHA-1 指纹**

加载或打印保存的服务器证书指纹。该指纹基于 SHA-1 哈希，正好 20 字节长。连接时，TLS 引擎会将收到的证书指纹与保存的值进行比较。确保设备连接到预期的服务器。成功连接后，指纹会被检查，此连接不再需要该指纹。

可以在浏览器中检查服务器证书时获取站点的 SHA-1 证书指纹。

**查询：**

**命令：**
```
AT+CIPSSLFP?
```

**响应：**
```
+CIPSSLFP:"4F:D5:B1:C9:B2:8C:CF:D2:D5:9C:84:5D:76:F6:F7:A1:D0:A2:FA:3D"

OK
```

**设置：**

**命令：**
```
AT+CIPSSLFP="4F:D5:B1:C9:B2:8C:CF:D2:D5:9C:84:5D:76:F6:F7:A1:D0:A2:FA:3D"
```

或者

```
AT+CIPSSLFP="4FD5B1C9B28CCFD2D59C845D76F6F7A1D0A2FA3D"
```

**响应：**
```
OK
```

指纹正好由 20 字节组成。它们以十六进制值设置，可以用 ':' 分隔。

### **AT+CIPSSLCERTMAX - 查询或设置最大证书加载数量**

目前一次最多可以加载 5 个证书。使用此命令可以调整通过 LittleFS 加载的证书数量。

**查询加载数量：**

**命令：**
```
AT+CIPSSLCERTMAX?
```

**响应：**
```
+CIPSSLCERTMAX:5
OK
```

**设置加载数量：**

**命令：**
```
AT+CIPSSLCERTMAX=6
```

**响应：**
```
+CIPSSLCERTMAX:6
OK
```

### **AT+CIPSSLCERT - 加载、查询或删除 TLS CA 证书**

加载、查询或删除用于 TLS 证书链验证的 CA 证书。目前一次最多可以加载 5 个证书。证书必须是 PEM 格式。成功连接后，证书会被检查，此连接不再需要该证书。

**查询第一个证书：**

**命令：**
```
AT+CIPSSLCERT?
```

**响应：**
```
+CIPSSLCERT:no cert

ERROR
```

或者

```
+CIPSSLCERT,1:DST Root CA X3
+CIPSSLCERT,2:DST Root CA X3

OK
```

**查询特定证书：**

**命令：**
```
AT+CIPSSLCERT?2
```

**响应：**
```
+CIPSSLCERT,2:DST Root CA X3

OK
```

**设置：**

**命令：**
```
AT+CIPSSLCERT
```

**响应：**
```
OK
>
```

现在可以发送证书（PEM 编码），不会给出回显。在最后一行（`-----END CERTIFICATE-----`）之后，证书会被解析和加载。证书应使用 \n 符号发送。例如 [isrg-root-x1-cross-signed.pem](https://letsencrypt.org/certs/isrg-root-x1-cross-signed.pem)：

```
-----BEGIN CERTIFICATE-----
MIIFYDCCBEigAwIBAgIQQAF3ITfU6UK47naqPGQKtzANBgkqhkiG9w0BAQsFADA/
MSQwIgYDVQQKExtEaWdpdGFsIFNpZ25hdHVyZSBUcnVzdCBDby4xFzAVBgNVBAMT
...（省略中间部分）...
MA0GCSqGSIb3DQEBCwUAA4IBAQAKcwBslm7/DlLQrt2M51oGrS+o44+/yQoDFVDC
-----END CERTIFICATE-----
```

应该这样发送：

```
-----BEGIN CERTIFICATE-----\nMIIFYDCCBEigAwIBAgIQQAF3ITfU6UK47naqPGQKtzANBgkqhkiG9w0BAQsFADA/\n...\n-----END CERTIFICATE-----
```

应用程序响应：

```
Read 1952 bytes

OK
```

或者返回错误消息。如果加载成功，证书就可以使用了，您可以开启证书检查（`AT+CIPSSLAUTH=2`）。

PEM 证书的总字符数限制为 4096 个。

**删除证书：**

**命令：**
```
AT+CIPSSLCERT=DELETE,1
```

**响应：**
```
+CIPSSLCERT,1:deleted

OK
```

证书已从内存中删除。

### **AT+CIPSSLMFLN - 检查给定站点是否支持 MFLN TLS 扩展**

最大片段长度协商扩展对于通过减少 TLS 连接上的接收缓冲区大小来降低 RAM 使用很有用。较新的 TLS 实现支持此扩展，但在更改 TLS 缓冲区大小并进行连接之前检查此功能是明智的。由于服务器不会即时更改此功能，因此只需测试一次 MFLN 功能。

**命令：**

AT+CIPSSLMFLN="*站点*",*端口*,*大小*

有效的大小为 512、1024、2048 和 4096。

```
AT+CIPSSLMFLN="www.github.com",443,512
```

**响应：**
```
+CIPSSLMFLN:TRUE

OK
```

### **AT+CIPSSLSTA - 检查 MFLN 协商的状态**

此命令检查打开的 TLS 连接上的 MFLN 状态。

**命令：**

AT+CIPSSLSTA[=linkID]

当多路复用开启时（AT+CIPMUX=1），*linkID* 值是必需的。多路复用关闭时不应输入。

```
AT+CIPSSLSTA=0
```

**响应：**
```
+CIPSSLSTA:1

OK
```

返回值 1 表示有 MFLN 协商。即使在设置了默认接收缓冲区大小的情况下也适用。

### **AT+SYSTIME - 返回当前 UTC 时间**

此命令以 Unix 时间形式返回当前时间（自 1970 年 1 月 1 日以来的秒数）。时区固定为 GMT (UTC)。时间是通过连接互联网后自动查询 NTP 服务器获得的。在连接互联网之前或与 NTP 服务器通信出错时，时间是未知的。这应该是暂时的。

**命令：**
```
AT+SYSTIME?
```

**响应：**
```
+SYSTIME:1607438042
OK
```

如果当前时间未知，将返回错误消息。

### **AT+CIPETH, AT+CIPETHMAC, AT+CEHOSTNAME**

这些指令像标准 AT 固件版本 2 及更新版本一样支持使用以太网接口。

这些以太网接口指令类似于 AT+CIPSTA、AT+CIPSTAMAC 和 AT+CWHOSTNAME 指令。

---

## MQTT AT Commands ⭐

本固件通过 [PubSubClient](https://github.com/knolleary/PubSubClient) 库支持 MQTT 3.1.1 协议。MQTT 允许 IoT 设备发布和订阅主题，实现轻量级消息传递。

### 功能特性

- **协议**：MQTT 3.1.1
- **认证**：支持用户名/密码
- **QoS**：支持 QoS 0、1、2
- **最大订阅数**：最多 8 个同时的主题订阅
- **最大发布长度**：每条消息最大 2048 字节
- **自动重订**：重连后自动重新订阅主题

### 使用流程

```
1. 连接 WiFi (AT+CWJAP)
2. 配置 MQTT 参数 (AT+MQTTUSERCFG)
3. 连接代理 (AT+MQTTCONN)
4. 订阅主题 (AT+MQTTSUB)
5. 发布消息 (AT+MQTTPUB)
6. 断开连接 (AT+MQTTCONN=0)
```

### **AT+MQTTUSERCFG - 配置 MQTT 连接**

配置 MQTT 连接参数，包括客户端 ID、用户名、密码和端口。必须在连接前调用。

**查询当前配置：**

**命令：**
```
AT+MQTTUSERCFG?
```

**响应：**
```
+MQTTUSERCFG:0,1,"client001","username","******",0,0,"",1883

OK
```

**设置配置：**

**命令：**
```
AT+MQTTUSERCFG=<LinkID>,<scheme>,<"client_id">,<"username">,<"password">,<cert_key_ID>,<CA_ID>,<path>,<port>
```

| 参数 | 描述 | 示例 |
| - | - | - |
| LinkID | 链路 ID（必须为 0）| 0 |
| scheme | 连接方式（1=TCP）| 1 |
| client_id | 客户端标识符字符串 | "ESP8266_001" |
| username | 认证用户名 | "user123" 或 "" |
| password | 认证密码 | "pass456" 或 "" |
| cert_key_ID | 证书密钥 ID（保留，使用 0）| 0 |
| CA_ID | CA 证书 ID（保留，使用 0）| 0 |
| path | 路径（保留，使用 ""）| "" |
| port | MQTT 代理端口号 | 1883 或 9501 |

**示例：**
```
AT+MQTTUSERCFG=0,1,"f9abbcca015846d69bc392a6c3187216","user","pass",0,0,"",1883
```

**响应：**

```
OK
```

### **AT+MQTTCONN - 连接或断开 MQTT 代理**

连接到 MQTT 代理或从其断开。

**查询连接状态：**

**命令：**
```
AT+MQTTCONN?
```

**响应：**
```
+MQTTCONN:1

OK
```
（返回 `1` 表示已连接，`0` 表示未连接）

**连接到代理：**

**命令：**
```
AT+MQTTCONN=<LinkID>,<host>,<port>
```

| 参数 | 描述 | 示例 |
| - | - | - |
| LinkID | 链路 ID（必须为 0）| 0 |
| host | 代理主机名或 IP 地址 | "broker.emqx.io" |
| port | 代理端口（可选，省略时使用配置中的端口）| 1883 |

**示例：**
```
AT+MQTTCONN=0,"bemfa.com",9501
```

**响应（成功）：**

```
CONNECT

OK
```

**响应（错误）：**
```
no ip           # WiFi 未连接
ERROR           # 连接失败
out of memory   # 内存分配失败
```

**从代理断开：**

**命令：**
```
AT+MQTTCONN=<LinkID>,0
```

**示例：**
```
AT+MQTTCONN=0,0
```

**响应：**
```
CLOSED

OK
```

### **AT+MQTTPUB - 发布消息**

使用数据模式（先发送长度，再发送数据）向主题发布消息。

**命令格式：**
```
AT+MQTTPUB=<LinkID>,<"topic">,<data_length>[,<qos>[,<retain>]]
```

| 参数 | 描述 | 范围 |
| - | - | - |
| LinkID | 链路 ID（必须为 0）| 0 |
| topic | 要发布的主题名称 | "sensor/data" |
| data_length | 数据长度（字节）| 1-2048 |
| qos | 服务质量等级（可选，默认 0）| 0, 1, 2 |
| retain | 保留标志（可选，默认 0）| 0, 1 |

**示例 - 发布 "hello world"（11 字节）：**

第 1 步：发送带数据长度的发布命令：
```
AT+MQTTPUB=0,"sensor01",11,1,0
```

**响应：**
```
OK
>
```

第 2 步：看到 `>` 提示后发送数据：
```
hello world
```

**响应：**
```
Recv 11 bytes
SEND OK
```

**错误响应：**
```
ERROR                   # 无效参数或未连接
ERROR: Not connected    # 未连接到 MQTT 代理
out of memory           # 缓冲区分配失败
SEND FAIL               # 发布操作失败
```

### **AT+MQTTSUB - 订阅主题**

订阅主题以接收在其上发布的消息。

**命令：**
```
AT+MQTTSUB=<LinkID>,<"topic">,<qos>
```

| 参数 | 描述 | 范围 |
| - | - | - |
| LinkID | 链路 ID（必须为 0）| 0 |
| topic | 要订阅的主题名称 | "sensor/#" |
| qos | 订阅的最大 QoS 等级 | 0, 1, 2 |

**示例：**
```
AT+MQTTSUB=0,"sensor01",1
```

**响应（成功）：**
```
+MQTTSUB:1

OK
```

**错误响应：**
```
ERROR                    # 无效参数
ERROR: Not connected     # 未连接到 MQTT 代理
max subscriptions reached # 达到订阅限制（最多 8 个）
subscribe failed         # 代理拒绝订阅
```

**接收订阅的消息：**

当在订阅的主题上收到消息时，固件会自动输出：

```
+MMTTSUBRECV:<topic>,<length>
<payload data>

OK
```

**示例：**
```
+MMTTSUBRECV:sensor01,11
hello world

OK
```

### **AT+MQTTUNSUB - 取消订阅主题**

取消对先前订阅的主题的订阅。

**命令：**
```
AT+MQTTUNSUB=<LinkID>,<"topic">
```

| 参数 | 描述 |
| - | - |
| LinkID | 链路 ID（必须为 0）|
| topic | 要取消订阅的主题名称 |

**示例：**
```
AT+MQTTUNSUB=0,"sensor01"
```

**响应（成功）：**
```
+MQTTUNSUB:1

OK
```

**错误响应：**
```
ERROR                    # 无效参数
ERROR: Not connected     # 未连接到 MQTT 代理
subscription not found   # 主题未被订阅
unsubscribe failed      # 代理拒绝取消订阅
```

## 完整示例

演示与 [Bemfa](https://bemfa.com) MQTT 平台通信的完整示例：

```bash
# 第 1 步：连接 WiFi
AT+CWJAP="MyWiFi","password"
# WIFI CONNECTED
# WIFI GOT IP
# OK

# 第 2 步：配置 MQTT（注意：最后必须有端口号！）
AT+MQTTUSERCFG=0,1,"your_client_id","your_username","",0,0,"",9501
# OK

# 第 3 步：连接到 MQTT 代理
AT+MQTTCONN=0,"bemfa.com",9501
# CONNECT
# OK

# 第 4 步：订阅主题
AT+MQTTSUB=0,"sensor01",0
# +MQTTSUB:1
# OK

# 第 5 步：发布消息（数据模式）
AT+MQTTPUB=0,"sensor01",5,0,0
# OK
> hello
# Recv 5 bytes
# SEND OK

# 第 6 步：在订阅主题上收到消息时（自动输出）：
# +MMTTSUBRECV:sensor01,12
# Hello World!!!
#
# OK

# 第 7 步：完成时断开连接
AT+MQTTCONN=0,0
# CLOSED
# OK
```

## 注意事项

1. **MQTTUSERCFG 中需要端口号**：`<port>` 参数必须作为 `AT+MQTTUSERCFG` 的最后一个参数包含在内。

2. **PUBLISH 仅支持数据模式**：`AT+MQTTPUB` 仅使用数据长度模式（不支持内联字符串）。必须先指定字节长度，然后在看到 `>` 提示后发送原始数据。

3. **连接顺序**：始终在连接（`MQTTCONN`）之前配置（`MQTTUSERCFG`），并在订阅/发布之前连接。

4. **单链接**：目前仅支持链路 ID `0`（单个 MQTT 连接）。

5. **内存限制**：在 RAM 有限的 ESP-01 上，请保持订阅数不超过 8 条，单条消息不超过 2048 字节。

---

## 许可证

本项目采用 [LGPL-2.1 许可证](LICENSE)。

## 致谢

感谢 [Jiri Bilek](https://github.com/JiriBilek) 创建的原始 ESP_ATMod 项目，以及 [knolleary](https://github.com/knolleary) 的 PubSubClient 库。

## 贡献

欢迎提交 Issue 和 Pull Request！如果您在使用过程中遇到问题或有改进建议，请在 GitHub 上提出。

---

<p align="center">
  <strong>如果这个项目对您有帮助，请给一个 ⭐ 支持！</strong>
</p>
