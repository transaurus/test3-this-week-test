---
title: Simulate Network Faults
---

本文介紹如何使用 Chaosd 模擬網路故障。這些模擬可透過修改網路路由與流量控制（使用 iptables、ipsets、tc 等工具）來完成。

:::note

請確保 Linux 核心已安裝 NET_SCH_NETEM 模組。若使用 CentOS，可透過 kernel-modules-extra 套件安裝。多數其他 Linux 發行版預設已安裝此模組。

:::

## 使用命令列模式建立網路故障實驗

本節介紹如何透過命令列模式建立網路故障實驗。

建立實驗前，可執行以下指令查看 Chaosd 支援的網路故障類型：

```bash
chaosd attack network --help
```

輸出如下：

```bash
Network attack related commands

Usage:
  chaosd attack network [command]

Available Commands:
  bandwidth   limit network bandwidth
  corrupt     corrupt network packet
  delay       delay network
  dns attack  DNS server or map specified host to specified IP
  duplicate   duplicate network packet
  loss        loss network packet
  partition   partition
  port        attack network port

Flags:
  -h, --help   help for network

Global Flags:
      --log-level string   the log level of chaosd, the value can be 'debug', 'info', 'warn' and 'error'

Use "chaosd attack network [command] --help" for more information about a command.
```

目前可使用 Chaosd 模擬四種實驗場景：網路損壞、網路延遲、網路重複傳輸及網路丟包。

### 使用命令列模式模擬網路損壞

可執行以下指令查看 Chaosd 模擬網路損壞的配置。

#### 網路損壞指令

指令如下：

```bash
chaosd attack network corrupt --help
```

輸出如下：

```bash
corrupt network packet

Usage:
  chaosd attack network corrupt [flags]

Flags:
  -c, --correlation string   correlation is percentage (10 is 10%) (default "0")
  -d, --device string        the network interface to impact
  -e, --egress-port string   only impact egress traffic to these destination ports, use a ',' to separate or to indicate the range, such as 80, 8001:8010. It can only be used in conjunction with -p tcp or -p udp
  -h, --help                 help for corrupt
  -H, --hostname string      only impact traffic to these hostnames
  -i, --ip string            only impact egress traffic to these IP addresses
      --percent string       percentage of packets to corrupt (10 is 10%) (default "1")
  -p, --protocol string      only impact traffic using this IP protocol, supported: tcp, udp, icmp, all
  -s, --source-port string   only impact egress traffic from these source ports, use a ',' to separate or to indicate the range, such as 80, 8001:8010. It can only be used in conjunction with -p tcp or -p udp

Global Flags:
      --log-level string   the log level of chaosd, the value can be 'debug', 'info', 'warn' and 'error'
```

#### 網路損壞相關配置項目

相關配置項目說明如下：

| Configuration item | Abbreviation | Description | Value |
| :-- | :-- | :-- | :-- |
| correlation | c | The correlation between the percentage of current corrupt occurrence and the previous occurrence. | int. It is a percentage ranging from 0 to 100 (10 is 10%) ("0" by default ). |
| device | d | Name of the impacted network interface card. | string, such as "eth0", required. |
| egress-port | e | The egress traffic that only impacts specific destination ports. It can only be configured when the protocol is TCP or UDP. | string. You need to use a `,` to separate the specific port or to indicate the range of the port, such as "80,8001:8010". |
| hostname | H | The host name impacted by traffic. `hostname` and `ip` cannot be empty at the same time. When `hostname` and `ip` are set at the same time, the configuration item affects both the specified `hostname` and `ip`. | string, such as "chaos-mesh.org". |
| ip | i | The IP address impacted by egress traffic. `hostname` and `ip` cannot be empty at the same time. When `hostname` and `ip` are set at the same time, the configuration item affects both the specified `hostname` and `ip`. | string, such as "123.123.123.123". |
| protocol | p | The IP protocol impacted by traffic. | string. Supported protocols: tcp, udp, icmp, all (all network protocols). |
| source-port | s | The egress traffic which only impact specific source ports. It can only be configured when the protocol is TCP or UDP. | string. Use a `,` to delimit the specific port or to indicate the range of the ports, such as "80,8001:8010". |
| percent |  | Ratio of network packet corruption. | string. It is a percentage which range is 0 to 100 (10 is 10%) (default "1"). |

#### 使用命令模式模擬網路損壞範例

執行以下指令模擬網路損壞：

