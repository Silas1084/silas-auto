# silas

Fingerprint + POC integrated scanner (web application focused).

- Six-dimensional fingerprint engine: title / body / header / icon_hash / cert / banner
- Three POC mapping types: direct / tag / general + AST DSL evaluation
- POC library: 88860 entries, peak memory ≤ 100MB (LRU sharding)
- 13-stage end-to-end pipeline: subdomain -> asset pre-identification -> fingerprint -> POC -> scoring -> deep probe -> risk-rated report

## Compatibility

| Platform | Status | Notes |
|---|---|---|
| Kali Linux | Native (primary dev env) | apt one-liner installs all upstream tools |
| Ubuntu 20.04+ | Native | subfinder via go install |
| CentOS 8 / RHEL 8+ | Usable | EPEL required, some tools built from Go source |
| Windows 10/11 | Usable | Native Python + tool binaries download |
| WSL2 | Linux-equivalent | Follow Ubuntu section |
| macOS | Usable | brew + go install |

Python ≥ 3.9. POC engine requires `requests`; fingerprint mmh3 requires gcc (Python C extension).

## Environment Setup

### 0. Common Steps (all platforms)

```bash
git clone <silas-repo> silas && cd silas
python3 -m venv venv
source venv/bin/activate           # Windows: venv\Scripts\activate
pip install --upgrade pip
pip install -r requirements.txt
pip install dnspython              # Required for deep pipeline IP reverse lookup + CDN detection
```

requirements.txt contents:

| Package | Purpose | C Extension |
|---|---|---|
| PyYAML | Config / POC parsing | No |
| requests | HTTP fetching | No |
| mmh3 | favicon hash fingerprint | **Yes** (requires gcc) |
| rich | Terminal UI | No |
| lxml | Partial XML parsing | **Yes** (requires libxml2-dev) |
| packaging | Version constraints | No |
| dnspython | DNS reverse lookup + CDN CNAME detection | No |

If C extensions fail to install, install the build toolchain first and retry.

---

### 1. Kali Linux

Kali ships subfinder / nmap / masscan / ffuf / dirsearch / httpx-toolkit natively.

```bash
# 1. System update + build toolchain
sudo apt update && sudo apt install -y \
    python3 python3-pip python3-venv \
    python3-dev gcc git \
    libxml2-dev libxslt1-dev libffi-dev

# 2. Upstream tools (one-liner)
sudo apt install -y subfinder nmap masscan ffuf dirsearch httpx-toolkit

# 3. Go (optional, if apt subfinder is outdated)
sudo apt install -y golang

# 4. venv + requirements
cd /opt/silas
python3 -m venv venv && source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
pip install dnspython
```

**OneForAll** (subdomain enumeration fallback):

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

Verify:

```bash
python3 silas.py --check-tools
# 7 tools all ✓ means ready
```

---

### 2. Ubuntu 20.04 / 22.04 / 24.04

```bash
# 1. System update + build toolchain + base tools
sudo apt update && sudo apt install -y \
    python3 python3-pip python3-venv \
    python3-dev build-essential git \
    libxml2-dev libxslt1-dev libffi-dev \
    nmap masscan ffuf dirsearch

# 2. Go (subfinder/httpx require it)
sudo apt install -y golang-go
# Or install latest:
# wget https://go.dev/dl/go1.22.0.linux-amd64.tar.gz
# sudo tar -C /usr/local -xzf go1.22.0.linux-amd64.tar.gz
# export PATH=$PATH:/usr/local/go/bin

# 3. subfinder + httpx (from Go source)
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

**Note**: On Ubuntu, projectdiscovery httpx binary is named `httpx` (not `httpx-toolkit`).
bin_manager auto-verifies via ELF header, skips Python same-name package.

**OneForAll**: See Kali section.

---

### 3. CentOS 8 / RHEL 8 / Rocky Linux 8+

```bash
# 1. Enable EPEL + PowerTools
sudo dnf install -y epel-release
sudo dnf config-manager --set-enabled powertools  # Rocky/Alma: crb

# 2. Build toolchain + base tools
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y \
    python3 python3-pip python3-devel \
    libxml2-devel libxslt-devel libffi-devel \
    git nmap

# 3. Go (system version may be outdated, manual install recommended)
sudo dnf install -y golang

