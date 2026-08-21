# silas

指纹 + POC 一体化扫描器(web 应用专用)。

- 六维指纹引擎:标题 / body / header / icon_hash / cert / banner
- 三类 POC 映射:direct / tag / general + AST DSL 求值
- POC 库 88860 条,内存峰值 ≤ 100MB(LRU 分片)
- 13 阶段端到端流程:子域 -> 资产预识别 -> 指纹 -> POC -> 综合评分 -> 深度探测 -> 风险评级报告

## 兼容性

| 平台 | 状态 | 备注 |
|---|---|---|
| Kali Linux | 原生 | apt 一行装齐所有上游工具 |
| Ubuntu 20.04+ | 原生 | subfinder 走 go install |
| CentOS 8 / RHEL 8+ | 可用 | 需 EPEL,部分工具走 Go 源码 |
| Windows 10/11 | 可用 | 原生 Python + 工具二进制下载 |
| WSL2 | 等同 Linux | 走 Ubuntu 章节即可 |
| macOS | 可用 | brew + go install |

Python ≥ 3.9。POC 引擎需要 `requests`,指纹 mmh3 需要 gcc(Python C 扩展)。

## 环境搭建

### 0. 通用步骤(所有平台)

```bash
git clone <silas-repo> silas && cd silas
python3 -m venv venv
source venv/bin/activate           # Windows: venv\Scripts\activate
pip install --upgrade pip
pip install -r requirements.txt
pip install dnspython              # 深度流程 IP 反查 + CDN 识别必装
```

requirements.txt 包含:

| 包 | 用途 | C 扩展 |
|---|---|---|
| PyYAML | 配置/POC 解析 | 否 |
| requests | HTTP 抓取 | 否 |
| mmh3 | favicon hash 指纹 | **是**(需 gcc) |
| rich | 终端 UI | 否 |
| lxml | 部分 XML 解析 | **是**(需 libxml2-dev) |
| packaging | 版本约束 | 否 |
| dnspython | DNS 反查 + CDN CNAME 识别 | 否 |

C 扩展装不上时,先装编译工具链再重试。

---

### 1. Kali Linux

Kali 自带 subfinder / nmap / masscan / ffuf / dirsearch / httpx-toolkit。

```bash
# 1. 系统更新 + 编译工具链
sudo apt update && sudo apt install -y \
    python3 python3-pip python3-venv \
    python3-dev gcc git \
    libxml2-dev libxslt1-dev libffi-dev

# 2. 上游工具(一行装齐)
sudo apt install -y subfinder nmap masscan ffuf dirsearch httpx-toolkit

# 3. Go(若 subfinder apt 版本旧,可选)
sudo apt install -y golang

# 4. venv + requirements
cd /opt/silas
python3 -m venv venv && source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
pip install dnspython
```

**OneForAll**(子域枚举备选):

```bash
cd Tools
git clone https://github.com/shmilylty/OneForAll.git
cd OneForAll
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
pip install --upgrade fire
deactivate
cd ../..
```

验证:

```bash
python3 silas.py --check-tools
# 7 个工具全部 ✓ 即就绪
```

---

### 2. Ubuntu 20.04 / 22.04 / 24.04

```bash
# 1. 系统更新 + 编译工具链 + 基础工具
sudo apt update && sudo apt install -y \
    python3 python3-pip python3-venv \
    python3-dev build-essential git \
    libxml2-dev libxslt1-dev libffi-dev \
    nmap masscan ffuf dirsearch

# 2. Go(subfinder/httpx 需要)
sudo apt install -y golang-go
# 或装最新版:
# wget https://go.dev/dl/go1.22.0.linux-amd64.tar.gz
# sudo tar -C /usr/local -xzf go1.22.0.linux-amd64.tar.gz
# export PATH=$PATH:/usr/local/go/bin

# 3. subfinder + httpx(Go 源)
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest
echo 'export PATH=$PATH:$HOME/go/bin' >> ~/.bashrc
source ~/.bashrc

# 4. venv + requirements
cd /opt/silas
python3 -m venv venv && source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
pip install dnspython
```

