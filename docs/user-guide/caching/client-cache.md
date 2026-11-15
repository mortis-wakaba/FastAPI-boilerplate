<a id="client-cache"></a>

# 客户端缓存

客户端通过 HTTP `Cache-Control` 告诉浏览器、CDN 和代理何时可复用响应而不访问服务器。模板提供小型中间件：API 明确不缓存，其他响应使用受控策略及可配置 max-age。

<a id="whats-built-in"></a>

## 内置能力

```text
infrastructure/middleware.py        ClientCacheMiddleware
infrastructure/app_factory.py       Wires it into the app at startup
infrastructure/config/settings.py   CLIENT_CACHE_ENABLED, CLIENT_CACHE_MAX_AGE
```

没有逐路由配置表或内置 ETag，更细致的逻辑由路由处理。

<a id="how-it-works"></a>

## 工作方式

```python
# infrastructure/middleware.py
class ClientCacheMiddleware(BaseHTTPMiddleware):
    def __init__(
        self,
        app: ASGIApp,
        max_age: int = 60,
        public_prefixes: tuple[str, ...] = PUBLIC_CACHE_PREFIXES,
        api_prefix: str = "/api",
    ) -> None:
        ...

    async def dispatch(self, request, call_next):
        response = await call_next(request)
        if "cache-control" in response.headers:
            return response

        path = request.url.path
        if path.startswith(f"{self.api_prefix}/"):
            response.headers["Cache-Control"] = "private, no-cache, no-store, must-revalidate"
        elif path.startswith(self.public_prefixes):
            response.headers["Cache-Control"] = f"public, max-age={self.max_age}"
        else:
            response.headers["Cache-Control"] = "private, no-store"
        return response
```

四条规则：

| 情况 | `Cache-Control` |
|------|-----------------|
| 路由已设置响应头 | 保留路由值 |
| `API_PREFIX` 下，默认 `/api` | `private, no-cache, no-store, must-revalidate` |
| 公开前缀，如 `/admin/statics/` | `public, max-age={CLIENT_CACHE_MAX_AGE}` |
| 其余 | `private, no-store` |

理由：

- **API** 常动态、需认证且个性化，浏览器或 CDN 缓存可能造成泄露或陈旧。规则跟随 `API_PREFIX`。
- **公开前缀** 对所有人字节一致，默认 SQLAdmin CSS / JS，可通过 `public_prefixes` 添加应用静态挂载。
- **其余**，如后台页面、文档、健康状态，可能因用户或状态变化，不应共享缓存。确定可缓存的路由自行设头。

<a id="configuration"></a>

## 配置

```env
# Enables the middleware. Set to false to skip the Cache-Control header entirely.
CLIENT_CACHE_ENABLED=true

# max-age (seconds) used for non-API paths.
CLIENT_CACHE_MAX_AGE=60
```

`CLIENT_CACHE_ENABLED=true` 时由 `infrastructure/app_factory.py` 添加，与服务端 `CACHE_ENABLED` 独立。关闭后中间件不设头，改由路由或代理负责。

<a id="overriding-for-a-specific-endpoint"></a>

## 覆盖特定接口

本节原文称中间件在处理器后覆盖 API 的缓存头，建议关闭全局中间件后逐路由设头，或使用路由级中间件：

```python
from fastapi import APIRouter, Response

router = APIRouter()


@router.get("/manifest.json")
async def manifest(response: Response) -> dict[str, str]:
    response.headers["Cache-Control"] = "public, max-age=86400, immutable"
    return {"name": "My App", "version": "1.0.0"}
```

原文说明该示例只对非 API 路由生效，API 需移出 `/api/` 或关闭 `CLIENT_CACHE_ENABLED` 自行管理。一般 CRUD API 优先采用[服务端缓存](redis-cache.md)，浏览器仍读取新响应。

> 译注：本节“覆盖已有头”的说法与上方规则表“保留路由值”不一致，请按实际代码核对。

<a id="a-quick-cache-control-primer"></a>

## Cache-Control 简介

