# 氢CMS 服务器监控上报 API 开发调用文档

> 版本：1.0　|　文档更新时间：2026-10-21
> 适用对象：后期的**统一管理系统（管理中心）** 开发人员
> 作用：远程读取每台被管节点的监控数据，用于集中展示、告警与审计。

---

## 1. 接口总览

| 接口 | 方法 | 说明 |
|------|------|------|
| 完整上报 | `GET /api.php` | 返回节点全部监控数据（server / php / disks） |
| 心跳探测 | `GET /api.php?ping=1` | 轻量上线状态检测，用于节点存活轮询 |
| 密钥鉴权 | `?key=` 或 Header `X-API-Key` | 可选鉴权方式，需在节点启用密钥 |

- 节点文件位置：节点站点根目录下的 `api.php`（依赖 `modules/collect.php` 采集层）。
- 所有请求均返回 JSON，编码为 `UTF-8`（`JSON_UNESCAPED_UNICODE`，中文不转义）。

### 基础地址

```
https://<节点域名或IP>/api.php
```

示例：`https://monitor.example.com/api.php`

---

## 2. 通用约定

### 2.1 请求头

接口已开启跨域支持，管理中心可在任意域名调用：

```
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, POST, OPTIONS
Access-Control-Allow-Headers: Content-Type, X-API-Key
```

如管理中心跨域请求，浏览器会先发送 `OPTIONS` 预检，节点返回 `204`，随后正常请求即可。

### 2.2 数据建议

- 上报数据含 `Cache-Control: no-store`，不允许被缓存，保证读到实时数据。
- 心跳接口不执行完整系统采集，开销极小，适合高频轮询（建议 30~60 秒/次）。

### 2.3 时间字段

- `timestamp`：Unix 时间戳（秒）。建议管理中心以此字段判断数据新旧，超时（如 >120 秒）则判定节点失联。

---

## 3. 接口一：完整上报数据

### 3.1 请求

```
GET /api.php
GET /api.php?key=<共享密钥>              # 若启用鉴权
```

可选鉴权方式二：请求头

```
X-API-Key: <共享密钥>
```

### 3.2 响应示例

```json
{
  "success": true,
  "api": "hcms/1.0",
  "id": "node-heicat-001",
  "node_name": "氢CMS-主服务器",
  "platform": "Windows",
  "timestamp": 1792512719,
  "generated_at": "2026-10-21 00:11:59",
  "uptime_seconds": 13980,
  "server": {
    "server_name": "HEICAT",
    "os": "Windows NT 10.0",
    "architecture": "AMD64",
    "uptime": "0天 3小时 53分",
    "cpu_cores": "16",
    "load_avg": "44, 44, 44",
    "mem_total": "15547 MB",
    "mem_used": "13853 MB",
    "mem_free": "1694 MB",
    "swap_total": "N/A",
    "swap_used": "N/A",
    "swap_free": "N/A",
    "net_rx": 1526.62,
    "net_tx": 280.73
  },
  "php": {
    "php_version": "8.5.8",
    "php_sapi": "apache2handler",
    "zend_version": "4.5.8",
    "memory_limit": "128M",
    "max_execution_time": "30",
    "upload_max_filesize": "2M",
    "post_max_size": "8M"
  },
  "disks": [
    {
      "name": "C盘",
      "total": 476.83,
      "used": 253.1,
      "free": 223.73,
      "percent": 53.1
    }
  ]
}
```

### 3.3 顶层字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `success` | bool | 是否成功。失败为 `false` 并有 `error` 字段 |
| `api` | string | API 版本标识，固定 `hcms/1.0` |
| `id` | string | 节点唯一标识（部署时在节点配置） |
| `node_name` | string | 节点显示名称 |
| `platform` | string | 节点平台：`Windows` 或 `Unix/Linux` |
| `timestamp` | int | 上报时间（Unix 秒） |
| `generated_at` | string | 上报时间的格式化字符串 |
| `uptime_seconds` | int | 运行时间（秒），用于管理中心计算百分比等 |
| `server` | object | 服务器资源信息（见 3.4） |
| `php` | object | PHP 运行环境（见 3.5） |
| `disks` | array | 磁盘分区列表（见 3.6） |

