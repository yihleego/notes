# Python 框架

## Flask

## FastAPI

### FastAPI 的核心特性有哪些？它与 Flask、Django REST Framework 的最大区别是什么？

#### 🚀 FastAPI 的核心特性

1. 高性能
    - 基于 Starlette（异步 web 框架）和 Pydantic（数据验证和序列化库），底层利用 uvicorn / ASGI，性能接近 Node.js 和 Go，远超 Flask/Django。
    - 原生支持 异步 async/await。
2. 自动生成 API 文档
    - 内置 OpenAPI (Swagger UI) 和 ReDoc 文档，几乎零配置即可获得交互式 API 文档。
3. 类型注解驱动
    - 利用 Python 3 的 类型提示 (type hints) 实现参数验证、请求体验证、响应序列化和文档生成。
    - 写代码时即能获得 IDE 智能提示和类型检查。
4. 数据验证与序列化
    - 集成 Pydantic，提供强大、声明式的数据模型定义，支持复杂嵌套结构验证和自动错误返回。
5. 依赖注入系统
    - 内置简洁的 依赖注入机制，方便复用认证、数据库会话管理、权限控制等逻辑。
6. 异步与同步兼容
    - 路由既支持同步函数，也支持异步函数，开发者可以按需选择。

#### 与 Flask 的区别

- Flask 是一个极简框架，提供最小内核（路由 + WSGI），其他功能需依赖扩展库。
- FastAPI 则是“自带增强功能”的现代框架，尤其是：
- 内置自动文档生成（Flask 需要手动集成 flasgger 或 apispec）。
- 原生 async/await 支持，而 Flask 直到 2.0 以后才逐步引入异步支持。
- 更强的类型系统集成，Flask 本身与 Python 类型注解无关。

#### 与 Django REST Framework (DRF) 的区别

- Django REST Framework (DRF) 构建在 Django 之上，适合传统 单体 Web 应用：
- 提供 ORM、Admin、权限系统、模板、认证 等“一揽子解决方案”。
- 学习曲线较陡，但对需要完整生态的大型项目很有优势。
- FastAPI 则更偏向 轻量级、高性能 API 服务：
- ORM/数据库层不是强制的（可选 SQLAlchemy、Tortoise ORM 等）。
- 更加现代化（异步原生支持，依赖注入，类型驱动开发）。
- 性能普遍高于 DRF（因为 Django/DRF 仍基于 WSGI 设计，异步支持较弱）。

#### ✅ 总结

- FastAPI：高性能、自动文档、类型安全、异步优先，适合现代 API 和微服务。
- Flask：轻量灵活，生态丰富，适合小项目或需要高度定制的场景。
- DRF：功能齐全，适合基于 Django 的大型企业项目，尤其是需要 ORM 和管理后台的情况。

### FastAPI 为什么使用 Pydantic 和 类型提示 (type hints)？它解决了什么问题？

#### 背景问题

在传统的 Python Web 框架（Flask、Django 等）里，通常需要手动解析请求参数、校验数据类型，并在错误时返回合适的响应。常见问题有：

- 缺少输入验证：开发者需要手动检查字符串、数字、Email 格式等。
- 重复代码多：写一堆 if/try-except 来做验证。
- 文档生成困难：API 文档常常依赖人工维护，容易和实际接口不一致。
- 类型信息缺失：Python 是动态类型语言，编辑器和 IDE 很难提前发现类型错误。

#### 类型提示 (type hints) 的作用

Python 3.6+ 引入了 类型提示（PEP 484、PEP 526），虽然在运行时不会强制检查，但：

- 可以被 静态分析工具 (如 mypy, pyright) 检查类型错误。
- IDE (PyCharm, VSCode) 可以提供智能提示和补全。
- 框架（如 FastAPI）可以利用类型提示来自动推断参数类型，并生成数据验证逻辑。

示例：

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items/{item_id}")
def read_item(item_id: int, q: str | None = None):
    return {"item_id": item_id, "q": q}
```

在这里，item_id: int 表示 FastAPI 会自动把路径参数转换为 int，并在传入非法数据（如 "abc"）时自动返回 422 Unprocessable Entity 错误。

#### Pydantic 的作用

Pydantic 是 FastAPI 的核心依赖之一，它用 数据模型 (BaseModel) 来做：

- 数据解析：自动把 JSON、表单数据转换为 Python 对象。
- 数据验证：根据类型提示和字段约束自动校验数据（如 str、int、EmailStr、长度限制）。
- 错误信息友好：返回清晰的错误响应，方便客户端调试。
- 数据序列化：自动把对象转换为 JSON 响应。
- 文档生成支持：结合 OpenAPI/Swagger，自动生成准确的 API 文档。

示例：

```python
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    price: float
    is_offer: bool | None = None

@app.post("/items/")
def create_item(item: Item):
    return item
```

请求体传入：

```json
{
  "name": "Laptop",
  "price": 1200.5,
  "is_offer": true
}
```

FastAPI 会自动：

1. 验证 JSON 字段是否正确。
2. 把数据转换成 Item 对象。
3. 如果缺少字段或类型错误，返回带详细错误的 422 响应。

#### FastAPI 的设计哲学

FastAPI 结合 类型提示 + Pydantic，解决了以下问题：

- 简化参数解析与验证：开发者不必手动写验证逻辑。
- 统一数据模型：请求与响应使用同一个 Pydantic 模型，避免重复定义。
- 自动生成 API 文档：利用类型提示和 Pydantic 模型，自动生成 OpenAPI/Swagger UI 文档。
- 更好的开发体验：IDE 支持补全，类型检查工具能提前发现错误。
- 高性能：底层基于 Starlette（异步框架）和 Pydantic（基于 Cython/高性能解析），性能很好。

✅ 总结一句话：
FastAPI 使用 Pydantic 和 类型提示，把 Python 的静态类型优势（可读性、可维护性、IDE支持）与动态 Web 开发需求（请求解析、验证、序列化）结合起来，自动化地解决了数据验证、转换、错误处理和文档生成 等繁琐问题。

### 解释 依赖注入 (Dependency Injection) 在 FastAPI 中的实现原理和作用。

#### 什么是依赖注入 (Dependency Injection, DI)

依赖注入是一种 设计模式，它的核心思想是：

- 不在函数/类内部直接创建或管理依赖，而是通过外部传入需要的依赖对象。
- 函数/类 声明自己需要什么，由框架或调用方负责提供。

在 FastAPI 中，这意味着：你可以在路径操作函数 (endpoint) 中声明依赖项，FastAPI 会在运行时自动解析并传入，而无需手动管理。

#### FastAPI 中依赖注入的实现方式

FastAPI 提供了一个 Depends 类来标记依赖。

基本示例

```python
from fastapi import FastAPI, Depends

