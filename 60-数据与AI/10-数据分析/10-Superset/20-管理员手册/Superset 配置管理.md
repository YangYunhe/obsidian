---
title: Superset 配置管理
date: 2026-10-22
tags:
  - superset
  - 运维
  - 配置
---

# Superset 配置管理

> 管理员视角的 Superset **6.1.0** 配置指南：`superset_config.py`、`SECRET_KEY`、生产元数据库、WSGI/HTTPS、反向代理、应用根路径、OAuth2 / LDAP、组到角色映射、Feature Flags、安全配置与 SQL Lab 历史清理。
> 原文：<https://superset.apache.org/admin-docs/6.1.0/configuration/configuring-superset/>

## 一、superset_config.py

Superset 通过 `config.py` 暴露数百个可配置参数（含合理的默认值与注释）。**不要修改核心模块**，而是创建自己的配置模块，通常命名为 `superset_config.py`。

让它被加载的方式：

- 加入 `PYTHONPATH`；或
- 用环境变量指向完整路径：

```bash
export SUPERSET_CONFIG_PATH=/app/superset_config.py
```

使用官方镜像自建 Dockerfile 时：

```dockerfile
COPY --chown=superset superset_config.py /app/
ENV SUPERSET_CONFIG_PATH /app/superset_config.py
```

> Docker Compose 部署有自己的配置约定，见其文档。

典型做法：只把想改的部分（连同注释）从核心 `superset/config.py` 复制到自己的文件。**至少需要修改 `SECRET_KEY` 与 `SQLALCHEMY_DATABASE_URI`。**

由于它本质是 Flask 配置模块，也可配置 Flask 本身及其捆绑的扩展：`flask-wtf`、`flask-caching`、`flask-migrate`、`flask-appbuilder`。

常见参数示例：

```python
# Superset specific config
ROW_LIMIT = 5000

# 生产必须设置，否则服务拒绝启动
SECRET_KEY = 'YOUR_OWN_RANDOM_GENERATED_SECRET_KEY'

# 元数据库（slices / connections / tables / dashboards …）
SQLALCHEMY_DATABASE_URI = 'sqlite:////path/to/superset.db?check_same_thread=false'

# CSRF
WTF_CSRF_ENABLED = True
WTF_CSRF_EXEMPT_LIST = []
WTF_CSRF_TIME_LIMIT = 60 * 60 * 24 * 365

# 启用 Mapbox 可视化
MAPBOX_API_KEY = ''
```

## 二、SECRET_KEY

Superset 从 2.1.0 起要求用户显式设置 `SECRET_KEY` 才能启动。

生成强密钥：

```bash
openssl rand -base64 42
```

写入配置或使用环境变量：

```bash
export SUPERSET_SECRET_KEY=$(openssl rand -base64 42)
```

> [!warning] 必须使用强密钥
> 该密钥用于签名会话 Cookie、加密存储在元数据库中的敏感信息，必须是复杂且唯一的。

**轮换密钥**：

1. 在 `superset_config.py` 中把旧值设为 `PREVIOUS_SECRET_KEY`，新值设为 `SECRET_KEY`。
2. 查看当前密钥（Docker 部署需进入应用容器）：

```python
superset shell
from flask import current_app; print(current_app.config["SECRET_KEY"])
```

3. 保存配置后执行：

```bash
superset re-encrypt-secrets
```

## 三、生产元数据库

Superset 需要一个数据库存放图表、仪表盘等元数据。

- 默认使用 **SQLite**（单文件、免安装，便于快速上手）。
- **生产环境强烈不建议 SQLite**（安全、扩展性、数据完整性）。
- 仅使用受支持的数据库引擎，建议独立主机/容器，或使用托管服务（如 Amazon RDS、Google Cloud Databases）来负责备份与运维。
- 通过 `SQLALCHEMY_DATABASE_URI` 指向元数据库。

## 四、运行在 WSGI HTTP 服务器上

推荐使用 **Gunicorn 的 async 模式**。一个经生产验证的配置：

```bash
gunicorn \
  -w 10 \
  -k gevent \
  --worker-connections 1000 \
  --timeout 120 \
  -b 0.0.0.0:6666 \
  --limit-request-line 0 \
  --limit-request-field_size 0 \
  --statsd-host localhost:8125 \
  "superset.app:create_app()"
```

