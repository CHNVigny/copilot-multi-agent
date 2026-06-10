# Tenon 平台开发文档 — 给 DevCoordinator Agent 团队

> 平台名称：**Tenon**
> 这是一份给 10 Agent 团队的开发启动文档。包含架构设计、开发规范、反模式清单和实现路径。

---

## 一、项目概述

**Tenon** — Django 平台化 + 可插拔架构。

核心思路：**Django 的 AppConfig.ready() 天然就是插件加载入口**，配合轻量 PluginRegistry，不引入大型插件框架即可实现真正的可插拔。

### 层级映射

| 业务层级 | Django 对应 | 说明 |
|---|---|---|
| 平台 | Django Project（manage.py、settings/、根 urls.py） | 全局配置、插件注册中心、中间件 |
| 分系统 | 可插拔 Django App（subsystems/xxx/） | 独立 app，有自己的 models、urls、services |
| 子系统 | App 内子包（subsystems/xxx/sub_xxx/） | 隶属于分系统的功能簇 |
| 模块 | Python 模块（modules/yyy/） | 单一职责的功能集合 |
| 组件 | 类/函数（components.py 或独立文件） | 最小可复用单元 |

### 核心设计决策

1. PluginRegistry — 模块级注册中心（非单例类），管理分系统元数据、依赖、生命周期
2. plugin.toml — 每个分系统的自描述文件，声明身份、依赖和配置
3. Signals — 分系统间事件总线，发送方和接收方零耦合
4. 接口抽象 ABC — 共性能力通过接口暴露，具体实现可替换
5. core/ — 纯技术组件，零业务属性，所有分系统的公共底座
6. apis/composite/ — 跨分系统 API 编排层，不塞在任何一个分系统里

---

## 二、项目目录结构

```
tenon/
├── manage.py
├── tenon/                             # Django 项目配置包
│   ├── settings/
│   │   ├── __init__.py
│   │   ├── base.py                    # 公共配置
│   │   ├── dev.py                     # 开发环境
│   │   └── prod.py                    # 生产环境
│   ├── urls.py                        # 根路由（插件自动发现）
│   ├── wsgi.py
│   └── asgi.py
│   └── celery.py                      # Celery 配置
│
├── core/                              # 共性基础组件（所有分系统共享）
│   ├── __init__.py
│   ├── apps.py
│   ├── base/
│   │   ├── __init__.py
│   │   ├── models.py                  # BaseModel（uuid 主键、时间戳、软删除）
│   │   ├── views.py                   # BaseViewSet、BaseAPIView
│   │   ├── serializers.py            # BaseSerializer
│   │   ├── services.py               # BaseService（事务管理、缓存）
│   │   ├── exceptions.py             # 统一异常类
│   │   ├── events.py                 # 标准事件定义（Signals）
│   │   └── interfaces.py             # 抽象接口（ABC）
│   ├── auth/
│   │   ├── backends.py               # 多认证后端
│   │   ├── permissions.py            # 细粒度权限
│   │   └── middlewares.py
│   ├── logging/
│   │   ├── formatters.py
│   │   └── audit.py                  # 审计日志
│   ├── task_queue/
│   │   ├── celery_app.py
│   │   └── base_task.py
│   └── utils/
│       ├── response.py               # 统一响应格式
│       ├── pagination.py
│       └── validators.py
│
├── plugin_registry/                   # 插件注册中心
│   ├── __init__.py
│   ├── registry.py                    # 模块级字典 + 纯函数
│   ├── discovery.py                   # 自动发现（黑名单防护）
│   └── exceptions.py                  # 依赖错误异常
│
├── subsystems/                        # 分系统目录
│   ├── __init__.py
│   ├── _template/                     # 脚手架目录（discovery 跳过）
│   ├── usercenter/                    # 分系统：用户中心
│   │   ├── __init__.py
│   │   ├── apps.py
│   │   ├── urls.py
│   │   ├── models.py
│   │   ├── signals.py
│   │   ├── plugin.toml
│   │   ├── profile/                   # 子系统：个人资料
│   │   │   ├── __init__.py
│   │   │   ├── urls.py
│   │   │   ├── models.py
│   │   │   ├── services.py
│   │   │   └── modules/
│   │   │       ├── avatar/
│   │   │       │   ├── __init__.py
│   │   │       │   ├── services.py
│   │   │       │   ├── serializers.py
│   │   │       │   └── components/
│   │   │       │       ├── uploader.py
│   │   │       │       └── processor.py
│   │   │       └── settings/
│   │   └── permission/               # 子系统：权限管理
│   │       ├── __init__.py
│   │       ├── urls.py
│   │       ├── models.py
│   │       ├── services.py
│   │       └── modules/
│   │           ├── role/
│   │           └── policy/
│   ├── billing/                       # 分系统：计费
│   │   ├── __init__.py
│   │   ├── apps.py
│   │   ├── urls.py
│   │   ├── plugin.toml
│   │   ├── order/
│   │   ├── invoice/
│   │   └── payment/
│   └── analytics/                     # 分系统：数据分析
│       ├── __init__.py
│       ├── apps.py
│       ├── plugin.toml
│       └── ...
│
├── apis/                              # API 聚合层
│   └── v1/
│       └── composite/                 # 组合 API（跨分系统编排）
│
├── static/
├── media/
├── templates/
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
└── ARCHITECTURE.md                    # 架构反模式文档
```

