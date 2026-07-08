# docplex 使用说明书

> **docplex** 是 IBM 官方的 Python 优化建模库，全称 *IBM Decision Optimization CPLEX Modeling for Python*。
> 它的定位是"建模层"：你用接近数学的语法描述问题，它在底层调用 **CPLEX 求解器**完成求解。
> 本手册覆盖日常建模 95% 的场景，全部代码可直接运行，并以电池调度为贯穿示例。

---

## 目录

1. [安装与环境](#1-安装与环境)
2. [核心工作流：五步建模](#2-核心工作流五步建模)
3. [创建模型 Model](#3-创建模型-model)
4. [变量：类型与批量创建](#4-变量类型与批量创建)
5. [表达式与约束](#5-表达式与约束)
6. [目标函数](#6-目标函数)
7. [求解与读取结果](#7-求解与读取结果)
8. [常用建模技巧（重点）](#8-常用建模技巧重点)
9. [求解器参数与性能调优](#9-求解器参数与性能调优)
10. [调试：模型不可行 / 结果不对怎么办](#10-调试模型不可行--结果不对怎么办)
11. [完整示例：电池套利模型](#11-完整示例电池套利模型)
12. [常见坑与最佳实践清单](#12-常见坑与最佳实践清单)

---

## 1. 安装与环境

```bash
pip install docplex
```

关于求解器，有三种情况：

| 场景 | 说明 |
|---|---|
| **社区版（免费）** | `pip install cplex` 附带社区版 CPLEX，限制 **1000 个变量 / 1000 条约束**。学习和小模型够用。 |
| **完整版（本地）** | 公司或学校有 CPLEX Optimization Studio 许可证，安装后 docplex 自动找到它，无规模限制。学术用户可免费申请。 |
| **云端求解** | 没有本地 CPLEX 时可以配置 IBM Cloud 的求解服务（较少用）。 |

验证安装：

```python
from docplex.mp.model import Model
m = Model()
print(m.environment)   # 打印 CPLEX 版本信息即成功
```

> **心智模型**：docplex 负责"把问题说清楚"，CPLEX 负责"找答案"。你写的每一行 docplex 代码，最终都会被编译成一个巨大的矩阵交给求解器。

---

## 2. 核心工作流：五步建模

任何 docplex 程序都是同一个骨架，背下来：

```python
from docplex.mp.model import Model

m = Model(name="my_problem")          # ① 建模型（拿一张白纸）
x = m.continuous_var(name="x")        # ② 声明决策变量
m.add_constraint(2*x + 3 <= 10)       # ③ 添加约束
m.maximize(5 * x)                     # ④ 设定目标
sol = m.solve()                       # ⑤ 求解
print(sol.objective_value, x.solution_value)
```

后面所有章节，都是在把这五步的每一步展开讲细。

---

## 3. 创建模型 Model

```python
m = Model(name="battery_arbitrage", log_output=True)
```

常用参数：

- `name`：模型名，导出文件、打印日志时用；
- `log_output=True`：求解时把 CPLEX 引擎日志打到屏幕上（看进度、看 MIP gap 非常有用，**调试时建议打开**）。

实用方法：

```python
m.print_information()   # 打印变量数、约束数、模型类型（LP/MIP）
m.export_as_lp("model.lp")   # 把模型导出成人类可读的 LP 文本文件（调试神器，见第10节）
```

`with` 写法可自动释放资源（脚本里可省略，Jupyter 反复运行时建议用）：

```python
with Model(name="demo") as m:
    ...
```

---

## 4. 变量：类型与批量创建

### 4.1 三种基本类型

| 类型 | 单个创建 | 取值 | 典型用途 |
|---|---|---|---|
| 连续 | `m.continuous_var(lb=0, ub=5, name="c")` | 任意实数 | 功率、电量、金额 |
| 整数 | `m.integer_var(lb=0, ub=100, name="n")` | 0,1,2,… | 台数、批次 |
| 0/1 | `m.binary_var(name="y")` | 0 或 1 | 开/关、是/否、模式选择 |

关键点：

- `lb` / `ub` 是**上下界约束的免费写法**——在声明变量时就把"功率 ≤ 5 MW"这类约束写完了，不必再 add_constraint;
- 连续变量默认 `lb=0`，`ub=+∞`。**如果变量允许为负（比如净持仓、温差），必须显式写 `lb=-m.infinity`**，这是新手第一大坑;
- 变量一定要起 `name`——报错信息、LP 导出文件里全靠名字定位问题。

### 4.2 批量创建：list 与 dict

真实模型的变量都是成组的。两种批量方式：

```python
T = range(24)

# ① var_list：按整数下标索引，返回 Python list
c = m.continuous_var_list(T, lb=0, ub=5, name="charge")
c[0], c[23]          # 直接下标访问，名字自动叫 charge_0 … charge_23

# ② var_dict：按任意 key 索引，返回 dict —— key 可以是字符串、元组
assets = ["battery_A", "battery_B"]
p = m.continuous_var_dict(assets, lb=0, name="power")
p["battery_A"]

# ③ 二维（矩阵）变量：var_matrix 或 用元组 key 的 dict
x = m.continuous_var_matrix(assets, T, lb=0, name="dispatch")
x["battery_A", 7]    # 资产 A 第 7 小时的出力
```

选择建议：**时间序列用 list（下标即小时），多维结构用 dict / matrix**。三种类型都有对应的 `integer_var_list`、`binary_var_dict` 等全套方法。

---

## 5. 表达式与约束

### 5.1 表达式：像写数学一样写

变量支持 `+ - *`（乘常数）运算，自然组合成线性表达式：

```python
revenue_t = price[t] * (d[t] - c[t])        # 一个线性表达式，还不是约束
```

**求和用 `m.sum(...)`，不要用 Python 内置 `sum()`**——两者结果等价，但 `m.sum` 在大模型上快一个数量级（内置 sum 会构造大量中间对象）：

```python
total = m.sum(price[t] * (d[t] - c[t]) for t in T)   # ✅
total = sum(price[t] * (d[t] - c[t]) for t in T)     # ⚠️ 能跑，但慢
```

点积的更快写法（超大模型时用）：

```python
total = m.dot(d, price)      # Σ price[t] * d[t]
```

### 5.2 约束：比较运算符 + add_constraint

用 `<=`、`>=`、`==` 把两个表达式连起来，就是一条约束；`add_constraint` 把它注册进模型：

```python
m.add_constraint(soc[t] == soc[t-1] + 0.9*c[t] - d[t]/0.9,  ctname=f"balance_{t}")
```

- `ctname` 给约束起名——**强烈建议给每条约束起名**，模型不可行时靠它定位（见第10节）；
- 批量添加用 `add_constraints`（复数），传一个生成器，比循环里逐条 add 快：

```python
m.add_constraints(
    (c[t] <= 5 * y[t] for t in T),
    names=[f"chg_switch_{t}" for t in T],
)
```

- **注意 `==` 是约束，`=` 是 Python 赋值**，写错不会报错但含义完全不同——第二大新手坑。

### 5.3 只支持线性（和少数扩展）

LP/MIP 框架内，约束两边必须是线性表达式。以下写法会报错或需要特殊处理：

```python
m.add_constraint(x * y <= 5)      # ❌ 变量相乘 = 非线性（除非转 QP）
m.add_constraint(x / y == 2)      # ❌ 变量做分母
m.add_constraint(x**2 <= 9)       # ❌ 平方
```

遇到逻辑或非线性需求，用第 8 节的建模技巧改写。

---

## 6. 目标函数

```python
m.maximize(m.sum(price[t] * (d[t] - c[t]) for t in T))
# 或
m.minimize(total_cost)
```

- 一个模型**只能有一个目标**。再次调用 maximize/minimize 会覆盖之前的；
- 多目标的常用做法是**加权合成**：

```python
m.maximize(revenue - 0.5 * degradation_cost)    # 收益 与 老化成本 的权衡
```

- 也可以分层（词典序）多目标：先最大化收益，再在收益不变的前提下最小化循环次数——docplex 支持 `m.set_multi_objective(...)`，进阶再学。

---

## 7. 求解与读取结果

### 7.1 求解

```python
sol = m.solve(log_output=True)

if sol is None:                       # ★ 必须检查！
    print(m.solve_details.status)     # 例如 'infeasible'
else:
    print("objective =", sol.objective_value)
```

**`m.solve()` 失败时返回 `None` 而不是抛异常**——不检查直接用 `sol.objective_value` 会莫名 AttributeError。这是第三大坑。

### 7.2 读取变量值

```python
# 方式一：从 solution 对象取
sol[c[3]]                    # 单个变量
sol.get_values(c)            # 一组变量 → list

# 方式二：从变量自身取（solve 之后可用）
c[3].solution_value

# 打印全部非零变量，快速人眼检查
sol.display()
```

### 7.3 求解状态与质量

```python
d = m.solve_details
print(d.status)          # optimal / feasible / infeasible / time limit exceeded ...
print(d.time)            # 求解耗时（秒）
print(d.mip_relative_gap)  # MIP gap：当前解离理论上界还差多少（0 = 证明了最优）
```

理解 **MIP gap**：MIP 求解是"不断找到更好的解 + 不断压低理论上界"的双向逼近。gap = (上界 − 当前解)/当前解。gap 为 0.5% 意味着"当前解距离理论最优最多差 0.5%"——实践中经常在 gap 足够小时就停，不必等到 0。

---

## 8. 常用建模技巧（重点）

真实业务规则往往不是天然线性的。以下模式覆盖了绝大多数"看似写不出来"的需求。

### 8.1 Big-M / 开关约束："只有开机才能出力"

最常用的技巧。用 binary 控制连续变量的开关：

```python
y = m.binary_var_list(T, name="on")           # 1 = 该小时处于充电模式
m.add_constraints(c[t] <= P_MAX * y[t]       for t in T)   # 关 → 充电压成 0
m.add_constraints(d[t] <= P_MAX * (1 - y[t]) for t in T)   # 开 → 放电压成 0
```

要点：系数 `P_MAX`（所谓 big-M）**要取"刚好够大"的物理上限，不要随手写 1e9**——M 越大，LP 松弛越松，MIP 越难解、数值越不稳定。

### 8.2 indicator 约束：更干净的"如果…就…"

docplex 原生支持指示约束，可读性比 big-M 好，且没有选 M 的烦恼：

```python
# 如果 y[t] == 0（非充电模式），则 c[t] == 0
m.add_indicator(y[t], c[t] == 0, active_value=0)
```

CPLEX 内部会做智能处理。小模型随意；超大模型上手工 big-M 有时反而更快，可以两种都试。

### 8.3 二选一 / 至多选一 / 恰好选一

```python
m.add_constraint(y1 + y2 <= 1)        # 至多一个为真
m.add_constraint(m.sum(y) == 1)       # 恰好选一个（例如：选且只选一个电价合约）
```

### 8.4 绝对值、最大最小值

`|x|`、`max`、`min` 都是非线性的，标准做法是引入辅助变量：

```python
# |x| ：引入 u ≥ 0，加两条约束，然后在目标里"惩罚" u
u = m.continuous_var(name="abs_x")
m.add_constraint(u >=  x)
m.add_constraint(u >= -x)
m.minimize(u)               # 只在 min |x| 型目标下成立！

# docplex 也提供了内置封装（内部自动做同样的事）：
m.add_constraint(m.abs(x) <= 3)
z = m.max(x1, x2, x3)       # z 是"三者最大值"的辅助变量
```

> 注意：`abs/min/max` 封装会**隐式引入整数变量**，可能让纯 LP 变成 MIP，规模大时留意。

### 8.5 固定成本："只要开机，就有 500 的启动费"

```python
start_cost = m.sum(500 * u[t] for t in T)     # u[t]: 该小时是否"新开机"
m.add_constraints(u[t] >= y[t] - y[t-1] for t in range(1, 24))  # 0→1 跳变时 u=1
```

这是机组组合（unit commitment）的核心写法，电池模型加"启停成本 / 循环成本"同理。

### 8.6 分段线性函数

阶梯电价、分段的老化成本，用 `piecewise`：

```python
f = m.piecewise(0, [(0, 0), (10, 50), (20, 200)], 1)   # 分段点列表
cost = f(x)      # cost 是 x 的分段线性函数值，可直接进目标或约束
```

### 8.7 软约束："尽量满足，违反了就罚钱"

硬约束太死会导致不可行。把它改造成"可以违反、但目标里罚款"：

```python
slack = m.continuous_var(lb=0, name="violation")
m.add_constraint(soc[23] >= 5 - slack)          # 允许收不回 5，缺口记入 slack
m.maximize(revenue - 1000 * slack)              # 每缺 1 MWh 罚 1000
```

罚金系数要显著大于正常经济价值，否则模型会"故意违反"。

---

## 9. 求解器参数与性能调优

```python
m.parameters.timelimit = 60                 # 最长求解 60 秒
m.parameters.mip.tolerances.mipgap = 0.005  # gap ≤ 0.5% 即停（默认 1e-4）
m.parameters.threads = 4                    # 限制线程数
m.parameters.randomseed = 42                # 固定随机种子（结果可复现）
```

实践经验：

1. **生产环境永远设 timelimit + mipgap**。MIP 的收敛是长尾的——90% 的质量往往在 10% 的时间内达到，最后 0.01% 的 gap 可能要跑一小时;
2. 到达时限后 `solve()` 仍返回**当前最好的可行解**（status 为 `time limit exceeded`），并非失败;
3. 模型慢的第一排查点不是参数，而是**建模本身**：binary 是否太多？big-M 是否太松？能否用 indicator/更紧的界收紧松弛？

---

## 10. 调试：模型不可行 / 结果不对怎么办

### 情况 A：`solve()` 返回 None，status = infeasible

约束之间互相矛盾。三板斧：

```python
# ① 冲突分析器：让 CPLEX 找出"最小矛盾约束集"
from docplex.mp.conflict_refiner import ConflictRefiner
cr = ConflictRefiner().refine_conflict(m)
cr.display()                     # 直接告诉你哪几条约束打架 —— 这就是给约束起名的意义

# ② 松弛器：最少违反多少约束能变可行？
from docplex.mp.relaxer import Relaxer
rx = Relaxer()
rs = rx.relax(m)
rx.print_information()           # 打印每条被松弛的约束及松弛量

# ③ 导出 LP 文件人眼检查
m.export_as_lp("debug.lp")       # 打开看：约束是不是写成了你以为的样子
```

### 情况 B：解出来了，但结果"不对劲"

- `sol.display()` 打印所有非零变量，逐个对物理直觉；
- 最常见的元凶：**漏了约束**（求解器一定会钻所有空子——它给出反直觉解，往往说明你的约束没把现实说全）；
- 其次：符号写反（`<=` 写成 `>=`）、单位不一致（MW 与 MWh 混用）、效率乘除方向搞反；
- 用小实例验证：把 T 缩到 3 个小时、价格设成手算可验证的数字，人肉对答案。

### 情况 C：数值告警 / 结果抖动

- 系数跨度不要超过 1e6（比如既有 0.001 又有 1e6）；做单位缩放；
- big-M 收紧到物理上限。

---

## 11. 完整示例：电池套利模型

可直接运行的完整脚本（社区版 CPLEX 即可，规模远小于 1000 变量）：

```python
"""24 小时电池日前套利：5 MW / 10 MWh / 效率 0.9"""
from docplex.mp.model import Model

# ---------- 数据 ----------
price = [45,42,40,38,37,39,48,65,80,72,60,52,
         48,45,44,50,68,95,120,110,85,65,55,48]   # £/MWh
T       = range(24)
P_MAX   = 5.0      # MW，最大充/放电功率
CAP     = 10.0     # MWh，容量
ETA     = 0.9      # 单程效率
SOC0    = 5.0      # 初始电量

# ---------- ① 模型 ----------
m = Model(name="battery_arbitrage")

# ---------- ② 变量（lb/ub 顺带写完了功率与容量约束） ----------
c   = m.continuous_var_list(T, lb=0, ub=P_MAX, name="charge")
d   = m.continuous_var_list(T, lb=0, ub=P_MAX, name="discharge")
soc = m.continuous_var_list(T, lb=0, ub=CAP,   name="soc")
y   = m.binary_var_list(T, name="is_charging")          # ← 从这行起是 MIP

# ---------- ③ 约束 ----------
for t in T:
    m.add_constraint(c[t] <= P_MAX * y[t],       ctname=f"chg_switch_{t}")
    m.add_constraint(d[t] <= P_MAX * (1 - y[t]), ctname=f"dis_switch_{t}")
    prev = SOC0 if t == 0 else soc[t-1]
    m.add_constraint(soc[t] == prev + ETA*c[t] - d[t]/ETA,
                     ctname=f"balance_{t}")              # 能量守恒（骨架方程）
m.add_constraint(soc[23] == SOC0, ctname="end_equals_start")

# ---------- ④ 目标 ----------
m.maximize(m.sum(price[t] * (d[t] - c[t]) for t in T))

# ---------- ⑤ 求解 ----------
m.parameters.timelimit = 30
sol = m.solve(log_output=False)

if sol is None:
    print("求解失败:", m.solve_details.status)
else:
    print(f"最优收益: £{sol.objective_value:.1f}")       # → £753.0
    print(f"{'hr':>3} {'price':>6} {'chg':>6} {'dis':>6} {'soc':>6}")
    for t in T:
        print(f"{t:>3} {price[t]:>6} {c[t].solution_value:>6.2f} "
              f"{d[t].solution_value:>6.2f} {soc[t].solution_value:>6.2f}")
```

运行结果：收益 **£753**，行为是凌晨（37–39 镑）与午间（44–48 镑）充电、早晚双峰（80 / 120 镑）满功率放电，一天两个完整循环——"低买高卖"是被解出来的，不是被编程进去的。

**练习建议**（每个都只改几行，感受"骨架不变、只加零件"）：

1. 把循环次数限制为每天 1 次：`m.add_constraint(m.sum(d) <= CAP/ETA)`;
2. 加每 MWh 吞吐 £2 的老化成本：目标改为 `revenue - 2 * m.sum(d[t] + c[t] for t in T)`;
3. 把结算颗粒度改成半小时：`T = range(48)`，功率约束乘 0.5 小时。

---

## 12. 常见坑与最佳实践清单

**五大坑（按踩坑频率排序）**

1. `solve()` 失败返回 **None** 而非抛异常 → 永远先判空再取值;
2. 连续变量默认 `lb=0` → 允许为负的变量必须显式 `lb=-m.infinity`;
3. 约束用 `==`，Python 赋值是 `=` → 写错不报错、含义天差地别;
4. 用内置 `sum()` 构建大表达式 → 用 `m.sum()`，大模型上快一个量级;
5. big-M 随手写 1e9 → 用刚好够的物理上限，否则又慢又数值不稳。

**最佳实践**

- 所有变量、约束都起有意义的名字（`ctname=`）——为不可行调试留后路;
- 数据（价格、参数）与模型代码分离，模型函数化：`build_model(prices, spec) -> Model`;
- 生产运行必设 `timelimit` + `mipgap`，并记录 `solve_details`;
- 上线前用手算可验证的 3 期小实例做单元测试;
- 结果反直觉时，第一反应是"我漏了哪条约束"，而不是"求解器错了"——求解器永远在钻你规则的空子;
- 模型改动后 `m.print_information()` 看一眼变量/约束数量是否符合预期。

---

## 附：速查表

| 我想… | 写法 |
|---|---|
| 建模型 | `m = Model(name="x", log_output=True)` |
| 连续/整数/0-1 变量 | `m.continuous_var / integer_var / binary_var` |
| 批量变量 | `m.continuous_var_list(T, lb, ub, name)` / `_dict` / `_matrix` |
| 加一条约束 | `m.add_constraint(expr <= rhs, ctname="...")` |
| 批量加约束 | `m.add_constraints((… for t in T), names=[…])` |
| 求和 | `m.sum(expr for t in T)`，点积 `m.dot(vars, coefs)` |
| 如果…就… | `m.add_indicator(y, x == 0, active_value=0)` 或 big-M |
| 目标 | `m.maximize(expr)` / `m.minimize(expr)` |
| 求解 | `sol = m.solve()`；**判空**；`sol.objective_value` |
| 取变量值 | `x.solution_value` / `sol.get_values(xs)` / `sol.display()` |
| 限时/限gap | `m.parameters.timelimit = 60`；`…mip.tolerances.mipgap = 0.005` |
| 不可行诊断 | `ConflictRefiner().refine_conflict(m).display()` |
| 导出模型 | `m.export_as_lp("model.lp")` |
