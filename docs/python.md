# Python

## Python 的内存管理机制

### 1. 引用计数机制

- 在 Python 中，每个对象都有一个引用计数器，记录有多少个变量或对象引用它。
- 当引用计数为 0 时，对象的内存会立即被回收。
- 常见影响引用计数的操作：
    - 增加计数：变量赋值、作为函数参数传递、存入容器（如 list, dict）。
    - 减少计数：变量被删除（del）、变量赋新值、对象被移出容器。

优点：简单、实时回收（不会长时间占用内存）。
缺点：无法处理循环引用（例如两个对象相互引用）。

### 2. 垃圾回收（GC）

为了解决循环引用问题，Python（CPython 实现）引入了 分代垃圾回收（Generational GC）：

- Python 把对象分成 三代：
    - 第 0 代：新创建的对象。
    - 第 1 代：多次 GC 后仍然存活的对象。
    - 第 2 代：长期存活的对象。
- 回收策略：
    - 第 0 代对象频繁检查。
    - 如果对象多次幸存，则晋升到更高一代，检查频率降低。

这样可以减少 GC 开销，因为大多数对象生命周期很短。

---

是否存在循环引用使用深度优先搜索（DFS）+ 回溯标记

- 用 DFS 遍历所有节点。
- 给每个节点维护一个三态标记：
    - 未访问（white）
    - 访问中（gray）
    - 已完成（black）
    - 在 DFS 时，如果遇到“访问中（gray）”的节点，说明存在一条回到自己祖先的路径，即检测到环。

复杂度：

- 时间复杂度：O(V+E)，V为对象数，E为引用关系数。
- 空间复杂度：O(V)（递归栈+标记数组）。

```python
def dfs_recursive(graph, node, visited=None):
    if visited is None:
        visited = set()
    if node not in visited:
        print(node, end=" ")
        visited.add(node)
        for neighbor in graph[node]:
            dfs_recursive(graph, neighbor, visited)

graph = {
    'A': ['B', 'C'],
    'B': ['D', 'E'],
    'C': ['F'],
    'D': [],
    'E': ['F'],
    'F': []
}
            
dfs_recursive(graph, 'A')
# A B D E F C 
```

BFS

```
from collections import deque

def bfs(graph, start):
    visited = set()         # 已访问的节点集合
    queue = deque([start])  # 使用队列存储待访问节点
    order = []              # 记录访问顺序

    while queue:
        node = queue.popleft()
        if node not in visited:
            visited.add(node)
            order.append(node)
            # 将未访问的相邻节点加入队列
            for neighbor in graph.get(node, []):
                if neighbor not in visited:
                    queue.append(neighbor)
    return order
```

### 3. 内存池机制（Memory Pool）

- CPython 使用了 内存池机制（pymalloc） 来提高小对象的分配效率。
- 内存分配过程：
    - 大对象（>512字节）直接向操作系统申请内存。
    - 小对象（≤512字节）由 Python 内部的内存池统一管理，减少频繁调用 malloc/free 的开销。
- 内存池被分为：
    - Arena：256 KB 的大块内存。
    - Pool：从 Arena 划分的小块。
    - Block：分配给对象的最小单位。

### 4. 内存管理相关模块

- gc 模块：可以手动调用垃圾回收、调整阈值、调试内存泄漏。
- sys.getrefcount(obj)：查看对象的引用计数。
- obj.__sizeof__()：查看对象的实际大小。

### 5. 优化内存使用的技巧

- 避免不必要的对象引用，及时使用 del 删除无用变量。
- 使用生成器（generator yield）代替列表保存大量数据。
- 大量数值计算时可用 numpy，因为它使用了更高效的内存管理。
- 对于循环引用的对象（如双向链表、树节点），可以使用 weakref 弱引用避免 GC 压力。

## Python 语言核心与原理

### is 与 == 的区别，id() 的作用

#### ==
- 比较的是 值是否相等。
- 即两个对象内容相同，就返回 True，不管它们是不是同一个对象。
#### is
- 比较的是 对象的身份是否相同。
- 即比较两者的 内存地址（id） 是否相同，只有指向同一个对象时才返回 True。
#### id() 的作用
- id(obj) 返回对象的 内存地址标识（唯一标识符）。
- 在 CPython 实现中，id() 的返回值就是对象的内存地址（十进制形式）。
- 可以用来判断两个变量是否引用同一个对象。