---

## 三、核心机制：插件注册中心

### 正确实现（模块级字典 + 纯函数，不用单例类）

```python
# plugin_registry/registry.py

from dataclasses import dataclass, field
from typing import Dict, List, Optional

@dataclass
class PluginMeta:
    """插件元数据"""
    name: str                        # 分系统唯一标识
    label: str                       # 显示名称
    version: str
    dependencies: List[str] = field(default_factory=list)
    optional_dependencies: List[str] = field(default_factory=list)
    enabled: bool = True


# 模块级注册表——Python import 机制天然单例，无热重载问题
_registry: Dict[str, PluginMeta] = {}
_loaded_order: List[str] = []        # 记录加载顺序


def register(meta: PluginMeta):
    """注册分系统。自动检查依赖是否已注册。"""
    # 依赖检查（在注册阶段做，不是事后拓扑排序）
    for dep in meta.dependencies:
        if dep not in _registry:
            from plugin_registry.exceptions import PluginDependencyError
            raise PluginDependencyError(
                f"Plugin '{meta.name}' requires '{dep}', "
                f"but '{dep}' is not installed or not yet loaded. "
                f"Load '{dep}' before '{meta.name}'."
            )
    _registry[meta.name] = meta
    _loaded_order.append(meta.name)


def get(name: str) -> Optional[PluginMeta]:
    return _registry.get(name)


def list_enabled() -> List[str]:
    return [n for n, m in _registry.items() if m.enabled]


def list_all() -> List[str]:
    return list(_loaded_order)
```

### 自动发现（带黑名单防护）

```python
# plugin_registry/discovery.py

import pkgutil
import importlib
import tenon.subsystems as subsystems
from django.conf import settings
from plugin_registry.registry import _registry, PluginMeta, register

_SKIP_PACKAGES = {'_template', '__pycache__', 'tests', 'migrations'}


def discover():
    """扫描 subsystems/ 目录，导入所有分系统的 apps.py。"""
    # 读取 settings 终极开关
    tenon_config = getattr(settings, 'TENON_PLUGINS', {})

    for _, name, is_pkg in pkgutil.iter_modules(subsystems.__path__):
        if not is_pkg or name in _SKIP_PACKAGES:
            continue
        try:
            importlib.import_module(f"tenon.subsystems.{name}.apps")
        except ImportError as e:
            # 分系统缺少 apps.py，跳过
            pass

    # settings 覆盖 toml 中的 enabled（settings 是终极开关）
    for meta_name in list(_registry.keys()):
        if meta_name in tenon_config:
            _registry[meta_name].enabled = tenon_config[meta_name]
```

---

## 四、分系统 AppConfig 写法

### plugin.toml（每个分系统的自描述文件）

