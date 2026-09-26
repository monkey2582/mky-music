# MKY音乐

一个**单文件 HTML 音乐播放器**。无广告、无需登录、无后端、无追踪。

双击 `index.html` 就能用，也可以把它扔到任何静态托管上。

**在线使用 →** https://mky-music.pages.dev

---

## 特性

**单文件交付**
整个播放器就是 `index.html` 一个文件，约 400KB，零构建、零依赖、零 `npm install`。拷进 U 盘、发到微信、丢进任何静态服务器，打开就是完整应用。

**逐字歌词 + 译文对齐**
同时解析网易云的三路歌词数据：

| 字段 | 内容 | 用途 |
| --- | --- | --- |
| `lrc` | 行级原文 | 基础歌词 |
| `yrc` | **逐字**原文（毫秒级时间戳） | 驱动进度光带精确到字 |
| `ytlrc` | 译文 | 显示在原文下方，按时间戳对齐 |

`yrc` 是网易云官方的逐字格式，格式为 `[行起始ms,行时长ms](字起始ms,字时长ms,0)字…`。有它就不必再按字符数均分瞎猜——唱得慢的字光带走得慢，唱得快的字光带走得快，与嘴型同步。

译文用 `±0.35s` 容差按时间戳对齐到原文行，避免"译文比原文快半行"的错位。

