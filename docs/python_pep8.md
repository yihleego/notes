# PEP8

## 基础理解

### 1. 什么是 PEP8？为什么它对 Python 开发很重要？

PEP 8（Python Enhancement Proposal 8）是 Python 官方的代码风格指南，它定义了编写 Python 代码时应遵循的格式和规范。它的目标是让 Python 代码具有更好的可读性、一致性和可维护性。

#### PEP 8 的主要内容

PEP 8 涉及 Python 代码风格的多个方面，包括：

- 缩进（Indentation）
    - 使用 4 个空格 作为缩进。
    - 不推荐使用 Tab（除非维护旧代码）。
- 行宽（Line Length）
    - 每行最多 79 个字符（文档字符串和注释可稍长）。
- 空行（Blank Lines）
    - 函数和类之间使用 两个空行 分隔。
    - 类中方法之间使用 一个空行。
- 导入（Imports）
    - 每个导入语句独占一行。
    - 导入顺序建议为：
        - 标准库
        - 第三方库
        - 自定义模块
        - 并在每组之间留一空行。
- 命名规范（Naming Conventions）
    - 函数名、变量名：lower_case_with_underscores
    - 类名：CamelCase
    - 常量名：UPPER_CASE_WITH_UNDERSCORES
- 空格使用（Whitespace）
    - 避免多余的空格，例如：a = b + c ✅ 而不是 a=b+c ❌。
    - 函数定义：def func(a, b): ✅ 而不是 def func( a , b ): ❌。

#### 为什么 PEP 8 很重要

- 提高可读性（Readability）
  Python 的核心哲学之一是“代码可读性至上”。遵守 PEP 8 让你的代码更容易被他人理解。
- 团队协作（Collaboration）
  当多个开发者共同维护一个项目时，统一的风格能减少沟通成本和代码审查时间。
- 工具支持（Tooling Support）
  有很多工具（如 flake8, black, pylint）基于 PEP 8 自动检查或格式化代码，保持风格一致。
- 专业性与规范化（Professionalism）
  遵循 PEP 8 被视为专业 Python 开发的基本标准之一。

### 2. PEP8 与 PEP20（The Zen of Python）有什么区别？

这是一个非常好的问题——PEP 8 和 PEP 20 都是 Python 社区的重要文档，但它们的作用完全不同。下面我来系统地对比说明两者的区别 👇
PEP 20：The Zen of Python（Python 之禅）

核心目标：传达 Python 的哲学与设计思想。

主要内容：
共有 19 条格言（输入 import this 可查看），如：

Beautiful is better than ugly.（优美胜于丑陋）

Simple is better than complex.（简单胜于复杂）

Explicit is better than implicit.（明确优于隐晦）

Readability counts.（可读性很重要）

There should be one—and preferably only one—obvious way to do it.（应该有一种——最好只有一种——显而易见的做法）

💡 一句话总结：

“PEP 20 告诉你 为什么要那样写 Python。”

### 3. PEP8 是否是强制性的？团队通常如何执行 PEP8？

PEP 8 并不是“强制性”的，但在 Python 社区和团队开发中，它几乎是事实上的代码风格标准。以下是详细说明：
PEP 8 是 Python 官方发布的代码风格指南，旨在让 Python 代码在不同项目中保持一致性与可读性。它涵盖命名约定、缩进、空格、导入顺序、行宽、注释规范等。

Python 解释器不会强制执行 PEP 8。即使代码不符合规范，只要语法正确，也能正常运行。
但在团队或开源项目中，PEP 8 通常被视为强制规范。许多公司或项目在代码审查（Code Review）阶段要求遵守 PEP 8。

团队通常通过以下方式确保 PEP 8 合规：

- 代码检查工具（Linter）
    - 常见工具：flake8, pylint, pycodestyle
    - 在提交前自动检查代码风格并给出警告或错误。
- 自动格式化工具（Formatter）
    - 常见工具：black, yapf, autopep8
    - 自动将代码格式化为符合 PEP 8 的风格，减少人为分歧。
    - black 最流行，被许多团队视为“不可配置”的统一标准。
