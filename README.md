# GFWList2AGH

将 GFWList 和中国域名列表转换为多种 DNS 服务器配置格式的工具。

## 📋 项目简介

GFWList2AGH 是一个自动化工具，用于从多个来源获取域名列表（GFWList、中国域名加速列表等），并将其转换为适用于不同 DNS 服务器的配置格式。该项目每日自动更新，确保域名列表始终保持最新。

### 主要特性

- 🔄 **每日自动更新**：通过 GitHub Actions 每天自动构建和发布
- 🌐 **多数据源支持**：整合来自 GFWList、Loyalsoldier、felixonmars 等多个权威来源
- 🛠️ **多格式输出**：支持 AdGuard Home、Bind9、DNSMasq、SmartDNS、Unbound 等主流 DNS 服务器
- 📝 **自定义规则**：支持通过配置文件自定义域名规则
- 🎯 **智能分流**：分离国内域名和国外域名，实现智能 DNS 分流
- 📦 **开箱即用**：生成的配置文件可直接导入使用

## 🚀 快速开始

### 方式一：直接使用生成的配置文件

#### AdGuard Home

```bash
# 下载黑名单（国外域名走代理）完整版
curl -O https://raw.githubusercontent.com/hezhijie0327/GFWList2AGH/main/gfwlist2adguardhome/blacklist_full.txt

# 下载白名单（国内域名走直连）完整版
curl -O https://raw.githubusercontent.com/hezhijie0327/GFWList2AGH/main/gfwlist2adguardhome/whitelist_full.txt
```

#### DNSMasq

```bash
# 下载黑名单配置
curl -O https://raw.githubusercontent.com/hezhijie0327/GFWList2AGH/main/gfwlist2dnsmasq/blacklist_full.conf

# 下载白名单配置
curl -O https://raw.githubusercontent.com/hezhijie0327/GFWList2AGH/main/gfwlist2dnsmasq/whitelist_full.conf
```

#### 其他 DNS 服务器

支持的格式包括：

- `gfwlist2bind9/` - Bind9 配置文件
- `gfwlist2smartdns/` - SmartDNS 配置文件
- `gfwlist2unbound/` - Unbound 配置文件
- `gfwlist2domain/` - 纯域名列表

### 方式二：本地构建

```bash
# 克隆仓库
git clone https://github.com/hezhijie0327/GFWList2AGH.git
cd GFWList2AGH

# 运行构建脚本
bash release.sh

# 生成的文件将保存在 gfwlist2* 目录中
```

## 📁 文件说明

每个 DNS 服务器目录包含以下文件：

### 文件类型

| 文件名                      | 说明                                             |
| --------------------------- | ------------------------------------------------ |
| `blacklist_full.{txt/conf}` | 黑名单完整版（包含所有子域名，约 3.4万 个域名）  |
| `blacklist_lite.{txt/conf}` | 黑名单精简版（仅顶级域名）                       |
| `whitelist_full.{txt/conf}` | 白名单完整版（包含所有子域名，约 11.7万 个域名） |
| `whitelist_lite.{txt/conf}` | 白名单精简版（仅顶级域名）                       |
| `blacklist_*_combine.txt`   | 单行合并格式（仅 AdGuard Home）                  |
| `whitelist_*_combine.txt`   | 单行合并格式（仅 AdGuard Home）                  |

### 列表说明

- **黑名单（Blacklist）**：GFW 列表，包含被屏蔽或需要代理访问的域名
- **白名单（Whitelist）**：中国域名列表，包含国内可直连的域名
- **完整版（Full）**：包含所有子域名，匹配精确但体积较大
- **精简版（Lite）**：仅包含顶级域名，覆盖范围广但可能误伤

## 🎯 使用场景

### AdGuard Home 配置示例

1. 进入 AdGuard Home 管理界面
2. 导航至 **设置** → **DNS 设置** → **上游 DNS 服务器**
3. 添加自定义 DNS 配置：

```ini
# 国外域名使用国外 DNS（黑名单）
[/google.com/youtube.com/facebook.com/]https://dns.opendns.com/dns-query

# 国内域名使用国内 DNS（白名单）
[/baidu.com/qq.com/taobao.com/]https://doh.pub/dns-query
```

4. 或直接导入生成的配置文件

### DNSMasq 配置示例

将下载的配置文件添加到 DNSMasq 配置目录：

```bash
# 复制配置文件
sudo cp blacklist_full.conf /etc/dnsmasq.d/gfwlist.conf
sudo cp whitelist_full.conf /etc/dnsmasq.d/chinalist.conf

# 重启 DNSMasq
sudo systemctl restart dnsmasq
```

### SmartDNS 配置示例

```bash
# 编辑 SmartDNS 配置
sudo nano /etc/smartdns/smartdns.conf

# 添加配置文件路径
conf-file /path/to/blacklist_full.conf
conf-file /path/to/whitelist_full.conf

# 重启 SmartDNS
sudo systemctl restart smartdns
```

## 🔧 自定义规则

您可以通过编辑 `data/data_modify.txt` 文件来自定义域名规则。

### 规则语法

| 标记    | 说明                                | 示例               |
| ------- | ----------------------------------- | ------------------ |
| `(@@@)` | 向 C（中国）及 G（GFW）列表添加域名 | `(@@@)example.org` |
| `(!!!)` | 从 C 及 G 列表移除域名              | `(!!!)example.org` |
| `(***)` | 从 C 及 G 列表排除域名（含子域名）  | `(***)example.org` |
| `(!**)` | 从 C 及 G 列表排除关键词            | `(!**)example`     |
| `(@%@)` | 仅向 C 列表添加域名                 | `(@%@)example.org` |
| `(!%!)` | 仅从 C 列表移除域名                 | `(!%!)example.org` |
| `(*%*)` | 仅从 C 列表排除域名                 | `(*%*)example.org` |
| `(!%*)` | 仅从 C 列表排除关键词               | `(!%*)example`     |
| `(@&@)` | 仅向 G 列表添加域名                 | `(@&@)example.org` |
| `(!&!)` | 仅从 G 列表移除域名                 | `(!&!)example.org` |
| `(*&*)` | 仅从 G 列表排除域名                 | `(*&*)example.org` |
| `(!&*)` | 仅从 G 列表排除关键词               | `(!&*)example`     |
| `(@%!)` | 向 C 列表添加并从 G 列表移除        | `(@%!)example.org` |
| `(!%@)` | 从 C 列表移除并向 G 列表添加        | `(!%@)example.org` |
| `(@&!)` | 向 G 列表添加并从 C 列表移除        | `(@&!)example.org` |
| `(!&@)` | 从 G 列表移除并向 C 列表添加        | `(!&@)example.org` |

### 规则优先级

```
移除(!) > 添加(@) > 排除(*)
```

### 示例

```bash
# 将 Apple 域名添加到中国列表（直连）
(@%!)apple.com
(@%!)icloud.com

# 将 GitHub 添加到 GFW 列表（代理）
(@&!)github.com
(@&!)githubusercontent.com

# 排除 .com.cn 域名
(*&*)com.cn
```

## 📄 许可证

本项目采用Apache License 2.0 with Commons Clause v1.0许可证 - 详见[LICENSE](LICENSE)文件