### Python 中 不可变对象和可变对象 的区别，以及对性能和多线程的影响。

pass

### Python 中的 作用域规则（LEGB 原则）。

LEGB 代表 Local → Enclosing → Global → Built-in，Python 会依次在这四个作用域中查找变量：

1. L（Local，本地作用域）
    - 指函数内部定义的变量（包括函数参数）。
    - 在函数运行时创建，函数结束后销毁。
2. E（Enclosing，闭包外层作用域）
    - 如果一个函数嵌套在另一个函数中，内层函数引用外层函数的变量，就属于 Enclosing。
    - Python 会向外逐层查找，直到找到该变量或到达 Global 为止。
3. G（Global，全局作用域）
    - 在模块文件的最外层定义的变量。
    - 可以在函数内部通过 global 关键字声明，修改全局变量。
4. B（Built-in，内置作用域）
    - Python 解释器内置的作用域，比如 len、print、sum 等。
    - 如果以上三个作用域都没有找到变量名，就会到内置作用域查找。

### 描述 装饰器（decorator） 的工作原理，并实现一个带参数的装饰器。

在 Python 中，装饰器本质上就是一个函数，它的作用是接收一个函数作为输入，并返回一个新的函数，从而在不修改原函数代码的情况下，给函数“添加”额外的功能。
它常用于：

- 日志记录
- 权限校验
- 性能测试（统计函数执行时间）
- 缓存
- 代码复用与解耦

装饰器的工作原理主要依赖于 函数是“一等公民” 这一特性：

1. 函数可以作为参数传递给另一个函数
2. 函数可以作为返回值返回
3. 函数可以赋值给变量

```python
def decorator(func):
    def wrapper(*args, **kwargs):
        print("在执行函数之前做一些事")
        result = func(*args, **kwargs)  # 调用原函数
        print("在执行函数之后做一些事")
        return result
    return wrapper

@decorator
def say_hello(name):
    print(f"Hello, {name}!")

say_hello("Alice")
```

使用 functools.wraps

如果不做处理，装饰器会导致原函数的元信息（如 __name__, __doc__）丢失。

```python
from functools import wraps

def decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper
```

### 一切皆对象：对象头、类型对象、类与元类

在 Python 中，不管是 整数、字符串、函数、类、模块，甚至连 类型本身，都是对象（object）。
对象的本质就是“内存里的一段数据 + 头部元信息”。所以你拿到的任何值，在 CPython 解释器里都是一个 PyObject * 指针。

### Python 的 元类（metaclass） 是什么，有哪些应用场景？

在 Python 中，一切都是对象，包括类本身。类是用来创建对象的“工厂”，而 元类就是用来创建类的工厂。
换句话说：

- 对象（instance） 是由 类（class） 创建的。
- 类（class） 又是由 元类（metaclass） 创建的。

默认情况下，Python 中的类是由内置的元类 type 生成的。

常见应用场景

- 单例模式：确保一个类只有一个实例。
- ORM 框架（比如 Django 的模型类）：在类定义时自动收集字段信息。
- 接口检查：确保子类实现了某些方法。
-

```python
# 定义一个元类
class MyMeta(type):
    def __new__(mcs, name, bases, attrs):
        print(f"正在创建类 {name}")
        attrs['say_hello'] = lambda self: print("Hello from metaclass!")
        return super().__new__(mcs, name, bases, attrs)

# 使用元类创建类
class MyClass(metaclass=MyMeta):
    pass

obj = MyClass()
obj.say_hello()  # 输出: Hello from metaclass! 
```

__new__ 方法会在类创建时被调用。
可以在这里动态修改类的属性和方法。

### Python 中的 描述符（descriptor）协议：__get__、__set__、__delete__。

在 Python 中，描述符（descriptor）协议是一种底层机制，它定义了对象如何控制对其属性的访问（读取、写入、删除）。这套机制让我们可以用类来定制属性行为，本质上是对 __get__、__set__、__delete__ 这三个方法的约定。