- 开发服务器（`superset run` / `flask run`）**不可用于生产**。
- 若不用 Gunicorn，可在配置中设 `COMPRESS_REGISTER = False` 关闭 `flask-compress`。
- **BigQuery 数据源与 `gevent` 不兼容**（gevent 对 Python 核心库的动态 monkeypatching），需使用非 gevent 的 worker 类型。

## 五、HTTPS 与反向代理

- 可在负载均衡或反向代理（如 nginx）上做 SSL/TLS 卸载，再转发到 Superset。
- 此架构下，Celery worker 为告警与报表生成图表快照时，可通过内网 `http://` 访问 Superset。
- 使用官方 Docker 镜像时，也可直接在 Gunicorn 配置 SSL。
- **健康检查端点**：`/health`，Web 服务正常时返回 200 与 `OK`。
- 若 LB 注入 `X-Forwarded-For` / `X-Forwarded-Proto`，设置：

```python
ENABLE_PROXY_FIX = True
```

- 反向代理提供 SSL 时，可能需显式设置 `X-Forwarded-Proto`（Apache 示例）：

```apache
RequestHeader set X-Forwarded-Proto "https"
```

## 六、配置应用根路径（BETA）

Superset 支持运行在非根路径下，前缀必须以 `/` 开头。三种方式：

1. 通过 `superset_app_root` 变量自定义 Flask 入口；
2. 设置环境变量 `SUPERSET_APP_ROOT`；
3. 在 `superset_config.py` 中设置 `APPLICATION_ROOT`。

示例（前缀 `/analytics`）：

```bash
FLASK_APP="superset:create_app(superset_app_root='/analytics')"
# 或
flask --app "superset.app:create_app(superset_app_root='/analytics')"
```

Docker Compose：在 `docker/.env-local` 中设置 `SUPERSET_APP_ROOT`，然后 `docker compose up --detach`。

## 七、自定义 OAuth2

Superset 基于 Flask-AppBuilder（FAB），内置 GitHub、Twitter、LinkedIn、Google、Azure 等提供方；也支持其它兼容「code」授权的 OAuth2 服务。**需在 Web 服务器安装 `Authlib`。**

```python
from flask_appbuilder.security.manager import AUTH_OAUTH

AUTH_TYPE = AUTH_OAUTH

OAUTH_PROVIDERS = [
    {
        'name': 'egaSSO',
        'token_key': 'access_token',
        'icon': 'fa-address-card',
        'remote_app': {
            'client_id': 'myClientId',
            'client_secret': 'MySecret',
            'client_kwargs': {'scope': 'read'},
            'access_token_method': 'POST',
            'access_token_params': {'client_id': 'myClientId'},
            'jwks_uri': 'https://myAuthorizationServer/adfs/discovery/keys',
            'access_token_headers': {
                'Authorization': 'Basic Base64EncodedClientIdAndSecret'
            },
            'api_base_url': 'https://myAuthorizationServer/oauth2AuthorizationServer/',
            'access_token_url': 'https://myAuthorizationServer/oauth2AuthorizationServer/token',
            'authorize_url': 'https://myAuthorizationServer/oauth2AuthorizationServer/authorize'
        }
    }
]

# 允许自注册
AUTH_USER_REGISTRATION = True
AUTH_USER_REGISTRATION_ROLE = "Public"   # 或 "Admin"
```

- 自注册为 `Admin` 后，若设置页无法列出用户，重新执行 `superset init` 同步权限：

```bash
docker-compose exec superset superset init
```

- 自定义用户信息：创建 `custom_sso_security_manager.py`（与 `superset_config.py` 同目录），继承 `SupersetSecurityManager` 并重写 `oauth_user_info`：

```python
import logging
from superset.security import SupersetSecurityManager

class CustomSsoSecurityManager(SupersetSecurityManager):
    def oauth_user_info(self, provider, response=None):
        logging.debug("Oauth2 provider: {0}.".format(provider))
        if provider == 'egaSSO':
            me = self.appbuilder.sm.oauth_remotes[provider].get('userDetails').data
            return {
                'name': me['name'], 'email': me['email'], 'id': me['user_name'],
                'username': me['user_name'], 'first_name': '', 'last_name': ''
            }
```

```python
from custom_sso_security_manager import CustomSsoSecurityManager
CUSTOM_SECURITY_MANAGER = CustomSsoSecurityManager
```