```toml
# tenon/subsystems/usercenter/plugin.toml

[plugin]
name = "usercenter"
label = "用户中心"
version = "1.0.0"
dependencies = []
optional_dependencies = ["notification", "storage"]
enabled = true

[plugin.settings]
DEFAULT_ROLE = "member"
```

### apps.py（在 ready() 中注册）

```python
# tenon/subsystems/usercenter/apps.py

import os
import tomllib
from django.apps import AppConfig
from plugin_registry.registry import PluginMeta, register


class UserCenterConfig(AppConfig):
    name = 'tenon.subsystems.usercenter'
    verbose_name = '用户中心'

    def ready(self):
        # 1. 读取插件元数据
        meta_path = os.path.join(os.path.dirname(__file__), 'plugin.toml')
        with open(meta_path, 'rb') as f:
            data = tomllib.load(f)
        meta = PluginMeta(**data['plugin'])

        # 2. 注册到注册中心
        register(meta)

        # 3. 导入 signals，让事件监听生效
        import tenon.subsystems.usercenter.signals  # noqa
```

---

## 五、分系统加载顺序控制

通过 settings 的 OrderedDict 控制加载顺序：

```python
# tenon/settings/base.py

from collections import OrderedDict

# Tenon 分系统开关（settings 是终极开关，覆盖 plugin.toml 的 enabled）
TENON_PLUGINS = OrderedDict([
    ('_template', False),       # 脚手架跳过
    ('usercenter', True),       # ← 最先加载
    ('notification', True),
    ('storage', True),
    ('billing', True),          # ← 依赖 usercenter，后加载
    ('analytics', False),       # ← 禁用
])
```

**settings 是终极开关**，plugin.toml 只是默认值。运维可一行代码关闭分系统。

---

## 六、分系统间解耦通信（三种模式）

### 模式 1：Django Signals（轻量事件总线）

适用场景：强一致性、同进程、同步场景（如创建用户必须同时创建账户）。

```python
# core/base/events.py —— 定义标准事件
import django.dispatch

user_created = django.dispatch.Signal()
order_paid = django.dispatch.Signal()
plugin_status_changed = django.dispatch.Signal()
```

**发送事件（发送方不 import 接收方的代码）**：

```python
# tenon/subsystems/usercenter/signals.py
from core.base.events import user_created

def broadcast_user_created(user):
    user_created.send(
        sender='usercenter',
        user_id=user.id,
        username=user.username,
    )
```

**订阅事件（接收方不 import 发送方的代码）**：

```python
# tenon/subsystems/billing/signals.py
from core.base.events import user_created
from django.dispatch import receiver
from plugin_registry.registry import list_enabled


@receiver(user_created)
def on_user_created(sender, **kwargs):
    """用户注册后自动创建计费账户。不 import usercenter。"""
    if 'billing' in list_enabled():
        from tenon.subsystems.billing.account.services import AccountService
        AccountService().create_account(kwargs['user_id'])
```

### 模式 2：服务层抽象 + 依赖注入

适用场景：需要调用另一个分系统提供的能力，但又不直接 import。

```python
# core/base/interfaces.py
from abc import ABC, abstractmethod

class INotificationService(ABC):
    """通知服务接口——分系统只依赖接口"""
    @abstractmethod
    def send(self, user_id: str, title: str, body: str, channel: str = 'in_app'):
        pass
```

```python
# tenon/subsystems/usercenter/profile/modules/avatar/services.py
from core.base.interfaces import INotificationService

class AvatarService:
    def __init__(self, notifier: INotificationService = None):
        self.notifier = notifier or self._get_notifier()

    def _get_notifier(self):
        from core.notification.dispatch import notification_service
        return notification_service

    def update_avatar(self, user, file):
        # ... 处理头像
        self.notifier.send(user.id, "头像更新", "您的头像已更新成功")
```

### 模式 3：REST API 调用

适用场景：分系统部署在不同进程中。