### 3.4 `server` 资源字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `server_name` | string | 主机名 |
| `os` | string | 操作系统及内核版本 |
| `architecture` | string | CPU 架构（如 x86_64 / AMD64） |
| `uptime` | string | 运行时间（人类可读："X天 X小时 X分"） |
| `cpu_cores` | string | CPU 逻辑核心数（"16"） |
| `load_avg` | string | 系统负载（1, 5, 15 分钟，逗号分隔；Windows 下为 CPU 占用近似值） |
| `mem_total` | string | 总内存（MB，如 "15547 MB"，值可能为 "N/A"） |
| `mem_used` | string | 已用内存（MB） |
| `mem_free` | string | 空闲内存（MB） |
| `swap_total` | string | Swap 总大小（MB，Windows 为 "N/A"） |
| `swap_used` | string | Swap 已用（MB） |
| `swap_free` | string | Swap 空闲（MB） |
| `net_rx` | number/string | 网络累计接收流量（MB）；未获取为 "N/A" |
| `net_tx` | number/string | 网络累计发送流量（MB）；未获取为 "N/A" |

> 说明：内存/负载等值可能为 `"N/A"` 字符串，管理中心解析时应做容错。

### 3.5 `php` 环境字段

| 字段 | 说明 |
|------|------|
| `php_version` | PHP 版本号 |
| `php_sapi` | PHP 运行方式（如 cgi-fcgi / apache2handler） |
| `zend_version` | Zend 引擎版本 |
| `memory_limit` | PHP 内存限制 |
| `max_execution_time` | 最大执行时间（秒） |
| `upload_max_filesize` | 上传限制 |
| `post_max_size` | POST 数据限制 |

### 3.6 `disks` 磁盘数组

每个元素：

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | 分区/挂载点（如 "C盘"、"/"、"/data"） |
| `total` | string/number | 总量（GB），Linux 可能为 "1.5G" 字符串 |
| `used` | string/number | 已用（GB） |
| `free` | string/number | 剩余（GB） |
| `percent` | number | 使用率百分比（0~100） |

---

## 4. 接口二：心跳探测

### 4.1 请求

```
GET /api.php?ping=1
```

### 4.2 响应示例

```json
{
  "success": true,
  "api": "hcms/1.0",
  "id": "node-heicat-001",
  "node_name": "氢CMS-主服务器",
  "status": "online",
  "timestamp": 1792512719
}
```

### 4.3 说明

- `status` 固定为 `online`，请求成功即表示节点在线。
- 该接口不执行完整采集，开销极小，适合管理中心定期保存用。
- 建议同时依据 `timestamp` 判断数据新鲜度。

---

## 5. 鉴权与安全

### 5.1 开启鉴权

在节点 `api.php` 顶部修改常量：

```php
const HCMS_NODE_ID   = 'node-heicat-001';   // 节点唯一标识
const HCMS_NODE_NAME = '氢CMS-主服务器';      // 节点显示名称
const HCMS_API_KEY   = '';                  // 为空则不校验；不为空则 ?key= 必须匹配
```

- `HCMS_API_KEY` 设置为非空字符串即启用密钥校验。
- `HCMS_NODE_ID` / `HCMS_NODE_NAME` 请在各节点配置为唯一值，作为管理中心的节点区分依据。

### 5.2 鉴权失败

- HTTP 状态码 `401`。
- 返回体：`{"success":false,"error":"unauthorized"}`。

### 5.3 安全建议

1. 生产环境务必设置 `HCMS_API_KEY`。
2. 管理中心使用 HTTPS 访问，避免密钥与数据明文泄露。
3. 管理中心应对请求结果校验 `success === true` 后再入库。
4. 各节点需保证 `api.php` 与 `modules/` 目录对外可访问（其余文件不对外暴露）。

---

## 6. 管理中心对接代码示例

### 6.1 PHP（cURL）