```bash
chaosd attack network corrupt -d eth0 -i 172.16.4.4 --percent 50
```

若指令執行成功，輸出如下：

```bash
Attack network successfully, uid: 4eab1e62-8d60-45cb-ac85-3c17b8ac4825
```

### 使用命令列模式模擬網路延遲

可執行以下指令查看 Chaosd 模擬網路延遲的配置。

#### 網路延遲指令

指令如下：

```bash
chaosd attack network delay --help
```

輸出如下：

```bash
delay network

Usage:
  chaosd attack network delay [flags]

Flags:
  -c, --correlation string   correlation is percentage (10 is 10%) (default "0")
  -d, --device string        the network interface to impact
  -e, --egress-port string   only impact egress traffic to these destination ports, use a ',' to separate or to indicate the range, such as 80, 8001:8010. It can only be used in conjunction with -p tcp or -p udp
  -h, --help                 help for delay
  -H, --hostname string      only impact traffic to these hostnames
  -i, --ip string            only impact egress traffic to these IP addresses
  -j, --jitter string        jitter time, time units: ns, us (or µs), ms, s, m, h.
  -l, --latency string       delay egress time, time units: ns, us (or µs), ms, s, m, h.
  -p, --protocol string      only impact traffic using this IP protocol, supported: tcp, udp, icmp, all
  -s, --source-port string   only impact egress traffic from these source ports, use a ',' to separate or to indicate the range, such as 80, 8001:8010. It can only be used in conjunction with -p tcp or -p udp

Global Flags:
      --log-level string   the log level of chaosd, the value can be 'debug', 'info', 'warn' and 'error'
```

#### 網路延遲相關配置項目

相關配置項目說明如下：

| Configuration item | Abbreviation | Description | Value |
| :-- | :-- | :-- | :-- |
| correlation | c | The correlation between the current latency and the previous one. | string. It is a percentage ranging from 0 to 100 (10 is 10%) ("0" by default). |
| device | d | Name of the impacted network interface card. | string, such as "eth0", required. |
| egress-port | e | The egress traffic which only impact specific destination ports. It can only be configured when the protocol is TCP or UDP. | string. You need to use a `,` to separate the specific port or to indicate the range of the port, such as "80,8001:8010". |
| hostname | H | The host name impacted by traffic. `hostname` and `ip` cannot be empty at the same time. When `hostname` and `ip` are set at the same time, the configuration item affects both the specified `hostname` and `ip`. | string, such as "chaos-mesh.org". |
| ip | i | The IP address impacted by egress traffic. `hostname` and `ip` cannot be empty at the same time. When `hostname` and `ip` are set at the same time, the configuration item affects both the specified `hostname` and `ip`. | string, such as "123.123.123.123". |
| jitter | j | Range of the length of network delay time. | string. The time units can be: ns, us (µs), ms, s, m, h, such as "1ms". |
| latency | l | Length of network delay time. | string. The time units can be: ns, us (μs), ms, s, m, h, such as "1ms". |
| protocol | p | The IP protocol impacted by traffic. | string. It supports the following protocol types: "tcp", "udp", "icmp", "all" (all network protocols). |
| source-port | s | The egress traffic that only impacts specified source ports. It can only be configured when the protocol is TCP or UDP. | string. You need to use a `,` to separate the specific port or to indicate the range of the port, such as "80,8001:8010". |

#### 使用命令列模式模擬網路延遲範例

執行以下指令模擬網路延遲：

```bash
chaosd attack network delay -d eth0 -i 172.16.4.4 -l 10ms
```

若指令執行成功，輸出如下：

```bash
Attack network successfully, uid: 4b23a0b5-e193-4b27-90a7-3e04235f32ab
```

### 使用命令列模式模擬網路重複傳輸

可執行以下指令查看 Chaosd 模擬網路重複傳輸的配置。

#### 網路重複傳輸指令

指令如下：

```bash
chaosd attack network duplicate --help
```

輸出如下：

```bash
duplicate network packet

Usage:
  chaosd attack network duplicate [flags]

Flags:
  -c, --correlation string   correlation is percentage (10 is 10%) (default "0")
  -d, --device string        the network interface to impact
  -e, --egress-port string   only impact egress traffic to these destination ports, use a ',' to separate or to indicate the range, such as 80, 8001:8010. It can only be used in conjunction with -p tcp or -p udp
  -h, --help                 help for duplicate
  -H, --hostname string      only impact traffic to these hostnames
  -i, --ip string            only impact egress traffic to these IP addresses
      --percent string       percentage of packets to duplicate (10 is 10%) (default "1")
  -p, --protocol string      only impact traffic using this IP protocol, supported: tcp, udp, icmp, all
  -s, --source-port string   only impact egress traffic from these source ports, use a ',' to separate or to indicate the range, such as 80, 8001:8010. It can only be used in conjunction with -p tcp or -p udp

Global Flags:
      --log-level string   the log level of chaosd, the value can be 'debug', 'info', 'warn' and 'error'
```

