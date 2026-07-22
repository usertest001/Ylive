# 播放 JS 脚本编写说明

本文说明如何在 **Y影音** 中编写、上传、使用「解析脚本」：  
- **频道行**：换台时由 App 用 QuickJS 执行脚本，得到真实播放地址再交给播放器。  
- **整源**：添加直播源链接为 `js://脚本.js?id=list` 时，拉源执行脚本，把返回文本解析为频道列表。

适合：需要按频道 ID 动态向门户/接口要流地址的场景（例如门户签名接口）。

---

## 1. 整体流程

```text
【整源】添加源：js://脚本.js?id=list
        ↓ QuickJS 返回 txt/m3u 正文 → 解析成多台（频道行常仍是 js://...?id=xxx）

【频道行】换台：js://脚本.js?id=频道ID
        ↓
App 读取 files/scripts/你的脚本.js
        ↓
QuickJS 执行 → main / resolve（或 fn=）
        ↓
返回 http(s) 播放地址 → 播放器拉流
```

- **不是**网页里打开的脚本（没有 `window` / 页面跳转）。
- 每次换台都会执行一次；是否复用旧地址由**脚本自己**用 `localStorage` / `cache` 决定。
- 整源列表会按普通直播源规则做磁盘缓存；改脚本后若未见更新，可清直播源缓存再刷新。

---

## 2. 直播源怎么写

### 2.1 整源：一个 `js://` 展开成频道列表

添加直播源时，链接可填：

```text
js://zg4k.js?id=list
```

App 拉源时执行脚本，把返回的**字符串当作 txt/m3u 列表正文**解析（与 http 源相同）。  
脚本应对 `id=list`（或你约定的参数）`return` 多行文本，例如：

```js
async function main(id) {
  if (id === "list") {
    return [
      "4K卫视,#genre#",
      "北京卫视4K,js://zg4k.js?id=btv4k",
      "浙江卫视4K,js://zg4k.js?id=zj4k",
    ].join("\n");
  }
  // ... 按频道 id 返回单条流地址
}
```

须先在 Web「脚本」页上传对应 `.js`。

### 2.2 频道行：换台时再解析

在 m3u / txt 直播源中，频道地址写成：

```text
CCTV1,js://demo.js?id=cctv1HD
CCTV2,js://demo.js?id=cctv2HD
```

| 部分 | 含义 |
|------|------|
| `js://` | 协议：走脚本解析 |
| `demo.js` | 脚本文件名（须已上传到设备） |
| `id=...` | 传给脚本的频道/节目 ID（常用） |
| `&fn=函数名` | 可选，指定入口函数；不写则按下方规则自动探测 |

也可带线路备注（与现有 IPTV 习惯一致）：

```text
CCTV1,js://demo.js?id=cctv1HD$线路1
```

`$` 后面不会进入脚本，只作显示名。

**指定入口函数示例：**

```text
js://demo.js?id=xxx&fn=resolve
```

---

## 3. 上传脚本

1. 手机/电脑浏览器打开：`http://<电视IP>:9876/advance`
2. 进入 **脚本**
3. 上传 `.js` 文件，或粘贴内容保存

脚本保存在设备：`应用私有目录/files/scripts/`  
文件名仅允许：`字母数字 . _ -`，且以 `.js` 结尾（如 `demo.js`）。

---

## 4. 入口函数约定

脚本在全局定义函数即可。App 按以下顺序查找默认入口：

1. URL 参数 `fn=` 指定的函数名（可选覆盖）  
2. `main`（推荐）  
3. `resolve`  

不写 `fn=` 时，只需提供 `main` 或 `resolve` 之一即可；名称不在上述列表时，须用 `&fn=函数名` 显式指定。

### 调用方式

- 若有 `id` 参数：`await 入口函数(id字符串)`
- 若无 `id`：`await 入口函数(params对象)`，`params` 为全部 query（如 `{ id: "...", fn: "..." }`）

### 返回值

任选其一：

```js
return "https://example.com/live.m3u8";           // 推荐
return { url: "https://example.com/live.m3u8" }; // 也可
```

失败请抛错：

```js
throw new Error("获取播放地址失败");
```

App 会提示错误，不会拿空地址去播。

### 最小示例

```js
async function main(id) {
  var res = await fetch("https://example.com/api/play?id=" + encodeURIComponent(id));
  var data = await res.json();
  if (!data.url) throw new Error(data.msg || "无播放地址");
  return data.url;
}
```

直播源：`测试台,js://demo.js?id=1001`

---

## 5. 运行环境里有什么

基于 **QuickJS**，支持常见现代 JS（含 `async` / `await`、`Promise`、`JSON`、`Date`、`encodeURIComponent` 等）。

### 5.1 `fetch`（已注入，接近浏览器）

```js
var response = await fetch(url, {
  method: "GET",           // 可选，默认 GET；也支持 POST / PUT / PATCH / DELETE
  headers: {
    "Content-Type": "application/json",
    // 可自定义头，如 appId、sign、timestamps
  },
  body: "...",             // POST 等时可用
});

var text = await response.text();
var data = await response.json();
// response.ok、response.status、response.url 可用
```