- **回调 URL**：`https://<superset-webserver>/oauth-authorized/<provider-name>`。
- **OpenID Connect 1.0**：只需配置 `server_metadata_url`（`.well-known/openid-configuration`），无需逐个填 `api_base_url` / `access_token_url` / `authorize_url` 等：

```python
'remote_app': {
    'client_id': 'myClientId',
    'client_secret': 'MySecret',
    'server_metadata_url': 'https://myAuthorizationServer/.well-known/openid-configuration'
}
```

- **PKCE**：不能安全保存 client secret 的公共客户端，可加 `code_challenge_method`：

```python
'code_challenge_method': 'S256',   # 启用 PKCE
```

> PKCE（`S256`）建议用于**所有** OAuth2 流程，即使存在 client secret，也能防止授权码被拦截。

## 八、LDAP 认证

FAB 支持对 LDAP 服务器认证用户；需安装 `python-ldap` 包，并参考 FAB 的 LDAP 文档。

## 九、LDAP / OAuth 组映射到 Superset 角色

`AUTH_ROLES_MAPPING` 是 FAB 的字典，把 LDAP/OAuth 组名映射到 FAB 角色。

**OAuth 组：**

```python
AUTH_ROLES_MAPPING = {
    "superset_users": ["Gamma", "Alpha"],
    "superset_admins": ["Admin"],
}
```

**LDAP 组（DN）：**

```python
AUTH_ROLES_MAPPING = {
    "cn=superset_users,ou=groups,dc=example,dc=com": ["Gamma", "Alpha"],
    "cn=superset_admins,ou=groups,dc=example,dc=com": ["Admin"],
}
```

> LDAP 映射需要设置 `AUTH_LDAP_SEARCH`。

**登录时同步角色**：`AUTH_ROLES_SYNC_AT_LOGIN`

- `True`：每次登录都同步用户角色；
- `False`：仅在首次注册时同步。

## 十、Flask app 配置钩子

`FLASK_APP_MUTATOR` 会接收 app 对象，可对其做任意修改。例如把会话 Cookie 有效期设为 24 小时：

```python
from flask import session
from flask import Flask

def make_session_permanent():
    session.permanent = True

PERMANENT_SESSION_LIFETIME = timedelta(hours=24)

def FLASK_APP_MUTATOR(app: Flask) -> None:
    app.before_request_funcs.setdefault(None, []).append(make_session_permanent)
```

## 十一、Feature Flags

部分功能默认关闭（面向不同安全要求的用户）。通过 `superset_config.py` 中的 `FEATURE_FLAGS` 字典开关：

```python
FEATURE_FLAGS = {
    'PRESTO_EXPAND_DATA': False,
}
```

当前可用清单见官方 Feature Flags 文档。

## 十二、安全配置：HASH_ALGORITHM

控制内部校验和与缓存键（缩略图、缓存键等）使用的哈希算法：

```python
HASH_ALGORITHM = "sha256"          # 默认；兼容旧部署可设为 "md5"
HASH_ALGORITHM_FALLBACKS = ["md5"] # 兼容回退；严格模式设为 []
```

- 默认 `sha256`，满足 FedRAMP 等更严格的合规要求。
- `HASH_ALGORITHM_FALLBACKS` 允许 UUID 查询回退到旧算法，便于渐进迁移；设为 `[]` 即严格模式。
- 仅影响 Superset 内部操作，**不影响用户密码或认证令牌**；已有部署修改后可能使缓存失效，但无需数据库迁移。

## 十三、SQL Lab 查询历史清理

SQL Lab 查询历史存在元数据库中，**默认不清理**。启用 `prune_query` Celery beat 任务并设置保留期：

```python
CELERY_BEAT_SCHEDULE = {
    "prune_query": {
        "task": "prune_query",
        "schedule": crontab(minute=0, hour=0, day_of_month=1),
        "kwargs": {"retention_period_days": 180},
    },
}
```

- `retention_period_days` 控制保留天数。
- 另有 `prune_logs`、`prune_tasks` 可选任务（见 `superset/config.py` 中的注释示例）。
- **不启用这些任务，元数据库会无限增长。**

## 相关

- [[60-数据与AI/10-数据分析/10-Superset/00-索引|10-Superset]]
- [[60-数据与AI/10-数据分析/10-Superset/10-用户使用手册/00-索引|10-用户使用手册]]
- Superset 官方文档：<https://superset.apache.org/>