app = FastAPI()

def common_params(q: str | None = None, page: int = 1, size: int = 10):
    return {"q": q, "page": page, "size": size}

@app.get("/items/")
async def read_items(commons: dict = Depends(common_params)):
    return commons
```

解释：

- common_params 是一个依赖函数，定义了一些通用的参数。
- 在 read_items 中通过 Depends(common_params) 声明依赖。
- FastAPI 会自动调用 common_params，并把返回值作为 commons 传给 read_items。

#### FastAPI 依赖注入的原理

FastAPI 的依赖注入系统基于 Python 类型注解 + Depends，其核心原理大致如下：

1. 分析函数签名
   FastAPI 在启动时会读取每个路径函数的参数签名，通过类型注解和 Depends 判断哪些参数是依赖。
2. 构建依赖图 (Dependency Graph)
   每个依赖函数本身也可以依赖其他函数。FastAPI 会递归解析 Depends，形成一张 有向无环图 (DAG)。
3. 按需执行依赖
   当请求到达时，FastAPI 会根据依赖图顺序调用依赖函数，并把返回值传递给需要它们的函数。
4. 作用域管理
    - 请求级依赖：默认在每个请求时执行一次。
    - 单例/缓存依赖：可以通过 Depends(..., use_cache=True) 缓存结果，在同一请求中多处使用时只执行一次。
    - 可以通过 yield 在依赖函数中管理资源的创建与释放（类似上下文管理器）。

#### 依赖注入的作用

FastAPI 的 DI 提供了很多实际价值：

1. 解耦
   业务逻辑与依赖的实现细节解耦，便于维护和扩展。
   例如数据库会话、认证逻辑、配置加载等都能作为依赖单独管理。

2. 复用
   依赖函数可以在多个路由中复用，避免重复代码。

3. 可测试性
   测试时可以替换依赖函数，方便做 mock，不用修改核心业务逻辑。
   `app.dependency_overrides[get_db] = override_get_db`
4. 层次化依赖
   依赖本身也能依赖其他依赖，能逐层构建复杂功能模块（例如认证 -> 用户信息 -> 权限检查）。

5. 资源管理
   依赖函数可以用 yield 实现资源的生命周期管理，比如数据库连接、文件句柄等：

```python
from fastapi import Depends

def get_db():
    db = DBSession()
    try:
        yield db
    finally:
        db.close()
```

#### 总结

- 原理：基于 Python 类型注解和 Depends，FastAPI 在运行时自动解析依赖函数，形成依赖图并按需执行。
- 作用：解耦、复用、可测试、层次化依赖、资源管理。
- 实践：广泛用于数据库会话管理、认证授权、全局参数复用、服务层解耦。

### FastAPI 的生命周期事件（startup、shutdown）是如何工作的？

在 FastAPI 中，应用程序的生命周期事件（startup 和 shutdown）提供了一个机制，让你在应用启动或关闭时执行特定的逻辑。这些事件最常用于：
• 启动时初始化资源（数据库连接池、缓存客户端、后台任务等）
• 关闭时清理资源（关闭数据库连接、释放文件句柄、终止后台任务等）

#### 生命周期事件的注册方式

FastAPI 使用装饰器或 add_event_handler 方法来注册事件处理函数。

使用装饰器

```python
from fastapi import FastAPI

app = FastAPI()

@app.on_event("startup")
async def startup_event():
    print("应用启动了！初始化资源...")

@app.on_event("shutdown")
async def shutdown_event():
    print("应用关闭了！清理资源...")
```

使用 add_event_handler

```python
def custom_startup():
    print("启动逻辑")

def custom_shutdown():
    print("关闭逻辑")

app = FastAPI()
app.add_event_handler("startup", custom_startup)
app.add_event_handler("shutdown", custom_shutdown)
```

#### 工作原理

- Startup 阶段
    - 在应用启动（uvicorn main:app）时执行。
    - 会在 第一次请求之前 调用。
    - 适合执行初始化操作，例如连接数据库、加载模型、创建缓存连接等。
- Shutdown 阶段
    - 在应用停止时执行（例如收到 SIGTERM 或 Ctrl+C 时）。
    - 适合执行清理操作，例如关闭数据库连接、清理缓存、保存状态等。

#### 异步与同步支持

FastAPI 的生命周期事件既支持同步函数，也支持异步函数：

```python
@app.on_event("startup")
async def startup_event():
    await some_async_init()

@app.on_event("shutdown")
def shutdown_event():
    sync_cleanup()
```

#### 高级用法：依赖注入 & Context Manager

从 FastAPI 0.95+ 开始推荐使用 lifespan context manager 来替代 @on_event，它能提供更清晰的生命周期控制。

```python
from fastapi import FastAPI
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup
    print("应用启动，初始化资源...")
    yield
    # shutdown
    print("应用关闭，清理资源...")

