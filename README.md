# Emby + STRM (115 / AList) 终极直链播放反代方案

> 彻底解决 Emby 播放 STRM 虚拟流时触发强制转码导致的 **403 Forbidden 拒播**与 **404 路径找不到**问题。  
> 采用 **Nginx + njs** 实现 STRM 内容直读、自动跟随 302 重定向获取 115 真实 CDN 直链，无需配置 AList 后台 Admin Token，实现客户端 DirectPlay 秒开。

---

## 痛点与核心原理解析

### 1. 为什么会出现 403 拒播？
- 在 Emby 媒体库中，`.strm` 文件属于虚拟流媒体引用。
- 当 Emby 服务端探测 STRM 虚拟流失败，或者客户端不具备完整解码能力时，Emby 会默认强制回退到服务端转码。
- 若媒体库或用户未开启转码权限，或者服务机无转码硬件/配置不支持时，Emby 会直接抛出 `403 Forbidden` 阻断播放。

### 2. 为什么之前的方案会报错 404 或 401？
- 普通的反代插件默认将 `.strm` 文件当作普通媒体文件，使用宿主机或 Emby 容器的虚拟挂载路径（如 `/media/...`）直接去请求 AList 的后台管理 API（`/api/fs/get`）。
- 一旦 AList 启用了访问权限控制且未配置有效 Admin Token，AList 会返回 `401 token is invalidated` 导致获取失败。
- 回退使用原始路径后，由于宿主机根本不存在该路径的实体视频文件，播放器直接报错 `404 Not Found`。

### 3. 本方案设计：纯粹 STRM 内容直读 + 302 跟随
- SmartStrm 生成的 `.strm` 文件内，已经写入了带防盗链签名的 AList 临时跳转链接：
  ```text
  http://<ALIST_IP>:<PORT>/d/115/downloads/xxx.mp4?sign=xxxx:0
  ```
- **核心逻辑**：
  1. Nginx 拦截客户端视频流请求；
  2. 直接读取宿主机本地对应的 `.strm` 文件第一行 URL；
  3. 跳过任何 AList 后台 API 调用（**完全不需要 AList Token**）；
  4. 向 AList 临时下载接口发起一次 302 跟随，直接拿到 115 真实 CDN（如 `cdnfhnfile.115cdn.net`）；
  5. 下发 302 重定向给客户端，播放器直接从 115 CDN 直连取流，实现极速 DirectPlay。

---

## 落地部署步骤

### 目录结构规划
```text
/opt/emby2alist/
├── conf.d/
│   ├── emby.conf
│   ├── emby.js                # 已打入 STRM 内容直读补丁
│   └── config/
│       ├── constant-strm.js   # 302 跟随规则
│       └── ...
├── log/                       # access.log & error.log
└── embyCache/
```

---

### 第一步：构建带 NJS 模块的 Nginx 镜像

Docker 官方纯净版 `nginx:1.27.1` 默认未预装 NJS 模块，直接运行会导致 `unknown directive "js_path"` 报错退出。使用以下 Dockerfile 秒级构建：

```dockerfile
# /tmp/Dockerfile.nginx-njs
FROM nginx:1.27.1
RUN apt-get update && apt-get install -y --no-install-recommends nginx-module-njs && rm -rf /var/lib/apt/lists/*
RUN sed -i '1i load_module modules/ngx_http_js_module.so;' /etc/nginx/nginx.conf
```

执行构建：
```bash
docker build -t nginx:1.27.1-njs -f /tmp/Dockerfile.nginx-njs /tmp
```

---

### 第二步：挂载并启动 Nginx 容器

> **关键点**：必须将宿主机存放 `.strm` 文件的实际物理路径挂载到容器内的 `/media`（只读 `:ro`），保证 Nginx 能直接读取 STRM 内容。同时采用 `--net=host` 模式打通内网与 Tailscale。

```bash
docker rm -f nginx-emby 2>/dev/null

docker run -d \
  --name nginx-emby \
  --restart always \
  --net=host \
  -v /opt/emby2alist/conf.d:/etc/nginx/conf.d \
  -v /opt/emby2alist/log:/var/log/nginx \
  -v /opt/emby2alist/embyCache:/var/cache/nginx/emby \
  -v /home/jellyfin-stack/alist/data/strm:/media:ro \
  nginx:1.27.1-njs
```
*(注：请将 `-v /home/jellyfin-stack/alist/data/strm` 替换为你宿主机真实的 STRM 根目录)*

---

### 第三步：为 `emby.js` 打入 STRM 内容直读补丁

在宿主机执行 Python 单行脚本，自动定位并注入 STRM 直读逻辑：

```bash
python3 -c '
with open("/opt/emby2alist/conf.d/emby.js", "r") as f:
    content = f.read()

target = "let mediaItemPath = util.doUrlMapping(r, embyRes.path, embyRes.notLocal, mediaPathMapping, \"mediaPathMapping\");"

patch = """let mediaItemPath = util.doUrlMapping(r, embyRes.path, embyRes.notLocal, mediaPathMapping, "mediaPathMapping");
  // 纯粹提取 STRM 内部真实链接
  if (embyRes.path && embyRes.path.toLowerCase().endsWith(".strm")) {
    try {
      const fs = require("fs");
      const realFilePath = embyRes.path;
      if (fs.existsSync(realFilePath)) {
        const fileContent = fs.readFileSync(realFilePath, "utf8");
        const strmUrl = fileContent.split(/\\r?\\n/).map(s => s.trim()).find(s => s.startsWith("http"));
        if (strmUrl) {
          r.warn(`[strm-direct] read internal url: ${strmUrl}`);
          mediaItemPath = strmUrl;
        }
      } else {
        r.warn(`[strm-direct] file not exists in nginx container: ${realFilePath}`);
      }
    } catch (e) {
      r.error(`[strm-direct] read strm file failed: ${e.message}`);
    }
  }"""

if target in content and "[strm-direct]" not in content:
    content = content.replace(target, patch)
    with open("/opt/emby2alist/conf.d/emby.js", "w") as f:
        f.write(content)
    print("Patch applied successfully!")
else:
    print("Patch already exists or target not found.")
'
```

---

### 第四步：配置 STRM 302 跟随（获取 115 真实 CDN）

修改 `/opt/emby2alist/conf.d/config/constant-strm.js`，开启单次 302 跳转跟随：

```javascript
// strm 内部直链重定向跟随规则
const redirectStrmLastLinkRule = [
  [0, "http", 1, false] // 遇到 http 链接，跟随 1 次 302 获取真实 115 CDN
];

export default {
  redirectStrmLastLinkRule,
};
```

---

### 第五步：重载生效与测试验证

```bash
docker exec nginx-emby nginx -s reload
```

播放任意 STRM 视频，在宿主机观察日志：
```bash
tail -f /opt/emby2alist/log/error.log
```

**成功日志特征**：
```text
[warn] js: [strm-direct] read internal url: http://100.119.231.71:25244/d/115/...
[warn] js: redirect to: https://cdnfhnfile.115cdn.net/...
```
客户端即刻秒开直连播放，不消耗服务器 CPU/转码资源，彻底告别 403 与 404！

---

## License
MIT License
