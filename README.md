# cn-dns-conf

将 [felixonmars/dnsmasq-china-list](https://github.com/felixonmars/dnsmasq-china-list) 的 **dnsmasq** 配置（`server=/域名/上游IP`）整理后输出为 **dnsmasq**、**SmartDNS** 与 **AdGuard Home** 可用的片段。脚本用 Bash 完成下载、去注释与格式转换。

## 依赖

- Bash、`curl`、`awk`（macOS / BSD `awk` 即可）

## 默认源文件

从 `BASE_URL`（默认同仓库 `master`）下载并合并：

| 文件 |
|------|
| `accelerated-domains.china.conf` |
| `google.china.conf` |
| `apple.china.conf` |

处理规则：

- 去掉行首（允许前导空格后）以 `#` 开头的行
- 去掉行尾 ` # ...` 形式的注释
- 只解析形如 `server=/example.com/1.2.3.4` 的行
- 三个文件按上述顺序合并；**同一域名保留首次出现的记录**

## 用法

```bash
./convert-dnsmasq-china.sh              # 下载并生成
./convert-dnsmasq-china.sh --no-download # 使用已有 upstream/ 下的文件
./convert-dnsmasq-china.sh -h            # 帮助
```

`--no-download` 可与其它参数任意顺序混写。

### 统一指定上游（可选）

将**所有域名**指向你指定的 DNS，并在 SmartDNS 里使用你给定的**上游组名**：

```bash
./convert-dnsmasq-china.sh 223.5.5.5 alidns
```

必须**同时**提供 `<dns_ip>` 与 `<dns_alias>`，或**都不**提供。  
指定后 **dnsmasq**、**AdGuard** 里每条规则的上游 IP 均使用该地址；**SmartDNS** 里 `server` 与组名仍按该参数生效。

## 环境变量

| 变量 | 默认 | 说明 |
|------|------|------|
| `BASE_URL` | `https://raw.githubusercontent.com/felixonmars/dnsmasq-china-list/refs/heads/master` | 列表下载根路径 |
| `INPUT_DIR` | `<脚本目录>/upstream` | 原始 `.conf` 存放处 |
| `OUT_DIR` | `<脚本目录>/out` | 生成文件输出目录 |
| `AG_BATCH` | `8` | **同一上游 IP** 下每行合并的域名个数：AdGuard（`[/d1/d2/.../]upstream`）与 dnsmasq（`server=/d1/d2/.../ip`）共用 |
| `SMARTDNS_DOMAINSET` | `yes` | `yes`：用 `domain-set` + 列表文件（紧凑）；`no`：每个域名一行 `nameserver` |
| `SMARTDNS_LIST_BASENAME` | `china-domains` | 仅域名列表的文件名前缀（与 DNS 无关）；多上游组时为 `basename-1.list`、`basename-2.list` 等 |

## 输出文件

均在 `OUT_DIR`（默认 `out/`）下。**本仓库已跟踪 `out/`**，可直接从 GitHub 浏览或下载（例如 Raw 链接），无需本地跑脚本。若上游列表有更新，可执行 `./convert-dnsmasq-china.sh` 后重新提交 `out/`。

| 文件 | 说明 |
|------|------|
| `dnsmasq-china.conf` | dnsmasq 片段：每行最多 `AG_BATCH` 个域名，`server=/d1/d2/.../IP`（与 AdGuard 分批规则相同）；`conf-file=/绝对路径/dnsmasq-china.conf` |
| `smartdns-china.conf` | SmartDNS 片段：在 `smartdns.conf` 里用 `conf-file /绝对路径/smartdns-china.conf` 引入 |
| `china-domains.list`（或可带 `-1`、`-2` 后缀） | **仅域名**，一行一个；由 `domain-set -file` 引用。文件名与上游无关，可改 `SMARTDNS_LIST_BASENAME` |
| `adguard-upstream-china.txt` | AdGuard Home 的「上游」列表或 `upstream_dns_file` 内容：`[/域名1/域名2/.../]IP`，每行最多 `AG_BATCH` 个域名（同一 IP） |

### dnsmasq 说明

- 与上游列表语义一致，仅去掉注释并按合并规则去重；使用 `conf-file` 引入即可。
- dnsmasq 的 `--server` 支持「多段域名 + 末尾上游」写法，与 `--address` 类似：`server=/a.com/b.com/1.2.3.4`。脚本在**连续且相同上游 IP** 的记录上按 `AG_BATCH`（默认 8）合并为一行。

### SmartDNS 说明

- 默认模式用官方 [域名集合（domain-set）](https://pymumu.github.io/smartdns/config/domain-rule/)：`domain-set` 指向列表文件，`nameserver /domain-set:集合名/上游组名` 把「中国域名」指到对应 `server … -g 组名`。
- 列表文件只含域名；**上游 IP 与组名**仍在 `server` / `nameserver` 的 **GROUP** 部分（例如 `g_114_114_114_114`）。

### AdGuard Home 说明

- 将生成文件内容粘贴到「DNS 上游服务器」，或配置为 `upstream_dns_file`（路径以你部署为准）。
- 多域名同行语法为官方支持的 `[/a/b/]1.1.1.1` 形式；仅当连续记录为**同一上游 IP** 时才会合并到一行。

## 示例

```bash
# 使用本地已下载的列表，统一走阿里 DNS，SmartDNS 组名为 alidns
./convert-dnsmasq-china.sh --no-download 223.5.5.5 alidns

# AdGuard 每行 12 个域名
AG_BATCH=12 ./convert-dnsmasq-china.sh --no-download

# SmartDNS 改为逐域名 nameserver（超大单文件）
SMARTDNS_DOMAINSET=no ./convert-dnsmasq-china.sh --no-download
```

## 克隆与生成

- `upstream/` 仍由 `.gitignore` 忽略（体积大，随脚本下载即可）。
- `out/` 已纳入 Git，克隆后可直接使用 `out/` 内文件。
- 需要与上游同步时在本机执行：

```bash
./convert-dnsmasq-china.sh
```

## 发布到 GitHub

1. 在 GitHub 新建空仓库（不要勾选添加 README，避免首次推送冲突）。
2. 在项目根目录执行：

```bash
git init
git add convert-dnsmasq-china.sh README.md .gitignore out/
git commit -m "Initial commit: dnsmasq-china-list converter"
git branch -M main
git remote add origin https://github.com/<你的用户名>/<仓库名>.git
git push -u origin main
```

将 `<你的用户名>`、`<仓库名>` 换成实际值。若使用 SSH，把 `origin` URL 改为 `git@github.com:用户名/仓库名.git`。

## 许可

脚本与生成配置的使用请同时遵守上游项目 [dnsmasq-china-list](https://github.com/felixonmars/dnsmasq-china-list) 的许可与声明。