app = FastAPI(lifespan=lifespan)
```

这样写可以避免分散的 startup/shutdown 装饰器，使生命周期逻辑集中在一个地方。

### BackgroundTasks 和 Depends 有什么区别和典型应用场景？

#### BackgroundTasks

功能

- 用于在 请求返回响应后 执行一些异步任务。
- 典型场景是用户不需要等待任务完成就能拿到响应。

```python
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()

def write_log(message: str):
    with open("log.txt", "a") as f:
        f.write(message + "\n")

@app.post("/send/")
async def send_notification(background_tasks: BackgroundTasks, email: str):
    background_tasks.add_task(write_log, f"Send email to {email}")
    return {"message": "Email scheduled"}
```

应用场景

- 写日志（例如：审计、请求统计）
- 发送邮件或通知（用户请求立即返回，但后台慢慢发）
- 调用第三方 API（非关键任务）
- 清理、缓存更新等辅助操作

#### Depends

功能

- 是 依赖注入（Dependency Injection） 的机制。
- 用来抽取、复用逻辑，比如认证、数据库连接、公共参数、权限控制等。

```python
from fastapi import Depends

def get_db():
    db = {"connection": "fake_db"}
    try:
        yield db
    finally:
        print("Close DB")

@app.get("/items/")
async def read_items(db=Depends(get_db)):
    return {"db_connection": db["connection"]}
``` 

应用场景

- 数据库会话管理
- 用户认证与权限校验
- 公共业务逻辑（如分页参数、配置注入）
- 封装重复性的前置处理逻辑

### 如何在 FastAPI 中实现 动态路由参数验证？比如：正则匹配、枚举值限制。

#### 基本的路径参数类型验证

FastAPI 默认会根据类型注解验证参数类型：

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items/{item_id}")
async def read_item(item_id: int):  # 这里会自动验证 item_id 是否为 int
    return {"item_id": item_id}
```

访问 /items/abc 会直接返回 422 错误，因为 abc 不能转换为整数。

#### 使用正则匹配动态路由参数

可以通过 Path 参数的 regex 选项来限制格式：

```python
from fastapi import FastAPI, Path

app = FastAPI()

@app.get("/users/{username}")
async def get_user(
    username: str = Path(..., regex="^[a-zA-Z0-9_-]{3,16}$")  # 用户名限制
):
    return {"username": username}
```

示例：

- ✅ /users/jack_99
- ❌ /users/ab （长度太短）
- ❌ /users/jack@ （包含非法字符）

#### 使用 Enum 限制参数值

如果某个参数只允许固定几个值，可以使用 Python 的 Enum：

```python
from enum import Enum
from fastapi import FastAPI

class UserRole(str, Enum):
    admin = "admin"
    user = "user"
    guest = "guest"

app = FastAPI()

@app.get("/roles/{role}")
async def get_role(role: UserRole):
    return {"role": role}
```

示例：

- ✅ /roles/admin
- ✅ /roles/user
- ❌ /roles/super （会返回 422 错误）

#### 动态组合：正则 + 枚举 + Pydantic 验证

如果需要更复杂的验证逻辑，可以用 Pydantic 模型 或 validator：

```python
from fastapi import FastAPI, Path
from pydantic import BaseModel, constr, validator

app = FastAPI()

class UserModel(BaseModel):
    username: constr(regex=r"^[a-zA-Z0-9_-]{3,16}$")
    role: str

    @validator("role")
    def validate_role(cls, v):
        if v not in {"admin", "user", "guest"}:
            raise ValueError("Invalid role")
        return v

@app.post("/users/")
async def create_user(user: UserModel):
    return user
```

- username 用正则约束。
- role 用 validator 手工校验。

### Request Body、Query Params、Form、Header、Cookie 在 FastAPI 中是如何区分和处理的？

#### Query Parameters（查询参数）

- 出现在 URL 中 ?key=value&key2=value2 部分。
- 默认情况：FastAPI 会把没有明确标注的 函数参数 当作 Query 参数处理。

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items/")
async def read_items(skip: int = 0, limit: int = 10):
    return {"skip": skip, "limit": limit}

# 或者

class ItemQuery(BaseModel):
    skip: int = 0
    limit: int = 10

@app.get("/items")
async def read_items(params: ItemQuery = Depends()):
    return {"params": params}
```

#### Request Body（请求体）

- 通常用于 POST / PUT 等方法，携带 JSON。
- 用 Pydantic 模型 或者 Body(…) 来声明。

```python
from fastapi import Body
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    price: float

@app.post("/items/")
async def create_item(item: Item):
    return {"item": item}
```

#### Form Data（表单数据）

- 常见于 HTML 表单提交 (application/x-www-form-urlencoded 或 multipart/form-data)。
- 需要用 Form(…)。

```python
from fastapi import Form

@app.post("/login/")
async def login(username: str = Form(...), password: str = Form(...)):
    return {"username": username}
```

#### Headers（请求头）

请求头数据，比如 User-Agent、Authorization 等。

- 用 Header(…) 来声明。

```python
from fastapi import Header

@app.get("/headers/")
async def read_headers(user_agent: str = Header(None)):
    return {"User-Agent": user_agent}
```

#### Cookies

- 从请求 Cookie 中获取值。
- 用 Cookie(…)。

```python
from fastapi import Cookie

@app.get("/cookies/")
async def read_cookie(session_id: str | None = Cookie(None)):
    return {"session_id": session_id}
```

### 如何在 FastAPI 中实现 自定义请求体验证？比如跨字段的复杂校验。

#### 基础：字段级验证

字段级别的验证器使用 @validator，只对单个字段生效。

```python
from pydantic import BaseModel, validator

class User(BaseModel):
    username: str
    password: str

    @validator("password")
    def password_length(cls, v):
        if len(v) < 6:
            raise ValueError("密码长度至少6位")
        return v