#### 網路重複傳輸相關配置項目

相關配置項目說明如下：

| Configuration item | Abbreviation | Description | Value |
| :-- | :-- | :-- | :-- |
| correlation | c | The correlation between the percentage of current duplication occurrence and the previous one. | string. It is a percentage which range is 0 to 100 (10 is 10%) (default "0"). |
| device | d | Name of the impacted network interface card. | string, such as "eth0", required. |
| egress-port | e | The egress traffic that only impacts specified destination ports. It can only be configured when the protocol is TCP or UDP. | string. You need to use a `,` to separate the specific port or to indicate the range of the port, such as "80,8001:8010". |
| hostname | H | The host name impacted by traffic. `hostname` and `ip` cannot be empty at the same time. When `hostname` and `ip` are set at the same time, the configuration item affects both the specified `hostname` and `ip`. | string, such as "chaos-mesh.org". |
| ip | i | The IP address impacted by egress traffic. `hostname` and `ip` cannot be empty at the same time. When `hostname` and `ip` are set at the same time, the configuration item affects both the specified `hostname` and `ip`. | string, such as "123.123.123.123". |
| percent | N/A | Ratio of network packet duplicate. | string. It is a percentage which range is 0 to 100 (10 is 10%) (default "1"). |
| protocol | p | The IP protocol impacted by traffic. | string. It supports the following protocol types: "tcp", "udp", "icmp", "all" (all network protocols). |
| source-port | s | The egress traffic which only impact specific source ports. It can only be configured when the protocol is TCP or UDP. | string. You need to use a `,` to separate the specific port or to indicate the range of the port, such as "80,8001:8010". |

#### 使用命令列模式模擬網路重複傳輸範例

執行以下指令模擬網路重複傳輸：

```bash
chaosd attack network duplicate -d eth0 -i 172.16.4.4 --percent 50
```

若指令執行成功，輸出如下：

```bash
Attack network successfully, uid: 7bcb74ee-9101-4ae4-82f0-e44c8a7f113c
```

### 使用命令列模式模擬網路丟包

可執行以下指令查看 Chaosd 模擬網路丟包的配置：

#### 網路丟包指令

指令如下：

```bash
chaosd attack network loss --help
```

輸出結果如下：

```bash
loss network packet

Usage:
  chaosd attack network loss [flags]

Flags:
  -c, --correlation string   correlation is percentage (10 is 10%) (default "0")
  -d, --device string        the network interface to impact
  -e, --egress-port string   only impact egress traffic to these destination ports, use a ',' to separate or to indicate the range, such as 80, 8001:8010. It can only be used in conjunction with -p tcp or -p udp
  -h, --help                 help for loss
  -H, --hostname string      only impact traffic to these hostnames
  -i, --ip string            only impact egress traffic to these IP addresses
      --percent string       percentage of packets to drop (10 is 10%) (default "1")
  -p, --protocol string      only impact traffic using this IP protocol, supported: tcp, udp, icmp, all
  -s, --source-port string   only impact egress traffic from these source ports, use a ',' to separate or to indicate the range, such as 80, 8001:8010. It can only be used in conjunction with -p tcp or -p udp

Global Flags:
      --log-level string   the log level of chaosd, the value can be 'debug', 'info', 'warn' and 'error'
```

#### 網路丟包相關配置項

相關配置項說明如下：