```php
<?php
function fetchNodeData($url, $apiKey = '') {
    $ch = curl_init($url);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_TIMEOUT, 10);
    if ($apiKey !== '') {
        curl_setopt($ch, CURLOPT_HTTPHEADER, array('X-API-Key: ' . $apiKey));
    }
    $resp = curl_exec($ch);
    $err  = curl_error($ch);
    curl_close($ch);
    if ($resp === false) return array('success' => false, 'error' => $err);
    return json_decode($resp, true);
}

$data = fetchNodeData('https://monitor.example.com/api.php', '你的密钥');
if ($data && $data['success'] === true) {
    echo '节点：' . $data['node_name'] . PHP_EOL;
    echo 'CPU核数：' . $data['server']['cpu_cores'] . PHP_EOL;
    echo '总内存：' . $data['server']['mem_total'] . PHP_EOL;
} else {
    echo '拉取失败：' . (isset($data['error']) ? $data['error'] : '无法解析响应') . PHP_EOL;
}
```

### 6.2 Node.js

```javascript
const https = require('https');

function fetchNodeData(host, apiKey = '') {
  const headers = {};
  if (apiKey) headers['X-API-Key'] = apiKey;
  return new Promise((resolve, reject) => {
    const req = https.get({ host, path: '/api.php', headers }, res => {
      let body = '';
      res.on('data', c => body += c);
      res.on('end', () => {
        try { resolve(JSON.parse(body)); }
        catch (e) { reject(new Error('JSON 解析失败: ' + e.message)); }
      });
    });
    req.on('error', reject);
    req.setTimeout(10000, () => req.destroy('timeout'));
  });
}

fetchNodeData('monitor.example.com', '你的密钥')
  .then(d => console.log(`节点 ${d.node_name} 在线，CPU ${d.server.cpu_cores} 核`))
  .catch(e => console.error('拉取失败', e.message));
```

### 6.3 Python

```python
import requests

def fetch_node_data(url, api_key=""):
    headers = {"X-API-Key": api_key} if api_key else {}
    r = requests.get(url, headers=headers, timeout=10)
    r.raise_for_status()
    return r.json()

data = fetch_node_data("https://monitor.example.com/api.php", "你的密钥")
if data.get("success"):
    print(f"节点：{data['node_name']}  CPU核数：{data['server']['cpu_cores']}")
else:
    print("拉取失败：", data.get("error"))
```

---

## 7. 管理中心建议字段映射（数据模型）

当管理中心入库时，建议保留以下关键字段，便于后续统一看板与告警：

| 库字段 | 来源 |
|--------|------|
| node_id | `id` |
| node_name | `node_name` |
| platform | `platform` |
| report_time | `timestamp`（转为 datetime） |
| hostname | `server.server_name` |
| os | `server.os` |
| cpu_cores | `server.cpu_cores` |
| load_avg | `server.load_avg` |
| mem_total | `server.mem_total` |
| mem_used | `server.mem_used` |
| mem_percent | `server.mem_used / server.mem_total`（自算，容错 N/A） |
| disk_percent_max | `disks` 中最大的 `percent` |
| uptime_seconds | `uptime_seconds` |
| net_rx | `server.net_rx` |
| net_tx | `server.net_tx` |
| php_version | `php.php_version` |

> 告警建议：`mem_percent > 85`、`disk_percent_max > 85`、`timestamp` 超过 120 秒视为失联。

---

## 8. 常见问题

**Q: 接口返回 `{"success":false,"error":"unauthorized"}`？**
A: 节点已启用密钥鉴权，请检查请求中的 `key` 或 `X-API-Key` 是否正确匹配节点 `HCMS_API_KEY`。

**Q: 部分字段为 "N/A"？**
A: 说明该平台未能采集到该项（例如 Windows 无 `/proc` 数据时 Swap 为 N/A）。管理中心应做容错，不应把 "N/A" 当作数值计算。

**Q: 是否需要高频率调用完整接口？**
A: 不建议。完整采集有系统命令开销，建议用心跳接口 `?ping=1` 高频探测，完整数据按需或低频（如每分钟）拉取。

**Q: 管理中心部署在不同域名，跨域会被拦截吗？**
A: 不会。接口已返回 `Access-Control-Allow-Origin: *` 并支持 `OPTIONS` 预检；服务端调用（cURL/requests）更是无跨域限制。

---

*本文档随 api.php 一同部署到各节点，管理中心开发以上述约定为准。*