**自建 CORS 代理**
不依赖任何第三方公共代理（它们全死了，见下方「踩坑记录」）。歌词请求走自建的 [mky-proxy](https://github.com/monkey2582/mky-proxy)，部署在 Cloudflare Pages 上。

**UI**
暗色 / 亮色双主题（首帧同步读 `localStorage`，无白闪）、毛玻璃质感、移动端适配。

---

## 架构

```
┌─────────────────┐
│  index.html     │  播放器本体（单文件，浏览器直接跑）
└────────┬────────┘
         │
         ├───────────────► node.api.xfabe.com     音源 / 搜索 / 榜单 / 封面
         │                 （自带 Access-Control-Allow-Origin: *，可直连）
         │
         └───────────────► mky-proxy.pages.dev    逐字歌词（需绕 CORS）
                           （Cloudflare Pages Functions，带密钥）
                                    │
                                    ▼
                           music.163.com/api/song/lyric
                           （无 CORS 头，浏览器无法直连）
```

**为什么歌词必须走代理**
`music.163.com` 不返回 `Access-Control-Allow-Origin`，浏览器同源策略下无法直接读取。而音源接口 `node.api.xfabe.com` 自带 `*`，所以不需要代理——这是两套不同的通道。

---

## 本地运行

**方式一：直接打开**

双击 `index.html`，或拖进浏览器。

> 注意：`file://` 协议下，浏览器不发送 `Origin` 请求头。这不影响本播放器（我们用的是自建代理，不依赖 `Origin`），但会让某些需要 `Origin` 的公共服务失效。

**方式二：起个本地服务器**（推荐，行为与线上一致）

```bash
python3 -m http.server 8080
# 然后访问 http://localhost:8080
```

---

## 部署

### 方案 A：Cloudflare Pages（推荐）

1. Fork 或克隆本仓库
2. Cloudflare Dashboard → **Workers & Pages** → **Create** → **Pages** → **Connect to Git**
3. 选择本仓库，构建配置全部留空：

   | 字段 | 值 |
   | --- | --- |
   | Framework preset | None |
   | Build command | *留空* |
   | Build output directory | *留空* |

4. Save and Deploy，约 10 秒后上线

因为 `index.html` 就在仓库根目录，Cloudflare 会把根目录直接作为产物发布，无需任何构建步骤。

### 方案 B：任何静态托管

GitHub Pages / Netlify / Vercel / 对象存储 / 自己的 nginx —— 把 `index.html` 放上去就行，没有服务端依赖。

---

## 配置

播放器顶部有两个常量需要按你的环境调整：

```js
// 自建代理地址。末尾不要带 ?url=，代码会自动拼接
const CF_PROXY = 'https://mky-proxy.pages.dev/api';

// 代理访问密钥。必须与 mky-proxy 里 functions/api.js 的 AUTH_KEY 一致
const CF_PROXY_KEY = 'your_key_here';
```

**换密钥的完整流程**

1. 打开 [mky-proxy](https://github.com/monkey2582/mky-proxy) → `functions/api.js` → 修改 `AUTH_KEY` 的值
2. 提交 → Cloudflare 自动重新部署（约 1 分钟）
3. 回到本仓库 `index.html` → 把 `CF_PROXY_KEY` 改成同一个值 → 提交 → 自动重新部署

两边都连着 Git，全程不需要碰命令行。

**为什么需要密钥**
`mky-proxy` 放行所有域名（真正意义上的"跳过所有 CORS"）。如果不设密钥，任何人扫到 `*.pages.dev` 就能把它当作免费跳板，刷爆每日额度。密钥是一道止损开关。

---

## 使用须知

- 音源、搜索、榜单、封面均来自第三方接口 `node.api.xfabe.com`，**本仓库不包含任何音频文件**，也不存储任何内容。
- 音频 URL 由接口动态签名（带时间戳），有有效期，播放器会在获取后缓存到内存。
- 仅供个人学习与技术研究使用。请遵守相关服务条款，勿用于商业用途。

---

## 踩坑记录

这些是开发过程中实测出来的结论，记下来供后来者少走弯路。

### `workers.dev` 在中国大陆被 DNS 污染

`*.workers.dev` 被解析到 `2a03:2880:...:face:b00c` —— 一个 Facebook 的 IPv6 地址段。直连必然超时。

而 `*.pages.dev` 解析正常，可直接访问。**这就是本项目用 Pages 而不是 Worker 的原因。**

判断方法：`workers.dev` 解析结果落在 Facebook 段（`157.240.x.x` / `2a03:2880::/32`）即为污染。

### 公共 CORS 代理全军覆没

2026-09 逐个实测 12 个，无一可用：

| 代理 | 死因 |
| --- | --- |
| corsfix | 需注册 token；`file://` 下 `400 invalid_origin`，https 域名下 `403`。两条路都堵死 |
| cors.sh / cors.x2u.in / thingproxy | `TypeError` |
| whateverorigin | `HTTP 200`，但返回它自己的 HTML 主页，不是转发结果 |
| test.cors.workers.dev | 必超时（`workers.dev` 被污染的又一例证） |
| cors.eu.org / codetabs / corsproxy.io | 域名解析失败 / 超时 / `401` |
| cors.lol | 曾唯一可用（1.2~2.4s），后按 IP 限流，连续几首即 `Rate limit exceeded` |
| allorigins | 探活就要 15s，`/raw` 时好时坏，`/get` 多包一层 `{contents:"..."}` |

**结论：不要指望公共 CORS 代理。** 自建一个 Pages Function 只要 30 行代码，且免费额度 10 万次/天。

细节：`cors.lol` 的地址是 `api.cors.lol?url=`，**没有斜杠**。写成 `api.cors.lol/?url=` 会 `net::ERR_FAILED`，这个失败长得很像"代理挂了"，极易误判。

### `file://` 下无法设置 `Origin`

`file://` 协议不发送 `Origin` 请求头，且 `Origin` 是浏览器**禁止 JS 修改**的请求头之一。任何依赖 `Origin` 鉴权的代理，在本地打开时都无法工作。

### Cloudflare Pages Direct Upload 不编译 Functions

用 API 直接上传文件（Direct Upload / `ad_hoc` 项目）时，**Functions 不会被编译**，`uses_functions` 恒为 `false`，即使 `functions/api.js` 确实出现在文件清单里。

只有 Git 连接的项目才会启用 Functions。且 Direct Upload 项目**无法通过 API 转换为 Git 项目**（返回 `8000069`）。

**所以：需要 Functions 就必须走 Git 部署。**

### Pages 部署 API 的两个坑

若用 API 做 Direct Upload，注意：

1. `manifest` 必须是**内联 JSON 字符串**表单字段，当文件上传会报 `8000096`
2. manifest 的 key **不能有前导斜杠** —— `"index.html"` ✅，`"/index.html"` ❌（会变成字面路径 `//index.html`，站点 404 但 `files` 列表看起来正常）

---

## 许可

MIT
