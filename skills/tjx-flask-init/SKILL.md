---
name: tjx-flask-init
description: Initialize a production-ready Flask project scaffold with app factory pattern, layered architecture, and configurable components. Use this skill whenever the user wants to create a new Flask project, set up a Flask backend, scaffold a Flask API, or initialize any Flask-based application — even if they don't explicitly say "scaffold" or "initialize". Also use it when the user mentions building a REST API with Flask, setting up Flask with SQLAlchemy, or creating a Flask project template.
---

# Flask Project Initializer

Generate a Flask project scaffold with a clean layered architecture. The project uses the app factory pattern, environment-based configuration, and a modular API layer. **Only generate framework skeleton code — no business logic implementation.** Files should contain class/function definitions, imports, and `pass` placeholders. The exception is the database connection: test MySQL connectivity on startup to verify the configuration is correct.

## Step 1: Gather Configuration

**All interaction with the user must be in Chinese.** Questions, explanations, prompts, and step descriptions should all use Chinese. Code comments should also be in Chinese.

如果用户的 prompt 中已经包含了所有配置信息（项目名、路由风格、组件选择、业务模块），直接跳过询问，按用户要求生成。

### 选择模式

首先询问用户选择哪种模式。使用 AskUserQuestion 工具：

**问题：选择配置模式**
- 选项 1：**默认配置（一键生成）** — 使用推荐配置直接生成，只需输入项目名和业务模块名
- 选项 2：**自定义配置** — 逐一选择各组件

### 默认配置

如果用户选择默认配置，只需确认两件事：

1. **项目名称** — 默认 `server`，用户可修改
2. **需要的业务模块** — 询问用户需要哪些模块（如 user、patient、product、order 等），每个模块生成 api/<module>/ + models/<module>.py

默认包含的组件（不需要询问）：
- 路由风格：Blueprint
- Celery + Redis：包含
- Flask-Migrate：包含
- 公共工具：全部包含（response/exceptions/auth/middleware/pagination/constants）

### 自定义配置

如果用户选择自定义，使用 AskUserQuestion 工具一次性展示所有配置选项让用户勾选：

**问题 1：项目名称** — 默认 `server`，用户可修改

**问题 2：路由风格**（单选）
- Blueprint（推荐）
- Flask-RESTful

**问题 3：包含的组件**（多选，默认全选）
- Celery + Redis（异步任务）
- Flask-Migrate（数据库迁移）
- 统一响应格式（common/response/）
- 异常处理（common/exceptions/）
- JWT 认证装饰器（common/utils/auth.py）
- 请求日志中间件（common/middleware/）
- 分页工具（common/utils/pagination.py）
- 常量文件（common/constants/）

**问题 4：需要的业务模块**
- 用户输入模块名称（如 user、patient、product、order 等）
- 每个模块生成对应的 api/<module>/ 目录和 models/<module>.py

## Step 2: Generate the Project