```python
class Age:
    def __get__(self, instance, owner):
        return instance.__dict__.get("_age", 0)

    def __set__(self, instance, value):
        if value < 0:
            raise ValueError("年龄不能为负数")
        instance.__dict__["_age"] = value


class Person:
    age = Age()  # 使用描述符

p = Person()
p.age = 25
print(p.age)   # 25
p.age = -5     # ValueError: 年龄不能为负数
```

实际应用场景

1. 属性验证（如上面 Age 示例）
2. 懒加载属性（在访问时才计算或加载）
3. ORM 框架：如 SQLAlchemy，用描述符管理数据库字段与类属性映射。
4. 函数与方法机制：Python 内置的 @property、方法绑定机制就是基于描述符协议实现的。

### __new__ 和 __init__ 的区别与使用场景。

#### __new__

- 作用：负责创建对象，是构造函数。
- 调用时机：在类实例化时，先调用 __new__ 来分配内存并返回实例对象。
- 返回值：必须返回类的实例（通常是 super().__new__(cls)），否则对象不会被创建。
- 常见用途：
    - 控制对象的创建过程（比如实现单例模式）。
    - 当需要返回不同类的实例时（如工厂模式）。
    - 在不可变对象（如 int, str, tuple）的子类中定制行为时。

#### __init__

- 作用：负责初始化对象，是初始化方法。
- 调用时机：__new__ 返回实例对象后，自动调用 __init__ 进行属性初始化。
- 返回值：始终返回 None，不能改变实例的返回值。
- 常见用途：
    - 为对象属性赋值。
    - 执行一些启动逻辑。

实例化 obj = MyClass() 的完整顺序是：

1. MyClass.__new__ 被调用 → 创建对象并返回实例。
2. MyClass.__init__ 被调用 → 对实例做进一步初始化。

如果 __new__ 没有返回实例（比如返回了 None 或其他类型对象），则 __init__ 不会被执行。

总结:

- __new__：负责对象创建，控制返回什么对象。
- __init__：负责对象初始化，填充对象内容。
- 一般情况：只写 __init__ 就足够。
- 特殊情况：需要改变实例化过程时（单例、工厂、不变对象）才会用到 __new__。

### Python 中的 数据模型魔法方法（如 __str__, __repr__, __iter__, __call__）。

#### __str__(self)

- 定义对象被 str() 或 print() 调用时的字符串表现形式。
- 主要用于“对用户友好”的可读性输出。

```python
class Person:
    def __init__(self, name):
        self.name = name
    
    def __str__(self):
        return f"Person(name={self.name})"

p = Person("Alice")
print(p)          # Person(name=Alice)   <-- 调用 __str__
print(str(p))     # Person(name=Alice) 
```

#### __repr__(self)

- 定义对象在 交互式解释器 或 repr() 函数中返回的字符串表现形式。
- 通常要求返回一个 能明确标识对象 的字符串（最好能用于重新创建该对象）。
- 开发者调试时更常用。

```python
class Person:
    def __init__(self, name):
        self.name = name
    
    def __repr__(self):
        return f"Person('{self.name}')"

p = Person("Alice")
print(repr(p))    # Person('Alice') 
```

#### __iter__(self)

- 使对象 可迭代（可用于 for ... in ...、解包、list() 等）。
- 必须返回一个迭代器对象（实现了 __next__ 的对象），通常是 self 或 iter(...)。

```python
class CountDown:
    def __init__(self, start):
        self.start = start
    
    def __iter__(self):
        self.current = self.start
        return self
    
    def __next__(self):
        if self.current <= 0:
            raise StopIteration
        self.current -= 1
        return self.current + 1

for num in CountDown(3):
    print(num)   # 3, 2, 1 
```

#### __call__(self, *args, **kwargs)

- 让对象 像函数一样可调用。
- 常用于回调、工厂模式或“函数对象”。

```python
class Greeter:
    def __init__(self, greeting):
        self.greeting = greeting
    
    def __call__(self, name):
        return f"{self.greeting}, {name}!"

hello = Greeter("Hello")
print(hello("Alice"))  # Hello, Alice!   <-- 调用 __call__ 
```

