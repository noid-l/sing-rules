# sing-rules

sing-box 规则集生成工具，基于 [Loyalsoldier/v2ray-rules-dat](https://github.com/Loyalsoldier/v2ray-rules-dat)。

将 v2ray geo 数据、Clash 规则列表转换为 sing-box rule-set 格式（`.json` + `.srs`），同时支持将 Clash/sing-box 订阅配置转换为 sing-box 完整出站配置。

## 技术栈

- **Python 3.14**：所有数据处理逻辑（使用项目内 `venv/` 虚拟环境）
- **Bash**：构建编排、外部工具调用
- **sing-box**：规则集编译（`rule-set compile/format`）、geo 导出、配置格式化/合并
- **geo**（MetaCubeX/geo，Go 编写）：v2ray geo 数据转换（`geo convert ip/site`）
- **GitHub Actions**：CI/CD 自动化构建与发布
- **sops**：secrets 加密管理

## Python 依赖

```
attrs, cattrs, pyyaml, requests, tldextract, typer
```

安装：`pip install -r requirements.txt`

## 项目结构

```
├── common/              # Python 公共库
│   ├── common.py        # Rule、merge、get_set、as_rule、domain_sort_key 等工具函数
│   ├── config.py        # sing-box 配置 dict 操作（remove_outbounds）
│   ├── io.py            # open_path（支持 - 标准输入/输出）
│   ├── object.py        # as_hashable、dedup、copy_without_tag
│   ├── outbound.py      # safe_find_country（IP 归属地检测）
│   └── yaml.py          # PyYAML 封装（优先 CLoader/CDumper）
├── config/              # 订阅配置与本地规则列表
│   ├── config.json      # 订阅源列表（ConfigFile 格式：name/cost/format/emby）
│   ├── subscribe.json   # 订阅下载配置（url_env/output/client/interval）
│   ├── secrets.env      # sops 加密的 secrets（订阅 URL、Token 等）
│   ├── *.list           # 本地 Clash 格式规则列表
│   ├── *.exclude        # 排除列表（用于 clash-merge.sh）
│   └── iphone/          # iPhone 专用配置片段（dns.json、outbounds.json 等）
├── bin/                 # 通用工具脚本
│   ├── install-github-binary.sh  # 从 GitHub Release 下载安装二进制
│   └── load-secrets.sh           # 解密 sops 加密的 dotenv 文件
├── rules/               # 输出目录：*.json（源码格式）+ *.srs（二进制格式）
├── preflight/           # 预处理缓存
│   ├── saved-countries.json      # 节点国家代码缓存
│   ├── *.sha1                    # 数据源变更检测摘要
│   └── v2ray-rules-dat.commit    # v2ray-rules-dat 版本记录
├── dat/                 # 下载的订阅文件（.gitignore，运行时生成）
├── geo/                 # MetaCubeX/geo 源码（.gitignore，运行时 clone）
├── private/             # Gitee 私有仓库（.gitignore，运行时 clone）
├── *.py                 # 主 Python 处理脚本
├── *.sh                 # 主 Bash 构建脚本
└── pyproject.toml       # PyRight 类型检查配置（仅 [tool.pyright]）
```

## 构建脚本

| 脚本 | 用途 |
|------|------|
| `build-all.sh` | **一站式本地构建脚本**：拉取更新 → 激活 venv → 解密 secrets → 下载订阅 → preflight → checkout geo → 构建配置/规则集 → 提交并推送 → 打 tag → Release |
| `build-sing-rules.sh` | 构建规则集：geo 转换 → 导出分类 → 下载 Clash 规则 → 合并 → 编译 `.srs` |
| `build-sing-config.sh <token>` | 构建 sing-box 完整配置（含出站、路由规则），需要 `private/` 目录 |
| `build-fakeip-filter.sh <url>` | 下载 fake_ip_filter.list，拆分 LAN/fakeip-bypass 并生成 JSON |
| `preflight.sh [configs...]` | 变更检测：对比 v2ray-rules-dat commit、bt-trackers sha1、订阅配置 sha1，决定是否触发 BUILD_RULES / BUILD_CONFIG |
| `cached-subscribe.sh` | 带缓存的订阅下载，依据 `config/subscribe.json` 配置 |
| `subscribe.sh <url> <output> [client]` | 下载订阅文件，保存流量信息到 `.info` |
| `clash-download.sh <list>` | 根据 `clash-list.txt` 批量下载 Clash 规则 |
| `clash-merge.sh [--enable-process] <dir>` | 将目录内 `.list` 文件合并到对应名称的 sing rule-set JSON |
| `commit-and-push.sh <message> [tag]` | 以 github-actions[bot] 身份提交并推送 |
| `npm-publish.sh <tag>` | 发布规则集到 npm（`@dkmoonfruit/sing-rules`）|
| `cleanup-old-tags.sh` | 删除 30 天前的日期格式 tag |

## Python 脚本

| 脚本 | 用途 |
|------|------|
| `geo-to-sing-rules.py` | 从 sing-box geo 数据库导出各分类规则集 JSON（geoip-cn、geosites-cn、ai、netflix 等） |
| `clash-to-sing-rules.py <list> <json> [-p]` | 将单个 Clash `.list` 文件合并到 sing rule-set JSON，支持 `--enable-process` 处理 PROCESS-NAME/PATH |
| `clash-to-sing.py` | 将 Clash/sing-box/Shadowrocket 订阅转换为 sing-box 出站+路由完整配置 |
| `filter-to-sing-rules.py` | 将 `fake_ip_filter.list` 格式转为 sing rule-set JSON（含通配符转正则） |
| `copy-config.py` | 从主配置提取精简版出站用于 iPhone 配置（移除 Block、高级节点等） |
| `split-outbounds.py` | 将 sing-box config 的 outbounds 拆分为 providers.json |
| `prune-config.py` | 大幅裁剪 sing-box 配置：仅保留白名单 rule_set，级联清理孤立 outbound |
| `fix-format.py` | 移除 sing-box outbound 中的零值 window 字段 |
| `locate.py` | 从 stdin 读取 outbound JSON，检测并输出国家代码 |

## 构建流程

### 规则集构建（`build-sing-rules.sh`）

1. 清理 `rules/` 下旧 `.json`/`.srs`
2. `geo convert ip/site` 将 v2ray geo 数据转为 sing-box 格式
3. `geo-to-sing-rules.py` 导出各分类规则 JSON（geosites-cn、direct、proxy、ai 等）
4. 下载 `clash-list.txt` 中的 Clash 规则列表
5. `clash-merge.sh` 合并 Clash 规则及 `config/` 中的本地 `.list` 规则
6. `build-fakeip-filter.sh` 生成 `lan.json`、`fakeip-bypass.json`
7. BT tracker 域名提取并合并到 `fakeip-bypass.json` 和 `direct.json`
8. `sing-box rule-set format/compile` 编译所有 `.json` 为 `.srs`
9. 若存在 `private/` 目录，同步 `rules/` 到其中

### 配置构建（`build-sing-config.sh`）

1. `clash-to-sing.py` 解析订阅 → 生成带出站和路由规则的 sing-box 配置
2. `sing-box format` 格式化配置
3. `fix-format.py` 清理零值字段 → 写入 `private/config.json`
4. `copy-config.py` 生成 iPhone 精简配置 → `sing-box merge` 生成 `private/config-iphone.json`
5. `prune-config.py` 裁剪为 Apple TV 版本 → `private/config-appletv.json`

## 代码规范

- **类型检查**：使用 PyRight（`pyproject.toml` 中配置 `typeCheckingMode = "standard"`）
- **Python 版本**：3.14，使用 match-case、type 别名等新特性
- **Shebang**：Python 脚本使用 `#!/usr/bin/env python`，Bash 脚本使用 `#!/usr/bin/env bash`
- **Bash 严格模式**：所有构建脚本以 `set -euo pipefail` 开头
- **注释与文档**：使用中文注释和 docstring
- **导入风格**：标准库 → 第三方库 → 本地 `common` 模块

## 关键数据格式

### Clash 规则列表（`.list`）

```
DOMAIN,example.com
DOMAIN-SUFFIX,example.com
DOMAIN-KEYWORD,example
IP-CIDR,1.2.3.0/24,no-resolve
PROCESS-NAME,example.exe
```

### sing-box rule-set JSON

```json
{
  "version": 1,
  "rules": [
    {
      "domain": ["example.com"],
      "domain_suffix": [".example.com"],
      "ip_cidr": ["1.2.3.0/24"]
    }
  ]
}
```

### 订阅配置（`config/config.json`）

```json
[
  {
    "name": "🌸 NanoCloud",
    "cost": 1.0,
    "path": "dat/nanocloud.json",
    "format": "sing-box"
  }
]
```

`format` 可选：`clash`、`shadowrocket`、`sing-box`。`emby` 字段用于 Emby 服务专用路由。

## 规则集命名约定

- `@CN` 后缀：国内版本（直连）
- `@!CN` 后缀：非国内版本（代理）
- `dev-cn.json`、`games-cn.json` 等：国内对应规则
- `*-proc.json`：含进程名规则（PROCESS-NAME/PATH）

## CI/CD（GitHub Actions）

- **文件**：`.github/workflows/build.yaml`
- **触发**：每日 0:00/12:00（cron）、main 分支 push、手动触发（workflow_dispatch）
- **流程**：
  1. 解密 secrets（SOPS_AGE_KEY）
  2. 缓存订阅下载
  3. Preflight 变更检测
  4. 按需构建配置和/或规则集
  5. 提交并推送主仓库 + private 仓库
  6. 清理旧标签
  7. GitHub Release（上传 `rules/*`）
  8. npm 发布（`@dkmoonfruit/sing-rules`）

## 安全与 Secrets 管理

- `config/secrets.env` 使用 **sops** 加密（`age` 密钥）
- `.sops.yaml` 定义加密规则
- 本地解密：`./bin/load-secrets.sh config/secrets.env`
- CI 中通过 `secrets.SOPS_AGE_KEY` 环境变量解密
- 订阅 URL 通过环境变量注入（`ASH_URL`、`NANOCLOUD_URL` 等），避免硬编码

## 开发环境设置

```bash
# 使用 direnv（.envrc 配置 layout python 3.14）
direnv allow

# 或手动激活 venv
source venv/bin/activate
pip install -r requirements.txt

# 安装外部依赖（本地开发需要）
# - sing-box: https://github.com/SagerNet/sing-box
# - geo (MetaCubeX): https://github.com/MetaCubeX/geo
# - sops: https://github.com/getsops/sops
```

## 常用操作

```bash
# 构建规则集（需要 dat/ 目录存在 v2ray geo 数据）
./build-sing-rules.sh

# 构建完整配置（需要 private/ 目录和 GITEE_TOKEN）
./build-sing-config.sh <gitee_token>

# 本地一站式构建（需要 secrets.env 和 private/ 目录）
./build-all.sh

# 下载订阅（带缓存）
./cached-subscribe.sh

# 检测节点国家
./locate.py <<< '{"type":"shadowsocks","server":"1.2.3.4",...}'
```

## 注意事项

- `rules/` 目录中的 `.srs` 文件被 `.gitignore` 忽略，只提交 `.json` 源码格式
- `dat/`、`geo/`、`private/`、`cache/` 均为运行时生成目录，不纳入版本控制
- 脚本中广泛使用临时文件和 `trap` 清理，避免残留
- `clash-to-sing.py` 中的国家检测依赖 `preflight/saved-countries.json` 缓存，首次运行可能较慢（需通过 sing-box 启动代理检测出口 IP）
