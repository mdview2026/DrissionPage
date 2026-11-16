# DrissionPage · Fork 增强版

> 本仓库是 [g1879/DrissionPage](https://github.com/g1879/DrissionPage) 的 **fork**，在上游基础上持续增加新功能。
> 所有新增能力均为 **opt-in**：不改变上游任何既有行为，旧代码零修改即可运行。
>
> 上游完整文档与官方网站：[https://DrissionPage.cn](https://drissionpage.cn)

DrissionPage 是一个基于 Python 的网页自动化工具，既能控制浏览器，也能收发数据包，还能把两者合而为一，可兼顾浏览器自动化的便利性和 requests 的高效率。

---

## ✨ 本 fork 新增功能

### 现代定位层：`?` 查询 DSL + `Q` Builder + 混合下推引擎

针对原定位语法的几个痛点——**简写必须放在最前**（`tag=` 后置即失效）、**`text()` 与 `@@text()` 层级易混**、**多条件没有显式逻辑运算符**——新增了一套词法无歧义、顺序无关、可组合、可调试的现代定位层。

以 `?` 开头的字符串或 `Q` 对象即触发新引擎，与旧语法并存互不影响。完整设计规范见 [docs/final-syntax.md](docs/final-syntax.md)。

#### 1. 条件顺序无关，逻辑显式

```python
# 旧写法：tag 必须在最前、@@tag()= 必须带括号，否则静默返回 NoneElement
header.ele('tag:input@@type=filter')

# 新写法：条件顺序随意，&& / || / ! / () 显式组合
header.ele('?input && type=filter')
header.ele('?type=filter && input')          # 同样命中
tab.ele('?(div || span) && .btn && !id=ad')  # 或、非、括号嵌套
```

#### 2. 文本层级与直觉对齐

```python
tab.ele('?text*=ABC')         # 深层文本（相当于 ele.text，含子孙元素）
tab.ele('?owntext=ABC')       # 仅直属文本节点
tab.ele('?a && text*=ABC')    # <a> 且任意层级文本包含 ABC
```

#### 3. 正则匹配 `/=` 与混合下推引擎

```python
tab.ele('?a && href/="/item/\\d+\\.html$"')          # 正则匹配
tab.ele(Q('a', href__regex=r'/item/\d+\.html$'))     # 等价的 Q 写法
```

纯 XPath 可表达的查询直接编译为**原生 XPath，零额外开销**；含正则等无法下推的节点时，自动切换为 **hybrid 模式**：XPath 安全超集预筛候选 + 本地精筛，绝不漏杀。

#### 4. `Q` Builder：动态条件免转义

变量中含引号、`&&`、括号也绝对安全，专为动态组装设计：

```python
from DrissionPage import Q

q = Q('a', text__contains=user_input, href__startswith='https://') & ~Q.id('ad')
q = (Q.tag('div') | Q.tag('span')) & Q.cls('btn')
q = Q.attr('data-testid').exists()      # 任意特殊属性名
q = Q.text.regex(r'^\s*登录')            # 流式调用
tab.ele(q)
```

#### 5. 调试友好，所见即所得

非法语法必报 `LocatorError`，带位置指针与修改建议（Did-You-Mean）：

```text
查询语法错误（位置 2）
  ?a!=b
    ^^
原因: 不支持 != 运算符
建议: 否定请统一使用 ! 前缀，如 !a=b
```

- 未命中时 `NoneElement` 自动回显解析结果：`<NoneElement ..., parsed=tag="buttton">` —— 打错标签名一眼即破
- `explain_loc('?...')` / `q.explain()` 打印原始输入、标准式、AST、执行策略与实际下推 XPath

#### 速查卡

```text
── 字符串 DSL（? 前缀）─────────────────────────────────────────
?input                        标签 input（位置无关；* 表示任意标签）
?#login                       id = login
?.el-input__inner             class 分词匹配（CSS 语义）
?type=filter                  属性全等
?type*=fil / ^= / $=          包含 / 开头 / 结尾
?text*=ABC                    深层文本包含（含子孙）
?owntext*=ABC                 仅直属文本节点
?text/="^\s*登录"             文本正则（含 () | 空格需加引号）
?@data-testid                 存在该属性；?!@data-testid 不存在
?@* / ?!@*                    有任意属性 / 没有任何属性
?@tag=x                       @ 转义保留键（匹配字面属性 tag）
?@"@click"*=submit            引号转义特殊属性名（如 Vue @click）
?input && type=filter         与（顺序随意；别名 &）
?(div || span) && !id=ad      或（别名 |）、非（!）、括号嵌套

── Python Q Builder（动态变量防注入）─────────────────────────────
Q('input', type='filter')                       # 基础构造
Q('a', text__contains=kw, href__startswith='/') # 关键字后缀
Q.attr('data-testid').exists()                  # 特殊属性名
Q.text.regex(r'\d+') & ~Q.id('ad')              # 流式 + 位运算 & | ~
explain_loc('?...') / q.explain()               # 打印 AST 与下推 XPath
```

#### 支持范围与边界

| 能力 | 支持 |
|---|---|
| 页面 / 元素 / ShadowRoot / iframe / Session 模式 | ✅ |
| `s_ele()` / `s_eles()` 快照查询 | ✅ |
| `offset()` 与 `east/south/west/north()` 方向定位 | ✅ |
| `wait.eles_loaded()` 等等待接口 | ✅ |
| 旧语法 | 100% 不变，零迁移成本 |
| `parent() / child() / next()` 等轴方法含 `/=` 正则的查询 | ⚠️ 暂不支持（纯 XPath 查询可用） |

---

## 安装

本仓库无独立打包文件，按源码方式使用：

```shell
git clone https://github.com/mdview2026/DrissionPage.git
cd DrissionPage
pip install -r requirements.txt
```

在仓库根目录下直接使用（`import DrissionPage` 解析到当前目录源码）：

```shell
# Linux / macOS
export PYTHONPATH=$PWD
# Windows (CMD)
set PYTHONPATH=%CD%

python -c "from DrissionPage import Q; print(Q('input', type='filter'))"
```

也可以把 `DrissionPage/` 目录拷入自己的项目，或自行补一份 `pyproject.toml` 安装。

## 测试

新增功能测试位于 `tests/feature_cases/test_query_locator.py`，不依赖浏览器，覆盖设计文档第 9 节验收矩阵：AST 往返不变量（含随机生成 300 例）、双前端等价性、新旧行为等价性、混合下推 soundness（零漏杀零误杀）、词法编码地狱、报错指针。

---

## 上游与致谢

- 原项目作者 g1879，项目地址：[github](https://github.com/g1879/DrissionPage) ｜ [gitee](https://gitee.com/g1879/DrissionPage) ｜ [gitcode](https://gitcode.com/g1879/DrissionPage)
- 官方网站与文档：[https://DrissionPage.cn](https://drissionpage.cn)
- 本 fork 遵循上游 [LICENSE](LICENSE)，感谢原作者的杰出工作 💖