**注意**:Ubuntu 上 projectdiscovery httpx 二进制叫 `httpx`(不是 `httpx-toolkit`)。
bin_manager 自动用 ELF 头校验,跳过 Python 同名包。

**OneForAll**:见 Kali 章节。

---

### 3. CentOS 8 / RHEL 8 / Rocky Linux 8+

```bash
# 1. 启用 EPEL + PowerTools
sudo dnf install -y epel-release
sudo dnf config-manager --set-enabled powertools  # Rocky/Alma: crb

# 2. 编译工具链 + 基础工具
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y \
    python3 python3-pip python3-devel \
    libxml2-devel libxslt-devel libffi-devel \
    git nmap

# 3. Go(系统版本可能旧,推荐手动装)
sudo dnf install -y golang

# 4. masscan(EPEL 可能没有,走源码)
sudo dnf install -y git libpcap-devel
cd /tmp && git clone https://github.com/robertdavidgraham/masscan.git
cd masscan && make -j$(nproc)
sudo make install
cd ..

# 5. ffuf + subfinder + httpx(Go 源)
go install -v github.com/ffuf/ffuf/v2@latest
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest
echo 'export PATH=$PATH:$HOME/go/bin' >> ~/.bashrc
source ~/.bashrc

# 6. dirsearch
sudo pip3 install dirsearch

# 7. venv + requirements
cd /opt/silas
python3 -m venv venv && source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
pip install dnspython
```

CentOS 7 默认 Python 3.6,**不满足 ≥3.9**:

```bash
sudo dnf install -y python39 python39-devel
python3.9 -m venv venv
source venv/bin/activate
```

---

#### 

```powershell
python silas.py --check-tools
```

---

### 

## 依赖(快速参考)

各平台一行装齐上游工具:

| 平台 | 命令 |
|---|---|
| Kali / Ubuntu | `sudo apt install -y subfinder nmap masscan ffuf dirsearch httpx-toolkit` |
| Ubuntu(无 subfinder) | `sudo apt install -y nmap masscan ffuf dirsearch && go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest` |
| CentOS / RHEL | `sudo dnf install -y nmap && go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest github.com/projectdiscovery/httpx/cmd/httpx@latest github.com/ffuf/ffuf/v2@latest && pip3 install dirsearch && (masscan 源码编译)` |
| Windows | 各工具 GitHub Release 下载二进制 + `pip install dirsearch` |

Python 依赖:

```bash
pip install -r requirements.txt
pip install dnspython
```

工具识别说明:
- `httpx-toolkit` 在 Kali 上是 projectdiscovery httpx 的 Go 二进制
- `/usr/bin/httpx` 是 Python 同名包(NodeJS httpx,silas 不用它)
- bin_manager 自动用 ELF 头校验,优先 `httpx-toolkit`
- OneForAll 目录名允许 `OneForAll/` 或 `OneForAll-master/`

检查工具是否齐:

```bash
python3 silas.py --check-tools
```

## 快速开始

```bash
# 单目标全流程(命中 < 阈值询问是否进深度)
python3 silas.py -w -d example.com

# 单目标仅指纹
python3 silas.py -t http://target -F

# 单目标仅 POC(指纹静默运行用于映射)
python3 silas.py -t http://target --poc-only

# 批量(每行一个 URL)
python3 silas.py -l targets.txt -o result.json --format json

# 严重度过滤(只跑 critical + high)
python3 silas.py -l targets.txt -G critical,high
```

输出格式 `text`(默认)/ `json` / `html`。

## 命令总览