# 4. masscan (EPEL may not have it, build from source)
sudo dnf install -y git libpcap-devel
cd /tmp && git clone https://github.com/robertdavidgraham/masscan.git
cd masscan && make -j$(nproc)
sudo make install
cd ..

# 5. ffuf + subfinder + httpx (from Go source)
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

CentOS 7 ships Python 3.6 by default, **does not meet ≥3.9**:

```bash
sudo dnf install -y python39 python39-devel
python3.9 -m venv venv
source venv/bin/activate
```

---

### 4. Windows 10 / 11

#### 4.1 Install Python 3.11+

Download from [python.org](https://www.python.org/downloads/windows/), check **Add Python to PATH**.

```powershell
python --version
# Python 3.11.x
```

#### 4.2 Install Visual C++ Build Tools (required by mmh3/lxml)

Download [Visual Studio Build Tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/),
check **Desktop development with C++**.

Or install pre-built wheels:

```powershell
pip install --only-binary :all: mmh3 lxml
```

#### 4.3 Create venv + install requirements

```powershell
cd C:\silas
python -m venv venv
venv\Scripts\activate
pip install --upgrade pip
pip install -r requirements.txt
pip install dnspython
```

#### 4.4 Install upstream tools

| Tool | Download URL | Install Method |
|---|---|---|
| nmap | https://nmap.org/download.html | exe installer, check Add to PATH |
| subfinder | https://github.com/projectdiscovery/subfinder/releases | Extract zip, add to PATH |
| httpx | https://github.com/projectdiscovery/httpx/releases | Extract zip, add to PATH |
| masscan | https://github.com/robertdavidgraham/masscan/releases | Extract zip, add to PATH |
| ffuf | https://github.com/ffuf/ffuf/releases | Extract zip, add to PATH |
| dirsearch | `pip install dirsearch` | Global or inside venv |

Recommended: put all in `C:\Tools\`, add to system PATH:

```powershell
# Temporary (current session)
$env:Path += ";C:\Tools"
# Permanent (admin PowerShell)
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\Tools", "Machine")
```

Or use [Scoop](https://scoop.sh/):

```powershell
scoop install nmap subfinder httpx masscan ffuf
pip install dirsearch
```

#### 4.5 OneForAll (Windows)

```powershell
cd Tools
git clone https://github.com/shmilylty/OneForAll.git
cd OneForAll
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
pip install --upgrade fire
deactivate
cd ..\..
```

**Windows known limitations**:
- OneForAll venv path is `venv\Scripts\python.exe` on Windows, silas hardcodes `venv/bin/python`, not recognized on native Windows (use Docker/WSL2 workaround)

#### 4.6 Verify

```powershell
python silas.py --check-tools
```

---

### 5. Docker (cross-platform one-liner)

```bash
# Dockerfile see PORTABILITY.md
docker build -t silas .
docker run --rm -it silas -w -d example.com --deep-auto
```

Mount config and workspace:

```bash
docker run --rm -it \
    -v "$PWD/config:/opt/silas/config" \
    -v "$PWD/silas_workspace:/opt/silas/silas_workspace" \
    silas -w -d example.com --deep-auto -k
```

## Dependencies (Quick Reference)

One-liner to install upstream tools per platform:

| Platform | Command |
|---|---|
| Kali / Ubuntu | `sudo apt install -y subfinder nmap masscan ffuf dirsearch httpx-toolkit` |
| Ubuntu (no subfinder) | `sudo apt install -y nmap masscan ffuf dirsearch && go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest` |
| CentOS / RHEL | `sudo dnf install -y nmap && go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest github.com/projectdiscovery/httpx/cmd/httpx@latest github.com/ffuf/ffuf/v2@latest && pip3 install dirsearch && (build masscan from source)` |
| Windows | Download binaries from each tool's GitHub Release + `pip install dirsearch` |

Python dependencies:

```bash
pip install -r requirements.txt
pip install dnspython
```

Tool identification notes:
- `httpx-toolkit` on Kali is the projectdiscovery httpx Go binary
- `/usr/bin/httpx` is the Python same-name package (NodeJS httpx, silas does not use it)
- bin_manager auto-verifies via ELF header, prefers `httpx-toolkit`
- OneForAll directory name accepts both `OneForAll/` and `OneForAll-master/`

Check tool readiness:

```bash
python3 silas.py --check-tools
```

## Quick Start

```bash
# Single target full pipeline (hits < threshold prompts for deep probe)
python3 silas.py -w -d example.com

# Single target fingerprint only
python3 silas.py -t http://target -F

# Single target POC only (fingerprint runs silently for mapping)
python3 silas.py -t http://target --poc-only

# Batch (one URL per line)
python3 silas.py -l targets.txt -o result.json --format json

# Severity filter (critical + high only)
python3 silas.py -l targets.txt -G critical,high
```

Output formats: `text` (default) / `json` / `html`.

## Command Reference

```
Target:     -t / -u / -l
Mode:       -F finger-only   --poc-only
Concurrency: -T threads  -W timeout
Output:     -o file  -O text|json|html
Paths:      --db-dir  --pocs-dir  --config-dir
Engine:     --audit / -A disable audit (default on)

Workflow (single-char mutex):
  -w                               Full pipeline (domain -> asset pre -> subdomain -> finger -> POC -> decision -> deep -> final report)
  -s                               Subdomain stage
  -f                               Fingerprint stage
  -p                               POC stage
  -d <domain>                      Root domain (-w / -s required, domain only)
  --tools-dir Tools                Upstream tools source dir
  --work-dir silas_workspace       Working directory
  -k                               Keep intermediate files (default: cleaned)

Deep probe control:
  --deep-threshold N               Hits < N triggers prompt (default 10)
  --deep-auto                      Auto deep probe when hits < threshold (CI friendly)
  --deep-yes                       Non-interactive yes (equivalent to --deep-auto)
  --deep-no                        Non-interactive no
  --no-deep                        Disable deep branch entirely

Port scan tuning:
  --port-top N                     nmap/masscan Top-N ports (default 100)
  --port-scan-tool nmap|masscan    Port scan tool (default nmap)
  --port-rate N                    masscan rate (default 1000)

Directory scan tuning:
  --dir-tool dirsearch|ffuf        Directory scan tool (default dirsearch)
  --dir-wordlist PATH              Wordlist (default dirb common.txt)
  --dir-threads N                  Directory scan concurrency (default 10)

Management (no scan):
  -M / --sync-pocs                 Scan config/pocs_inbox/ reverse-sync fingerprints
  --sync-pocs-from DIR             Specify different inbox
  --sync-pocs-dry-run              Print plan only, no DB write
  -C / --check-tools               Check upstream tools
```

## Workflow Orchestration

`-w` is the end-to-end entry point. Domain to POC exploitation in one shot; auto-enters deep probe when composite score is low:

```bash
# Full pipeline (score < 30 auto deep, >= 70 skip, 30-70 prompt)
python3 silas.py -w -d example.com

# Equivalent to pre-refactor -w (no deep branch)
python3 silas.py -w -d example.com --no-deep

# CI/batch: no prompt, auto deep when score < threshold
python3 silas.py -w -d example.com --deep-auto

# Custom threshold + port/directory scan tuning
python3 silas.py -w -d example.com --deep-auto \
    --deep-threshold 5 \
    --port-scan-tool masscan --port-rate 500 \
    --dir-tool ffuf --dir-threads 20
```

### Stage Sequence (13 steps)

| # | Stage | Upstream | Downstream | Tool |
|---|---|---|---|---|
| 1 | subdomain | domain | subdomains.txt | subfinder / OneForAll |
| 1.5 | asset_pre | subdomains.txt | asset_pre.json (CDN/WAF/tech stack) | httpx + dnspython |
| 2 | finger | subdomains.txt | urls.txt + fingerprints (alive only) | silas FingerEngine |
| 3 | poc | urls.txt | poc_hits.json + verified.json + audit_results.json | silas PocRunner |
| 4 | quick_report | poc_hits.json + asset_pre.json | quick_report.json (with composite score) | - |
| 5 | prompt | quick_report.json | User choice | input() |
| 6 | ip_resolve | subdomains.txt | ips.txt (CDN subdomains filtered) + cdn_subs.txt | dnspython |
| 7 | ip_alive | ips.txt | alive_ips.txt | nmap -sn |
| 8 | port_scan | alive_ips.txt | ip_ports.txt + ip_services.json | nmap -sT -sV / masscan |
| 9 | web_probe | ip_ports.txt | web_targets.txt | httpx |
| 10 | dir_scan | web_targets.txt | dir_hits.json | dirsearch / ffuf |
| 11 | deep | web_targets.txt + dir_hits.json | deep_report.json | silas (dual concurrent) |
| 12 | final_report | all artifacts | final_report.json (risk rating) | - |

### Scoring Decision

`quick_report` stage computes a 4-dimensional composite score (max 100):

| Dimension | Score | Basis |
|---|---|---|
| Asset value | 0-30 | Subdomain count + high-value prefixes (api/admin/mail/vpn) |
| Exposure | 0-30 | Alive URL count + subdomain alive rate |
| High-risk fingerprints | 0-25 | Hits on struts/log4j/weblogic/shiro/nginx/aliyun/oss keywords |
| POC success rate | 0-15 | poc_hits / urls_count banded |

Decision logic:

```
Score >= 70    -> High asset value, end pipeline (sufficient results)
Score <  30    -> Low asset value, auto-trigger deep probe
30 <= score < 70 -> Fall back to hit-count vs threshold logic
                   hits >= threshold -> end pipeline
                   hits <  threshold -> prompt (or --deep-auto auto deep)
```

Non-TTY environments (CI/pipe) exit by default; add `--deep-auto` to auto-enter deep probe.

### Asset Pre-identification (asset_pre)

Lightweight pre-identification right after subdomain scan, avoiding invalid port scans on CDN node IPs:

- **CNAME -> CDN**: Match known CDN suffixes (cloudflare/alicdn/qcloud/akamai/fastly etc.)
- **Server header -> CDN**: Tengine/AliyunOSS/Cloudflare keywords
- **tech-detect -> CDN**: httpx-detected Alibaba Cloud OSS / Tengine etc.
- **WAF detection**: Response header keywords (cloudflare/yundun/safedog/anquanbao/safeline etc.)

Artifact `asset_pre.json`:

```json
{
  "records": [
    {
      "host": "boatai-img.boatai.me",
      "cname": "boatai-img.boatai.me.queniuaa.com",
      "cdn": "alibaba",
      "waf": "",
      "tech": ["Alibaba Cloud Object Storage Service", "Tengine"],
      "title": "",
      "status": 403,
      "alive": true
    }
  ],
  "summary": {"total": 4, "alive": 3, "cdn": 1, "waf": 0}
}
```

### CDN Filter Pipeline

```
asset_pre identifies CDN subdomains
    |
ip_resolve skips CDN subdomains, no IP reverse lookup for them
    |
ips.txt contains only real origin IPs (avoids scanning CDN node IPs)
```

Filtered CDN subdomains recorded in `cdn_subs.txt`.

### Deep Dual-Concurrent

`deep` stage uses `ThreadPoolExecutor(max_workers=2)` for two subtasks, isolated artifacts:

| Subtask | Input | Output Dir |
|---|---|---|
| A | web_targets.txt (root URLs) | `silas_workspace/deep_a/` |
| B | dir_hits.json path URLs + web_targets feedback (info backflow) | `silas_workspace/deep_b/` |

Info backflow: subtask B merges dir_hits + web_targets inputs, re-runs fingerprinting on new paths discovered by directory scan.

Merge dedup:
- Fingerprint dedup key: `url|finger_id`
- Vulnerability dedup key: `url|poc_id`
- Cross-stage dedup: reads `verified.json` cumulative records

Output `cross_discover` stats for A/B complementary findings (A-only / B-only / shared).

### Final Report

Merges all stage artifacts -> `final_report.json`, sorted by risk rating:

```json
{
  "summary": {
    "total_vulns": 2,
    "by_severity": {"high": 1, "medium": 1},
    "poc_vulns": 0,
    "audit_hits": 2,
    "total_assets": 4,
    "alive_assets": 3,
    "cdn_assets": 1,
    "waf_assets": 0,
    "services_detected": 3,
    "quick_score": 34
  },
  "risk_sorted_vulns": [
    {"poc_id": "leak-source", "severity": "high", "affected_urls": [...], "source": "audit"},
    {"poc_id": "leak-swagger", "severity": "medium", "affected_urls": [...], "source": "audit"}
  ],
  "audit_hits": [...],
  "assets_summary": {"top_tech": [...]},
  "services_summary": [...]
}
```

Same POC across multiple assets merged into one record, with affected URL list.

### Workspace Artifacts

```
silas_workspace/
├── subdomains.txt          # [1] subdomains
├── subfinder.json          # [1] upstream raw (cleaned after scan)
├── oneforall_results/      # [1] OneForAll fallback output
├── asset_pre.json          # [1.5] asset pre-identification (CDN/WAF/tech) -k retained
├── asset_pre_httpx.jsonl   # [1.5] httpx raw output
├── urls.txt                # [2] URL-normalized subdomains (alive only)
├── poc_hits.json           # [3] POC hits
├── audit_results.json      # [3] config audit hits -k retained
├── verified.json           # [3] global verified vulns (cross-stage dedup) -k retained
├── quick_report.json       # [4] quick summary (with score) -k retained
├── ips.txt                 # [6] IP reverse lookup (CDN subdomains filtered)
├── cdn_subs.txt            # [6] filtered CDN subdomains
├── alive_ips.txt           # [7] alive IPs
├── ip_ports.txt            # [8] ip:port
├── ip_services.json        # [8] port service detection details
├── nmap.xml / masscan.json # [8] upstream raw
├── web_targets.txt         # [9] http(s)://ip:port
├── dir_hits.json           # [10] directory scan hits
├── deep_a/                 # [11-A] dual-concurrent subtask A artifacts
├── deep_b/                 # [11-B] dual-concurrent subtask B artifacts
├── deep_report.json        # [11] deep report -k retained
└── final_report.json       # [12] risk-rated final report -k retained
```

`-k` mode retains all intermediate files; default retains only `asset_pre.json` / `audit_results.json` / `verified.json` / `quick_report.json` / `deep_report.json` / `final_report.json`.

### Single-Stage Execution

Each stage usable independently (resume from existing artifacts):

```bash
# Subdomain (requires -d domain)
python3 silas.py -s -d example.com

# Fingerprint (requires -l subdomain list)
python3 silas.py -f -l silas_workspace/subdomains.txt

# POC (requires -l URL list)
python3 silas.py -p -l silas_workspace/urls.txt

# Asset pre-identification (requires -d domain, resume from existing subdomains.txt)
python3 silas.py -w -d example.com --no-deep
```

`-d` is strictly validated as a domain (rejects IP/URL), preventing subfinder 0-result fallback to stale fixtures causing "scan A actually scan B".

## POC Fingerprint Reverse Sync

Scans yaml files in POC inbox (`config/pocs_inbox/`), reverse-parses fingerprint rules from fofa-query / shodan-query and fingerprint blocks, writes to `db/fingerprints.json` + builds mapping:

```bash
# Dry-run see plan
python3 silas.py -M --sync-pocs-dry-run

# Actually write to DB
python3 silas.py -M

# Specify different inbox
python3 silas.py -M --sync-pocs-from /path/to/pocs
```

Reverse-synced fingerprints written to `config/fingerprints/reversed_fingerprints.json`, auto-merged into main fingerprint DB on next launch.

## User Config Directory

`config/` is your own config area, physically isolated from `db/` / `pocs/`, runtime three-layer merge:

| Case | Behavior |
|---|---|
| Same id / same key | User config overrides built-in |
| New id / new key | Append |
| List field | Dedup union |
| File missing | Skip, no impact on other files |

```
config/
├── fingerprints/
│   ├── custom_fingerprints.json   # Custom fingerprints (override/append to db/fingerprints.json)
│   ├── active_probes.json         # Active probe paths (grouped by app_name)
│   └── reversed_fingerprints.json # POC reverse-synced fingerprints (--sync-pocs auto-generated)
├── workflows/
│   ├── direct_mappings.json       # Fingerprint name -> POC direct mapping
│   ├── tag_mappings.json          # app_name -> POC tag fallback
│   └── general_pocs.json          # No-fingerprint fallback POCs (Shiro/Log4j2 etc.)
├── pocs/
│   ├── custom_pocs_index.json     # POC index (must match custom_pocs.json id)
│   └── custom_pocs.json           # POC content (IR format)
├── audit/
│   └── custom_audit_rules.json    # Config audit rules
└── pocs_inbox/                    # POC sources to reverse-sync (--sync-pocs)
```

Environment variable `SILAS_CONFIG_DIR` overrides default `config/` path.

## POC Format

silas internally unifies two format categories (both via template engine):

| Format | Description |
|---|---|
| `silas_tpl_v2` | nuclei v2 style (http block + variable substitution + multi-request) |
| `silas_tpl_v1` | nuclei v1 style (requests block) |
| `silas_tpl` / `unknown` | Generic template (equivalent to v2) |
| `silas_rules` | Single request + DSL expression |

POC IR is pre-compiled by `build/poc_embedder.py`, runtime loaded on-demand from LRU shards, peak memory ≤ 100MB (vs ~800MB-1.2GB for single-file full load).

DSL expressions evaluated by `engines/silas_dsl.py` hand-written lexer + recursive descent parser, **no eval/exec**. Matcher supports word/regex/status/binary/dsl, extractor supports regex/kval/json/xpath/dsl.

## Build Database

Run once on first deploy or after updating fingerprints/POCs:

```bash
bash build.sh [template_poc_dir] [finger_yaml_dir] [finger_json_file] [output_dir]
# Default reads from ../template-poc ../finger-yaml ../finger-json, outputs to db/
```

To keep only web application data (delete non-web service fingerprints and network-format POCs):

```bash
python3 -m build.filter_web_only
```

Filter rules:
- Delete POCs with `format=network` (pure TCP protocol probes)
- Delete fingerprints using only `banner/cert/port/protocol` rules (pure non-web service fingerprints)
- Cascade cleanup mapping / tag_index / pocs_version_constraints / shards

## Real-World Examples

### Example 1: Single Target Full Pipeline

```bash
# Scan example.com, auto deep when score < 30
python3 silas.py -w -d example.com --deep-auto -k

# View final report
cat silas_workspace/final_report.json | python3 -m json.tool
```

### Example 2: Batch Target Quick Fingerprint

```bash
# Prepare targets.txt (one URL per line)
cat > targets.txt <<EOF
http://target1.com
http://target2.com
https://target3.com
EOF

# Batch fingerprint, JSON output
python3 silas.py -l targets.txt -F -T 20 -O json -o fingers.json
```

### Example 3: High-Severity POC Only

```bash
# Severity filter (critical + high only)
python3 silas.py -l targets.txt -G critical,high

# Disable audit + POC only
python3 silas.py -l targets.txt --poc-only -A
```

### Example 4: Large IP Range Deep Scan

```bash
# masscan high-speed port scan + ffuf high-concurrency directory scan
python3 silas.py -w -d example.com --deep-auto \
    --port-scan-tool masscan --port-rate 500 \
    --dir-tool ffuf --dir-threads 20 \
    --port-top 1000
```

### Example 5: POC Fingerprint Reverse Sync

```bash
# Reverse-sync POCs in config/pocs_inbox/
python3 silas.py -M --sync-pocs-dry-run    # See plan first
python3 silas.py -M                       # Actually write to DB

# Next launch auto-merges into main fingerprint DB
python3 silas.py -t http://target -F
```

### Example 6: Single-Stage Resume from Existing Artifacts

```bash
# Have subdomains.txt, run fingerprint directly
python3 silas.py -f -l silas_workspace/subdomains.txt

# Have urls.txt, run POC directly
python3 silas.py -p -l silas_workspace/urls.txt -o poc_hits.json -O json
```

## FAQ

**Q: Single target scan returns nothing?**
A: Run `-F` first to check fingerprint hits. If fingerprints hit but POC doesn't run, check `config/workflows/direct_mappings.json` whether fingerprint is mapped to POC.

**Q: POC changes not taking effect?**
A: `config/pocs/custom_pocs.json` and `custom_pocs_index.json` must have matching ids. After editing POC content, update the index too.

**Q: `-d` errors "needs domain, got IPv4"?**
A: `-d` only accepts domains. For IP scanning use `-l <ip_list>` single-stage, not `-w` full pipeline (subfinder returns 0 results for IP).

**Q: High memory usage?**
A: Check if `pocs/pocs_content.shards/` is generated. Without sharding, falls back to single-file full load (~800MB-1.2GB). Run `bash build.sh` to rebuild.

**Q: Deep pipeline stuck at prompt?**
A: Non-TTY environments (pipe/CI) exit by default. Add `--deep-auto` to auto-enter deep, or `--deep-no` to exit directly, or `--no-deep` to disable deep branch entirely.

**Q: Composite score too low?**
A: Check `asset_pre.json` tech field for detected components. `high_risk_fingers` keywords are in `core/workflow/stages_quick.py` `_HIGH_RISK_KEYWORDS`, add as needed.

**Q: web_probe returns 0 web targets?**
A: Check `silas_workspace/httpx.jsonl`. httpx by default probes 80/443/8080/8443. If target uses non-standard ports, ensure `--port-top` covers them.

**Q: Port service detection all tcpwrapped?**
A: Target has WAF/cloud firewall blocking nmap banner grabbing. Try:
- `--port-scan-tool masscan` (faster but no service detection)
- Increase `--version-intensity` in `core/workflow/stages_ip.py`
- Accept tcpwrapped, rely on httpx tech-detect for service hints

**Q: OneForAll runs slowly?**
A: 5-10 minutes per target is normal (30+ passive sources + brute force + DNS resolution). Prefer subfinder when it returns results; OneForAll is fallback only.

**Q: CDN subdomains not filtered?**
A: Check `asset_pre.json` `cdn` field. If empty, CDN detection missed. CDN detection uses three sources: CNAME suffix + Server header + tech-detect. Custom CDN domains (e.g. queniuaa.com) rely on Server header / tech field.

## Project Structure

```
silas.py                      CLI entry (route dispatch)
core/
├── scanner.py                Main scan orchestration (finger -> audit -> POC -> audit dump)
├── finger_engine.py         Six-dimensional fingerprint engine
├── rule_parser.py            Fingerprint DSL parser
├── bool_eval.py              Rule evaluation
├── poc_runner.py             POC scheduler
├── poc_loader.py             Sharding + LRU loading
├── mapping_engine.py         Fingerprint -> POC three-type mapping
├── user_config.py            Three-layer merge loading
├── audit_engine.py           Config audit
├── poc_finger_reverse.py     POC fingerprint reverse-sync
├── http_client.py            HTTP fetching + active probe
├── output.py / colors.py / banner.py / progress.py  Terminal UI
└── workflow/                 13-stage workflow orchestration
    ├── stages.py             subdomain / finger / poc
    ├── stages_prefinger.py   asset_pre (CDN/WAF/tech)
    ├── stages_quick.py       quick_report (score) + prompt
    ├── stages_ip.py          ip_resolve (CDN filter) / ip_alive / port_scan (-sV)
    ├── stages_web.py         web_probe / dir_scan
    ├── stages_deep.py        deep (dual concurrent + info backflow)
    ├── stages_report.py     final_report (risk rating)
    ├── runner.py             WorkflowRunner orchestration
    ├── bin_manager.py        Tool discovery (httpx prefers httpx-toolkit ELF)
    ├── invoker.py            Subprocess call (stdin=DEVNULL prevents fd leak)
    ├── parsers.py            nmap XML/masscan JSON/httpx JSONL/dirsearch JSON
    └── workspace.py          Artifact management
engines/
├── silas_template_engine.py  Template engine + IR normalization
├── silas_template_helpers.py Placeholder system
└── silas_dsl.py              AST DSL evaluation
build/                        Build-time data production
db/                           Fingerprint DB (build artifact)
pocs/                         POC DB (build artifact)
config/                       User config (top layer of three-way merge)
```

## Design Constraints (non-negotiable)

1. `core/user_config.py` three-layer merge - multi-tenant / private fingerprint basis
2. `build/clean_external_traces.py` - commercial desensitization (remove ehole/dddd/nuclei traces)
3. `build/shard_pocs.py` removed (sharding inlined into build/pipeline.py + filter_web_only rewrite)
4. `core/finger_engine.py` confidence noise reduction - false positive control
5. `engines/silas_dsl.py` AST evaluation - prevent RCE
6. `core/workflow/stages.py` stale file pre-cleanup - prevent "scan A actually scan B" target drift
7. `core/workflow/bin_manager.py` tool discovery - httpx prefers `httpx-toolkit` ELF binary, auto-skips Python same-name package; OneForAll dir name auto-match
8. `core/workflow/parsers.py` stdlib only (`xml.etree`/`json`) - no third-party deps, XML parse failure tolerant return empty list, avoids pipeline interruption
9. `core/workflow/stages_quick.py` composite scoring - asset value + exposure + high-risk fingerprints + POC rate
10. `core/workflow/stages_prefinger.py` CDN three-source detection - CNAME + Server header + tech-detect
11. `core/workflow/stages_report.py` risk rating - sort by severity + merge same POC across assets

## License

Internal tool, not open source.