由 App 用 OkHttp 代发请求（含跟随重定向）。**没有**浏览器 Cookie 罐、**没有** CORS 限制。

### 5.2 `console`

```js
console.log("调试信息");
console.warn("警告");
console.error("错误");
```

输出到 App 的 logcat（标签相关日志中可见）。

### 5.3 `localStorage`（磁盘持久化）

与浏览器用法相同，数据写在设备 `files/script-kv/`，**杀进程、换台后仍在**。

```js
localStorage.setItem("demo_cache_" + playId, JSON.stringify({
  url: playUrl,
  time: Date.now(),
}));

var raw = localStorage.getItem("demo_cache_" + playId);
localStorage.removeItem("demo_cache_" + playId);
localStorage.clear(); // 清空全部脚本 KV（慎用）
```

适合：`demo.js` 这类「按 ID 缓存播放地址、自定过期时间」的逻辑。

### 5.4 `cache`（同一套存储，另一种写法）

与 `localStorage` **共用** `files/script-kv/`，key 空间相同。

```js
cache.write("mykey", "内容");
var v = cache.read("mykey");      // 没有则 null / undefined
cache.exists("mykey");            // true / false
cache.remove("mykey");
```

### 5.5 配额

| 项 | 限制 |
|----|------|
| 单个 key 长度 | ≤ 128 字符 |
| 单条 value | ≤ 256 KB |
| 全部合计 | ≤ 16 MB |

超出时 `setItem` / `cache.write` 会抛错，请在脚本里 `try/catch`。

### 5.6 没有的东西

以下**不要**依赖（App 未提供）：

- `window` / `document` / `location` / `URLSearchParams`（浏览器段可保留，App 不会执行）
- Node 的 `fs`、`require`
- 任意路径读写文件
- 调用 Android / Java API

浏览器专用入口可这样包起来（App 会跳过）：

```js
if (typeof window !== "undefined") {
  // 仅浏览器调试用
}
```

---

## 6. 缓存建议

1. **推荐脚本自己缓存**（`localStorage` 或 `cache`），自行决定过期时间。  
2. App **不会**再自动缓存「解析出的播放地址」。  
3. 清除缓存：Web → **脚本** →「清除全部缓存」。

示例（约 23 小时）：

```js
var cacheTime = 23 * 60 * 60 * 1000;
var cachePrefix = "demo_cache_";

function getCachedPlayUrl(playId) {
  try {
    var cache = JSON.parse(localStorage.getItem(cachePrefix + playId) || "{}");
    if (cache.url && cache.time && Date.now() - cache.time < cacheTime) {
      return cache.url;
    }
  } catch (e) {
    localStorage.removeItem(cachePrefix + playId);
  }
  return "";
}

function setCachedPlayUrl(playId, playUrl) {
  try {
    localStorage.setItem(
      cachePrefix + playId,
      JSON.stringify({ url: playUrl, time: Date.now() }),
    );
  } catch (e) {
    localStorage.removeItem(cachePrefix + playId);
  }
}

async function main(playId) {
  var cached = getCachedPlayUrl(playId);
  if (cached) return cached;
  // ... fetch 拿地址 ...
  setCachedPlayUrl(playId, playUrl);
  return playUrl;
}
```

---

## 7. 超时与换台

- 单次脚本解析最长约 **20 秒**，超时会失败提示。  
- 解析过程中换台，旧请求会取消，避免串台。  
- 每次执行是**新的** JS 环境：全局变量不会跨换台保留；要持久化请用 `localStorage` / `cache`。

---

## 8. 安全注意（写脚本的人）

- 脚本里的 `appId`、`secret` 等会明文存在设备上，勿把含密钥的脚本随意传播。  
- `fetch` 可访问任意 URL（含局域网），请只写可信逻辑。  
- 局域网 Web 上传脚本当前无登录鉴权，请在可信网络下管理脚本。

---

## 9. 调试 checklist

1. Web「脚本」页确认文件已上传且文件名与 `js://` 一致。  
2. 直播源行为：`js://demo.js?id=...`。  
3. logcat 过滤：`MainContentState` / `JsPlayScript`  
   - `JS 脚本解析开始`  
   - `JS 脚本解析成功: http...`  
   - 或 `JS 脚本解析失败` + 错误信息。  
4. 若用了缓存，第二次同 ID 换台应明显更快；Web「脚本缓存」里能看到对应 key。

---

## 10. 完整能力一览（速查）

```text
协议：  js://demo.js?id=...&fn=可选
        整源：js://demo.js?id=list（返回列表正文）
上传：  Web → 脚本
入口：  fn → main → resolve
返回：  string 或 { url }（整源时为 txt/m3u 文本；频道行为流地址）
网络：  fetch / response.text() / response.json()
日志：  console.log / warn / error
缓存：  localStorage.*  或  cache.read/write/remove/exists
落盘：  files/scripts/*.js
        files/script-kv/   （缓存）
清除：  Web 脚本页
```