| Configuration item | Abbreviation | Description | Value |
| :-- | :-- | :-- | :-- |
| correlation | c | The correlation between the percentage of the current network loss and the previous one. | string. It is a percentage which range is 0 to 100 (10 is 10%) (default "0"). |
| device | d | Name of the impacted network interface card. | string, such as "eth0", required. |
| egress-port | e | The egress traffic that only impacts specified destination ports. It can only be configured when the protocol is TCP or UDP. | string. You need to use a `,` to separate the specific port or to indicate the range of the port, such as "80,8001:8010". |
| hostname | H | The host name impacted by traffic. `hostname` and `ip` cannot be empty at the same time. When `hostname` and `ip` are set at the same time, the configuration item affects both the specified `hostname` and `ip`. | string, such as "chaos-mesh.org". |
| ip | i | The IP address impacted by egress traffic. `hostname` and `ip` cannot be empty at the same time. When `hostname` and `ip` are set at the same time, the configuration item affects both the specified `hostname` and `ip`. | string, such as "123.123.123.123". |
| percent | N/A | Ratio of network packet loss. | string. It is a percentage which range is 0 to 100 (10 is 10%) (default "1"). |
| protocol | p | Only impact traffic using this IP protocol. | string. It supports the following protocol types: "tcp", "udp", "icmp", "all" (all network protocols). |
| source-port | s | The egress traffic which only impact specific source ports. It can only be configured when the protocol is TCP or UDP. | string. You need to use a `,` to separate the specific port or to indicate the range of the port, such as "80,8001:8010". |

#### 命令列模式模擬網路丟包範例

執行以下指令模擬網路丟包：

```bash
chaosd attack network loss -d eth0 -i 172.16.4.4 --percent 50
```

若指令執行成功，輸出結果如下：

```bash
Attack network successfully, uid: 1e818adf-3942-4de4-949b-c8499f120265
```

### 命令列模式模擬網路分割

可執行以下指令查看 Chaosd 模擬網路分割的配置：

#### 網路分割指令

指令如下：

```bash
chaosd attack network partition --help
```

輸出結果如下：

```bash
partition

Usage:
  chaosd attack network partition [flags]

Flags:
      --accept-tcp-flags string   only the packet which match the tcp flag can be accepted, others will be dropped. only set when the protocol is tcp.
  -d, --device string             the network interface to impact
      --direction string          specifies the partition direction, values can be 'to', 'from' or 'both'. 'from' means packets coming from the 'IPAddress' or 'Hostname' and going to your server, 'to' means packets originating from your server and going to the 'IPAddress' or 'Hostname'. (default "both")
  -h, --help                      help for partition
  -H, --hostname string           only impact traffic to these hostnames
  -i, --ip string                 only impact egress traffic to these IP addresses
  -p, --protocol string           only impact traffic using this IP protocol, supported: tcp, udp, icmp, all

Global Flags:
      --log-level string   the log level of chaosd. The value can be 'debug', 'info', 'warn' and 'error'
      --uid string         the experiment ID
```

#### 網路分割相關配置項

相關配置項說明如下：

| Configuration item | Abbreviation | Description | Value |
| :-- | :-- | :-- | :-- |
| accept-tcp-flags | N/A | Only the packet which matches the tcp flag can be accepted, others will be dropped. Only set when the protocol is tcp. | string, such as "SYN,ACK SYN,ACK" |
| device | d | the network interface to impact | string, such as "eth0", required |
| direction | d | Specifies the partition direction, values can be 'to', 'from' or 'both'. 'from' means packets coming from the 'ip' or 'hostname' and going to your server, 'to' means packets originating from your server and going to the 'ip' or 'hostname'. | string, values can be "to", "from" or "both" (default "both") |
| hostname | H | Only impact traffic to these hostnames. `hostname` and `ip` cannot be empty at the same time. When `hostname` and `ip` are set at the same time, the configuration item affects both the specified `hostname` and `ip`. | string, such as "chaos-mesh.org". |
| ip | i | Only impact egress traffic to these IP addresses. `hostname` and `ip` cannot be empty at the same time. When `hostname` and `ip` are set at the same time, the configuration item affects both the specified `hostname` and `ip`. | string, such as "192.168.123.123". |
| protocol | p | Only impact traffic using this IP protocol | string. It supports the following protocol types: "tcp", "udp", "icmp", "all" (all network protocols). |

#### 命令列模式模擬網路分割範例

執行以下指令模擬網路分割：

```bash
chaosd attack network partition -i 172.16.4.4 -d eth0 --direction from
```

### 命令列模式模擬 DNS 故障

可執行以下指令查看 Chaosd 模擬 DNS 故障的配置：

#### DNS 故障指令

指令如下：

```bash
chaosd attack network dns --help
```

輸出結果如下：

```bash
attack DNS server or map specified host to specified IP

Usage:
  chaosd attack network dns [flags]

Flags:
  -d, --dns-domain-name string   map this host to specified IP
  -i, --dns-ip string            map specified host to this IP address
      --dns-server string        update the DNS server in /etc/resolv.conf with this value (default "123.123.123.123")
  -h, --help                     help for dns

Global Flags:
      --log-level string   the log level of chaosd. The value can be 'debug', 'info', 'warn' and 'error'
      --uid string         the experiment ID
```