```
目标:      -t / -u / -l
模式:      -F 仅指纹   --poc-only 仅POC
并发:      -T threads  -W timeout
输出:      -o file  -O text|json|html
路径:      --db-dir  --pocs-dir  --config-dir
引擎:      --audit / -A 关审计(默认开)

工作流(单字符互斥):
  -w                               全流程(主域名->资产预识别->子域->指纹->POC->决策->深度->最终报告)
  -s                               子域名阶段
  -f                               指纹阶段
  -p                               POC 阶段
  -d <domain>                      主域名(-w / -s 必填,仅 domain)
  --tools-dir Tools                上游工具源码目录
  --work-dir silas_workspace       工作目录
  -k                               保留中间文件(默认清理)

深度探测控制:
  --deep-threshold N               命中数 < N 触发询问(默认 10)
  --deep-auto                      命中 < 阈值自动进深度(CI 友好)
  --deep-yes                       非交互模式选是(等价 --deep-auto)
  --deep-no                        非交互模式选否
  --no-deep                        完全禁用深度分支

端口扫描调优:
  --port-top N                     nmap/masscan Top-N 端口(默认 100)
  --port-scan-tool nmap|masscan    端口扫描工具(默认 nmap)
  --port-rate N                    masscan 速率(默认 1000)

目录扫描调优:
  --dir-tool dirsearch|ffuf        目录扫描工具(默认 dirsearch)
  --dir-wordlist PATH              目录字典(默认 dirb common.txt)
  --dir-threads N                  目录扫描并发(默认 10)

管理(不扫描):
  -M / --sync-pocs                 扫 config/pocs_inbox/ 反写指纹入库
  --sync-pocs-from DIR             指定其他收件箱
  --sync-pocs-dry-run              只打印计划不写库
  -C / --check-tools               检查上游工具
```

## 工作流编排

`-w` 是端到端入口。从域名到 POC 利用一条龙,综合评分低时自动进深度:

```bash
# 全流程(综合分 < 30 自动进深度,>= 70 跳过,30-70 询问)
python3 silas.py -w -d example.com

# 等价改造前 -w(不进深度分支)
python3 silas.py -w -d example.com --no-deep

# CI/批量:免询问,综合分 < 阈值自动进深度
python3 silas.py -w -d example.com --deep-auto

# 自定义阈值 + 端口/目录扫描调优
python3 silas.py -w -d example.com --deep-auto \
    --deep-threshold 5 \
    --port-scan-tool masscan --port-rate 500 \
    --dir-tool ffuf --dir-threads 20
```

### 

### 

### 单阶段执行

各阶段独立可用(从已有产物继续):

```bash
# 子域(需 -d domain)
python3 silas.py -s -d example.com

# 指纹(需 -l 子域列表)
python3 silas.py -f -l silas_workspace/subdomains.txt

# POC(需 -l URL 列表)
python3 silas.py -p -l silas_workspace/urls.txt

# 资产预识别(需 -d domain,从已有 subdomains.txt 继续)
python3 silas.py -w -d example.com --no-deep
```

`-d` 强制校验为 domain(拒 IP/URL),防止 subfinder 产 0 结果后误读陈旧 fixture
导致"扫 A 实际扫 B"。

## POC 反写指纹

把 POC 收件箱(`config/pocs_inbox/`)下的 yaml 批量扫,从 fofa-query / shodan-query
和 fingerprint 块反向解析出指纹规则,入 `db/fingerprints.json` + 建 mapping:

```bash
# 干跑看计划
python3 silas.py -M --sync-pocs-dry-run

# 实际写库
python3 silas.py -M

# 指定其他收件箱
python3 silas.py -M --sync-pocs-from /path/to/pocs
```

反写指纹写入 `config/fingerprints/reversed_fingerprints.json`,下次启动自动合并入主指纹库。

## 用户配置目录

`config/` 是你自己的配置区,与 `db/` / `pocs/` 物理隔离,运行时三层合并:

| 情况 | 行为 |
|---|---|
| 同 id / 同键 | 用户配置覆盖内置 |
| 新 id / 新键 | 追加 |
| 列表字段 | 去重 union |
| 文件缺失 | 跳过,不影响其他文件 |

