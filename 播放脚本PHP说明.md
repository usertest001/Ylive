# 播放脚本 PHP 编写说明

与 [播放脚本 JS 编写说明](./播放脚本JS编写说明.md) 并列：PHP 用 **顶层脚本 + HTTP 输出**，不用入口函数。

## 地址格式

| 用途 | 示例 |
|------|------|
| 频道行 | `php://demo.php?id=cctv1HD` |
| 整源列表 | `php://demo.php?id=list`（或 `?list=txt` 等自定参数） |
| 脚本回看 | `php://demo.php?id=cctv1HD&catchup=php` |
| 节目单（按天） | `php://epg.php`（可省略查询串；App 自动补 `ch`/`date`）→ 输出与 HTTP 按天 EPG 相同的 JSON |

脚本放在设备 `scripts/`（可用 Web 管理页上传 `.php`，或从 URL 拉取）。

## 脚本约定

1. **无入口函数**：文件从顶层执行。
2. **参数**：query → `$_GET`（含 `id`、`catchup`、`start`、`end`）。
3. **结果（换台播放）**  
   - 有 `header('Location: …')` → 用该地址播放；  
   - 正文以 `#EXTM3U` 开头 → 本机 HTTP 刷新播；  
   - JSON 含可播 `url` → 用该字段；整段绝对 URL 正文亦可；  
   - JSON `error` 或其它非 URL 正文 → 报错（不会把错误文案当播放地址）。  
   - 需要特定播放 UA（如央视 CDN）时：`header('X-Ylive-User-Agent: qqlive')`，或 JSON `header`/`headers` 里写 `User-Agent`。  
   浏览器调试 `http://电视:8888/脚本.php?list=txt` 会**原样显示**脚本输出；列表里的 `http://php.local/…` 在 App 拉源时会改成 `php://…`。
4. **缓存**：需要时自行写文件。`sys_get_temp_dir()` 指向设备 `files/php-tmp/`；也可写 `__DIR__` 下子目录（如 `script_cache`）。Web「脚本」页有 **PHP 运行时** 可查看 / 清除临时、Session、脚本旁目录（不会删 `.php` 本身）。
5. **回看**：线路带 `catchup=php` 时，App 会注入当前节目的 `start`/`end`（Unix 秒），由脚本拼回看地址。

样例见仓库 `play-scripts/`（上传到设备后可用 `php://demo.php?...`）。

## 与服务器脚本的差异

多数门户 PHP 可直接拷到设备，把直播源里的 `http://服务器/xxx.php?id=` 改成 `php://xxx.php?id=`。  
注意：无外部 MySQL；相对路径缓存目录需脚本自己 `mkdir`。