#### DNS 故障相關配置項

相關配置項說明如下：

| Configuration item | Abbreviation | Description | Value |
| :-- | :-- | :-- | :-- |
| dns-domain-name | d | Map this host to specified IP(dns-ip) | string, such as "chaos-mesh.org". |
| dns-ip | i | Map specified host(dns-domain-name) to this IP address | string, such as "123.123.123.123" |
| dns-server | N/A | Update the DNS server in /etc/resolv.conf with this value | string, default is "123.123.123.123" |

#### 命令列模式模擬 DNS 故障範例

執行以下指令將指定主機映射至指定 IP 來模擬 DNS 故障：

```bash
chaosd attack network dns --dns-ip 123.123.123.123 --dns-domain-name chaos-mesh.org
```

執行以下指令使用錯誤 DNS 伺服器模擬 DNS 故障：

```bash
chaosd attack network dns --dns-server 123.123.123.123
```

### 命令列模式模擬網路頻寬

可執行以下指令查看 Chaosd 模擬網路頻寬的配置：

#### 網路頻寬指令

指令如下：

```bash
chaosd attack network bandwidth --help
```

輸出結果如下：

```bash
limit network bandwidth

Usage:
  chaosd attack network bandwidth [flags]

Flags:
  -b, --buffer uint32     the maximum amount of bytes that tokens can be available for instantaneously
  -d, --device string     the network interface to impact
  -h, --help              help for bandwidth
  -H, --hostname string   only impact traffic to these hostnames
  -i, --ip string         only impact egress traffic to these IP addresses
  -l, --limit uint32      the number of bytes that can be queued waiting for tokens to become available
  -m, --minburst uint32   specifies the size of the peakrate bucket
      --peakrate uint     the maximum depletion rate of the bucket
  -r, --rate string       the speed knob, allows bps, kbps, mbps, gbps, tbps unit. bps means bytes per second

Global Flags:
      --log-level string   the log level of chaosd. The value can be 'debug', 'info', 'warn' and 'error'
      --uid string         the experiment ID
```

#### 網路頻寬相關配置項

相關配置項說明如下：

| Configuration item | Abbreviation | Description | Value |
| :-- | :-- | :-- | :-- |
| buffer | b | The maximum amount of bytes that tokens can be available for instantaneously | int, such as `10000`, required |
| device | d | The network interface to impact | string, such as "eth0", required |
| hostname | H | Only impact traffic to these hostnames. `hostname` and `ip` cannot be empty at the same time. When `hostname` and `ip` are set at the same time, the configuration item affects both the specified `hostname` and `ip`. | string, such as "chaos-mesh.org". |
| ip | i | Only impact egress traffic to these IP addresses. `hostname` and `ip` cannot be empty at the same time. When `hostname` and `ip` are set at the same time, the configuration item affects both the specified `hostname` and `ip`. | string, such as "123.123.123.123". |
| limit | l | The number of bytes that can be queued waiting for tokens to become available | int, such as `10000`, required |
| minburst | `m` | Specifies the size of the peakrate bucket | int, such as `10000` |
| peakrate | N/A | The maximum depletion rate of the bucket | int, such as `10000` |
| rate | r | The speed knob, allows bps, kbps, mbps, gbps, tbps unit. The bps unit means bytes per second. | string, such as "1mbps", required |

#### 命令列模式模擬網路頻寬範例

執行以下指令模擬網路頻寬：

```bash
chaosd attack network bandwidth --buffer 10000 --device eth0 --limit 10000 --rate 10mbps
```

### 命令列模式模擬端口佔用

可執行以下指令查看模擬端口佔用的配置：

#### 端口佔用指令

指令如下：

```bash
chaosd attack network port --help
```

輸出結果如下：

```bash
attack network port

Usage:
  chaosd attack network port [flags]

Flags:
  -h, --help          help for port
  -p, --port string   this specified port is to occupied

Global Flags:
      --log-level string   the log level of chaosd. The value can be 'debug', 'info', 'warn' and 'error'
      --uid string         the experiment ID
```

#### 端口佔用相關配置項

相關配置項說明如下：

| Configuration item | Abbreviation | Description                       | Value                       |
| :----------------- | :----------- | :-------------------------------- | :-------------------------- |
| port               | p            | The specified port to be occupied | int, such as 8080, required |