- CI/CD 集成
    - 在 GitHub Actions、GitLab CI 等 CI 流程中加入风格检查步骤。
    - 若代码未通过 PEP 8 检查，CI 构建会失败，阻止合并。
- Pre-commit Hook
    - 使用 pre-commit 在提交前运行 black 和 flake8。
    - 开发者在本地提交代码时，自动执行格式化和检查。

## 代码格式相关

### 4. PEP8 推荐的代码行最大长度是多少？为什么？

- 提高可读性
    - 较短的行便于在各种设备和编辑器中查看，不需要频繁水平滚动。
    - 当多文件并排显示（例如在代码审查或差异比较时），短行更易阅读。
- 便于终端和打印输出
    - 历史上，终端和打印机的宽度通常是 80 列。保留 1 列用于边框或行号，故推荐 79 字符。
    - 虽然现代显示器更宽，但保持这一标准有助于兼容性与统一性。
- 代码风格一致
    - 遵循统一长度上限使代码库风格更整齐，利于协作和审查。

### 5. PEP8 对缩进（indentation）有什么要求？Tab 和空格如何选择？

强烈推荐使用空格（spaces）而不是制表符（tabs）。
PEP 8 规定：

“Spaces are the preferred indentation method.”

不要混用 Tab 和空格。
在同一个文件中混用会导致：

不同编辑器显示不一致；

Python 3 中直接抛出 TabError。

如果你必须维护旧代码（使用 Tab 缩进），保持一致即可。
PEP 8 建议：

“When invoking the Python 3 command line interpreter with the -t option, it issues warnings about inconsistent tab usage.”

### 6. 在 PEP8 中，如何书写合适的空行来分隔类、函数、逻辑段落？

顶层函数、类定义 之间应 使用两个空行 分隔。
类中的方法定义 之间使用 一个空行 分隔。
在函数体或方法体中，为了分隔逻辑上不同的代码块，可以 使用单个空行 来增强可读性。

### 7. PEP8 对导入语句的顺序有什么建议？

PEP 8 建议将导入语句分为 三个分组，并且每个分组之间要空一行：

1. 标准库导入（Standard library imports）
2. 第三方库导入（Related third-party imports）
3. 本地应用或库的特定导入（Local application/library specific imports）

### 8. 如何在一行代码过长时进行换行？举例说明。

当一行代码太长时，可以在逻辑上尚未结束的位置加上 \，让下一行继续：

```python
result = first_value + second_value + third_value + \
         fourth_value + fifth_value
```

Python 在 ()、[]、{} 内可以自动识别换行，不需要加 \：

```python
numbers = [
    1, 2, 3, 4, 5,
    6, 7, 8, 9, 10
]

total = (
    first_value
    + second_value
    + third_value
)
```