```

#### 跨字段校验：@root_validator

当你需要多个字段之间的复杂逻辑时，可以用 @root_validator。

```python
from pydantic import BaseModel, root_validator

class RegisterRequest(BaseModel):
    password: str
    confirm_password: str
    age: int

    @root_validator
    def check_passwords_and_age(cls, values):
        pw = values.get("password")
        cpw = values.get("confirm_password")
        age = values.get("age")

        if pw != cpw:
            raise ValueError("两次密码不一致")
        if age < 18:
            raise ValueError("必须年满18岁")
        return values
```

#### Pydantic v2 的写法（推荐）

如果你用的是 Pydantic v2（FastAPI 0.100+ 默认支持），跨字段校验应使用 @model_validator：

```python
from pydantic import BaseModel, field_validator, model_validator

class RegisterRequest(BaseModel):
    password: str
    confirm_password: str
    age: int

    @field_validator("password")
    def check_password_length(cls, v):
        if len(v) < 6:
            raise ValueError("密码长度至少6位")
        return v

    @model_validator(mode="after")
    def check_passwords_and_age(self):
        if self.password != self.confirm_password:
            raise ValueError("两次密码不一致")
        if self.age < 18:
            raise ValueError("必须年满18岁")
        return self
```

#### 高级用法：自定义依赖做复杂校验

如果你的校验逻辑不仅依赖请求体，还涉及数据库或外部条件，可以在 FastAPI 依赖项中实现：

```python
from fastapi import Depends, HTTPException

def validate_register(data: RegisterRequest):
    # 例如检查用户名是否存在于数据库
    if data.password != data.confirm_password:
        raise HTTPException(status_code=400, detail="两次密码不一致")
    return data

@app.post("/register")
async def register(data: RegisterRequest = Depends(validate_register)):
    return {"msg": "注册成功"}
```

### WebSocket 在 FastAPI 中是如何支持的？请描述典型的使用场景。

在 FastAPI 中，WebSocket 是原生支持的（基于 Starlette 的实现），可以用来处理全双工、实时通信的场景。下面我分几个部分说明：

#### WebSocket 在 FastAPI 中的支持方式

FastAPI 提供了专门的接口来处理 WebSocket 请求：

- 使用 WebSocket 对象来表示一个连接。
- 在路由函数中，通过 await websocket.accept() 来接受连接。
- 使用 await websocket.send_text(...) 或 await websocket.send_json(...) 发送数据。
- 使用 await websocket.receive_text() 或 await websocket.receive_json() 接收数据。
- 最后可调用 await websocket.close() 来关闭连接。

```
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import List
import uvicorn

app = FastAPI()

# 管理连接的类
class ConnectionManager:
    def __init__(self):
        self.active_connections: List[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)

    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)

    async def broadcast(self, message: str):
        for connection in self.active_connections:
            await connection.send_text(message)

manager = ConnectionManager()

@app.get("/")
async def get():
    return {"message": "WebSocket 聊天室服务已启动，请连接 /ws"}

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await manager.connect(websocket)
    try:
        while True:
            data = await websocket.receive_text()
            await manager.broadcast(f"用户说: {data}")
    except WebSocketDisconnect:
        manager.disconnect(websocket)
        await manager.broadcast("有一个用户离开了聊天室")

if __name__ == "__main__":
    uvicorn.run("chat:app", host="0.0.0.0", port=8000, reload=True)
```

#### 典型使用场景

WebSocket 常用于需要 实时通信 的场合，而 FastAPI 因为异步特性，非常适合处理这些应用。常见场景包括：

1. 聊天应用
    - 用户之间通过 WebSocket 建立长连接，服务器负责转发消息。
    - 可以实现单聊、群聊，以及消息已读/输入中提示。
2. 实时通知 / 推送
    - 比如股票价格更新、新闻推送、系统消息提醒。
    - 客户端无需反复轮询，而是服务端主动推送。
3. 在线协作应用
    - 如文档协作、白板协作，多个用户的操作需要实时同步。
4. 实时数据可视化
    - 如 IoT 设备的状态监控、日志流监控、传感器数据展示。
    - WebSocket 用来将服务端不断更新的数据推送到前端图表中。
5. 在线游戏
    - 多玩家游戏需要实时同步状态（位置、分数、操作事件等）。

#### WebSocket 与 HTTP 的区别

- HTTP：请求/响应模式，短连接或伪长连接（如轮询、SSE）。
- WebSocket：真正的全双工长连接，客户端和服务端可以随时互发消息，适合高频、低延迟的实时应用。

### 如何处理 文件上传/下载，并保证性能与安全？

#### 文件上传

避免直接 await file.read() 读取整个文件，而是分块写入磁盘：

```python
@app.post("/uploadfile/")
async def upload_file(file: UploadFile = File(...)):
    with open(f"uploads/{file.filename}", "wb") as f:
        while chunk := await file.read(1024 * 1024):  # 每次1MB
            f.write(chunk)
    return {"status": "success", "filename": file.filename}
```

- 限制文件大小：可在 Nginx/反向代理层或 FastAPI 中设置请求体大小限制。
- 校验文件类型：不要仅依赖前端的 MIME type，应使用 python-magic 或自定义校验。
- 路径安全：禁止直接使用用户提供的 filename，需用 os.path.basename 或 uuid 重命名，防止目录穿越攻击。
- 权限控制：上传目录应仅对应用可写，避免被直接访问。

#### 文件下载

OSS

### 如何组织大型 FastAPI 项目的代码结构（例如分层架构：routers、services、repositories）？

pass

### 如何在 FastAPI 中集成 SQLAlchemy 并实现依赖注入？

pass

### 描述如何使用 FastAPI 构建 微服务架构。

目前没有使用Python写分布式服务的场景。

### 如何在 FastAPI 中实现 全局异常处理 和 自定义异常类？

FastAPI 提供了 @app.exception_handler 装饰器，可以为指定的异常类型注册一个全局处理函数。比如：

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()

@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    return JSONResponse(
        status_code=500,
        content={"message": f"服务器内部错误: {str(exc)}"}
    )
```