#### 命令列模式模擬端口佔用範例

執行以下指令以模擬網路頻寬限制：

```bash
chaosd attack network port --port 8080
```

## 使用服務模式建立網路故障實驗

要使用服務模式建立實驗，請遵循以下說明：

1. 在服務模式下執行 Chaosd：

   ```bash
   chaosd server --port 31767
   ```

2. Send a `POST` HTTP request to the `/api/attack/process` path of the Chaosd service.

   ```bash
   curl -X POST 172.16.112.130:31767/api/attack/process -H "Content-Type:application/json" -d '{fault-configuration}'
   ```

   In the above command, you need to configure `fault-configuration` according to the fault types. For the corresponding parameters, refer to the parameters and examples of each fault type in the following sections.

:::note

When running an experiment, remember to record the UID of the experiment. When you want to end the experiment corresponding to the UID, you need to send a `DELETE` HTTP request to the `/api/attack/{uid}` path of the Chaosd service.

:::

### 使用服務模式模擬網路損壞

#### 模擬網路損壞的參數

| Parameter | Description | Value |
| --- | --- | --- |
| action | Action of the experiment. | set to "corrupt" |
| correlation | The correlation between the current latency and the previous one. | string. It is a percentage ranging from 0 to 100 (10 is 10%) ("0" by default). |
| device | Name of the impacted network interface card. | string, such as "eth0", required. |
| egress-port | The egress traffic which only impact specific destination ports. It can only be configured when the protocol is TCP or UDP. | string. You need to use a `,` to separate the specific port or to indicate the range of the port, such as "80,8001:8010". |
| hostname | The host name impacted by traffic. `hostname` and `ip-address` cannot be empty at the same time. when `hostname` and `ip-address` are set at the same time, the configuration item affects both the specified `hostname` and `ip-address`. | string, such as "chaos-mesh.org". |
| ip-address | The IP address impacted by egress traffic. `hostname` and `ip-address` cannot be empty at the same time. when `hostname` and `ip-address` are set at the same time, the configuration item affects both the specified `hostname` and `ip-address`. | string, such as "123.123.123.123". |
| ip-protocol | The IP protocol impacted by traffic. | string. Supported protocols: tcp, udp, icmp, all (all network protocols). |
| source-port | The egress traffic which only impact specific source ports. It can only be configured when the protocol is TCP or UDP. | string. Use a `,` to delimit the specific port or to indicate the range of the ports, such as "80,8001:8010". |
| percent | Ratio of network packet corruption. | string. It is a percentage which range is 0 to 100 (10 is 10%) (default "1"). |

#### 使用服務模式模擬網路損壞的範例

```bash
curl -X POST 172.16.112.130:31767/api/attack/network -H "Content-Type:application/json" -d '{"action":"corrupt","device":"eth0","ip-address":"172.16.4.4","percent":"50"}'
```

### 使用服務模式模擬網路延遲

#### 模擬網路延遲的參數

| Parameter | Description | Value |
| --- | --- | --- |
| action | Action of the experiment. | set to "delay" |
| correlation | The correlation between the current latency and the previous one. | string. It is a percentage ranging from 0 to 100 (10 is 10%) ("0" by default). |
| device | Name of the impacted network interface card. | string, such as "eth0", required. |
| egress-port | The egress traffic which only impact specific destination ports. It can only be configured when the protocol is TCP or UDP. | string. You need to use a `,` to separate the specific port or to indicate the range of the port, such as "80,8001:8010". |
| hostname | The host name impacted by traffic. `hostname` and `ip-address` cannot be empty at the same time. When `hostname` and `ip-address` are set at the same time, the configuration item affects both the specified `hostname` and `ip-address`. | string, such as "chaos-mesh.org". |
| ip-address | The IP address impacted by egress traffic. `hostname` and `ip-address` cannot be empty at the same time. When `hostname` and `ip-address` are set at the same time, the configuration item affects both the specified `hostname` and `ip-address`. | string, such as "123.123.123.123". |
| jitter | Range of the length of network delay time. | string. The time units can be: ns, us (µs), ms, s, m, h, such as "1ms". |
| latency | Length of network delay time. | string. The time units can be: ns, us (μs), ms, s, m, h, such as "1ms". |
| ip-protocol | The IP protocol impacted by traffic. | string. It supports the following protocol types: "tcp", "udp", "icmp", "all" (all network protocols). |
| source-port | The egress traffic that only impacts specified source ports. It can only be configured when the protocol is TCP or UDP. | string. You need to use a `,` to separate the specific port or to indicate the range of the port, such as "80,8001:8010". |