```python
# core/base/api_client.py
import requests
from django.conf import settings

class SubsystemAPIClient:
    def __init__(self, base_url: str, auth_token: str = None):
        self.base_url = base_url
        self.auth_token = auth_token or settings.SERVICE_AUTH_TOKEN

    def get(self, path, **kwargs):
        return requests.get(
            f"{self.base_url}{path}",
            headers={"Authorization": f"Bearer {self.auth_token}"},
            **kwargs
        )
```

---

## 七、URL 自动发现

```python
# tenon/urls.py
from django.urls import path, include
from plugin_registry.registry import list_enabled
from plugin_registry.discovery import discover

discover()

urlpatterns = [
    path('api/auth/', include('core.auth.urls')),
    path('api/common/', include('core.urls')),
]

for plugin_name in list_enabled():
    try:
        urlpatterns.append(
            path(f'api/{plugin_name}/',
                 include(f'tenon.subsystems.{plugin_name}.urls'))
        )
    except ImportError:
        pass
```

每个分系统按子系统组织路由：

```python
# tenon/subsystems/usercenter/urls.py
from django.urls import path, include

app_name = 'usercenter'
urlpatterns = [
    path('profile/', include('tenon.subsystems.usercenter.profile.urls')),
    path('permission/', include('tenon.subsystems.usercenter.permission.urls')),
]
```

---

## 八、共性基础组件设计原则

放在 core/ 中的东西必须满足：

| 原则 | 说明 |
|---|---|
| 零业务属性 | 不包含任何业务领域概念，纯技术能力 |
| 接口优于实现 | 暴露抽象基类/接口，具体实现可替换 |
| 向后兼容 | 接口变更走版本化，不破坏已有分系统 |
| 自文档化 | 每个组件有清晰的 docstring 和使用示例 |

**core 判定标准**：如果做第二个平台项目，这个东西要不要带过去？要→ core。不要→ 独立分系统。

**组件归类决策**：

| 组件 | 位置 | 原因 |
|---|---|---|
| auth/backends.py | ✅ core | 所有分系统都要认证 |
| logging/ | ✅ core | 纯基础设施 |
| base/models.py | ✅ core | 抽象基类 |
| notification/ | ⚠️ 独立分系统 | 模板、渠道、偏好、退订——这是业务域 |
| storage/ | ⚠️ 独立分系统 | CDN、水印、转码有独立演进需求，放 core 会让 core 膨胀 |

### BaseModel

```python
# core/base/models.py
import uuid
from django.db import models

class BaseModel(models.Model):
    """所有模型的抽象基类"""
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_deleted = models.BooleanField(default=False, db_index=True)

    class Meta:
        abstract = True
        ordering = ['-created_at']

    def soft_delete(self):
        self.is_deleted = True
        self.save(update_fields=['is_deleted'])


class BaseManager(models.Manager):
    """自动过滤已删除记录"""
    def get_queryset(self):
        return super().get_queryset().filter(is_deleted=False)
```

---

## 九、PlugManager 生命周期

```
Django 启动
  │
  ▼
core/apps.py::ready()                          ← 基础组件初始化
  │
  ▼
PluginRegistry.discover()                      ← 扫描 subsystems/（跳过 _SKIP_PACKAGES）
  │
  ├── usercenter/apps.py::ready()
  │   ├── 读取 plugin.toml
  │   ├── PluginRegistry.register(meta)        ← 注册时做依赖检查
  │   └── 导入 signals
  │
  ├── billing/apps.py::ready()
  │   ├── 读取 plugin.toml
  │   ├── PluginRegistry.register(meta)        ← 检查 usercenter 已注册
  │   └── 导入 signals（连接事件监听）
  │
  └── analytics → settings 中 enabled=false，不注册
  │
  ▼
tenon/urls.py                                  ← 自动组装已启用分系统的路由
  │
  ▼
平台就绪 ✓
```

---

## 十、分系统启用/禁用

在 settings 中一行控制：

```python
# tenon/settings/base.py
TENON_PLUGINS = OrderedDict([
    ('usercenter', True),
    ('billing', True),
    ('analytics', False),     # ← 禁用，不加载、不注册路由
])
```

---

## 十一、关键约束与反模式清单

### 硬性约束（Code Review 红线，违反即驳回）