### 如何在 FastAPI 中实现 中间件 (Middleware)，举例一个请求耗时统计的中间件。

#### 使用函数装饰器方式

```python
from fastapi import FastAPI, Request
import time

app = FastAPI()

@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start_time = time.time()       # 请求开始时间
    response = await call_next(request)   # 执行下一个中间件或路由处理函数
    process_time = time.time() - start_time   # 计算耗时
    response.headers["X-Process-Time"] = str(process_time)  # 添加到响应头
    return response

@app.get("/")
async def root():
    return {"message": "Hello World"}
```

#### 使用类方式 (BaseHTTPMiddleware)

```python
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware
import time

app = FastAPI()

class ProcessTimeMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        start_time = time.time()
        response = await call_next(request)
        process_time = time.time() - start_time
        response.headers["X-Process-Time"] = str(process_time)
        return response

# 注册中间件
app.add_middleware(ProcessTimeMiddleware)

@app.get("/")
async def root():
    return {"message": "Hello World"}
```

### FastAPI 是基于 Starlette 和 Pydantic 的，解释它在性能上的优势。

#### 基于 Starlette 的高性能网络层

- Starlette 是一个轻量级的 ASGI 框架，它本身就以高性能著称。ASGI（Asynchronous Server Gateway Interface）是 WSGI 的异步升级版，支持异步 I/O。
- 因为 FastAPI 构建在 Starlette 之上，它继承了以下优点：
    - 异步请求处理：支持 async/await，能够同时处理更多请求，而不是阻塞在 I/O 上。
    - 高吞吐量与低延迟：与 Node.js、Go 语言框架相比较，Starlette（进而 FastAPI）在性能基准测试里非常接近甚至超过它们。
    - 轻量而灵活：Starlette 没有冗余的抽象层，直接操作 ASGI 协议，使得响应速度更快。

#### 基于 Pydantic 的数据解析与验证

- Pydantic 提供快速的数据验证与序列化。相比 Python 的传统 dataclasses 或手写校验逻辑，Pydantic 使用 Cython 优化和缓存机制，速度更快。
- 优势体现在：
    - 数据解析快：请求中的 JSON 数据会直接被转换成 Python 类型（如 int, str, datetime），无需手动转换。
    - 零额外开销的验证：请求参数在进入业务逻辑前已经过验证，避免了额外的防御性编程逻辑。
    - 高效的序列化：响应时，模型可以直接导出为 JSON，减少数据转换的性能损耗。

#### 异步 + 高效数据处理的结合

- FastAPI 将 Starlette 的异步网络层 和 Pydantic 的高效数据层结合：
    - 在请求进来时，Starlette 快速接收并调度请求。
    - 请求体会立即交给 Pydantic 进行高效的解析和验证。
    - 开发者拿到的就是已经验证过的 Python 对象，可以直接使用。
    - 响应时，同样由 Pydantic 高效序列化，Starlette 快速返回响应。

这意味着在 高并发场景下，FastAPI 不仅能高效利用 CPU 和 I/O，还能减少 Python 层面的数据处理开销。

#### 其他性能相关优势

- 自动生成文档：虽然这是功能层面优势，但因为基于 Pydantic 的数据模型定义，FastAPI 可以零额外开销地生成 OpenAPI/Swagger 文档。
- 更少的 bug 风险：类型验证自动完成，减少手写错误和运行时异常，这在高性能系统中间接提升了稳定性。
- 可扩展性强：依赖注入系统轻量高效，不会成为性能瓶颈。

### 如何使用 async/await 提升 FastAPI 的吞吐量？在什么情况下不要使用 async？

#### 什么时候使用 async/await 能提升吞吐量

FastAPI 基于 Starlette + Uvicorn (ASGI)，它的事件循环由 asyncio 驱动。使用 async/await 可以在 等待 I/O 时不阻塞事件循环，从而提升并发处理能力。适用场景包括：

1. 数据库访问
    - 使用支持异步的驱动（如 asyncpg、databases 库）访问 PostgreSQL、MySQL。
    - 在等待 SQL 查询返回时，可以处理其他请求。
2. HTTP 请求调用
    - 用 httpx.AsyncClient 代替 requests。
    - 调用外部 API 时无需阻塞线程。
3. 文件/对象存储
    - 异步上传/下载到 AWS S3、MinIO、GCS。
4. 消息队列
    - 使用异步客户端访问 Kafka、RabbitMQ、Redis Stream。

👉 总结：当你的函数逻辑 主要耗时在 I/O 等待 上时，改成 async def 并使用 await 可以显著提升吞吐量。

#### 什么时候不要使用 async

有些情况下使用 async 反而没有意义，甚至会拖累性能：

1. CPU 密集任务
    - 如大数据计算、机器学习推理、视频编码、加密解密等。
    - 即使函数是 async def，CPU 运算也会占满事件循环，导致其他请求无法及时调度。
    - ✅ 解决办法：
        - 放到后台任务（BackgroundTasks）。
        - 或使用 concurrent.futures.ThreadPoolExecutor / ProcessPoolExecutor。
        - 或者用 Celery、RQ 等任务队列。
2. 阻塞库
    - 如果库本身是同步的（如 requests、大多数 Python ORM 的同步版本），即使你写了 async def，内部依然是阻塞的。
    - 这会导致整个事件循环停顿，吞吐量下降。
    - ✅ 解决办法：
        - 要么用异步版本（如 httpx.AsyncClient、SQLAlchemy 2.x async）。
        - 要么用 run_in_threadpool() 把同步代码丢到线程池。