| 指令 | 含义 |
|------|------|
| `public` | 浏览器、CDN、代理均可存储 |
| `private` | 仅终端浏览器可存储，共享缓存不可 |
| `no-cache` | 可存储，但复用前需向服务器验证 |
| `no-store` | 不存储到浏览器、磁盘或 CDN |
| `must-revalidate` | 过期后必须向上游验证再使用 |
| `max-age=<seconds>` | 有效秒数 |
| `s-maxage=<seconds>` | 仅共享缓存有效秒数 |
| `immutable` | 内容不会变化，客户端可跳过验证 |
| `stale-while-revalidate=<s>` | 过期后在指定时间内可返回旧副本并后台更新 |

API 的多项禁止缓存指令有意采用保守默认值，适用于认证后的动态数据。

<a id="when-to-reach-for-etags"></a>

## 何时使用 ETag

中间件不生成 ETag 或 `Last-Modified`。需要未变化返回 `304 Not Modified` 时自行设置：

```python
import hashlib
from fastapi import APIRouter, Request, Response, status

router = APIRouter()


@router.get("/manifest.json")
async def manifest(request: Request, response: Response) -> dict[str, str] | Response:
    payload = {"name": "My App", "version": "1.0.0"}
    body = str(payload).encode()
    etag = f'"{hashlib.md5(body).hexdigest()}"'

    if request.headers.get("if-none-match") == etag:
        return Response(status_code=status.HTTP_304_NOT_MODIFIED)

    response.headers["ETag"] = etag
    response.headers["Cache-Control"] = "public, max-age=300"
    return payload
```

适合大型较静态内容、清单、配置、生成 PDF、读多写少公开接口，以及跨多次请求稳定的计算结果。普通 CRUD 通常不值得额外复杂度，服务端 `@cache` 和静态资源短 max-age 已覆盖多数场景。

<a id="reverse-proxy-cdn-considerations"></a>
<a id="reverse-proxy--cdn-considerations"></a>

## 反向代理与 CDN

Nginx、Caddy、CloudFront、Cloudflare 会读取缓存头：API 禁止边缘缓存，每次到源站。本节原文称非 API 响应可共享缓存 `CLIENT_CACHE_MAX_AGE` 秒。

> 译注：上方规则表仅对公开前缀允许共享缓存，其余非 API 响应仍为 `private, no-store`。

需要 CDN 长 TTL、浏览器短 TTL 时，处理器同时设 `s-maxage` 和 `max-age`，或在代理移除并替换缓存头。

<a id="disabling-the-middleware"></a>

## 禁用中间件

```env
CLIENT_CACHE_ENABLED=false
```

重启后模板不再设置该头，由路由和代理完全管理。适用于已有 CDN 策略、逐路由策略，或排查缓存问题建立基线。

<a id="troubleshooting"></a>

## 故障排查

<a id="my-api-response-is-still-being-cached-by-the-browser"></a>

### API 仍被浏览器缓存

确认响应头：

```bash
curl -I http://localhost:8000/api/v1/users/me \
    -H "Cookie: session_id=..."
# look for: Cache-Control: private, no-cache, no-store, must-revalidate
```

缺失时检查 `CLIENT_CACHE_ENABLED=true`，它决定是否挂载。

<a id="i-want-to-cache-an-api-response-but-the-middleware-overrides-it"></a>

### 中间件覆盖 API 的缓存头

本节沿用上文的旧描述，建议服务端缓存、关闭客户端中间件后逐路由管理，或将真正静态资源移出 `/api/*`。与规则表的差异见译注。

<a id="static-assets-arent-being-cached-aggressively-enough"></a>

### 静态资源缓存不够久

提高 `CLIENT_CACHE_MAX_AGE`，或在处理器、代理设置资源专属头。修改部署后新请求读取最新值。

<a id="key-files"></a>

## 关键文件

| 组件 | 位置 |
|------|------|
| 中间件 | `backend/src/infrastructure/middleware.py` |
| 组装 | `backend/src/infrastructure/app_factory.py` |
| 配置 | `backend/src/infrastructure/config/settings.py` 的 `CacheSettings` |

<a id="next-steps"></a>

## 后续步骤

- **[Redis 缓存](redis-cache.md)**：服务端装饰器。
- **[缓存策略](cache-strategies.md)**：键名与失效。
- **[环境变量](../configuration/environment-variables.md#cache)**：完整配置。