#### 使用服務模式模擬網路延遲的範例

```bash
curl -X POST 172.16.112.130:31767/api/attack/network -H "Content-Type:application/json" -d '{"action":"delay","device":"eth0","ip-address":"172.16.4.4","latency":"10ms"}'
```

### 使用服務模式模擬網路封包重複

#### 模擬網路封包重複的參數

| Parameter | Description | Value |
| --- | --- | --- |
| action | Action of the experiment. | set to "duplicate" |
| correlation | The correlation between the percentage of current duplication occurrence and the previous one. | string. It is a percentage which range is 0 to 100 (10 is 10%) (default "0"). |
| device | Name of the impacted network interface card. | string, such as "eth0", required. |
| egress-port | The egress traffic that only impacts specified destination ports. It can only be configured when the protocol is TCP or UDP. | string. You need to use a `,` to separate the specific port or to indicate the range of the port, such as "80,8001:8010". |
| hostname | The host name impacted by traffic. `hostname` and `ip-address` cannot be empty at the same time. when `hostname` and `ip-address` are set at the same time, the configuration item affects both the specified `hostname` and `ip-address`. | string, such as "chaos-mesh.org". |
| ip-address | The IP address impacted by egress traffic. `hostname` and `ip-address` cannot be empty at the same time. when `hostname` and `ip-address` are set at the same time, the configuration item affects both the specified `hostname` and `ip-address`. | string, such as "123.123.123.123". |
| percent | Ratio of network packet duplicate. | string. It is a percentage which range is 0 to 100 (10 is 10%) (default "1"). |
| ip-protocol | The IP protocol impacted by traffic. | string. It supports the following protocol types: "tcp", "udp", "icmp", "all" (all network protocols). |
| source-port | The egress traffic which only impact specific source ports. It can only be configured when the protocol is TCP or UDP. | string. You need to use a `,` to separate the specific port or to indicate the range of the port, such as "80,8001:8010". |

#### 使用服務模式模擬網路封包重複的範例

```bash
curl -X POST 172.16.112.130:31767/api/attack/network -H "Content-Type:application/json" -d '{"action":"duplicate","ip-address":"172.16.4.4","device":"eth0","percent":"50"}'
```

### 使用服務模式模擬網路丟包

#### 模擬網路丟包的參數

| Parameter | Description | Value |
| --- | --- | --- |
| action | Action of the experiment. | set to "loss" |
| correlation | The correlation between the percentage of the current network loss and the previous one. | string, it is a percentage which range is 0 to 100 (10 is 10%) (default "0"). |
| device | Name of the impacted network interface card. | string, such as "eth0", required. |
| egress-port | The egress traffic that only impacts specified destination ports. It can only be configured when the protocol is TCP or UDP. | string. You need to use a `,` to separate the specific port or to indicate the range of the port, such as "80,8001:8010". |
| hostname | The host name impacted by traffic. `hostname` and `ip-address` cannot be empty at the same time. when `hostname` and `ip-address` are set at the same time, the configuration item affects both the specified `hostname` and `ip-address`. | string, such as "chaos-mesh.org". |
| ip-address | The IP address impacted by egress traffic. `hostname` and `ip-address` cannot be empty at the same time. when `hostname` and `ip-address` are set at the same time, the configuration item affects both the specified `hostname` and `ip-address`. | string, such as "123.123.123.123". |
| percent | Ratio of network packet loss. | string. It is a percentage which range is 0 to 100 (10 is 10%) (default "1"). |
| ip-protocol | Only impact traffic using this IP protocol. | string, it supports the following protocol types: "tcp", "udp", "icmp", "all" (all network protocols). |
| source-port | The egress traffic which only impact specific source ports. It can only be configured when the protocol is TCP or UDP. | string. You need to use a `,` to separate the specific port or to indicate the range of the port, such as "80,8001:8010". |

#### 使用服務模式模擬網路丟包的範例

```bash
curl -X POST 172.16.112.130:31767/api/attack/network -H "Content-Type:application/json" -d '{"action":"loss","ip-address":"172.16.4.4","device":"eth0","percent":"50"}'
```

### 使用服務模式模擬網路分割

#### 模擬網路分割的參數