3. 简单逻辑（无需并发）
    - 如果接口只是返回一些计算结果（轻量 CPU）或内存查找，不涉及 I/O，写 async 没有意义，普通 def 更简单。

#### 🔑 实战建议

混用 async 和 sync 是完全可以的
FastAPI 可以自动调度：

```python
from fastapi import FastAPI
from fastapi.concurrency import run_in_threadpool
import requests

app = FastAPI()

# 异步 I/O
@app.get("/async")
async def async_api():
    return {"msg": "async good for I/O!"}

# 同步 CPU 任务
@app.get("/sync")
def sync_api():
    return {"msg": "sync is fine if CPU task is small"}

# 阻塞 I/O，用线程池包装
@app.get("/blocking")
async def blocking_api():
    data = await run_in_threadpool(lambda: requests.get("https://example.com").text)
    return {"data": data}
```

### FastAPI 如何与 Uvicorn / Gunicorn 配合部署？常见优化参数有哪些？

#### Uvicorn 的角色

- Uvicorn 是一个 ASGI 服务器，负责运行你的 FastAPI、Starlette、Django（ASGI 模式）等应用。
- 它的特点是：轻量、高性能、支持异步。
- 但它本身只是一个单进程单 worker 的实现（虽然有 --workers 参数，但并不完全等同于生产环境的多进程管理）。

#### Gunicorn 的角色

- Gunicorn 是一个 WSGI/ASGI 应用服务器管理器，主要负责：
- 进程管理：可以启动多个 worker 进程，支持预派生模型，能更好利用多核 CPU。
- 稳定性与容错：如果某个 worker 挂掉，Gunicorn 会自动拉起新的 worker。
- 信号处理：支持优雅重启、平滑更新，适合生产环境。
- Gunicorn 自身最初是 WSGI 服务器，但可以通过 worker 类支持 ASGI。

#### 为什么 Uvicorn + Gunicorn 一起用

- 直接用 Uvicorn 启动应用，虽然能跑，但缺乏进程管理和容错能力。
- 在生产环境中，一般会选择 Gunicorn 作为进程管理器，然后用 Uvicorn 作为 worker 来处理请求：`gunicorn -k uvicorn.workers.UvicornWorker myapp:app -w 4 -b 0.0.0.0:8000`
- -k uvicorn.workers.UvicornWorker 表示使用 Uvicorn 作为 worker。
- -w 4 启动 4 个 worker，能利用多核 CPU。
- Gunicorn 负责管理 worker 的生命周期，Uvicorn 专注于高性能处理请求。

### 如何处理数据库的 连接池 管理（例如 SQLAlchemy async engine）？

### 在高并发场景下，如何避免 阻塞 I/O 造成的性能瓶颈？

### 如何在 FastAPI 中实现 OAuth2 + JWT 认证？

### 如何在 FastAPI 中保护 API 接口免受 CSRF / XSS / SQL 注入？

#### 防止 CSRF（跨站请求伪造）

API 本身如果是 RESTful + JWT / OAuth2 模式，通常不容易受到 CSRF，因为：
CSRF 多依赖浏览器自动携带 Cookie，而 JWT 常通过 Authorization: Bearer <token> 头传递，不会被跨站请求自动附带。

#### 防止 XSS（跨站脚本攻击）

XSS 主要来自 用户输入未过滤后直接渲染到前端页面。FastAPI 本身是后端 API，不直接渲染 HTML，风险相对较低。

#### 防止 SQL 注入

SQL 注入风险主要来自拼接 SQL 语句时未做参数化。FastAPI 常用 SQLAlchemy / Tortoise ORM 等库，能自动处理参数化查询。

### FastAPI 的 Security 模块与 Depends 的区别是什么？

#### Depends

- 作用：最通用的依赖注入机制。你可以用它声明任何逻辑依赖，比如数据库会话、业务逻辑层、配置注入等。
- 特点：
    - 完全通用，不局限于认证/鉴权。
    - 如果依赖函数返回值，则会传递给路由处理函数。
    - 可以嵌套依赖（一个依赖中再使用 Depends）。

```python
from fastapi import Depends, FastAPI

app = FastAPI()

def get_db():
    db = "数据库连接"
    return db

@app.get("/items/")
def read_items(db=Depends(get_db)):
    return {"db": db}
```

#### Security

- 作用：Security 是 FastAPI 专门为 安全/认证相关依赖 提供的一个特殊依赖注入工具。
- 区别于 Depends 的关键点：它允许你在一个依赖中声明 安全作用域（scopes），并与 OAuth2/OpenAPI 安全规范 自动集成。
- 适用场景：当你需要和 OpenAPI 的 安全方案 (security schemes) 对齐时，比如 OAuth2 scopes、API key、JWT 认证等。
- 示例：

```python
from fastapi import FastAPI, Security
from fastapi.security import OAuth2PasswordBearer

app = FastAPI()
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token", scopes={"items": "Read items"})

def get_current_user(token: str = Security(oauth2_scheme, scopes=["items"])):
    return {"token": token}

@app.get("/items/")
def read_items(current_user=Depends(get_current_user)):
    return {"user": current_user}
```

### 如何为 FastAPI 接口添加 速率限制 (Rate Limiting)？

#### 简单实现（内存级，适合测试）

你可以在 FastAPI 的依赖函数里使用 dict 来记录请求次数和时间戳。

```python
from fastapi import FastAPI, Request, HTTPException, Depends
from time import time

app = FastAPI()

# 存储请求计数 {ip: [时间戳]}
request_counts = {}
RATE_LIMIT = 5  # 每个窗口最多请求次数
WINDOW = 60     # 时间窗口（秒）

def rate_limiter(request: Request):
    ip = request.client.host
    now = time()

    if ip not in request_counts:
        request_counts[ip] = []
    
    # 清理过期请求
    request_counts[ip] = [t for t in request_counts[ip] if t > now - WINDOW]

    if len(request_counts[ip]) >= RATE_LIMIT:
        raise HTTPException(status_code=429, detail="Too Many Requests")
    
    request_counts[ip].append(now)

@app.get("/items/")
def get_items(dep=Depends(rate_limiter)):
    return {"msg": "请求成功"}
```