### GIL（全局解释器锁）：为什么存在，如何影响多线程？

#### GIL 为什么存在

GIL 最初设计的主要原因是 简化 CPython 的内存管理：

- CPython 使用了 引用计数（reference counting） 作为垃圾回收的一部分，每次对象引用数的增加/减少都需要保证线程安全。
- 如果没有全局锁，解释器就必须在所有对象操作中加上细粒度的锁，这会大大增加实现的复杂性和运行开销。
- 因此，GIL 的出现使得 单个线程在任意时刻独占解释器执行字节码，从而避免了并发修改对象引用计数时的数据竞争问题。

简而言之：
👉 GIL 是一个历史遗留的 实现权衡，它让 Python 在多线程情况下更简单、更安全，但牺牲了多核并行的性能潜力。

#### GIL 如何影响多线程

1. CPU 密集型任务
    - 因为 GIL 的存在，同一进程内的多个线程不能同时利用多个 CPU 核心执行 Python 字节码。
    - 即便开了很多线程，实质上在同一时刻也只有一个线程在执行，其他线程被阻塞。
    - 所以在需要大量计算的任务中，多线程并不会加速，甚至可能因为频繁上下文切换而变慢。
2. I/O 密集型任务
    - 当线程执行 I/O 操作（如文件读写、网络请求）时，GIL 会被释放，这样其他线程可以运行。
    - 因此，多线程在 I/O 密集型场景（如爬虫、网络服务）仍然能带来性能提升。
3. 多进程替代方案
    - 对于 CPU 密集型任务，常用的办法是使用 多进程（multiprocessing），每个进程有独立的 Python 解释器和 GIL，可以真正利用多核。
    - 当然，这会带来更高的内存开销和进程间通信成本。

### __slots__ 的作用与限制

#### 作用

1. 节省内存
    - 默认情况下，Python 对象的实例属性存储在一个 __dict__ 中，这个字典会占用额外内存。
    - 使用 __slots__ 声明后，实例不会再使用 __dict__ 存储属性，而是通过更紧凑的结构（类似数组）来存放指定的属性，从而减少内存开销。
    - 对于需要大量创建实例的场景（如数百万对象），效果尤其明显。
2. 提高属性访问速度
    - 因为跳过了字典查找，__slots__ 中的属性访问通常会更快一些。
3. 防止动态添加新属性
    - 使用 __slots__ 限定后，实例只能有定义好的属性，无法随意添加新的属性，减少了错误的可能性。

#### 限制

1. 不能随意添加新属性
    - 除非在 __slots__ 中定义了 "__dict__"，否则无法再为实例动态添加未声明的属性。
2. 不支持默认继承 __dict__ 和 __weakref__
    - 如果类需要支持弱引用，需要在 __slots__ 中显式声明 __weakref__。
    - 否则，实例不能作为 weakref 的目标。
3. 继承的复杂性
    - 子类没有定义 __slots__ 时会重新获得 __dict__，从而失去内存优化效果。
    - 多重继承时，如果父类们都定义了 __slots__，可能会出现兼容性问题。
4. 某些动态特性受限
    - 无法给类随意绑定属性。
    - 不便于某些依赖动态属性操作的库（如某些 ORM）使用。

```python
class Point:
    __slots__ = ("x", "y")

    def __init__(self, x, y):
        self.x = x
        self.y = y

p = Point(1, 2)
print(p.x, p.y)

# p.z = 3   # AttributeError: 'Point' object has no attribute 'z'
```

总结：__slots__ 适合 需要大量实例、注重内存和性能优化 的场景（如数据结构、游戏对象、科学计算中轻量对象），但会牺牲一些灵活性（动态属性和继承特性）。

### 闭包与自由变量

#### 什么是闭包（Closure）

包指的是：一个函数内部定义了另一个函数，并且内部函数引用了外部函数的变量，那么这个内部函数就被称为闭包。

- 闭包的本质是 函数 + 环境 的组合。
- 当外部函数返回了内部函数时，内部函数依然“记住”了外部函数作用域中的变量，即使外部函数已经结束执行。