| Parameter | Description | Value |
| --- | --- | --- |
| action | Action of the experiment. | set to "partition" |
| accept-tcp-flags | Only the packet which match the tcp flag can be accepted, others will be dropped. Only set when the protocol is tcp. | string, such as "SYN,ACK SYN,ACK" |
| device | The network interface to impact | string, such as "eth0", required |
| direction | Specifies the partition direction, values can be 'to', 'from' or 'both'. 'from' means packets coming from the 'ip-address' or 'hostname' and going to your server, 'to' means packets originating from your server and going to the 'ip-address' or 'hostname'. | string, values can be "to", "from" or "both" (default "both") |
| hostname | Only impact traffic to these hostnames. `hostname` and `ip-address` cannot be empty at the same time. when `hostname` and `ip-address` are set at the same time, the configuration item affects both the specified `hostname` and `ip-address`. | string, such as "chaos-mesh.org". |
| ip-address | Only impact egress traffic to these IP addresses. `hostname` and `ip-address` cannot be empty at the same time. when `hostname` and `ip-address` are set at the same time, the configuration item affects both the specified `hostname` and `ip-address`. | string, such as "192.168.123.123". |
| ip-protocol | Only impact traffic using this IP protocol | string. It supports the following protocol types: tcp, udp, icmp, all (all network protocols). |

#### 使用服務模式模擬網路分割的範例

```bash
curl -X POST 172.16.112.130:31767/api/attack/network -H "Content-Type:application/json" -d '{"action":"partition","ip-address":"172.16.4.4","device":"eth0","direction":"from"}'
```

### 使用服務模式模擬 DNS 故障

#### 模擬 DNS 故障的參數

| Parameter | Description | Value |
| --- | --- | --- |
| action | Action of the experiment. | set to "dns" |
| dns-domain-name | Map this host to specified IP(dns-ip) | string, such as "chaos-mesh.org". |
| dns-ip | Map specified host(dns-domain-name) to this IP address | string, such as "123.123.123.123" |
| dns-server | Update the DNS server in /etc/resolv.conf with this value | string, such as "123.123.123.123" (default "123.123.123.123") |

#### 使用服務模式模擬 DNS 故障的範例

```bash
curl -X POST 172.16.112.130:31767/api/attack/network -H "Content-Type:application/json" -d '{"action":"dns","dns-domain-name":"chaos-mesh.org","dns-ip":"123.123.123.123"}'
```

### 使用服務模式模擬網路頻寬限制

#### 模擬網路頻寬限制的參數

| Parameter | Description | Value |
| --- | --- | --- |
| action | Action of the experiment. | set to "bandwidth" |
| buffer | The maximum amount of bytes that tokens can be available for instantaneously | int, such as `10000`, required |
| device | The network interface to impact | string, such as "eth0", required |
| hostname | Only impact traffic to these hostnames. `hostname` and `ip-address` cannot be empty at the same time. when `hostname` and `ip-address` are set at the same time, the configuration item affects both the specified `hostname` and `ip-address`. | string, such as "chaos-mesh.org". |
| ip-address | Only impact egress traffic to these IP addresses. `hostname` and `ip-address` cannot be empty at the same time. When `hostname` and `ip-address` are set at the same time, the configuration item affects both the specified `hostname` and `ip-address`. | string, such as "123.123.123.123". |
| limit | The number of bytes that can be queued waiting for tokens to become available | int, such as `10000`, required |
| minburst | Specifies the size of the peakrate bucket | int, such as `10000` |
| peakrate | The maximum depletion rate of the bucket | int, such as `10000` |
| rate | The speed knob, allows bps, kbps, mbps, gbps, tbps unit. The bps unit means bytes per second. | string, such as "1mbps", required |

#### 使用服務模式模擬網路頻寬限制的範例

```bash
curl -X POST 172.16.112.130:31767/api/attack/network -H "Content-Type:application/json" -d '{"action":"bandwidth","buffer":10000,"limit":10000,"rate":"10mbps","device":"eth0"}'
```

### 使用服務模式模擬端口佔用

#### 模擬端口佔用的參數

| Parameter | Description                        | Value                       |
| --------- | ---------------------------------- | --------------------------- |
| action    | Action of the experiment.          | set to "occupied"           |
| port      | The specified port to be occupied. | int, such as 8080, required |

#### 使用服務模式模擬端口佔用的範例

```bash
curl -X POST 172.16.112.130:31767/api/attack/network -H "Content-Type:application/json" -d '{"action":"occupied","port":8080}'
```