#### 使用中间件实现

可以写一个 FastAPI Middleware 来统一处理所有请求。

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from time import time

app = FastAPI()

request_counts = {}
RATE_LIMIT = 5
WINDOW = 60

@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    ip = request.client.host
    now = time()

    if ip not in request_counts:
        request_counts[ip] = []

    request_counts[ip] = [t for t in request_counts[ip] if t > now - WINDOW]

    if len(request_counts[ip]) >= RATE_LIMIT:
        return JSONResponse(content={"detail": "Too Many Requests"}, status_code=429)
    
    request_counts[ip].append(now)
    response = await call_next(request)
    return response
```

### 在多租户系统中，如何保证 用户隔离与权限控制？

#### 租户

ORM自动根据当前登录用户获取租户ID，自动拼接到SQL中

#### 数据权限

pass

### 如何为 FastAPI 应用编写 单元测试 / 集成测试？常用工具有哪些（pytest, unittest）？

#### 常用测试工具

- pytest
  主流的 Python 测试框架，支持强大的 fixture 机制、插件系统。
- httpx
  一个支持异步的 HTTP 客户端，可用于模拟请求，和 FastAPI 的 TestClient 类似。
- starlette.testclient.TestClient
  FastAPI 基于 Starlette，因此提供了一个内置的测试客户端，封装在 fastapi.testclient.TestClient 中（实际上是基于 requests）。
- pytest-asyncio
  如果你在测试异步函数（async def），可以用它来支持异步测试。
- pytest-cov
  生成覆盖率报告，方便了解测试覆盖情况。

#### 单元测试 vs 集成测试

单元测试（Unit Test）

- 目标：测试单个函数、类或逻辑，避免依赖数据库、外部 API。
- 方法：通过 依赖注入 或 mock 替换掉外部依赖。
- 工具：pytest + unittest.mock 或 pytest-mock。

```
from fastapi.testclient import TestClient
from myapp.main import app

client = TestClient(app)

def test_read_main():
    response = client.get("/")
    assert response.status_code == 200
    assert response.json() == {"message": "Hello World"}
```

集成测试（Integration Test）

- 目标：测试路由、数据库、依赖项等组件的组合。
- 方法：使用测试数据库（SQLite 内存库 / docker 里的临时 DB）、启动完整 app。
- 工具：pytest + TestClient 或 httpx.AsyncClient。

异步测试示例（使用 httpx + pytest-asyncio）：

```
import pytest
from httpx import AsyncClient
from myapp.main import app

@pytest.mark.asyncio
async def test_get_items():
    async with AsyncClient(app=app, base_url="http://test") as ac:
        response = await ac.get("/items/")
    assert response.status_code == 200
    assert isinstance(response.json(), list)
```

### 如何模拟依赖注入对象以进行测试？

#### 简单 Function 依赖：用 dependency_overrides

示例：有个依赖 get_current_user，在测试时替换为返回测试用户。

```python
# app.py
from fastapi import FastAPI, Depends

app = FastAPI()

def get_current_user():
    # 真实实现（例如读取 token 并查 DB）
    return {"id": 1, "name": "alice"}

@app.get("/me")
def read_me(user = Depends(get_current_user)):
    return {"user": user}
```

测试中替换：

```python
# test_app.py
from fastapi.testclient import TestClient
from app import app

def test_read_me_override_dependency():
    def fake_user():
        return {"id": 42, "name": "test-user"}

    app.dependency_overrides.clear()
    app.dependency_overrides[get_current_user] = fake_user

    client = TestClient(app)
    r = client.get("/me")
    assert r.status_code == 200
    assert r.json() == {"user": {"id": 42, "name": "test-user"}}

    app.dependency_overrides.clear()  # 清理
```

#### 使用 fixture 管理 overrides（推荐）

把 override 放进 pytest fixture，自动清理。

```python
import pytest
from fastapi.testclient import TestClient
from app import app, get_current_user

@pytest.fixture
def client_with_user_override():
    def fake_user():
        return {"id": 99, "name": "fixture-user"}

    app.dependency_overrides[get_current_user] = fake_user
    client = TestClient(app)
    yield client
    app.dependency_overrides.clear()

def test_me(client_with_user_override):
    r = client_with_user_override.get("/me")
    assert r.json()["user"]["id"] == 99
```

#### 类（对象）依赖或工厂依赖

常见场景：依赖返回一个类实例（例如 Service / DB）。直接 override 以返回“测试替身”对象。

```python
# service.py
class Greeter:
    def greet(self):
        return "hi"

def get_greeter():
    return Greeter()

# app.py
from fastapi import Depends, FastAPI
from service import get_greeter

app = FastAPI()

@app.get("/greet")
def greet(g = Depends(get_greeter)):
    return {"msg": g.greet()}
```

测试中用 stub 对象：

```python
class FakeGreeter:
    def greet(self):
        return "hello-test"

def test_greet_override():
    app.dependency_overrides[get_greeter] = lambda: FakeGreeter()
    client = TestClient(app)
    r = client.get("/greet")
    assert r.json() == {"msg": "hello-test"}
    app.dependency_overrides.clear()
```

#### 异步依赖（async def） & AsyncMock

如果依赖是异步函数，用 async def 的替代函数，或者用 AsyncMock（Python 3.8+）：

```python
# app.py
async def get_async_user():
    return {"id": 1}

@app.get("/async-me")
async def async_me(user = Depends(get_async_user)):
    return user