```python
def outer(x):
    def inner(y):
        return x + y
    return inner

f = outer(10)  # outer 执行后返回 inner
print(f(5))    # 输出 15
```

#### 什么是自由变量（Free Variable）

自由变量是指在函数中使用，但既不是该函数的参数，也不是函数内部定义的局部变量，而是来自外部作用域的变量。

在闭包中，自由变量通常来自外部函数的作用域。

```python
def outer():
    x = 10  # x 就是自由变量
    def inner(y):
        return x + y
    return inner

f = outer()
print(f(5))  # 输出 15
```

#### 闭包与自由变量的关系

- 闭包产生的前提：内部函数引用了外部函数的自由变量。
- 自由变量的存储：Python 会将这些自由变量保存在函数的 __closure__ 属性中。

```python
def outer(a):
    b = 20
    def inner(c):
        return a + b + c
    return inner

f = outer(10)
print(f(5))                  # 输出 35
print(f.__closure__[0].cell_contents)  # 10
print(f.__closure__[1].cell_contents)  # 20 
```

总结：

- 闭包 = 内部函数 + 外部作用域的自由变量。
- 自由变量 = 非局部变量、非参数，但在函数中被使用。
- 闭包常用于数据隐藏、延迟计算、函数工厂、装饰器等场景。

### 异常传播机制与上下文（__context__、__cause__）

#### 异常传播机制

1. 异常抛出
   当 raise 抛出一个异常时，解释器会中断当前代码执行，沿调用栈逐层回溯，直到找到合适的 except 块来处理。
    - 如果找到处理代码 → 执行对应的 except
    - 如果未找到 → 程序终止，并输出 traceback
2. 嵌套异常
   在 except 块中可能再次抛出新的异常，这时候原始异常信息会被“链式”保留下来，用来提示开发者“这个异常是由前一个异常导致的”。

#### 异常上下文属性

Python 为了保留这种“链式”关系，引入了几个特殊属性：

1. __context__
    - 当一个异常在 except 块中处理另一个异常时，如果又抛出了新的异常，新异常对象的 __context__ 会指向前一个异常。
    - 这是“隐式链”，即只要是在异常处理过程中触发了新的异常，就会自动建立。

```python
try:
    1 / 0
except ZeroDivisionError:
    raise ValueError("出现了新的错误")
```

此时：

- 新异常是 ValueError
- ValueError.__context__ → ZeroDivisionError
- traceback 会显示类似：

```
During handling of the above exception, another exception occurred:
```

2. __cause__
    - 可以通过 raise ... from ... 语法手动指定“直接原因”。
    - 当显式使用 from 时，新的异常对象的 __cause__ 属性会被设置为你指定的异常，而不是默认的 __context__。

```
try:
    1 / 0
except ZeroDivisionError as e:
    raise ValueError("出现了新的错误") from e
```

- ValueError.__cause__ → ZeroDivisionError
- traceback 会显示类似：

```
The above exception was the direct cause of the following exception:
```

这样可以让错误链更清晰地表达“因果关系”。

3. __suppress_context__

- 当你用 raise ... from ... 时，Python 会自动将 __suppress_context__ = True，表示不再显示 __context__ 的 traceback，只显示 __cause__。
- 如果你想手动抑制 __context__ 的显示，也可以设置该属性。

### 自定义异常的最佳实践

1. 继承 Exception，避免直接继承 BaseException。
2. 定义项目级别基类异常，分层管理。
3. 在异常中存储上下文信息，便于调试。
4. 写清楚文档字符串，避免模糊异常。
5. 能用内置异常时不要重复造轮子。
6. 结合 logging 输出错误信息。
7. 使用 raise ... from ... 保留上下文。

## 性能与优化

### 如何分析 Python 程序的性能瓶颈？（cProfile, line_profiler 等工具）
### 列举几种 Python 优化性能的手段（如 Cython、NumPy 向量化、缓存、并发）。
### deepcopy vs copy 的区别和性能影响。
### Python 中的内存泄漏可能由哪些情况引起？如何定位？
### 如何减少 Python 中函数调用的开销？
### list vs tuple 的性能与使用场景
### set、dict 的哈希实现与优化
### collections 模块（deque、Counter、defaultdict、OrderedDict）
### 内存优化
### 内存池机制（small object allocator）
### sys.intern、字符串驻留
### __slots__、生成器（generator）、迭代器的内存特性
### 性能调优
### 使用 timeit、cProfile、line_profiler 做性能分析
### C 扩展与 Cython 提升性能
### 内建函数与列表推导的性能差异