```
config/
├── fingerprints/
│   ├── custom_fingerprints.json   # 自定义指纹(覆盖/追加到 db/fingerprints.json)
│   ├── active_probes.json         # 主动探测子路径(按 app_name 分组)
│   └── reversed_fingerprints.json # POC 反写指纹(--sync-pocs 自动生成)
├── workflows/
│   ├── direct_mappings.json       # 指纹名 -> POC 直接映射
│   ├── tag_mappings.json          # app_name -> POC 标签兜底
│   └── general_pocs.json          # 无指纹兜底 POC(Shiro/Log4j2 等)
├── pocs/
│   ├── custom_pocs_index.json     # POC 索引(必须与 custom_pocs.json 同 id)
│   └── custom_pocs.json           # POC 内容(IR 格式)
├── audit/
│   └── custom_audit_rules.json    # 配置审计规则
└── pocs_inbox/                    # 待反写的 POC 源(--sync-pocs)
```

环境变量 `SILAS_CONFIG_DIR` 可覆盖默认 `config/` 路径。

## POC 格式

silas 内部统一两类格式(均走模板引擎):

| 格式 | 说明 |
|---|---|
| `silas_tpl_v2` | nuclei v2 风格(http 块 + 变量替换 + 多请求) |
| `silas_tpl_v1` | nuclei v1 风格(requests 块) |
| `silas_tpl` / `unknown` | 通用模板(等价 v2) |
| `silas_rules` | 单请求 + DSL 表达式 |

POC IR 由 `build/poc_embedder.py` 预编译,运行时按需从 LRU 分片加载,内存峰值
≤ 100MB(对比单文件全量 ~800MB-1.2GB)。

DSL 表达式由 `engines/silas_dsl.py` 手写 lexer + 递归下降 parser 求值,
**不用 eval/exec**。matcher 支持 word/regex/status/binary/dsl,extractor 支持
regex/kval/json/xpath/dsl。

## 构建数据库

首次部署或更新指纹/POC 后跑一次:

```bash
bash build.sh [template_poc_dir] [finger_yaml_dir] [finger_json_file] [output_dir]
# 默认从 ../template-poc ../finger-yaml ../finger-json 读,输出到 db/
```

如需只保留 web 应用数据(删除非 web 服务指纹与 network 格式 POC):

```bash
python3 -m build.filter_web_only
```

筛选规则:
- 删除 `format=network` 的 POC(纯 TCP 协议探测)
- 删除规则只用 `banner/cert/port/protocol` 的指纹(纯非 web 服务指纹)
- 级联清理 mapping / tag_index / pocs_version_constraints / shards

## 实战案例

### 案例 1: 单目标全流程

```bash
# 扫 example.com,综合分 < 30 自动进深度
python3 silas.py -w -d example.com --deep-auto -k

# 查看最终报告
cat silas_workspace/final_report.json | python3 -m json.tool
```

### 案例 2: 批量目标快速指纹

```bash
# 准备 targets.txt(每行一个 URL)
cat > targets.txt <<EOF
http://target1.com
http://target2.com
https://target3.com
EOF

# 批量指纹,JSON 输出
python3 silas.py -l targets.txt -F -T 20 -O json -o fingers.json
```

### 案例 3: 仅跑高危 POC

```bash
# 严重度过滤(只跑 critical + high)
python3 silas.py -l targets.txt -G critical,high

# 关闭审计 + 仅 POC
python3 silas.py -l targets.txt --poc-only -A
```

### 案例 4: 大段 IP 深度扫描

```bash
# 用 masscan 高速扫端口 + ffuf 高并发目录扫描
python3 silas.py -w -d example.com --deep-auto \
    --port-scan-tool masscan --port-rate 500 \
    --dir-tool ffuf --dir-threads 20 \
    --port-top 1000
```

### 案例 5: POC 反写指纹入库

```bash
# 把 config/pocs_inbox/ 下的 POC 反写指纹
python3 silas.py -M --sync-pocs-dry-run    # 先看计划
python3 silas.py -M                       # 实际写库

# 下次启动自动合并入主指纹库
python3 silas.py -t http://target -F
```

### 案例 6: 单阶段从已有产物继续

```bash
# 已有 subdomains.txt,直接跑指纹
python3 silas.py -f -l silas_workspace/subdomains.txt

# 已有 urls.txt,直接跑 POC
python3 silas.py -p -l silas_workspace/urls.txt -o poc_hits.json -O json
```