- [ ] ❌ `from tenon.subsystems.xxx.models import ...` 出现在另一个分系统中
- [ ] ❌ 分系统的 views.py 直接调用另一个分系统的 services.py
- [ ] ❌ core/ 里出现业务概念（User、Order、Invoice 等业务模型名）
- [ ] ❌ 跨分系统的 ForeignKey
- [ ] ❌ 分系统不声明 plugin.toml

### 推荐约束

- [ ] ✅ 跨分系统引用用 user_id（UUID 字符串），不持有外键
- [ ] ✅ models 只定义在自己的分系统内（数据所有权明确）
- [ ] ✅ 数据库共享但 Schema 隔离（每个分系统用自己的表前缀）
- [ ] ✅ 共性能力放 core/（DRY，统一演进）
- [ ] ✅ 跨分系统编排逻辑放 apis/composite/，不塞在分系统里

### composite API 正确写法

```python
# ❌ 错误：直接写业务逻辑
class DashboardView(APIView):
    def get(self, request):
        total_users = User.objects.count()
        total_orders = Order.objects.count()
        return Response({...})

# ✅ 正确：通过接口调用各分系统 Service
class DashboardView(APIView):
    def get(self, request):
        user_svc = PluginRegistry.get_service(IUserStatsService)
        order_svc = PluginRegistry.get_service(IOrderStatsService)
        return Response({
            'users': user_svc.get_stats(),
            'orders': order_svc.get_stats(),
        })
```

---

## 十二、Docker 部署

```yaml
# docker-compose.yml
version: "3.8"
services:
  web:
    build: .
    command: gunicorn tenon.wsgi -b 0.0.0.0:8000
    volumes:
      - .:/app
      - media:/app/media
    depends_on:
      - postgres
      - redis

  celery_worker:
    build: .
    command: celery -A tenon worker -l info
    volumes:
      - .:/app
    depends_on:
      - redis
      - postgres

  celery_beat:
    build: .
    command: celery -A tenon beat -l info
    volumes:
      - .:/app
    depends_on:
      - redis
      - postgres

  postgres:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: tenon
      POSTGRES_USER: platform
      POSTGRES_PASSWORD: ***

  redis:
    image: redis:7-alpine

  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    volumes:
      - minio_data:/data
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: ***

  nginx:
    image: nginx:alpine
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - media:/media
    ports:
      - "80:80"
    depends_on:
      - web

volumes:
  pgdata:
  minio_data:
  media:
```

Django settings 中 ENGINE 配 `django.db.backends.postgresql`。

---

## 十三、反模式文档（放在 core/__init__.py 或 ARCHITECTURE.md）

```markdown
## 反模式（Code Review 红线）

请实现以下检查清单，确保每个分系统的代码不违反架构原则：

- [ ] `from tenon.subsystems.xxx.models import ...` 出现在另一个分系统中
- [ ] 分系统的 views.py 直接调另一个分系统的 services.py
- [ ] core/ 里出现业务概念（User、Order、Invoice）
- [ ] 跨分系统的 ForeignKey
- [ ] plugin.toml 里声明了依赖但实际没用到
- [ ] composite view 里直接写业务逻辑而非调 Service 接口
- [ ] 分系统缺少 plugin.toml
- [ ] discovery 扫描到了 _template 等脚手架目录
```

---

## 十四、Agent 团队开发顺序

Coordinator 按以下顺序分派实现任务：

1. **搭建脚手架** — core/ + plugin_registry/ + subsystems/_template/ + ARCHITECTURE.md
2. **实现 usercenter 分系统** — 验证完整链路（plugin.toml → apps.py ready() → register → URL 自动发现）
3. **实现 notification 分系统** — 验证跨分系统通信（接口抽象 + 依赖注入）
4. **实现 storage 分系统** — 验证文件管理链路
5. **实现 billing 分系统** — 验证跨分系统事件通信（Signals 解耦）
6. **实现 composite API 层** — 验证跨分系统数据聚合
7. **Docker 部署验证** — 完整 compose 启动

每个阶段完成后，Reviewer 审查确认不违反反模式清单。