## 并发与异步

### Python 中 多进程 vs 多线程 vs 协程 的区别和适用场景。
### Python 的 multiprocessing 和 threading 模块差异。
### asyncio 的工作机制：事件循环、协程调度。
### 如何在 asyncio 中进行任务取消、超时处理？
### Python 中实现生产者消费者模型的几种方法。
### concurrent.futures 模块的使用。
### GIL 的影响与适用场景（IO 密集 vs CPU 密集）
### threading、concurrent.futures.ThreadPoolExecutor
### 多进程
### 进程间通信：Queue、Pipe、共享内存
### multiprocessing 与 joblib
### 协程与异步编程
### asyncio 原理：事件循环、任务调度
### await 与 async 的底层机制（协程对象、本质是生成器）
### Trio、Curio、gevent 与 asyncio 的对比

## 常用数据结构与算法

### Python 内置数据结构（list, dict, set, tuple）的底层实现原理。
### Python 字典在 3.6+ 后的 有序性是如何实现的？
### list 扩容机制与性能特点。
### 如何实现一个 LRU 缓存？（可用 functools.lru_cache 或自定义 OrderedDict）
### 实现一个线程安全的队列（Queue 原理）。
### 基础数据结构
### 栈、队列、堆、哈希表的 Python 实现
### Python 内置排序算法（Timsort）原理
### 高阶数据结构
### B 树、Trie、图的常见实现
### LRU/LFU 缓存（functools.lru_cache 实现）
### 算法与复杂度
### 常见算法复杂度分析（Big-O）
### 动态规划、回溯、贪心的 Python 实现
### 海量数据处理（分治、Bloom Filter、LRU Cache）

## 工程与架构

### Python 项目常见的目录结构与模块组织。
### Python 的 包管理：pip、Poetry、Conda 的区别。
### 如何在大型 Python 项目中做 依赖注入？
### 如何保证代码质量（lint、mypy、pytest、tox 等工具）。
### Python 项目的日志管理最佳实践。
### 单元测试与 Mock 的应用场景。
### CI/CD 中 Python 项目的常见自动化流程。
  测试与调试
### unittest、pytest、doctest
### mock 与 patch 技巧
### logging、traceback、pdb
### 代码质量与规范
### PEP8、typing（静态类型检查）
### mypy、pylint、flake8
### 框架与库
### Web 框架（Django、Flask、FastAPI）
### 科学计算（NumPy、Pandas、PyTorch）
### ORM 与数据库（SQLAlchemy、peewee、Django ORM）
### 架构设计
### 设计模式在 Python 中的应用（单例、工厂、观察者）
### 依赖注入与模块解耦
### 高并发架构与微服务中的 Python 角色

## 系统设计与场景题

### 如果让你实现一个 高并发爬虫系统，会怎么设计？
### 如何实现一个支持 百万级并发的 WebSocket 服务？
### Python 在 微服务架构中的角色与优化。
### 如何设计一个 大规模数据处理 pipeline？
### Python 在 消息队列系统（Kafka/RabbitMQ） 中的应用。
### 设计一个高并发爬虫（反爬应对、数据存储、异步请求）
### 设计一个分布式日志采集系统（Kafka + Python 消费）
### 如何实现一个限流器（令牌桶/漏桶）在 Python 中
### 如何优化一个大规模 ETL 任务（Pandas vs PySpark）

## 额外加分题

### C 扩展、Cython、Numba 的使用场景。
### Python 如何与 C/C++ 交互（ctypes、cffi、cython、pybind11）。
### PEP 484 类型注解与 mypy 的关系。
### Python 3.10+ 的新特性（如模式匹配 match）。
### Python 3.14+ 的新特性。
### Python 解释器（CPython、PyPy、Jython）的区别。
### CPython 的内存管理机制（引用计数、垃圾回收、GC Root）