Create all files inside the project directory. The structure below is the **canonical layout** — generate exactly these files, nothing more, nothing less (based on user's component choices).

```
<project_name>/
├── app/
│   ├── __init__.py              # App factory: create_app()
│   ├── config/
│   │   ├── __init__.py          # Config loader + base config
│   │   ├── development.py
│   │   ├── production.py
│   │   └── testing.py
│   ├── extensions/
│   │   ├── __init__.py          # Re-exports all extension instances
│   │   ├── db.py
│   │   ├── redis.py             # (if Celery chosen)
│   │   ├── jwt.py
│   │   └── cors.py
│   ├── models/
│   │   ├── __init__.py          # Import all models here for Alembic discovery
│   │   └── <module>.py          # One file per user-specified module
│   ├── api/
│   │   ├── __init__.py          # register_blueprints() or register_resources()
│   │   └── <module>/
│   │       ├── __init__.py
│   │       ├── views.py         # Routes / endpoints
│   │       ├── service.py       # Business logic
│   │       └── schemas.py       # Serialization / validation
│   ├── common/
│   │   ├── __init__.py
│   │   ├── utils/               # (if chosen)
│   │   │   ├── __init__.py
│   │   │   ├── auth.py          # JWT decorator
│   │   │   └── pagination.py
│   │   ├── middleware/           # (if chosen)
│   │   │   ├── __init__.py
│   │   │   └── request_logger.py
│   │   ├── response/            # (if chosen)
│   │   │   ├── __init__.py
│   │   │   └── response.py
│   │   ├── exceptions/          # (if chosen)
│   │   │   ├── __init__.py
│   │   │   └── errors.py
│   │   └── constants/           # (if chosen)
│   │       └── __init__.py
│   ├── templates/               # Empty dir with .gitkeep
│   ├── static/                  # Empty dir with .gitkeep
│   └── tasks/                   # (if Celery chosen)
│       ├── __init__.py
│       ├── celery_app.py
│       └── example_task.py
├── migrations/                  # (if Migrate chosen, run flask db init after)
├── tests/
│   ├── __init__.py
│   ├── conftest.py              # Pytest fixtures (test client, db setup)
│   └── test_<module>.py         # One test file per module
├── logs/                        # Empty dir with .gitkeep
├── requirements.txt
├── .env
├── .env.example
├── .gitignore
├── run.py                       # Dev server entry point
├── wsgi.py                      # Production WSGI entry (gunicorn/uwsgi)
└── README.md
```

## Step 3: Code Standards for Generated Files

**核心原则：只生成框架骨架，不写业务逻辑实现。** 每个文件包含必要的 import、类/函数定义、`pass` 占位。唯一例外是数据库连接测试 — 启动时验证 MySQL 连接是否正常。

### app/__init__.py (App Factory)

App factory 只做三件事：加载配置、初始化扩展、注册蓝图。**启动时测试数据库连接**：

```python
from flask import Flask
from app.config import config_by_name
from app.extensions import init_extensions
from app.api import register_blueprints


def create_app(config_name="development"):
    app = Flask(__name__)
    app.config.from_object(config_by_name[config_name])

    init_extensions(app)
    register_blueprints(app)

    # 启动时测试数据库连接
    _test_db_connection(app)

    return app


def _test_db_connection(app):
    """启动时测试 MySQL 连接是否正常"""
    from app.extensions.db import db
    with app.app_context():
        try:
            db.engine.connect()
            print("[OK] 数据库连接成功")
        except Exception as e:
            print(f"[ERROR] 数据库连接失败: {e}")
```

### app/config/__init__.py

Must export a `config_by_name` dict mapping `"development"`, `"production"`, `"testing"` to their config classes. The base config class should include:
- `SECRET_KEY` from env
- Database connection using separate env vars — **do NOT use a single DATABASE_URL**:
  - `DB_HOST` (default: `localhost`)
  - `DB_PORT` (default: `3306`)
  - `DB_USER` (default: `root`)
  - `DB_PASSWORD` (default: empty)
  - `DB_NAME` (default: `flask_db`)
  - Compose `SQLALCHEMY_DATABASE_URI` from these: `f"mysql+pymysql://{DB_USER}:{DB_PASSWORD}@{DB_HOST}:{DB_PORT}/{DB_NAME}?charset=utf8mb4"`
  - For testing config, override to use sqlite: `SQLALCHEMY_DATABASE_URI = "sqlite:///:memory:"`
- `SQLALCHEMY_TRACK_MODIFICATIONS = False`
- `JWT_SECRET_KEY` from env
- `JSON_AS_ASCII = False` (for Chinese character support)
- CORS, Redis, Celery settings as needed

### app/extensions/__init__.py

Must export an `init_extensions(app)` function that initializes each extension with the app. Each extension gets its own file (db.py, jwt.py, etc.) that creates the instance:

```python
# db.py
from flask_sqlalchemy import SQLAlchemy
db = SQLAlchemy()
```

### app/models/

**只定义模型结构，不写方法和业务逻辑。** 根据用户指定的模块名称生成对应的模型文件，每个模型包含常用字段定义和 `__tablename__`。如果多个模块之间有关系，应展示外键关联。

```python
# models/user.py — 只定义字段，不写方法
from app.extensions.db import db
from datetime import datetime


class User(db.Model):
    __tablename__ = "users"

    id = db.Column(db.Integer, primary_key=True, autoincrement=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(256), nullable=False)
    is_active = db.Column(db.Boolean, default=True)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    updated_at = db.Column(db.DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
```

`models/__init__.py` must import all models so Alembic can detect them.

### app/api/ — Blueprint style

Each module's `views.py` creates a Blueprint and defines routes with `pass` placeholders. The top-level `api/__init__.py` has `register_blueprints(app)` that imports and registers each module's blueprint with a versioned prefix (`/api/v1/`).

```python
# api/user/views.py
from flask import Blueprint

user_bp = Blueprint("user", __name__)


@user_bp.route("/users", methods=["GET"])
def get_users():
    """获取用户列表"""
    pass


@user_bp.route("/users/<int:user_id>", methods=["GET"])
def get_user(user_id):
    """获取单个用户"""
    pass


@user_bp.route("/users", methods=["POST"])
def create_user():
    """创建用户"""
    pass


@user_bp.route("/users/<int:user_id>", methods=["PUT"])
def update_user(user_id):
    """更新用户"""
    pass


@user_bp.route("/users/<int:user_id>", methods=["DELETE"])
def delete_user(user_id):
    """删除用户"""
    pass
```

### app/api/ — Flask-RESTful style

Each module's `views.py` defines Resource classes with `pass` in each method. The top-level `api/__init__.py` has `register_resources(app)` that creates an `Api` instance and adds resources with versioned prefixes.

```python
# api/user/views.py
from flask_restful import Resource

class UserList(Resource):
    def get(self):
        """获取用户列表"""
        pass

    def post(self):
        """创建用户"""
        pass

class UserDetail(Resource):
    def get(self, user_id):
        """获取单个用户"""
        pass

    def put(self, user_id):
        """更新用户"""
        pass

    def delete(self, user_id):
        """删除用户"""
        pass
```

### app/api/*/service.py

**只定义类和方法签名，方法体用 `pass`。** 导入 app.models 和 app.extensions.db。

```python
# api/user/service.py
from app.models.user import User
from app.extensions.db import db


class UserService:
    @staticmethod
    def get_all():
        """获取所有用户"""
        pass

    @staticmethod
    def get_by_id(user_id):
        """根据 ID 获取用户"""
        pass

    @staticmethod
    def create(data):
        """创建用户"""
        pass

    @staticmethod
    def update(user_id, data):
        """更新用户"""
        pass

    @staticmethod
    def delete(user_id):
        """删除用户"""
        pass
```

### app/api/*/schemas.py

**只定义 schema 类/函数签名，不写具体序列化逻辑。**

```python
# api/user/schemas.py


class UserSchema:
    """用户序列化"""

    @staticmethod
    def dump(user):
        """模型 -> 字典"""
        pass

    @staticmethod
    def load(data):
        """字典 -> 模型"""
        pass
```

### common/response/response.py

统一响应格式 — 这个需要完整实现，因为是基础设施：

```python
from flask import jsonify


def success(data=None, message="ok", code=200):
    return jsonify({"code": code, "message": message, "data": data}), code


def error(message="error", code=400, data=None):
    return jsonify({"code": code, "message": message, "data": data}), code
```

### common/exceptions/errors.py

自定义异常类 + 错误处理器注册 — 需要完整实现：

```python
from flask import jsonify


class AppError(Exception):
    def __init__(self, message="Internal Error", code=500):
        self.message = message
        self.code = code


def register_error_handlers(app):
    @app.errorhandler(AppError)
    def handle_app_error(e):
        return jsonify({"code": e.code, "message": e.message, "data": None}), e.code

    @app.errorhandler(404)
    def handle_not_found(e):
        return jsonify({"code": 404, "message": "资源不存在", "data": None}), 404

    @app.errorhandler(500)
    def handle_internal_error(e):
        return jsonify({"code": 500, "message": "服务器内部错误", "data": None}), 500
```

### common/utils/auth.py

JWT 认证装饰器 — 需要完整实现：

```python
from functools import wraps
from flask_jwt_extended import verify_jwt_in_request


def login_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        verify_jwt_in_request()
        return f(*args, **kwargs)
    return decorated
```

### common/middleware/request_logger.py

请求日志中间件 — 只定义框架，不写具体日志逻辑：

```python
def register_request_logger(app):
    """注册请求日志中间件"""
    pass
```

### common/utils/pagination.py

分页工具 — 只定义函数签名：

```python
def paginate(query, page, per_page):
    """分页查询"""
    pass
```

### common/constants/__init__.py

常量文件 — 只定义常量结构：

```python
# 角色常量
ROLE_ADMIN = "admin"
ROLE_USER = "user"

# 分页默认值
DEFAULT_PAGE = 1
DEFAULT_PER_PAGE = 20
```

### tests/conftest.py

Pytest fixtures for test client and database:

```python
import pytest
from app import create_app
from app.extensions.db import db as _db


@pytest.fixture
def app():
    app = create_app("testing")
    with app.app_context():
        _db.create_all()
        yield app
        _db.drop_all()


@pytest.fixture
def client(app):
    return app.test_client()
```

### tests/test_*.py

**只定义测试类和测试方法，方法体用 `pass`。**

```python
class TestUserAPI:
    def test_get_users(self, client):
        """测试获取用户列表"""
        pass

    def test_create_user(self, client):
        """测试创建用户"""
        pass
```

### run.py

```python
from app import create_app

app = create_app("development")

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
```

### wsgi.py

```python
from app import create_app

app = create_app("production")
```

### requirements.txt

Pin major versions. Always include:
```
Flask>=3.0
Flask-SQLAlchemy>=3.1
Flask-JWT-Extended>=4.6
Flask-CORS>=4.0
flask-migrate>=4.0  # if Migrate chosen
celery>=5.3         # if Celery chosen
redis>=5.0          # if Celery chosen
python-dotenv>=1.0
pymysql>=1.1
gunicorn>=21.2
pytest>=8.0
```

### .env

Template with all required env vars, commented:

```
FLASK_APP=run.py
FLASK_ENV=development
SECRET_KEY=change-me-in-production

# 数据库配置
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your_password_here
DB_NAME=flask_db

JWT_SECRET_KEY=change-me-in-production
# REDIS_URL=redis://localhost:6379/0
# CELERY_BROKER_URL=redis://localhost:6379/1
```

### .gitignore

Standard Python + Flask gitignore (include `__pycache__/`, `*.pyc`, `.env`, `logs/*.log`, `instance/`, `.pytest_cache/`, `migrations/versions/`).

### README.md

Brief setup instructions: install deps, set env vars, run migrations (if applicable), run dev server, run tests.

## Step 4: Self-Check and Auto-Fix

After generating all files, run a comprehensive self-check to verify the project is error-free. **This step is mandatory — do not skip it.** Use Bash 工具执行检查命令。

### Check 1: Python Syntax Errors

对项目中所有 `.py` 文件运行语法检查：

```bash
cd <project_name>
python -m py_compile run.py
python -m py_compile wsgi.py
# 遍历 app/ 下所有 .py 文件逐一检查
find app -name "*.py" -exec python -m py_compile {} \;
```

如果有语法错误，立即修复并重新检查该文件。

### Check 2: Circular Import Detection

编写并运行一个脚本检测循环导入：

```bash
cd <project_name>
python -c "
import sys
sys.path.insert(0, '.')
from app import create_app
app = create_app('testing')
print('[OK] 应用创建成功，无循环导入')
"
```

如果出现 `ImportError` 或循环导入错误，分析导入链，调整 import 语句（如将导入移入函数内部），然后重新检查。

### Check 3: Configuration Validation

验证配置加载是否正常：

```bash
cd <project_name>
python -c "
import sys
sys.path.insert(0, '.')
from app.config import config_by_name
for name, cfg in config_by_name.items():
    print(f'  {name}: SQLALCHEMY_DATABASE_URI = {cfg.SQLALCHEMY_DATABASE_URI[:30]}...')
print('[OK] 配置加载成功')
"
```

### Check 4: Extension Initialization

验证所有扩展能正常初始化：

```bash
cd <project_name>
python -c "
import sys
sys.path.insert(0, '.')
from app import create_app
app = create_app('testing')
with app.app_context():
    # 检查数据库引擎
    from app.extensions.db import db
    print(f'  db engine: {db.engine}')
    # 检查蓝图注册
    for name, bp in app.blueprints.items():
        print(f'  blueprint: {name}')
print('[OK] 扩展初始化成功')
"
```

### Check 5: Model Registration

验证所有模型都被正确注册：

```bash
cd <project_name>
python -c "
import sys
sys.path.insert(0, '.')
from app import create_app
app = create_app('testing')
with app.app_context():
    from app.extensions.db import db
    for name, model in db.Model._decl_class_registry.items():
        if hasattr(model, '__tablename__'):
            print(f'  model: {name} -> {model.__tablename__}')
print('[OK] 模型注册成功')
"
```

### Auto-Fix Loop

如果任何检查失败：

1. **分析错误信息** — 确定是哪个文件、哪行代码的问题
2. **修复问题** — 使用 Edit 工具修改文件
3. **重新运行失败的检查** — 验证修复是否有效
4. **重新运行所有检查** — 确保修复没有引入新问题
5. **重复以上步骤** — 直到所有检查全部通过

常见问题及修复方案：
- **循环导入**：将 `from xxx import y` 改为在函数内部导入
- **蓝图未注册**：检查 `api/__init__.py` 中是否导入并注册了所有蓝图
- **模型未导入**：检查 `models/__init__.py` 是否导入了所有模型
- **配置键错误**：检查 config 类中的键名是否与 extensions 中使用的一致

### 最终确认

所有检查通过后，输出：

```
✅ 自检完成，全部通过：
  - 语法检查：X 个文件，无错误
  - 循环导入：无
  - 配置加载：正常
  - 扩展初始化：正常
  - 模型注册：X 个模型
```

## Step 5: Present the Result

After all self-checks pass:

1. Print a tree of the created structure
2. Print the self-check results (from Step 4)
3. Print the next steps (in Chinese):
   - `cd <project_name>`
   - `pip install -r requirements.txt`
   - `cp .env.example .env` (然后编辑填写数据库密码等配置)
   - 如果选了 Migrate: `flask db init && flask db migrate -m "initial" && flask db upgrade`
   - `python run.py`
4. 询问用户是否需要添加更多模型、接口或调整配置

## Important Notes

- **只生成框架骨架，不写业务逻辑实现** — views/service/schemas 的方法体用 `pass`，测试方法体用 `pass`。
- **例外：数据库连接测试必须完整实现** — app factory 启动时调用 `db.engine.connect()` 测试 MySQL 连接，成功打印 `[OK]`，失败打印 `[ERROR]` 和错误信息。
- **例外：基础设施代码必须完整实现** — 统一响应（`common/response/`）、异常处理（`common/exceptions/`）、JWT 装饰器（`common/utils/auth.py`）需要完整可用的代码。
- **自检必须通过** — 生成项目后必须运行 Step 4 的所有检查，发现问题立即修复，直到全部通过才能展示给用户。
- **所有与用户的交互必须使用中文** — 包括问题、解释、提示和步骤说明。
- **代码注释使用中文** — 所有生成的代码文件中的注释都用中文。
- Model 只定义字段和 `__tablename__`，不写方法。
- Each API module follows the **three-layer pattern**: views (HTTP) → service (logic) → model (data). Schemas handle serialization between layers.
- When the user says "both" for routing style, generate Blueprint style as default and mention they can switch to Flask-RESTful by modifying `api/__init__.py`.
