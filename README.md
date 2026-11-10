# wopanupload

> 联通云盘（沃盘 / `pan.wo.cn`）App 通道上传工具 —— 秒传 + 分片并发上传，绕开网页端的 200 KB/s 限速，附带实时进度条与文件管理 CLI。

![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?logo=python&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green)
![CLI](https://img.shields.io/badge/Interface-CLI-blue)

`wopanupload` 是一个单文件的 Python 命令行工具（`ltpan.py`），通过复用**联通云盘 Android App 的上传通道**来上传文件，同时提供目录浏览、新建目录、删除文件、容量查询等常用操作。

---

## 为什么会有这个项目

网页端上传到联通云盘时会被限制在约 **200 KB/s**，而 App 走的是独立的上传专用节点 `hyupload.pan.wo.cn/openapi/client/upload2C`，并不受这条限速约束。本项目直接复用该通道，因此可以获得接近带宽上限的上传速度，并且支持**秒传**（服务端已有相同文件时无需传输数据）。

脚本只依赖 `requests` 与 `pycryptodome`，无需浏览器、无需 App、无需额外服务，一条命令即可上传。

---

## 功能特性

| 特性 | 说明 |
| --- | --- |
| ⚡ 秒传 | 先计算文件 SHA-256 并向服务端校验，命中即秒完成，不消耗上传流量 |
| 🚀 分片并发上传 | 默认 16 MB 分片、4 线程并发，可调至更高；单片失败自动重试 3 次 |
| 📊 实时进度条 | 百分比、已传/总量、瞬时速度、总耗时与平均速度 |
| 📁 文件管理 | `ls` 列目录（自动翻页）、`mkdir` 新建目录、`rm` 按 fileId 删除、`info` 查容量 |
| 👨‍👩‍👧 家庭空间 | `--space 1 --family-id <id>` 上传到家庭空间 |
| 🌐 代理支持 | `--proxy http://127.0.0.1:7890`，方便调试与特殊网络环境 |
| 🔐 完整协议实现 | AES-128-CBC 加解密、dispatcher 的 MD5 签名、App 请求头模拟 |
| 🐍 可当库用 | `from ltpan import WoPan`，直接在自己的 Python 代码里调用 |
| 🔧 灵活的 Token 来源 | 命令行参数 / 环境变量 / `~/.wopan.json`，三选一 |

---

## 快速开始

### 1. 环境要求

- **Python 3.10+**（代码使用了 `dict | None` 等 PEP 604 类型注解）
- 一个有效的联通云盘 App `access token`

### 2. 安装

```bash
git clone https://github.com/jeffreys125/wopanupload.git
cd wopanupload
pip install -r requirements.txt
# 或者手动安装：pip install requests pycryptodome
```

### 3. 准备 access token

> ⚠️ 本项目**不包含登录/授权流程**，需要你自行提供 App 的 `Access-Token`（形如 `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`）。通常可在已登录的联通云盘 App 发出的请求头中找到。

Token 等同账号凭证，**切勿提交到仓库、写进代码或分享给他人**。三种配置方式任选其一：

```bash
# 方式一：命令行参数
python3 ltpan.py --token xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx info

# 方式二：环境变量（推荐）
export WOPAN_TOKEN=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

# 方式三：配置文件 ~/.wopan.json
# {"access_token": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"}
```

全局参数 `--token` / `--proxy` / `--device-id` 也可以写在子命令后面，例如 `python3 ltpan.py ls 0 --token xxxx`。

### 4. 跑起来

```bash
python3 ltpan.py info          # 查看账号容量信息，验证 token 是否可用
python3 ltpan.py ls 0          # 列出根目录，拿到目录 id
python3 ltpan.py upload ./movie.mkv
```

---

## 命令与示例

### `upload` — 上传文件

```bash
# 上传到根目录（默认 16MB 分片 / 4 并发 / 先尝试秒传）
python3 ltpan.py upload ./movie.mkv

# 上传到指定目录，32MB 分片、8 并发
python3 ltpan.py upload ./movie.mkv --dir 3083ba2e47774559a8dfef6a14dcd6ea --chunk 32 --workers 8

# 跳过秒传检查，强制走分片上传
python3 ltpan.py upload ./movie.mkv --no-quick

# 上传到家庭空间
python3 ltpan.py upload ./movie.mkv --space 1 --family-id 123456

# 走代理 / 自定义上传节点
python3 ltpan.py upload ./movie.mkv --proxy http://127.0.0.1:7890
python3 ltpan.py upload ./movie.mkv --host https://hyupload.pan.wo.cn
```

输出为 JSON，例如：

```json
{"fid": "a1b2c3...", "quick": false, "size": 4831838208, "name": "movie.mkv", "parts": 289}
```

`quick: true` 表示命中秒传；`fid` 是服务端返回的文件 ID。

### `ls` — 列出目录

```bash
python3 ltpan.py ls 0          # 0 表示根目录；目录 id 可从 ls 结果获得
```

```
类型     大小  fileId                            名称
目录       0B  3083ba2e47774559a8dfef6a14dcd6  我的电影
文件    1.2GB  c0ffee...                        movie.mkv
共 2 项
```

### `mkdir` / `rm` / `info`

```bash
python3 ltpan.py mkdir 0 我的电影     # 在根目录新建文件夹，输出目录 id
python3 ltpan.py rm <fileId> [<fileId> ...]   # 按 fileId 删除，可传多个
python3 ltpan.py info                 # 查询云盘容量使用情况
```

`mkdir` 省略第一个参数时默认在根目录创建：`python3 ltpan.py mkdir 我的电影`。

### 参数一览

**全局参数**

| 参数 | 说明 | 默认值 |
| --- | --- | --- |
| `--token` | access token | `WOPAN_TOKEN` 环境变量 → `~/.wopan.json` |
| `--device-id` | 请求头 `X-YP-Device-Id` | 随机生成 32 位 |
| `--proxy` | HTTP/HTTPS 代理地址 | 无 |

**`upload` 专用参数**

| 参数 | 说明 | 默认值 |
| --- | --- | --- |
| `--dir` | 目标目录 id | `0`（根目录） |
| `--chunk` | 分片大小（MB） | `16` |
| `--workers` | 并发线程数 | `4` |
| `--space` | `0` 个人空间 / `1` 家庭空间 | `0` |
| `--family-id` | 家庭空间 id（`--space 1` 时生效） | `0` |
| `--no-quick` | 跳过秒传检查 | 关闭 |
| `--host` | 自定义上传节点 | `https://hyupload.pan.wo.cn` |

---

## 作为 Python 库使用

```python
from ltpan import WoPan

cli = WoPan("你的 access token")

print(cli.info())                       # 容量信息

for f in cli.ls("0"):                   # 列目录
    print(f["id"], f["name"])

res = cli.upload(
    "./movie.mkv",
    dir_id="0",
    chunk_mb=16,
    workers=8,
    use_quick=True,
)
print(res)                              # {"fid": ..., "quick": ..., "size": ..., "name": ...}

did = cli.mkdir("0", "我的电影")        # 新建目录
cli.rm(["<fileId>"])                   # 删除文件
```

`WoPan` 构造参数：`token`、`device_id`、`client_id`、`proxies`、`timeout`。
另外提供 `quick_transfer()`、`refresh(refresh_token)` 等底层方法，其中 `refresh()` 尚未实测，仅供研究。

---

## 工作原理

脚本涉及三个服务端点，均为 App 实际使用的接口：

| 环节 | 端点 | 作用 |
| --- | --- | --- |
| 业务网关 | `https://s.pan.wo.cn/{channel}/dispatcher` | 查询文件、新建目录、删除、容量信息 |
| 秒传校验 | `https://b.smartont.net/openapi/transfer/quickTransfer` | 用 SHA-256 判断服务端是否已有同一文件 |
| 分片上传 | `https://hyupload.pan.wo.cn/openapi/client/upload2C` | 上传分片数据，服务端负责合并 |

**加解密**：dispatcher 的请求体 `param` 使用 AES-128-CBC 加密，密钥取 `token` 的前 16 字节、固定 IV `wNSOYIB1k1DjY5lA`，响应数据用同样方式解密；请求头中的 `sign` 为 `md5(key + resTime + reqSeq + channel + version)` 的大写值。

**上传流程**：

1. 计算文件 SHA-256，调用秒传接口；若 `hasFile=1` 且返回 `fileId`，直接完成。
2. 未命中则按 `--chunk` 切分，生成批次号与加密后的 `fileInfo`。
3. 以 multipart 并发 POST 每个分片（`partIndex` / `partSize`），单片失败按 2s、4s、6s 间隔重试。
4. 服务端在最后一片返回 `fid` 并完成合并，进度条输出总耗时与平均速度。

---

## 常见问题

**Q：速度没有明显提升？**
确认输出是否显示 `⚡ 秒传命中`（那样本来就不传数据）；检查 `--host` 是否被改到了网页端节点、`--workers` 是否过小、代理是否成为瓶颈。服务端策略也可能随时调整。

**Q：报 `xxxx 失败` / 解密异常 / token 无效？**
AES 密钥取自 token 前 16 位，token 格式不对或已过期会导致解密失败；确认使用的是 App 通道的 `Access-Token`，且长度不少于 16 位。

**Q：分片上传总是失败？**
适当调小 `--chunk`（如 8）、降低 `--workers`（如 2），或配置 `--proxy` 后重试。

**Q：支持断点续传 / 下载吗？**
目前**不支持**。中断后重新执行即可，若服务端已存有该文件则会命中秒传。下载不在本项目范围内。

**Q：token 过期了怎么办？**
`WoPan.refresh(refresh_token)` 实现了刷新逻辑但未实测，建议重新获取 access token。

---

## 注意事项与免责声明

- 本项目是**第三方非官方工具**，与联通云盘官方无关；接口、签名与限速策略随时可能变更，脚本可能因此失效。
- 请**遵守联通云盘的服务条款及当地法律法规**，仅用于上传和管理你有权处置的个人文件，不要用于分发侵权或违规内容，不要对服务端发起滥用性请求。
- `access token` 属于敏感凭证，泄露可能造成账号与数据风险，请自行妥善保管。
- 使用本项目所产生的任何账号限制、数据丢失或其他后果，由使用者自行承担；作者不对工具的可用性、稳定性与适用性作任何担保。
- 本项目仅供个人学习与技术研究使用。

---

## License

[MIT](LICENSE) © 2026 jeffreys125
