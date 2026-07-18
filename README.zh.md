# Hermes 搜索生存指南

[English](README.md) | [中文](README.zh.md)

给 [Hermes Agent](https://hermes-agent.nousresearch.com) 的一套完整、实战验证过的搜索配置。
别再调试那些永远不会工作的搜索引擎了。直接用这两个能跑的。

---

## 引擎生死表

在自建 SearXNG 实例上经过数周测试：

| 引擎 | 状态 | 结果 | 结论 |
|------|:----:|:----:|------|
| **bing** | ✅ | ~10条/次 | **用这个。** 需要 `timeout: 15.0`。 |
| **mojeek** | ✅ | ~9-10条/次 | **用这个。** 独立索引，几乎不会被封。 |
| brave | ❌ | 0 | 已死。自建实例被永久封禁。 |
| duckduckgo | ❌ | 0 | 已死。所有自建实例触发 CAPTCHA。 |
| startpage | ❌ | 0 | 已死。所有自建实例触发 CAPTCHA。 |
| presearch | ❌ | 0 | 已死。拒绝访问。 |
| qwant | ❌ | 0 | 已死。拒绝访问。 |
| google | 🔒 | 0 | 需要付费的 Google Custom Search API key。 |

**结论**：7 个引擎，活下来 2 个。每个自建 SearXNG 的人都得自己踩一遍。这个仓库就是捷径。

---

## 仓库内容

```
hermes-search-survival-guide/
├── README.md                              ← 英文版
├── README.zh.md                           ← 你在这里
├── AGENTS.md                              ← AI agent 自动加载
├── docker-compose.yml                     ← Hermes + SearXNG + Valkey
├── config/
│   └── config.yaml.template               ← Hermes 搜索配置模板
├── docker/
│   └── searxng/
│       └── settings.yml                   ← 完整 2782 行配置，能跑
├── skills/
│   ├── hermes-web-search-debugging/       ← 搜索故障诊断
│   │   ├── SKILL.md
│   │   └── references/
│   ├── web-search-strategy/               ← 多轮搜索方法论
│   │   ├── SKILL.md
│   │   └── references/
│   └── searxng-config/                    ← SearXNG 配置指南（独立使用）
│       ├── SKILL.md
│       └── templates/
│           └── settings.yml               ← 最小化配置：只开 2 个能用的引擎
├── .gitignore
└── LICENSE
```

---

## 快速开始

### 前置条件

- Docker + Docker Compose
- 已安装 [Hermes Agent](https://hermes-agent.nousresearch.com)（`~/.hermes/` 目录存在）
- 端口 8085 未被占用

### 1. 克隆

```bash
git clone https://github.com/wu1chenghui/hermes-search-survival-guide.git
cd hermes-search-survival-guide
```

### 2. 添加搜索配置到 ~/.hermes

**如果你已经有 `~/.hermes/config.yaml`**（大多数 Hermes 用户都有），在现有内容下添加这几行：

```yaml
web:
  search_backend: "searxng"
  extract_backend: "firecrawl"
```

**如果是全新安装**，复制模板：

```bash
cp config/config.yaml.template ~/.hermes/config.yaml
```

然后复制 skills：

```bash
mkdir -p ~/.hermes/skills
cp -r skills/hermes-web-search-debugging ~/.hermes/skills/
cp -r skills/web-search-strategy ~/.hermes/skills/
cp -r skills/searxng-config ~/.hermes/skills/
```

### 3. 复制 SearXNG 设置

```bash
mkdir -p docker-data/searxng/config docker-data/searxng/cache
cp docker/searxng/settings.yml docker-data/searxng/config/
```

如果只想用最小化配置（bing + mojeek + 非网页引擎）：
```bash
cp skills/searxng-config/templates/settings.yml docker-data/searxng/config/
```

### 4. 启动

```bash
docker compose up -d
```

### 5. 安装 ddgs（可选，双后端方案的第二个后端）

```bash
docker compose exec hermes bash -c "uv pip install --target /opt/data/pip-packages ddgs h2 httpcore brotli fake-useragent lxml primp"
docker compose exec hermes bash -c "uv pip install --target /opt/data/pip-packages 'scrapling[all]'"
```

### 6. 验证

```bash
curl -s --max-time 30 "http://localhost:8085/search?q=test&format=json" | python3 -c "
import sys,json
d=json.load(sys.stdin)
print(f'结果数: {len(d.get(\"results\",[]))} | 无响应引擎: {d.get(\"unresponsive_engines\",[])}')
# 预期: 17-20 条结果，无响应列表为空或只有 presearch
"
# 如果结果为 0 或报错，参见下方「已知陷阱」。
```

你的 Hermes 容器已经配置了 `SEARXNG_URL` 和 `search_backend: searxng`。
现在所有 web_search 调用都通过 SearXNG。不需要额外配置。

---

## 包含的 Skills

| Skill | 功能 |
|-------|------|
| `hermes-web-search-debugging` | 诊断任何 web_search 故障——8 个后端全覆盖、静默回退检测、引擎健康检查 |
| `web-search-strategy` | 四阶段搜索方法论、通过 Scrapling 绕过 Cloudflare、搜索预算控制 |
| `searxng-config` | 独立的 SearXNG 配置指南——引擎可靠性表、最小化配置、诊断命令 |

---

## 架构

```
┌─────────────┐     ┌───────────────┐     ┌──────────┐
│   Hermes    │────▶│  searxng-core │────▶│   bing   │
│  (Docker)   │     │   (Docker)    │     │  mojeek  │
│             │     │    :8080      │     │  arxiv   │
│             │     └───────┬───────┘     │   ...    │
└──────┬──────┘             │             └──────────┘
       │              ┌─────▼──────┐
       │              │   valkey   │
       ▼              │  (cache)   │
  ~/.hermes/          └────────────┘
  config.yaml
  skills/
  pip-packages/
```

---

## 双后端策略

本仓库支持两个互补的搜索后端：

| 后端 | 部署方式 | 稳定性 | 结果数 | 使用场景 |
|------|---------|:------:|:------:|---------|
| **SearXNG** | Docker（已包含） | ⭐⭐⭐⭐⭐ | 17-20 | 默认。零限速。 |
| **ddgs** | `pip install ddgs` | ⭐⭐⭐ | 3-5 | 备用。不需要 Docker。约 25% 空结果率。 |

切换（编辑 `~/.hermes/config.yaml`）：

```yaml
web:
  search_backend: "searxng"   # 稳定，推荐
  # search_backend: "ddgs"     # 零基础设施备用方案
```

---

## 网页内容提取 (web_extract)

SearXNG 只处理 `web_search`，不处理 `web_extract`（SearXNG 不返回完整页面内容）。
默认的提取后端是 **Firecrawl**，Hermes 自带，提供**无 key 免费层**（1000 credits/月，
不需要注册）。开箱即用。

如果用完免费额度或需要更高质量的提取：

```bash
# 方案 A：在 ~/.hermes/.env 中添加 Firecrawl API key
# （extract_backend: firecrawl 已经是默认值）
echo 'FIRECRAWL_API_KEY=fc-your-key' >> ~/.hermes/.env

# 方案 B：用浏览器作为后备（始终免费，不需要 key）
# hermes-web-search-debugging skill 中记录了 browser_navigate 作为
# web_extract 替代方案的使用方法，适用于 arxiv、docs.python.org、github、wikipedia。
```

详细诊断：加载 `hermes-web-search-debugging` skill。

---

## 已知陷阱（调试前请先读）

### 1. 静默回退陷阱

如果在 `config.yaml` 里写 `search_backend: ddgs` 但 ddgs 没安装，Hermes 不会报错——
它会静默回退到 firecrawl。搜索仍然能跑，你永远不会发现配置是错的。检查实际使用的是哪个后端：

```python
from agent.web_search_registry import get_active_search_provider
print(get_active_search_provider().name)
```

### 2. ddgs 的 LAZY_DEPS 缺口

Hermes 对 firecrawl、exa、parallel 会自动安装 SDK，但 ddgs 不在自动安装白名单里。
Docker 重建后 ddgs 静默消失。修复方案：把包持久化到 `/opt/data/pip-packages/` + `PYTHONPATH`。

### 3. Cloudflare 封锁 AI Agent

2025 年 7 月起，Cloudflare 封锁 AI agent 流量。`browser_navigate` 和 `web_extract`
在 Cloudflare 保护的网站上都会静默失败（Indeed、Glassdoor、Crunchbase 等）。
使用 Scrapling + Playwright Chromium 作为后备方案。

### 4. SearXNG 密钥问题

`settings.yml` 中的 `secret_key` 被有意注释掉了。SearXNG 理应在首次启动时自动生成一个，
但这取决于 Docker entrypoint 脚本版本。如果重启 SearXNG 后遇到 session 错误或 CSRF 拒绝，
生成一个固定密钥：

```bash
# 在 docker-compose.yml 中 searxng-core → environment 下添加：
#   SEARXNG_SECRET: a94a8fe5ccb19ba61c4c0873d391e987982fbbd3
# （用 openssl rand -hex 32 自己生成一个）
```

---

## 不包含的内容

- API keys（`.env`）——用你自己的
- 运行时数据（`sessions/`、`memories/`、`state.db`）
- 项目专属 skills（本仓库只包含搜索相关）
- Playwright Chromium 下载（一次性约 114MB，见快速开始）

---

## 许可证

MIT