如果字符串太长，可以用三引号 """ 或括号自动拼接：

```python
long_text = (
    "This is a very long string that will be "
    "automatically concatenated by Python "
    "without any special symbols."
)
```

函数参数太长时换行

```python
def send_email(
    recipient,
    subject,
    message,
    cc=None,
    attachments=None
):
    pass
```

## 命名规范

### 9. 变量、函数、类、常量的命名规范分别是什么？

- 变量命名：snake_case
- 函数命名：snake_case
- 类命名：UpperCamelCase
- 常量命名：UPPER_CASE_WITH_UNDERSCORES
- 其他补充：
    - 模块名：全小写，可包含下划线
    - 包名：全小写，尽量无下划线
    - 私有属性/函数：以下划线开头
    - 强制私有：双下划线开头

### 10. 模块名和包名的命名规范是怎样的？

#### 一、模块命名（module names）

模块是一个 .py 文件。
✅ 规范要求：

- 模块名应全部使用小写字母。
- 可以使用下划线 _ 分隔单词（但尽量避免，除非能提高可读性）。
- 模块名应尽量简短且具有描述性。
- 避免与 Python 标准库模块同名（例如 email.py、json.py、re.py 等），否则会引发导入冲突。

```
# 推荐
math_utils.py
stringtools.py
parser.py

# 不推荐
MathUtils.py     # 使用了驼峰命名
String-Tools.py  # 包含非法字符
parser123.py     # 不具描述性
```

#### 二、包命名（package names）

包是一个包含 __init__.py 文件的目录。
✅ 规范要求：

- 包名也应全部使用小写字母。
- 不建议使用下划线 _，除非有助于可读性。
- 包名应简短、清晰地表达功能。

```
# 推荐
mypackage/
datautils/
networking/

# 不推荐
MyPackage/      # 驼峰命名
data_utils/     # 一般不建议使用下划线（除非确实能提升可读性）
```

### 11. Python 内置方法通常以双下划线开头和结尾（如 __init__），为什么？

在 Python 中，这类以双下划线开头和结尾的方法（例如 __init__、__str__、__len__ 等）被称为 “特殊方法” 或 “魔术方法（magic methods）”。它们存在的原因和命名方式有着明确的设计目的：

#### 设计目的：区分“语言机制”与“普通方法”

Python 的设计哲学之一是“避免命名冲突”。
双下划线（也叫 dunder，来自 double underscore）是一种 命名约定，表示这个名字是由解释器内部使用或保留的，不是给普通用户随意调用或覆盖的。

```
class MyClass:
    def __init__(self, value):
        self.value = value
```

这里的 __init__ 并不是随便起的名字，而是 Python 解释器在创建对象时 自动调用 的方法名。如果它只是 init，那么用户自己的普通方法就可能和它冲突。

#### “魔术方法”实现语言特性

这些方法定义了对象在特定语义下的行为，比如：

方法名 触发场景 例子
__init__    创建对象时调用 obj = MyClass()
__str__    调用 str(obj) 或 print(obj)    print(obj)
__len__    调用 len(obj)    len(mylist)
__getitem__    用 obj[key] 取值 mylist[0]
__add__    使用 + 运算符 a + b

这些方法让开发者可以自定义类的行为，使用户自定义对象能像内置类型那样自然地参与表达式运算。

#### 为什么不用单下划线？

- 单下划线（_var）在 Python 中表示“内部使用”或“不建议外部访问”，但并没有任何语言机制强制限制。
- 双下划线前缀（如 __var）触发 名称改写（name mangling），防止子类覆盖。
- 双下划线前后缀（如 __init__）是 系统保留格式，不参与名称改写，由解释器识别。

#### 不要自创“dunder”名字

PEP 8（Python 官方风格指南）明确指出：
“Never invent names that start and end with double underscores.”
也就是说，只有 Python 解释器定义的双下划线名字才应该存在，用户不应创造新的此类名字，以免与未来的语言特性冲突。

## 空格和对齐

### 12. 在 PEP8 中，在哪些地方应该避免多余的空格？

### 13. 以下两种写法哪一个符合 PEP8？为什么？

```python
spam(ham[1], {eggs: 2})
spam( ham[ 1 ], { eggs : 2 } )
```

第一种写法符合 PEP8，因为它遵守了括号、方括号、花括号内不留多余空格，以及字典冒号左右空格的规范。

### 14. 如何在函数参数过多时进行换行？请写一个符合 PEP8 的例子。

## 文档与注释

### 15. PEP8 对 docstring 有哪些推荐？

### 16. 单行注释和块注释应该如何使用？

### 17. TODO 注释通常应该如何书写？

## 最佳实践 & 工具

### 18. 如何在项目中自动检测是否符合 PEP8？

### 19. 请列举几个常见的 Python 代码格式化或检查工具。

### 20. 在实际开发中，如果某些情况需要违反 PEP8，该如何处理？

📖 加分题

### 21. 解释一下 E501、E302、E231 分别表示什么错误？

E501 – Line too long
E302 – Expected 2 blank lines, found X
E231 – Missing whitespace after ‘,’, ‘;’, or ‘:’

### 22. 你在团队项目中遇到过哪些常见的 PEP8 不符合情况？你是怎么解决的？

### 23. 为什么 PEP8 建议尽量避免使用单字符变量名？