```

```python
import pytest
from unittest.mock import AsyncMock

@pytest.mark.asyncio
async def test_async_dep(monkeypatch):
    async def fake_async_user():
        return {"id": 7}
    app.dependency_overrides[get_async_user] = fake_async_user

    client = TestClient(app)
    r = client.get("/async-me")  # TestClient 会处理 ASGI
    assert r.json() == {"id": 7}
    app.dependency_overrides.clear()
```

### 如何利用 FastAPI 的 自动生成 OpenAPI 文档 进行回归测试？

利用 FastAPI 自动生成的 OpenAPI 文档，你可以将其作为回归测试的规范来源，结合 契约测试（contract testing） 和 差异检测（schema diff） 工具，确保 API 在版本迭代中保持一致性和兼容性。

### 如何实现 接口版本管理？（v1, v2 API 兼容问题）

#### 兼容性处理策略

1. 向后兼容优先
    - 新版本添加字段时，保证旧字段继续存在（只新增，不删除）。
    - 如果必须删除字段，应提前标记为 deprecated，并给出迁移文档。
2. API 网关做适配层
    - 可以在 API 网关层面做路由、参数映射，把 v1 请求转发到 v2 实现，进行兼容转换。
    - 避免在业务服务中处理太多兼容逻辑。
3. 数据格式兼容
    - 返回 JSON 时，确保新字段不会影响旧客户端解析。
    - 推荐约定：客户端忽略未知字段。
4. 逐步迁移
    - 保留 v1 但只做安全性修复，不再加新功能。
    - 引导用户逐步迁移到 v2，最终下线 v1。

#### 最佳实践

- 明确版本生命周期：例如 v1 保持 1 年支持，之后废弃。
- 文档齐全：Swagger / OpenAPI 中标注不同版本的接口。
- 测试覆盖：对多版本接口分别进行测试，避免出现兼容性 bug。
- 沟通机制：对外发布版本更新计划，提前通知使用方。

#### 适用场景建议

- 中小型项目：推荐 URL 路径版本化（简单直观）。
- 企业级/对外 API：推荐 Header 版本化（更符合 RESTful）。
- 快速试验：可以先用 query 参数，后续再迁移到规范化方案。

### 如何保证大型团队协作下 FastAPI 项目的 可维护性和可扩展性？

1. 合理的架构，天然防止底层出现蜘蛛网式调用
2. 代码规范
3. 配置与环境管理
4. 测试和质量保障
5. 代码审查

### 设计一个电商系统的订单接口，要求：

- 提交订单时校验商品库存
- 支持异步下单（任务队列）
- 返回订单处理状态（pending/confirmed/failed）

1. 根据SKU_ID生成一个唯一的临时订单ID，例如：$SKU_ID_$UUD
2. 将临时订单ID存入Redis，设置超时时间，订单状态设置为PENDING
3. 将临时订单ID发送给MQ
4. 将临时订单ID直接返回给前端，前端接口轮询获取订单状态，进入等待状态
5. MQ消费者查询Redis，如果不存在则直接结束，如果存在则设置看门狗续期，获取分布式锁，然后进行生成订单流程
6. 扣减库存，生成订单，将Redis中订单状态设置为已生成，然后将真实的订单信息存入Redis，提交MQ事务
7. 前端获取到订单信息为已生成，则可以直接跳转到订单详情页面（可以调一下接口清理Redis释放内存）

### 如果 FastAPI 服务在生产环境中出现 请求延迟升高，你会如何排查？

1. 先看整体资源（CPU/内存/连接数）。
2. 再看外部依赖（数据库、缓存、第三方 API）。
3. 接着检查应用配置（workers 并发、阻塞代码）。
4. 最后用 APM/Profiling 定位代码级别的瓶颈。

### 描述一个 WebSocket 实时通知系统 的设计思路（如聊天、消息推送）。

### 假设你要把 FastAPI 部署到 Kubernetes，如何配置 健康检查 (liveness/readiness)？

在 Kubernetes 中部署 FastAPI 应用时，通常需要配置 livenessProbe 和 readinessProbe 来保证 Pod 健康和流量调度正常。FastAPI 自身没有内置的健康检查端点，但你可以轻松加上。下面分两步说明：

1. 在 FastAPI 中添加健康检查端点
   最简单的方式就是定义两个路由：

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/healthz")
def healthz():
    # 活性检查：只要进程活着就返回 200
    return {"status": "ok"}

@app.get("/readyz")
def readyz():
    # 就绪检查：比如检查数据库、缓存等依赖
    # 这里演示假设数据库连接正常
    db_ok = True  
    if db_ok:
        return {"status": "ready"}
    return {"status": "not ready"} 
```

实际生产环境中，/readyz 可以做数据库 ping、Redis 检查等。

2. 在 Kubernetes Deployment 中配置探针
   在 Deployment 或 Pod 里配置容器的探针，例如：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: fastapi
  template:
    metadata:
      labels:
        app: fastapi
    spec:
      containers:
        - name: fastapi
          image: your-registry/fastapi:latest
          ports:
            - containerPort: 8000
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8000
            initialDelaySeconds: 5   # 容器启动后等待 5 秒再检查
            periodSeconds: 10        # 每 10 秒检查一次
            timeoutSeconds: 2
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3 
```

3. 实战注意事项

- /healthz：只要 FastAPI 进程正常运行，就返回 200。用于 livenessProbe。
- /readyz：要检查依赖是否可用，例如数据库连接、缓存服务。用于 readinessProbe，只有准备好时，Kubernetes Service 才会将流量转发到该 Pod。
- 建议加上 initialDelaySeconds，避免应用启动慢时被 Kubernetes 判定为失败。
- 如果应用是异步的，也可以用 async def 定义健康检查路由。

## SQLAlchemy
